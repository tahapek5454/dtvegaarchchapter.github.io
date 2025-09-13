---
layout: post
title: "DDD Aggregate ve EF Core Concurrency (Optimistic Locking) Yönetimi"
categories: [mimari, ddd]
tags: [ddd, aggregate, ef core, concurrency, optimistic locking, polly]
lang: tr
author: QuickOrBeDead
excerpt: DDD Aggregate'lerde concurrency riskleri, invariant koruma, EF Core RowVersion ile optimistic concurrency + Polly retry, ölçüm & gözlemlenebilirlik pratikleri.
date: 2025-09-12
last_modified_at: 2025-09-13 12:30:00 +0300
---

## TL;DR
- Aggregate; bir grup entity/value object üzerindeki iş kurallarını (domain invariants) tek transaction içinde tutarlı kılar. Bu yüzden güncellemeler çoğunlukla geniş kapsamlı olur ve aynı Aggregate'e eşzamanlı erişimler çakışma riskini artırır.
- Concurrency çakışmaları; kurallar (ör: yüzde toplamı 100 olmalı, statü geçiş sırası, min < mid < max) sağlanıyormuş gibi görünse de son yazan kazanır (lost update) problemiyle iş kuralları bozulabilir.
- Çözüm: Optimistic concurrency (RowVersion ya da concurrency stamp) + domain seviyesinde ConcurrencyException + üst katmanda kontrollü retry (örn. Polly) + kullanıcıya anlamlı geri bildirim ("Bu hedef az önce değişti, tekrar dene") + idempotent davranış.

DDD kavramlarına yabancıysan önce şu yazıya göz atmanı öneririm: [DDD Nedir? Goal Management Örnek Uygulama](/mimari/ddd/2025/08/19/ddd-nedir-goal-management-ornek-uygulama.html).

> Concurrency problemleri Aggregate'in sorumlu olduğu iş kurallarını (invariants) sessizce bozabilecek en tehlikeli senaryolardan biridir; erken tespit ve kontrollü retry ile bu risk minimize edilir.

## Neden Aggregate Kullanımı Concurrency Riskini Yükseltir?
Aggregate; dış dünyaya tek bir tutarlılık kapısı sunar. 

Bir güncelleme genellikle:
1. Aggregate Root ve Aggregate Root'a bağlı bütün child entity koleksiyonuyla birlikte veri tabanından çeker.
2. İş kurallarını uygulayarak domain metodunu çağırır (ör: `goalSet.UpdateGoal(...)`).
3. Tek seferde veri tabanında günceller.

Bu model; "bir satırı güncelle ve çık" (narrow update) yerine birden fazla satırın (örn. GoalSet + Goals) aynı anda versiyonlanmasına yol açar. Aynı GoalSet'e farklı kullanıcıların saniyeler içinde hedef ekleme / güncelleme yapması çakışma olasılığını artırır ve iş kurallarının bozulmasına neden olur.

## Concurrency Problemleri Domain Invariant'larını Nasıl Bozar?
Örnek invariant'lar (Goal Management):
- Bir GoalSet altındaki hedef yüzdelerinin toplamı = 100.
- `min < mid < max` (GoalValue tutarlılığı).
- Onay akışında statü sekansının ihlali olmamalı.

İki kullanıcı aynı anda farklı hedeflerin yüzdesini güncelliyor olsun:
- Kullanıcı A: 20'yi 25 yapıyor (toplam 100 kalmasını sağlıyor).
- Kullanıcı B: 15'i 10 yapıyor (kendi anlık snapshot'ında yine 100 kalıyor).
- Sıra: A kaydediyor (veritabanı toplam 100). B (eski toplam üzerinden) kaydediyor, toplam 95'e düşüyor. Invariant bozuldu; çünkü B'nin verileri eski (stale) idi. (Lost update)

Optimistic concurrency ile B'nin kaydı reddedilir (RowVersion uyuşmaz) ve böylece invariant'ın bozulması engellenir. Kullanıcıya yeniden denemesi için güncel veri gösterilir. 

