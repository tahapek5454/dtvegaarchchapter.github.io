---
layout: post
title: "DDD Aggregate ve EF Core Concurrency (Optimistic Locking) Yönetimi"
categories: [mimari, ddd]
tags: [ddd, aggregate, ef core, concurrency, optimistic locking, polly]
lang: tr
author: QuickOrBeDead
excerpt: DDD Aggregate'lerde neden concurrency problemleri daha sık görülür, domain invariant'ları nasıl bozar ve EF Core ile optimistic concurrency (RowVersion + retry) nasıl yönetilir?
date: 2025-09-10
---

## TL;DR
- Aggregate; bir grup entity/value object üzerindeki iş kurallarını (domain invariants) tek transaction içinde tutarlı kılar. Bu yüzden güncellemeler çoğunlukla geniş kapsamlı olur ve aynı Aggregate'a eşzamanlı erişimler çakışma riskini artırır.
- Concurrency çakışmaları; kurallar (ör: yüzde toplamı 100 olmalı, statü geçiş sırası, min < mid < max) sağlanıyormuş gibi görünse de son yazan kazanır (lost update) problemiyle iş kuralları bozulabilir.
- Çözüm: Optimistic concurrency (RowVersion ya da concurrency stamp) + domain seviyesinde ConcurrencyException + üst katmanda kontrollü retry (örn. Polly) + kullanıcıya anlamlı geri bildirim ("Bu hedef az önce değişti, tekrar dene") + idempotent davranış.

> Concurrency problemleri Aggregate'in sorumlu olduğu "iş kurallarını" (invariants) sessizce bozabilecek en tehlikeli senaryolardan biridir; erken tespit ve kontrollü retry ile bu risk minimize edilir.

## Neden Aggregate Kullanımı Concurrency Riskini Yükseltir?
Aggregate; dış dünyaya tek bir tutarlılık kapısı sunar. 

Bir güncelleme genellikle:
1. Aggregate Root ve Aggrate Root'a bağlı bütün child entity koleksiyonuyla birlikte veri tabanından çeker.
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

Alternatif olarak pessimistic concurrency (ör: SELECT ... FOR UPDATE / satır kilitleme) kullanılırsa ikinci kullanıcının işlemi ilk transaction bitene kadar bloklanır. Yoğun trafikte bekleme süreleri artar, throughput düşer ve potansiyel deadlock riskleri doğar.

## Optimistic Concurrency Nedir?
Optimistic Concurrency; aynı veriyi nadiren çakışacak şekilde güncellediğimizi varsayarak (conflict olasılığı düşük varsayımı) kilit (lock) tutmadan ilerleyip, güncelleme anında bir versiyon karşılaştırmasıyla (ör: RowVersion, ETag, ExpectedVersion) çakışmayı tespit eden yaklaşımdır.

Karşıt yaklaşım olan Pessimistic Locking'de ise güncellemeden önce satır(lar) kilitlenir; bu da yüksek bekleme (blocking), deadlock ve throughput düşüşü riskini artırır. DDD aggregate senaryolarında çoğu güncelleme kısa ve atomiktir; bu yüzden optimistic model daha yüksek paralellik (concurrency) ve daha iyi ölçeklenme sunar.

Avantajlar:
- Yüksek okuma-yazma paralelliği, lock yok.
- Bulut ve yatay ölçekli (stateless) API pod'larında iyi çalışır.

Dezavantaj / Dikkat:
- Çakışma oranı yüksekse sürekli retry maliyeti artar (pessimistic daha iyi olabilir).
- Kullanıcı deneyimi tasarımı gerekir ("kayıt değişti" mesajı, otomatik yeniden yükleme, merge akışı).
- Retry stratejisi yoksa kayıp güncellemeler kullanıcıya hata olarak döner (kaybın sebebi anlaşılmayabilir).

