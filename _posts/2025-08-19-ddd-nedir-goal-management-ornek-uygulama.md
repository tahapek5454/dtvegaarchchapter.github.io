---
layout: post
title: "Domain-Driven Design (DDD) - Kısa Bir Giriş ve Goal Management Örnek Uygulaması"
categories: [mimari, ddd]
tags: [ddd, event storming, strategic design, tactical design, ubiquitous language]
lang: tr
author: QuickOrBeDead
excerpt: Domain-Driven Design (DDD) nedir? Kısa bir giriş, avantaj/dezavantajlar ve Goal Management örnek uygulamamızla DDD’yi adım adım görün.
date: 2025-08-19
last_modified_at: 2025-08-19 00:00:00 +0300
---

## TL;DR
- DDD; hizasızlık, belirsiz sınırlar, tutarlılık ihlalleri, kontrolsüz yan etkiler ve düşük test edilebilirliği hedef alır.
- Çözüm: Ubiquitous Language + Bounded Context + aggregate kapsülleme.
- Nerede iyi: Karmaşık domain ve uzun yaşayan sistemler. Nerede değil: Basit CRUD.
- Örnek: DT Vega Goal Management ile uçtan uca akış (Event Storming → Strategic → Tactical).

## DDD Nedir?
Domain-Driven Design (DDD), karmaşık iş problemlerini yazılım tasarımının merkezine alarak çözmeyi amaçlayan bir yaklaşımdır. Temel fikir, iş alanını (domain) daha iyi anlayıp modellemek, bu modeli doğrudan koda yansıtmak ve teknik kararları iş kuralları etrafında şekillendirmektir.

DDD sayesinde:

- İş kavramlarıyla birebir örtüşen bir model kurulur (Ubiquitous Language).
- Model, Bounded Context'lere ayrılarak karmaşıklık kontrollü bir şekilde yönetilir.
- Stratejik (büyük resim) ve taktiksel (detay tasarım) seviyelerde rehberlik sunulur.

## Neden Ortaya Çıktı? Hangi Problemi Çözüyor?

Eric Evans, DDD terimini ortaya koyup sistematikleştirdi; amacı kurumsal yazılımlarda artan domain karmaşıklığını yönetmekti. 2003'te yayımladığı "Domain‑Driven Design" kitabı ile karmaşıklığın merkezine domain modelini koyarak tasarımı yönlendirmeyi savunur. Kitabında; "Ubiquitous Language", "Strategic Design", "Bounded Context", "Tactical Design" gibi birçok kavramı açıklamıştır.

2000'lerin başında kurumsal uygulamalarda farklı yaklaşım tarzları hakimdi:

- **Transaction Script:** Her use case'i bir prosedürle çözen, basit ve doğrudan yaklaşım.
- **Active Record:** Veri satırı ile nesneyi birleştirir; CRUD ağırlıklıdır.
- **Domain Model (DDD):** Zengin iş kuralları ve davranışları modelin içine taşır.

Martin Fowler, basit süreçler için Transaction Script ya da Active Record'un yeterli olabileceğini; fakat kurallar ve iş akışları karmaşıklaştıkça davranış odaklı, kapsüllenmiş bir Domain Model'e ihtiyaç duyulduğunu vurgular. Ayrıca "Anemic Domain Model" (sadece veri taşıyan, davranışı servislere yükleyen modeller) yaklaşımının, nesne yönelimli tasarımın özünü zayıflattığını; iş kurallarının entity/aggregate'lerin içinde ifade edilmesi gerektiğini belirtmiştir.

**Çözdüğü problem**: Karmaşık iş kurallarını dağınık servisler ve anemik modeller yerine, sınırları net, davranışları kapsüllenmiş bir domain modeli içinde toplayarak, dağınık servisler ve anemik modellerin yol açtığı kural tekrarlarının, çelişkilerin, tutarlılık ve veri bütünlüğü ihlallerinin önüne geçmek.

## Avantajlar / Dezavantajlar - Nerede Kullanmalı/Kullanmamalı?

### Avantajlar
- Karmaşık kuralların net sınırlar ve kapsülleme ile yönetimi
- Ubiquitous Language sayesinde iş birimi ile teknik ekip hizalanması
- Modülerlik (Bounded Context)
- Test edilebilirlik ve davranış odaklı tasarım

### Dezavantajlar
- Öğrenme eğrisi ve başlangıç maliyeti
- Diyagram/keşif (Event Storming vb.) için ekip zamanına ihtiyaç
- Aşırı mühendislik (Overengineering) riski (basit problemlerde gereksiz karmaşıklık)