Alternatif olarak pessimistic concurrency (ör: `SELECT ... FROM ... WITH (ROWLOCK, XLOCK) WHERE Id = @Id` / satır locklama) kullanılırsa ikinci kullanıcının işlemi ilk transaction bitene kadar bloklanır. Yoğun trafikte bekleme süreleri artar, throughput düşer ve potansiyel deadlock riskleri doğar.

## Optimistic Concurrency Nedir?
Optimistic Concurrency; aynı veriyi nadiren çakışacak şekilde güncellediğimizi varsayarak (conflict olasılığı düşük varsayımı) lock kullanmadan ilerleyip, güncelleme anında bir versiyon karşılaştırmasıyla (ör: RowVersion, ETag, ExpectedVersion) çakışmayı tespit eden yaklaşımdır.

Karşıt yaklaşım olan Pessimistic Locking'de ise güncellemeden önce satır(lar) locklanır; bu da yüksek bekleme (blocking), deadlock ve throughput düşüşü riskini artırır. DDD Aggregate senaryolarında çoğu güncelleme kısa ve atomiktir; bu yüzden optimistic model daha yüksek paralellik (concurrency) ve daha iyi ölçeklenme sunar.

Avantajlar:
- Yüksek okuma-yazma paralelliği, lock yok.
- Yatay ölçekli (stateless) API'lerde iyi çalışır.

Dezavantaj / Dikkat:
- Çakışma oranı yüksekse sürekli retry maliyeti artar (pessimistic daha iyi olabilir).
- Kullanıcı deneyimi tasarımı gerekir ("kayıt başka bir kullanıcı tarafından değiştirildi" mesajı, otomatik yeniden yükleme, merge akışı).
- Retry stratejisi yoksa kayıp güncellemeler kullanıcıya hata olarak döner (kaybın sebebi anlaşılmayabilir).