Ne Zaman Tercih Etmeli?
- Yazma çakışma oranı düşük/orta (örn. aynı Aggregate'e saniyede çok nadir eş zamanlı update).
- Yüksek okuma hacmi.
- Kısa süreli transaction'lar.

Uygulama Prensipleri:
1. Versiyon Kolonu: RowVersion (timestamp / byte[])
2. Aggregate Root Okuma: Komut başında versiyon belleğe alınır.
3. İş Kuralı Uygulama: Domain metodları invariant'ları kontrol eder.
4. Persist: UPDATE ... WHERE Id = @id AND RowVersion = @orijinal.
5. Etkilenen satır 0 → ConcurrencyException → Retry veya kullanıcıya mesaj.
6. Başarılıysa yeni RowVersion istemciye (veya response body event'ine) eklenebilir.

## EF Core ile Optimistic Concurrency Mekanizması
1. Entity üzerinde concurrency token (genelde `byte[] RowVersion`) tanımlanır.
2. EF Core `UPDATE ... WHERE Id = @id AND RowVersion = @originalRowVersion` üretir.
3. Etkilenen satır sayısı 0 ise `DbUpdateConcurrencyException` fırlatır.
4. Infrastructure katmanı domain'e özgü bir `ConcurrencyException` oluşturup fırlatır.
5. Application/UseCase seviyesi bunu yakalar; gerekirse retry veya kullanıcıya mesaj.

### Örnek Entity (Basitleştirilmiş)
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

#### Entity Ef Core Konfigurasyonu
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
Altyapıdaki `DbUpdateConcurrencyException` yakalanıp domain'e anlamlı bir `ConcurrencyException` olarak yeniden fırlatılır; application katmanı bu sayede retry kararını doğru verir.

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
`Polly` ile yalnızca gerçekten concurrency kaynaklı hatalarda (domain kuralı değil) exponential backoff retry uygulanır.

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

### Neden Jitter Gerekli?
Exponential backoff tek başına yeterli değildir; aynı anda çakışma yaşayan birçok istek aynı deterministik gecikme süreleriyle (50ms, 100ms, 200ms, ...) tekrar denediğinde "thundering herd" oluşur. Jitter (rastgele ufak sapma) ekleyerek retry çağırımlarını zamana yayarız.

Faydalar:
- Trafiği Yayma: Aynı milisaniyeye yığılmayı engelleyip veritabanı / cache üzerindeki ani pikleri azaltır.
- Lock Contention Azalması: Aggregate üzerindeki eşzamanlı yeniden denemelerin çakışma olasılığını düşürür.
- Kuyruk Boşalması: Downstream servis geçici yavaşsa jitter, kuyruktaki retry'ların tek blok halinde yeniden çökmesini engeller.
- Adalet (Fairness): Farklı isteklerin başarı şansını dengeleyerek starvation riskini azaltır.

Yanlış Uygulama Örneği:
- Sabit gecikme (fixed delay) + no jitter → çakışmalar fazla.
- Tüm retry'lara geniş aralıkta (örn. 0–Delay*3) aşırı jitter → Ortalama gecikme gereksiz artar.

Öneri:
- Exponential backoff: baseDelay * 2^attempt.
- Jitter: Örneğin "Full Jitter" (AWS önerisi) → delay = Random(0, base * 2^attempt).
- Üst sınır (max cap) koy: Özellikle UI isteklerinde 1–2 saniye üstü total bekleme kullanıcı deneyimini bozar.

Polly Opsiyonları:
- `UseJitter = true` varsayılan (Polly V8 Resilience pipeline) küçük, kontrollü jitter uygular.
- Gelişmiş senaryoda `AddRetry(new RetryStrategyOptions { DelayGenerator = ... })` ile custom jitter.

Ölçüm:
- ConcurrencyException sayısı / toplam update oranı.
- Ortalama retry attempt sayısı.
- Jitter devredeyken ve değilken DB CPU / lock wait karşılaştırması.

Bu metrikleri log + metrics (Prometheus / OpenTelemetry) ile zaman serisi halinde toplayıp threshold aşımlarında alarm üretebilirsiniz. Çünkü yükselen oran veya artan ortalama retry denemesi yeniden tasarım ihtiyacını gösterir.

## Kullanıcı Deneyimi (UX) ve Versiyonlama
İyi pratikler:
- Concurrency sonucu hata mesajında kaybın sebebini açıkla ("bu kayıt güncellendi").
- İstersen: Değişiklik diff'ini göster (eski vs yeni değerler) ve merge akışı tasarla.

## Özet Öneriler
- Her aggregate için optimistic concurrency token ekle (ör: RowVersion).
- Infrastructure: EF Core `DbUpdateConcurrencyException` -> domain `ConcurrencyException` dönüştür.
- Application: ConcurrencyException için limitli, jitter'lı exponential retry (Polly) uygula.
- Domain metodları invariant ihlallerini bildir; retry etme.
- İzleme (Observability): ConcurrencyException metriklerini (count, retry attempts) ölç.

## Kaynaklar
- DDD Goal Management örnek repo (kod parçaları uyarlanmıştır): https://github.com/DTVegaArchChapter/Architecture/tree/main/ddd/goal-management-system
- Microsoft Docs – EF Core Concurrency: https://learn.microsoft.com/ef/core/saving/concurrency
- Polly: https://www.thepollyproject.org/
- Eric Evans – Domain-Driven Design (Aggregate konsepti)