### Kullanım Önerileri
- **Kullan**: Karmaşık iş akışları, yoğun domain kuralları, farklı alt alanlar (sub-domain'ler) ve uzun yaşayan sistemler.
- **Kaçın**: Basit CRUD, raporlama/ETL ağırlıklı sistemler, kısa ömürlü MVP'ler, domain karmaşıklığının düşük olduğu projeler.

## DDD Süreci
- **Ubiquitous Language**: İş ve teknik ekiplerin paylaştığı tek dil. İsimlendirmeler koda aynen yansıtılır; yanlış anlaşılmalar azalır.
- **Event Storming:** Domain uzmanlarıyla birlikte, olaylar (domain events), komutlar (command) ve politikalar (policy) üzerinden süreci keşfetme atölyesi.
- **Strategic Design**: Bounded Context'leri, sub-domain'leri (core/supporting/generic) ve bağlamlar arası ilişkileri (Context Map) tasarlama.
- **Tactical Design**: Kod seviyesinde `Aggregate Root`, `Entity`, `Value Object`, `Domain Event`, `Repository`, `Specification` gibi yapı taşlarıyla modelleme.

## Tactical Design Öğeleri
- **Aggregate Root**: Veri tutarlılığını koruyan kök nesnedir. İçindeki entity ve value object'lere ait iş kurallarının tutarlılığını ve veri bütünlüğünü korur. Dışarıdan erişim ve değişiklikler çoğunlukla bu nesne üzerinden yapılır.
- **Entity**: Benzersiz kimliği (ID - identity) olan ve zamanla durumu değişebilen nesne (oluşturulur, güncellenir, silinir).
- **Value Object**: Kimliği yoktur (identity yok); içeriği aynıysa aynı kabul edilir (değerine göre eşitlik). Genellikle değiştirilmez (immutable) tasarlanır. Örnek: Miktar, Para, Tarih Aralığı, Adres.
- **Domain Event**: Domain'de gerçekleşmiş bir olaydır (geçmiş zaman, olmuş bitmiş bir gerçek). Çoğunlukla bir komutun (Command) başarılı işlenmesi ve durum değişikliği sonrası üretilir. Diğer süreçleri/bağlamları bilgilendirmek için yayınlanır; ana iş akışını bozmadan ek işlemleri bağımsız olarak tetikler ve gevşek bağlılığı artırır.
- **Repository**: Aggregate’leri veri tabanına (veya başka bir depoya) kaydetmek, okumak ve güncellemek için kullanılan arayüzdür. Uygulaması altyapı (infrastructure) katmanındadır.
- **Specification**: Sorgu ve kural ifadelerini yeniden kullanılabilir kılar.

## Hızlı Başlangıç
- Reponun README’sini açın: https://github.com/DTVegaArchChapter/Architecture/tree/main/ddd/goal-management-system#readme
- Event Storming ve Strategic Design görsellerine göz atın (aşağıdaki başlıklarda).
- Aggregate’leri (GoalSet, Goal, GoalProgress) ve Value Object’leri inceleyin.
- “Goal ekle” akışını ve üretilen Domain Event’leri takip edin.
- Kavramları pekiştirmek için SSS bölümündeki kısa tanımlara bakın.

## Örnek Projemiz: DDD Goal Management (DT Vega Architecture Chapter)
Açık kaynak örnek proje: Goal Management System (DT Vega Architecture Chapter ekibi tarafından geliştirildi)
- Repo: https://github.com/DTVegaArchChapter/Architecture/tree/main/ddd/goal-management-system
- Amaç: Organizasyon, hedef belirleme, ilerleme/onarım-onay akışları ve dönem sonu performans değerlendirme süreçlerini DDD ile modellemek.

### Event Storming
![Event Storming diyagramı - Goal Management DDD keşfi](https://raw.githubusercontent.com/DTVegaArchChapter/Architecture/main/ddd/goal-management-system/docs/eventstorming.png)

### Strategic Design
![Strategic Design bounded context haritası - Goal Management](https://raw.githubusercontent.com/DTVegaArchChapter/Architecture/main/ddd/goal-management-system/docs/StrategicDesign.png)

### Tactical Design - Örnek Aggregate ve Öğeleri (Goal Management)

![Goal Management Class Diagram - aggregate ve ilişkiler](https://raw.githubusercontent.com/DTVegaArchChapter/Architecture/main/ddd/goal-management-system/docs/goal-management-class-diagram.png)

- **Aggregate Roots**: GoalSet (kullanıcı/takım üyesinin hedef kümesi), GoalPeriod (dönem)
- **Entities**: Goal, GoalProgress
- **Value Objects**: GoalValue (min/mid/max ve tip)
- **Domain Events**: GoalAddedEvent, GoalProgressUpdatedEvent, GoalProgressApprovedEvent, GoalSetSentToApprovalEvent, GoalSetApprovedEvent ...
- **Örnek İş Kuralları**: Bir GoalSet altında en fazla 10 hedef; toplam yüzde 100 olmalı; statü bazlı onay/red akışları; min < mid < max tutarlılığı.

### Ekran örneği
![Goal Management ekranı - Kullanıcının Hedef Listesi](https://raw.githubusercontent.com/DTVegaArchChapter/Architecture/main/ddd/goal-management-system/docs/012-list-user-goals.png)

Daha detaylı dokümantasyon için [README](https://github.com/DTVegaArchChapter/Architecture/tree/main/ddd/goal-management-system#readme) dosyasını inceleyebilirsiniz.

Kodları incelemek için [tıklayın](https://github.com/DTVegaArchChapter/Architecture/tree/main/ddd/goal-management-system)

## SSS (DDD Hakkında)
- DDD (Domain-Driven Design) nedir? İş kurallarını merkeze alan, domain modelini kodla hizalayan bir yazılım tasarım yaklaşımıdır.
- Ne zaman DDD kullanılır? Karmaşık kurallar, uzun yaşayan sistemler, birden çok alt alan ve yoğun iş akışları olduğunda.
- Bounded Context nedir? Terimlerin ve kuralların değişmediği, modelin tutarlı olduğu sınır; her bağlam kendi modeline sahiptir.
- Aggregate Root nedir? Veri tutarlılığını koruyan kök nesne; kuralların uygulanmasını sağlar ve dış erişim genelde bu kök üzerinden yapılır.

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "DDD (Domain-Driven Design) nedir?",
      "acceptedAnswer": { "@type": "Answer", "text": "İş kurallarını merkeze alan, domain modelini kodla hizalayan bir yazılım tasarım yaklaşımıdır." }
    },
    {
      "@type": "Question",
      "name": "Ne zaman DDD kullanılır?",
      "acceptedAnswer": { "@type": "Answer", "text": "Karmaşık iş kuralları, uzun yaşayan sistemler, birden çok alt alan ve yoğun iş akışları olduğunda tercih edilir." }
    },
    {
      "@type": "Question",
      "name": "Bounded Context nedir?",
      "acceptedAnswer": { "@type": "Answer", "text": "Terimlerin ve kuralların değişmediği, modelin tutarlı olduğu sınırdır; her bağlam kendi modeline sahiptir." }
    },
    {
      "@type": "Question",
      "name": "Aggregate Root nedir?",
      "acceptedAnswer": { "@type": "Answer", "text": "Veri tutarlılığını koruyan kök nesnedir; kuralların uygulanmasını sağlar ve dış erişim çoğunlukla bu kök üzerinden yapılır." }
    }
  ],
  "inLanguage": "tr-TR"
}
</script>

## Sonuç
DDD; karmaşık iş kurallarını sürdürülebilir biçimde modellemek için güçlü bir çerçeve sunar. Doğru yerde uygulandığında hem ekip hizalanmasını hem de yazılımın evrimini kolaylaştırır. Basit problemlerde ise fazladan karmaşıklık getirebilir. Örnek projede, Event Storming'den stratejik ve taktik tasarıma kadar uçtan uca bir akış gösterilmektedir.

## Kaynaklar
- Eric Evans - Domain-Driven Design (kitap)
- Vaughn Vernon - Implementing Domain-Driven Design (kitap)
- Martin Fowler - Anemic Domain Model: https://martinfowler.com/bliki/AnemicDomainModel.html
- Martin Fowler - Transaction Script: https://martinfowler.com/eaaCatalog/transactionScript.html
- Martin Fowler - Active Record: https://martinfowler.com/eaaCatalog/activeRecord.html
- Martin Fowler - Domain Model: https://martinfowler.com/eaaCatalog/domainModel.html
- DDD Goal Management System (README, diyagramlar ve kodlar): https://github.com/DTVegaArchChapter/Architecture/tree/main/ddd/goal-management-system
