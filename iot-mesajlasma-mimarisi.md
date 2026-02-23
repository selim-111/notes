# IoT Mesajlaşma Mimarisi

---

## 1) Problem Tanımı ve Hedefler

### Senaryo
- Akıllı şehir trafik altyapısı
- 1.000.000+ cihaz (kamera, sensör, kontrol modülü)
- Tepe yükte 200.000+ mesaj/sn
- Gerçek zamanlı karar üretimi (ör. kavşak sinyal süresi)
- Operasyonel bildirimler (SMS, e-posta, push)

### Başarı Kriterleri
- Mesaj kaybını minimize etmek
- Yük altında düşük gecikme korumak
- Katmanları bağımsız ölçekleyebilmek
- Arıza anında servisi ayakta tutmak (HA)
- Analitik/AI tarafına replay edilebilir veri sunmak

---

## 2) Mimariyi Neden Katmanlara Ayırdım?

Tek bir mesajlaşma sistemi ile her problemi çözmeye çalışmak yerine, her katmana farklı bir sorumluluk verdim:

1. **MQTT** → Cihazdan güvenilir/ekonomik veri alma (edge ingestion)
2. **Kafka** → Kalıcı event omurgası (streaming backbone)
3. **RabbitMQ** → İş/komut kuyruğu (task processing)
4. **ZeroMQ** → Çok düşük gecikmeli kontrol hattı (realtime control)
5. **Kubernetes** → Dağıtım, ölçekleme, failover ve operasyon

Bu yaklaşımın ana avantajı: her katman kendi doğasına göre ölçekleniyor ve failure domain’leri ayrılıyor.

---

## 3) Uçtan Uca Veri Akışı (Mental Model)

```text
IoT Devices
	↓
MQTT Broker Cluster (Edge)
	↓
Ingestion Services
	↓
Kafka Topics (Source of Truth)
	↓
┌──────────────────────────────┬─────────────────────────────┐
│ Stream Processing / AI       │ Async Business Tasks        │
│ (Flink/Spark/Consumers)      │ (RabbitMQ + Workers)        │
└──────────────────────────────┴─────────────────────────────┘
	↓                                           ↓
Realtime Control Decisions               Notifications / Billing
	↓
ZeroMQ Control Plane
	↓
Physical Controllers
```

---

## 4) MQTT Katmanı (Edge Ingestion)

### Neden MQTT?
- Düşük bant genişliği tüketimi
- Güvenilmez ağlarda çalışabilme
- Düşük kaynaklı cihazlarla uyumluluk
- QoS seviyeleri ile teslimat esnekliği

### Pratik Tasarım Notlarım
- Telemetri için çoğunlukla QoS 1, kritik komut yanıtlarında QoS 2 düşünülür
- Topic hiyerarşisi cihaz türü/bölgeye göre standardize edilir
- Keep-alive ve session timeout değerleri sahaya göre optimize edilir

### Riskler ve Önlemler
- **Risk:** Broker kaybında bağlantı kırılır  
  **Önlem:** 3+ broker, StatefulSet, anti-affinity
- **Risk:** Tek broker’a dengesiz yük binmesi  
  **Önlem:** Client tarafında DNS/LB stratejisi ve bağlantı dağıtımı
- **Risk:** Uzun kesintide cihaz buffer taşması  
  **Önlem:** Cihaz tarafında bounded queue + geri basınç politikası

---

## 5) Kafka Katmanı (Streaming Backbone)

Kafka’yı sistemin **kalıcı olay günlüğü** olarak konumladım. Bu sayede replay, audit ve event-sourcing mümkün oluyor.

### Neden Kafka?
- Yüksek throughput
- Partition temelli paralellik
- Replication ile dayanıklılık
- Consumer group modeli ile esnek tüketim

### Partition Planlama (Örnek Hesap)

Tepe trafik: $200{,}000\ \text{msg/sn}$  
Ortalama güvenli partition kapasitesi: $15{,}000\ \text{msg/sn}$

$$
	ext{Minimum Partition} = \frac{200000}{15000} \approx 14
$$

Operasyonel headroom ve yeniden dengeleme maliyeti düşünülerek başlangıçta **48 partition** ile başlamak daha güvenli.

### Kritik Konfigürasyonlar
- `replication.factor = 3`
- `min.insync.replicas = 2`
- Rack awareness aktif
- SSD zorunlu (özellikle broker log diskleri)
- Retention ve segment ayarları kullanım amacına göre ayrılmalı

### Topic Stratejisi
- Ham veri topic’leri (`raw.*`)
- Zenginleştirilmiş veri topic’leri (`enriched.*`)
- Komut/event topic’leri (`command.*`, `event.*`)
- DLQ topic’leri (`dlq.*`)

---

## 6) RabbitMQ Katmanı (Task Processing)

Kafka “veriyi saklar”, RabbitMQ “işi yaptırır”. Bu ayrım, sistemi sadeleştirir.

### RabbitMQ’yu Nerede Kullanıyorum?
- E-posta/SMS/push gönderimi
- Faturalama ve rapor üretimi
- Webhook tetikleme
- Retry ve DLQ gerektiren asenkron işler

### Neden Bu Katman Ayrı?
- İş yürütme semantiği farklıdır
- ACK/NACK akışı daha nettir
- Retry policy yönetimi kolaydır

### Operasyon Notları
- DLQ her kritik kuyruk için zorunlu
- TTL + dead-letter pattern kullanımı
- Worker autoscale metrikleri: queue depth + processing latency
- Çok yoğun tek kuyruk yerine sharding yaklaşımı

---

## 7) ZeroMQ Katmanı (Gerçek Zamanlı Kontrol)