Ne Zaman Tercih Etmeli?
- Yazma çakışma oranı düşük/orta (örn. aynı Aggregate'e saniyede çok nadir eş zamanlı update).
- Yüksek okuma hacmi.
- Kısa süreli transaction'lar.

Uygulama Prensipleri:
1. Versiyon kolonu: RowVersion (timestamp / byte[])
2. Aggregate Root okuma: Komut başında versiyon belleğe alınır.
3. İş kuralı uygulama: Domain metodları invariant'ları kontrol eder.
4. Veri tabanına kaydetme: UPDATE ... WHERE Id = @id AND RowVersion = @original.
5. Etkilenen satır 0 → ConcurrencyException → Retry veya kullanıcıya mesaj.
6. Başarılıysa yeni RowVersion istemciye (veya response body event'ine) eklenebilir.

## EF Core ile Optimistic Concurrency Mekanizması
1. Entity üzerinde concurrency token (genelde `byte[] RowVersion`) tanımlanır.
2. EF Core `UPDATE ... WHERE Id = @id AND RowVersion = @originalRowVersion` üretir.
3. Etkilenen satır sayısı 0 ise `DbUpdateConcurrencyException` fırlatır.
4. Infrastructure katmanı domain'e özgü bir `ConcurrencyException` oluşturup fırlatır.
5. Application/UseCase seviyesi bunu yakalar; gerekirse retry veya kullanıcıya mesaj.

### Örnek Aggregate Root (Basitleştirilmiş)
GoalSet Aggregate Root'u için RowVersion kolonu ile optimistic concurrency kontrolü ve basit bir invariant (yüzde toplamı = 100) gösterimi.

```csharp
public class GoalSet : EntityBase, IAggregateRoot
{
    public int Id { get; private set; }
    public string Title { get; private set; } = string.Empty;
    public List<Goal> Goals { get; private set; } = new();

    // Optimistic concurrency token
    public byte[] RowVersion { get; private set; } = [];

    public Result UpdateGoal(int goalId, string? title, GoalType type, GoalValueType valueType, int percentage)
    {
        var goal = Goals.SingleOrDefault(g => g.Id == goalId);
        if (goal is null) return Result.Error("Goal not found");

        goal.Update(title, type, valueType, percentage);
        if (Goals.Sum(g => g.Percentage) != 100)
            return Result.Error("Percentage total must be 100");

        return Result.Success();
    }
}
```

#### Entity EF Core Konfigürasyonu
RowVersion alanını EF Core'a concurrency token olarak tanıtır; böylece UPDATE cümlesine WHERE RowVersion = @original eklenir ve çakışmada etkilenen satır 0 olur.

```csharp
internal sealed class GoalSetEvaluationConfiguration : IEntityTypeConfiguration<GoalSet>
{
  public void Configure(EntityTypeBuilder<GoalSet> builder)
  {
    builder.HasKey(gs => gs.Id);

    // Configure optimistic concurrency token
    builder.Property(gs => gs.RowVersion)
      .IsRowVersion()
      .ValueGeneratedOnAddOrUpdate();
  }
}
```

### Repository ConcurrencyException Çevirisi
Aşağıdaki repository, altyapıda oluşan `DbUpdateConcurrencyException` istisnasını domain katmanına anlamlı bir `ConcurrencyException` olarak çevirir.

```csharp
public class EfRepository<T>(AppDbContext dbContext) : RepositoryBase<T>(dbContext), IReadRepository<T>, IRepository<T> where T : class, IAggregateRoot
{
  public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
  {
    try
    {
      return await base.SaveChangesAsync(cancellationToken).ConfigureAwait(false);
    }
    catch (DbUpdateConcurrencyException ex)
    {
      // Log + domain'e özel exception
      throw new ConcurrencyException("A concurrency conflict occurred.", ex);
    }
  }
}
```

### Use Case (Application) Katmanı: Retry + Invariant Ayrımı
`Polly` ile yalnızca gerçekten concurrency kaynaklı hatalarda (domain kuralı değil) exponential backoff + jitter retry uygulanır.

```csharp
internal sealed class UpdateGoalCommandHandler(IRepository<GoalSet> goalSetRepository) : ICommandHandler<UpdateGoalCommand, Result<(int GoalSetId, int GoalId)>>
{
  private static readonly ResiliencePipeline RetryPipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
      ShouldHandle = new PredicateBuilder().Handle<ConcurrencyException>(),
      MaxRetryAttempts = 3,
      Delay = TimeSpan.FromMilliseconds(50),
      BackoffType = DelayBackoffType.Exponential,
      UseJitter = true
    })
    .Build();

  public async Task<Result<(int GoalSetId, int GoalId)>> Handle(UpdateGoalCommand request, CancellationToken cancellationToken)
  {
    var spec = new GoalSetWithGoalsByGoalSetIdSpec(request.GoalSetId);

    try
    {
      return await RetryPipeline.ExecuteAsync(async token =>
      {
        var goalSet = await goalSetRepository.SingleOrDefaultAsync(spec, token).ConfigureAwait(false);
        if (goalSet == null)
        {
          return Result.Error($"Goal set not found for id: {request.GoalSetId}");
        }

        var updateGoalResult = goalSet.UpdateGoal(request.GoalId, request.Title, request.GoalType, request.GoalValueType, request.Percentage);
        if (!updateGoalResult.IsSuccess)
        {
          // Domain kuralı (invariant) ihlali: retry anlamsız
            return updateGoalResult.ToResult();
        }

        await goalSetRepository.UpdateAsync(goalSet, token).ConfigureAwait(false);
        return Result.Success((goalSet.Id, request.GoalId));
      }, cancellationToken).ConfigureAwait(false);
    }
    catch (ConcurrencyException)
    {
      return Result.Error("Another user modified this goal set at the same time. Your change could not be applied automatically. Please try again.");
    }
  }
}
```

## Retry Ne Zaman Anlamlı / Ne Zaman Değil?
- Anlamlı: Sadece ConcurrencyException alındığında (RowVersion mismatch). Retry ile çoğunlukla ikinci denemede success alınır.
- Anlamsız: İş kuralı ihlali (invariant bozuk), validasyon hatası, yetki hatası. Bunlar deterministiktir; tekrar denenmekle değişmez.

## Neden Jitter Gerekli?
Exponential backoff tek başına yeterli değildir; aynı anda çakışma yaşayan birçok istek aynı deterministik gecikme süreleriyle (50ms, 100ms, 200ms, ...) tekrar denediğinde "thundering herd" oluşur. Jitter (rastgele ufak sapma) ekleyerek retry çağırımlarını zamana yayarız.

Faydalar:
- Trafiği Yayma: Aynı milisaniyeye yığılmayı engelleyip veritabanı / cache üzerindeki ani pikleri azaltır.
- Lock Contention Azalması: Aggregate üzerindeki eşzamanlı yeniden denemelerin çakışma olasılığını düşürür.
- Kuyruk Boşalması: Downstream servis geçici yavaşsa jitter, kuyruktaki retry'ların tek blok halinde yeniden çökmesini engeller.
- Adalet (Fairness): Farklı isteklerin başarı şansını dengeleyerek starvation riskini azaltır.

Yanlış Uygulama Örnekleri:
- Sabit gecikme (fixed delay) + jitter yok → çakışmalar artar.
- Tüm retry'lara geniş aralıkta (örn. 0–Delay*3) aşırı jitter → Ortalama gecikme gereksiz artar.

Öneri:
- Exponential backoff: baseDelay * 2^attempt.
- Jitter: "Full Jitter" → delay = Random(0, base * 2^attempt).
- Üst sınır (max cap) koy: Özellikle UI isteklerinde 1–2 saniye üstü total bekleme kullanıcı deneyimini bozar.

Basit örnek (base 50ms):
- Attempt 0 → 0–50 ms arası (ortalama ~25 ms)
- Attempt 1 → 0–100 ms arası (ortalama ~50 ms)
- Attempt 2 → 0–200 ms arası (ortalama ~100 ms)

Polly Opsiyonları:
- `UseJitter = true` (Polly V8) küçük, kontrollü jitter uygular.
- Gelişmiş senaryoda `AddRetry(new RetryStrategyOptions { DelayGenerator = ... })` ile custom jitter.

## Ölçüm ve Gözlemlenebilirlik
İzlenmesi önerilen metrikler:
- ConcurrencyException sayısı / toplam update oranı.
- Ortalama retry attempt sayısı.
- Jitter devredeyken ve devre dışıyken DB CPU / lock wait / deadlock sayıları.
- P95/P99 işlem süresi (retry etkisi var mı?).

Toplama Yöntemi:
- Application katmanında RetryPipeline içine retry attempt counter/log.
- Prometheus/OpenTelemetry ile zaman serisi + uyarı eşikleri (örn. ConcurrencyException oranı > %5 ise alert).
- Log korelasyonu için traceId + aggregateId + originalRowVersion bilgisi.

Yorumlama:
- Oran sürekli yükseliyorsa: Daha küçük aggregate sınırları, komutların parçalanması veya pessimistic yaklaşım değerlendirmesi.
- Retry attempt ortalaması 1.5+ ise: UI otomatik refresh / merge akışı ekleme zamanı gelmiş olabilir.

## Kullanıcı Deneyimi (UX) ve Versiyonlama
İyi pratikler:
- Concurrency sonucu hata mesajında kaybın sebebini açıkla ("bu kayıt az önce başka kullanıcı tarafından güncellendi").
- Gerekirse değişiklik diff'ini göster (eski vs yeni değerler) ve merge akışı tasarla.
- Otomatik yeniden yükleme (silent refresh) yaparken kullanıcı odak kaybı yaratmamaya dikkat et.

## Özet Öneriler
- Her aggregate için optimistic concurrency token ekle (ör: RowVersion).
- Infrastructure: EF Core `DbUpdateConcurrencyException` -> domain `ConcurrencyException` dönüştür.
- Application: ConcurrencyException için limitli, jitter'lı exponential retry (Polly) uygula.
- Domain metodları invariant ihlallerini bildir; retry etme.
- İzleme (Observability): ConcurrencyException metriklerini (count, retry attempts, oran) ölç ve eşik aşımlarında alarm kur.

## Sık Sorulan Sorular (FAQ)
**Optimistic ve Pessimistic concurrency farkı nedir?** Optimistic model lock kullanmaz, çakışmayı versiyon uyuşmazlığı ile sonradan yakalar; Pessimistic model güncellenecek kayıtları baştan locklayarak bekleme ve potansiyel deadlocklar yaratır.

**EF Core RowVersion nasıl çalışır?** UPDATE cümlesinin WHERE kısmına orijinal RowVersion eklenir. Etkilenen satır 0 ise concurrency çakışması kabul edilip `DbUpdateConcurrencyException` fırlatılır.

**Polly ile retry ne zaman yapılmalı?** Yalnızca geçici (transient) çakışmalarda: `ConcurrencyException`. Domain invariant veya validasyon hatasında retry zaman kaybıdır.

**Domain invariant ihlali neden retry edilmez?** Çünkü deterministik; aynı giriş yeniden işlendiğinde aynı kural bozulacaktır.

**Hangi metrikler kritik?** ConcurrencyException oranı, ortalama retry attempt, P95/P99 süreleri, deadlock veya lock wait süreleri.

## Devam / Uygulama Adımı
Repo'yu klonlayıp ([Repo](https://github.com/DTVegaArchChapter/Architecture/tree/main/ddd/goal-management-system), [Goal Management örnek uygulaması ile ilgili blog yazısı](/mimari/ddd/2025/08/19/ddd-nedir-goal-management-ornek-uygulama.html)) aynı GoalSet üzerinde iki paralel update senaryosu çalıştırarak RowVersion çakışmasını tetikle ve retry metriklerini gözlemle. Dilersen jitter'i kapatarak (UseJitter=false) farkı ölçebilirsin.

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "Optimistic concurrency nedir?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Optimistic concurrency, veriyi güncellemeden önce locklamak yerine RowVersion gibi bir versiyon alanı ile çakışmayı güncelleme anında tespit eden yaklaşımdır. Çakışma yoksa günceller, varsa güncelleme reddedilir."
      }
    },
    {
      "@type": "Question",
      "name": "Pessimistic concurrency farkı nedir?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Pessimistic concurrency güncelleme öncesi satırı locklar; bu bloklanma, bekleme süresi ve deadlock riskini artırabilir. Optimistic model lock kullanmadığı için paralellik daha yüksektir ancak çakışma sonrası retry gerekir."
      }
    },
    {
      "@type": "Question",
      "name": "EF Core RowVersion nasıl çalışır?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Entity'e eklenen RowVersion alanı concurrency token olarak işaretlenir. EF Core UPDATE ... WHERE RowVersion = @original üretir. Etkilenen satır 0 ise DbUpdateConcurrencyException fırlatır."
      }
    },
    {
      "@type": "Question",
      "name": "Polly ile retry ne zaman uygulanmalı?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "Sadece geçici concurrency çakışmalarında (ConcurrencyException). Domain invariant ihlali, validasyon veya yetki hatalarında retry anlamsızdır."
      }
    },
    {
      "@type": "Question",
      "name": "Hangi metrikleri izlemeliyim?",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "ConcurrencyException oranı, ortalama retry attempt, P95/P99 süreleri, lock wait ve deadlock sayıları kritik metriklerdir. Yükselen trend tasarım veya aggregate sınırlarını yeniden gözden geçirmeyi gerektirir."
      }
    }
  ]
}
</script>

## Kaynaklar
- DDD Goal Management örnek repo (kod parçaları uyarlanmıştır): [https://github.com/DTVegaArchChapter/Architecture/tree/main/ddd/goal-management-system](https://github.com/DTVegaArchChapter/Architecture/tree/main/ddd/goal-management-system)
- Microsoft Docs – EF Core Concurrency: [https://learn.microsoft.com/ef/core/saving/concurrency](https://learn.microsoft.com/ef/core/saving/concurrency)
- Polly: [https://github.com/App-vNext/Polly](https://github.com/App-vNext/Polly)
- Eric Evans – Domain-Driven Design (Aggregate konsepti)
