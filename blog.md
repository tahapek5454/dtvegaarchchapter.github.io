---
layout: page
title: Blog / Yazılar
permalink: /blog
---

Bu alanda teknik notlar, mimari karar kayıtları (ADR) ve kısa öğretici yazılar yayınlanacaktır.

## Planlanan Başlıklar

- Mimari Karar: Mesaj Kuyruğu Seçimi (RabbitMQ vs Kafka)
- Observability Minimum Gereksinimleri
- DDD Aggregate Tasarım İpuçları
- Go ile Oyun Döngüsü Basit Örneği
- Olay Tabanlı Entegrasyon Stratejileri

## Son Yazılar

{% for post in site.posts %}

### {{ post.date | date: "%Y-%m-%d" }} - [{{ post.title }}]({{ post.url | relative_url }}){% if post.tags %} • {{ post.tags | join: ", " }}{% endif %}

{{ post.excerpt | strip_html | truncate: 180 }}

---
{% endfor %}

## ADR (Architecture Decision Records)

İleride /adr klasörü (ayrı depo veya site içerisi) linkleri burada listelenecek.

> Öneri eklemek için issue açabilirsiniz.