Bu katmanı yalnızca **milisaniye seviyesinde karar iletimi** gereken yerlerde kullanıyorum.

### Kullanım Örneği
AI, “Kavşak 42’de yeşil süre +7 sn” kararı verdiğinde komutun 5–10 ms penceresinde iletilmesi gerekiyor.

### Güçlü Yanlar
- Broker olmadan doğrudan socket iletişimi
- Düşük gecikme
- Esnek mesajlaşma pattern’leri

### Sınırlar
- Kalıcılık doğal olarak yok
- Retry/ordering/idempotency uygulamada tasarlanmalı
- Node kaybında mesaj düşebilir; fallback şart

---

## 8) Kubernetes Dağıtım Yaklaşımım

### Node Pool Ayrımı
- **Infra pool:** Kafka, MQTT, RabbitMQ
- **App pool:** Consumer/worker servisleri
- **Low-latency pool:** Realtime control servisleri

### Workload Türleri
| Bileşen | K8s Türü | Not |
|---|---|---|
| MQTT | StatefulSet | Kalıcı bağlantı ve durum |
| Kafka | StatefulSet | Disk ve broker kimliği kritik |
| RabbitMQ | StatefulSet | Queue state korunmalı |
| ZeroMQ servisleri | Deployment | Stateless kontrol katmanı |

### HA İlkeleri
- En az 3 AZ
- Pod anti-affinity
- PodDisruptionBudget
- Kaynak limit/request disiplinli tanımlanmalı
- HPA/VPA kararları metrik bazlı verilmeli

---

## 9) Arıza Senaryoları (Runbook Özeti)

### MQTT Broker Down
- Client otomatik reconnect
- QoS 1/2 ile yeniden iletim
- Uzun kesintide cihaz tarafı buffer politikası kritik

### Kafka Broker Down
- Leader election devreye girer
- ISR sağlıklıysa veri kaybı beklenmez
- Gecikme geçici artabilir

### RabbitMQ Worker Crash
- ACK alınmayan mesaj tekrar kuyruklanır
- İdempotent consumer yazımı şart

### ZeroMQ Node Down
- Paket kaybı olabilir
- Fallback kanal (örn. Kafka command topic) tanımlı olmalı

---

## 10) Event Sourcing ve CQRS Uygulama Notu

Kafka üzerinde tüm olaylar tutulduğu için, sistem durumunu olaylardan tekrar üretebiliyorum (event sourcing).  
Komut (write) ve sorgu (read) yollarını ayırarak CQRS modelini uyguluyorum.

### Örnek
- **Command:** `intersection/42/signal/update`
- **Query:** Dashboard’da “son 15 dk yoğunluk trendi”

Bu ayrım, hem performans hem de domain netliği sağlar.

---

## 11) Multi-Region ve Felaket Kurtarma

### Model
- **Region A:** Primary
- **Region B:** Replica (standby)

### Strateji
- Topic/DB replikasyonu
- DNS veya Global LB ile failover
- Active-passive başlangıç için daha düşük operasyon maliyeti

### Dikkat Noktaları
- RPO/RTO hedeflerini baştan net koymak
- Saat senkronizasyonu ve event ordering etkisini analiz etmek
- Failover tatbikatını düzenli yapmak

---

## 12) Gözlemlenebilirlik (Monitoring & Alerting)

### İzlediğim Temel Metrikler

**Kafka**
- Consumer lag
- Under-replicated partitions
- ISR değişimleri

**RabbitMQ**
- Queue depth
- Unacked messages
- Publish/ack oranı

**MQTT**
- Aktif bağlantı sayısı
- Disconnect/reconnect oranı
- Mesaj düşüm oranı

**ZeroMQ**
- P95/P99 gecikme
- Socket error oranı

### Stack
- Prometheus
- Grafana
- Alertmanager

### Alarm Prensibi
- Sadece “gürültü” üreten alarm değil, aksiyon doğuran alarm
- SLO bazlı eşik tanımları

---

## 13) Kapasite Planlama (Kısa Formül Defteri)

### Trafik Varsayımları
- 1M cihaz
- Cihaz başına tepe $5\ \text{msg/sn}$
- Teorik üst sınır: $5M\ \text{msg/sn}$

### Kafka Disk Hesabı (Kabaca)
- Ortalama mesaj: $1\ \text{KB}$
- Akış: $5M\ \text{msg/sn} \Rightarrow 5\ \text{GB/sn}$
- Saatlik: $5 \times 3600 = 18\ \text{TB/saat}$
- 24 saat retention: $18 \times 24 = 432\ \text{TB}$ (replikasyon hariç ham hesap)

### Çıkarım
- Sıkıştırma zorunlu
- Retention katmanlara göre ayrıştırılmalı
- Tiered storage ciddi maliyet avantajı sağlar

---

## 14) Güvenlik ve Uyum Notlarım

- Cihaz kimlik doğrulama (sertifika/anahtar yönetimi)
- TLS zorunlu taşıma güvenliği
- Topic/queue bazlı yetkilendirme (least privilege)
- Secrets yönetimi (K8s secrets + dış kasalar)
- Audit log ve erişim kaydı

---

## 15) Uygulama Sırası (Pratik Yol Haritası)

1. MQTT ingestion’ı stabilize et
2. Kafka topic ve şema stratejisini kilitle
3. Consumer idempotency + DLQ modelini kur
4. RabbitMQ task katmanını devreye al
5. Realtime kontrol yolunu (ZeroMQ) sadece kritik path’te aç
6. SLO/alert/runbook üçlüsünü canlıya çıkmadan tamamla