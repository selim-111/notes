# TCP Window Optimization (TCP Pencere Optimizasyonu) - Detaylı Rehber

## İçindekiler

1.  TCP Penceresi Nedir?
2.  Optimizasyon Yöntemleri
3.  Window Scaling (Pencere Ölçekleme)
4.  TCP Fast Open (TFO)
5.  Selective Acknowledgments (SACK)
6.  Congestion Control Algorithms
7.  Pratik Uygulama ve Sistem Yapılandırması
8.  Performans Ölçümleri ve Analiz

------------------------------------------------------------------------

# 1) TCP Penceresi Nedir?

## Tanım

TCP penceresi (window), gönderici tarafından onay (ACK) beklemeden gönderilebilecek maksimum veri miktarıdır. Bu, ağda "uçuş" halindeki veri miktarını kontrol eder.

## Neden Önemlidir?

- **Ağ Verimliliği**: Çok küçük pencere = düşük verim
- **Uzun Bağlantılar**: Uydu interneti (yüksek gecikme) = büyük pencere gerekir
- **Yüksek Hızlı Bağlantılar**: 10 Gbps link = çok fazla veri "uçuş" halinde

## Pencere Boyutu Hesaplaması

Optimum pencere boyutu şu formül ile hesaplanır:

```
Window Size = Bandwidth × RTT (Round Trip Time)
```

### Örnek Hesaplama

**Senaryo 1: Yerel Ağ**
- Bandwidth: 1 Gbps = 125 MB/s
- RTT: 1 ms (0.001 saniye)
- Pencere Boyutu = 125 MB/s × 0.001s = 125 KB

Bu neden önemli? Çünkü gönderici RTT kadar beklemeksizin veri gönderebilmesi gerekir.

**Senaryo 2: Uydu İnterneti (ABD-Avrupa)**
- Bandwidth: 1 Gbps = 125 MB/s
- RTT: 300 ms (0.3 saniye) - uydu gecikmesi
- Pencere Boyutu = 125 MB/s × 0.3s = 37.5 MB

Standart TCP penceresi: 64 KB
Gereken pencere: 37.5 MB
**Problem**: 64 KB / 37.5 MB = %0.17 verim sadece!

------------------------------------------------------------------------

# 2) Optimizasyon Yöntemleri - Genel Bakış

TCP pencere optimizasyonu 4 ana teknik ile gerçekleştirilir:

```
┌─────────────────────────────────────────────────┐
│         TCP Window Optimization                 │
├─────────────────────────────────────────────────┤
├─ Window Scaling (RFC 1323)                      │
│  └─ 64 KB → 1 GB                               │
├─ TCP Fast Open (RFC 7413)                       │
│  └─ Handshake sırasında veri gönder            │
├─ Selective ACK (RFC 2018)                       │
│  └─ Seçici onay mekanizması                    │
└─ Congestion Control                             │
   ├─ BBR (Google)                               │
   ├─ CUBIC (Linux varsayılan)                   │
   └─ Reno (Eski)                                │
```

------------------------------------------------------------------------

# 3) Window Scaling (Pencere Ölçekleme) - RFC 1323

## Nedir?

TCP penceresi alanı 16 bit = 65.535 (64 KB) ile sınırlıdır. Window Scaling, bir ölçek faktörü ekleyerek bu sınırı aşar.

## Formül

```
Gerçek Pencere Boyutu = Header'daki Değer × 2^(Scale Factor)
```

Maksimum ölçek faktörü: 14

```
Maksimum Pencere Boyutu = 65.535 × 2^14 = 65.535 × 16.384 = 1.073.676.288 bytes
                         ≈ 1 GB
```

## Nasıl Çalışır?

### Aşama 1: 3-Way Handshake Sırasında Anlaşma

```
Client                              Server
  |--- SYN (Scale Factor=7) ------->|
  |<------- SYN-ACK (Scale=7) ------|
  |----------- ACK ----------------->|

Scale Factor = 7 anlamına gelir:
Ölçek = 2^7 = 128

Sunucu karşı tarafa der: "Benim pencerem 64 KB ama scala 128, 
yani gerçek = 64 KB × 128 = 8 MB demektir."
```

### Aşama 2: Veri Transferi

```
Gönderilen pencere: 60.000 bytes (header'da)
Ölçek faktörü: 7 (2^7 = 128)
Gerçek pencere: 60.000 × 128 = 7.680.000 bytes = 7.5 MB
```

## Ölçek Faktörü Nasıl Seçilir?

Ölçek faktörü SYN kütüğüne yazılır ve değiştirilemez.

### Optimal Ölçek Faktörü Seçimi

```
Gerçek Pencere =  Bandwidth × RTT
Maksimum = 65.535

Ölçek Faktörü = ceil(log2(Gerçek Pencere / 65.535))
```

#### Örnek 1: Yerel Ağ (1 Gbps, RTT=1 ms)

```
Pencere = 125 MB/s × 0.001s = 125 KB
Ölçek = log2(125.000 / 65.535) = log2(1.9) = 0.9 ≈ 1
Seçilecek Ölçek = 2^1 = 2
Gerçek Pencere = 65.535 × 2 = 131.070 bytes ≈ 131 KB ✓
```

#### Örnek 2: Uydu İnterneti (1 Gbps, RTT=300 ms)

```
Pencere = 125 MB/s × 0.3s = 37.5 MB = 37.500.000 bytes
Ölçek = log2(37.500.000 / 65.535) = log2(572) = 9.15 ≈ 10
Seçilecek Ölçek = 2^10 = 1.024
Gerçek Pencere = 65.535 × 1.024 = 67.104.320 bytes ≈ 64 MB ✓
```

#### Örnek 3: Transonik Bağlantı (100 Gbps, RTT=10 ms)

```
Pencere = 12.500 MB/s × 0.01s = 125 MB = 125.000.000 bytes
Ölçek = log2(125.000.000 / 65.535) = log2(1.908) = 10.9 ≈ 11
Seçilecek Ölçek = 2^11 = 2.048
Gerçek Pencere = 65.535 × 2.048 = 134.209.280 bytes ≈ 128 MB ✓
```

## Linux'ta Aktivasyon

Window Scaling Linux'ta varsayılan olarak etkindir.

```bash
# Kontrol et
cat /proc/sys/net/ipv4/tcp_window_scaling

# Çıktı:
# 1 = Aktif
# 0 = Deaktif

# Etkinleştir (devre dışıysa)
sudo sysctl -w net.ipv4.tcp_window_scaling=1

# Kalıcı yap
echo "net.ipv4.tcp_window_scaling = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Windows'ta Aktivasyon

```cmd
# Kontrol et (PowerShell - Admin)
netsh int tcp show global

# Çıktı örneği:
# Receive-Side Scaling State          : enabled
# Receive Window Auto-Tuning Level    : normal

# Etkinleştir
netsh int tcp set global autotuninglevel=normal
```

## ethtool ile Linux'ta Özellik Kontrol

```bash
# TCP özelliklerini listele
ethtool -k eth0 | grep tcp

# Çıktı:
# tcp-segmentation-offload: on
# tcp-mangled-offload: off
```

## Sınırlamalar ve Dikkat Edilecekler

1. **Esca Yaşlı Ağ Cihazları**: Eski router/switch'ler Window Scaling'i desteklemeyebilir
   - Semptom: Bağlantı kurulamıyor
   - Çözüm: `tcp_window_scaling=0` yap

2. **NAT Arkasında**: Bazı firewall'lar Window Scaling seçeneğini bozabilir

3. **Simetrik Kullanım**: İki taraf da Window Scaling desteklemeli

------------------------------------------------------------------------

# 4) TCP Fast Open (TFO) - RFC 7413

## Nedir?

Normalde TCP, 3-aşamalı handshake'ten sonra veri gönderilebilir:

```
1) SYN gönder
2) SYN-ACK al (bekle)
3) ACK gönder
4) ÜZERİNDE Veri göndermeye başla
```

**Sorun**: 3 round trip = 3 × RTT = faydasız gecikme

TFO, **ilk SYN paketi ile veri göndermek** için özel bir mekanizmaydır.

## Nasıl Çalışır?

### Aşama 1: TFO Token Alımı (İlk Bağlantı)

```
Client                              Server
  |--- SYN (TFO Req, No Data) ----->|
  |<---- SYN-ACK (TFO Token) --------|
  |------- ACK --------------------->>| (Bu şimdiye kadarki standart)

Token = Sunucunun imza attığı, müşterinin sakla mesajı
```

### Aşama 2: TFO ile Bağlantı (Aynı Sunucuya Tekrar)

```
Client                              Server
  |--- SYN (TFO Token + DATA) ----->| ← VERİ BAŞLADI!
  |            :                     |   (Sunucu başlıyor)
  |            :                     |
  |<----------SYN-ACK + Response------|
  |----------- ACK ----------------->|

Zaman kazanç = 1 × RTT
```

## Faydası

Senaryo: Google'a bağlanmak, RTT = 100 ms

```
Standart TCP:
- SYN gönder: 100 ms
- SYN-ACK bekle: 100 ms
- ACK & İstek gönder: 100 ms
- Cevap başla
Toplam = 300 ms başlamadan önce

TFO:
- SYN + İstek: 100 ms
- Cevap başlıyor (parallel)
Toplam = 100 ms + processing
Kazanç = 200 ms (67% hızlanma)
```

## Linux'ta Aktivasyon

```bash
# TFO'yu etkinleştir
# Değerler:
# 0 = Deaktif
# 1 = İstemci (client) olarak TFO gönder
# 2 = Sunucu (server) olarak TFO kabul et
# 3 = Her ikisi de (Tavsiye edilen)

sudo sysctl -w net.ipv4.tcp_fastopen=3

# Kalıcı yap
echo "net.ipv4.tcp_fastopen = 3" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Kontrol et
cat /proc/sys/net/ipv4/tcp_fastopen
```

## Uygulamada Aktivasyon

### Python (HTTP istemcisi)

Standard HTTP kütüphaneleri TFO'yu otomatik kullanmaz. AsyncIO ile:

```python
import asyncio
import socket

async def fetch_with_tfo():
    # Socket oluştur
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    
    # TFO seçeneğini etkinleştir
    try:
        sock.setsockopt(socket.SOL_TCP, socket.TCP_FASTOPEN_CONNECT, 1)
    except AttributeError:
        print("TFO desteklenmiyor")
    
    # Bağlan + veri gönder
    sock.connect(("google.com", 80))
    sock.sendall(b"GET / HTTP/1.1\r\nHost: google.com\r\n\r\n")
    
    # Cevap al
    response = sock.recv(4096)
    sock.close()
    
    return response

# Çalıştır
result = asyncio.run(fetch_with_tfo())
print(result)
```

### Node.js (TFO desteği)

```javascript
const net = require('net');

const socket = new net.Socket();

// TCP_FASTOPEN seçeneğini ayarla
socket.setNoDelay(true); // Nagle'ı deaktif et

socket.connect({
  host: 'google.com',
  port: 80,
  family: 4  // IPv4
}, () => {
  console.log('Bağlı (TFO aktif)');
  socket.write('GET / HTTP/1.1\r\nHost: google.com\r\n\r\n');
});

socket.on('data', (data) => {
  console.log('Cevap alındı:', data.toString());
  socket.end();
});
```

## Sınırlamalar

1. **Token Süresi**: Sunucu tarafı token'ları kontrollü bir şekilde dağıtır
2. **Yeniden Oynatma (Replay) Riski**: İlk paket yeniden gönderilebilir
3. **UDP gibi Davranış**: İlk paket kaybolabilir (garanti yok)

------------------------------------------------------------------------

# 5) Selective Acknowledgments (SACK) - RFC 2018

## Nedir?

Paket kaybı olduğunda, TCP orjinal davranışta tüm pencereyi baştan göndermek zorundadır. SACK, sadece kaybolan paketlerin tekrar gönderilmesini sağlar.

## Sorunu Anlayalım

### Standart TCP (SACK'sız)

```
Gönderici: 1 2 3 4 5 6 7 8 →
Alıcı:     1 2 3 X 5 6 7 8 (4. paket kaybı)
Alıcı:     ← "ACK 4" (4. paketi istiyorum)

Gönderici: 4 5 6 7 8 → (Tüm penceri tekrar!)
Alıcı:     4 5 6 7 8
```

**Problem**: 4, 5, 6, 7, 8 tekrar gönderildi ama 5-8 zaten alınmıştı!

### SACK ile

```
Gönderici: 1 2 3 4 5 6 7 8 →
Alıcı:     1 2 3 X 5 6 7 8 (4. paket kaybı)
Alıcı:     ← "ACK 4, SACK 5-8" (4'ü gönder ama 5-8 alındı!)

Gönderici: 4 → (Sadece 4!)
Alıcı:     4
```

**Kazanç**: Saçataşz trafik %60 azalabilir!

## SACK İstatistiği

Tipik senaryoda paket kaybı oranı %0.1 ile %1 arasında:

```
1000 paket gönderilebilir
%0.5 kaybı = 5 paket kaybarı
SACK'sız dünyada = 5 × (pencere boyutu) = 5 × 100 = 500 paket tekrar
SACK'lı dünyada = 5 paket tekrar
Kazanç = (500-5)/500 = %99 trafik azalması
```

## Linux'ta Aktivasyon

SACK Linux'ta varsayılan olarak açıktır.

```bash
# Kontrol et
cat /proc/sys/net/ipv4/tcp_sack

# Çıktı:
# 1 = Aktif
# 0 = Deaktif

# Etkinleştir
sudo sysctl -w net.ipv4.tcp_sack=1

# Kalıcı yap
echo "net.ipv4.tcp_sack = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## SACK Mekanizması Detaylı

### SACK Bloğu Formatı

```
TCP Seçeneği: SACK
Tutacak türü: Başlangıç (4 bytes) - Bitiş (4 bytes)

Örnek: SACK [5-8] [10-12]
Anlamı: 5-8 arasındaki paketler alındı, 10-12 arasındaki paketler alındı
```

### Senaryo: Çoklu Aralık Kaybı

```
Gönderici:  1  2  3  4  5  6  7  8  9  10 11 12 →
Alıcı:      1  2  X  4  5  X  7  8  X  10 11 12

Alıcı gönderir:
"ACK 2, SACK [4-5], [7-8], [10-12]"
(3, 6, 9. paketleri gönder yeterli)
```

## D-SACK (Duplicate SACK) - RFC 2883

Hatalı paket tespiti için. Eğer paket zaten alındığı halde tekrar gelirse:

```bash
# D-SACK etkinleştir
sudo sysctl -w net.ipv4.tcp_dsack=1
echo "net.ipv4.tcp_dsack = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Performans Ölçümleri

### tcpdump ile SACK Paketleri Analiz Et

```bash
# Paketleri yakala
sudo tcpdump -i eth0 -w capture.pcap host 192.168.1.100

# Analiz et
tcpdump -r capture.pcap | grep SACK
```

Çıktı örneği:
```
14:32:45.123456 IP A.1234 > B.80: Flags [.], ack 1000, 
               win 512, options [sack 1 {900:1000}]
```

------------------------------------------------------------------------

# 6) Congestion Control Algorithms (Başlangıç Kontrol Algoritmaları)

## Nedir?

TCP gönderici, ağda tıkanıklık (congestion) tespit ettiğinde pencere boyutunu dinamik olarak ayarlar. Bu, internet'te trafik hayvanat bahçesi olmasını önler.

## Büyük Resim

```
┌─────────────────────────────────────────────────┐
│         Congestion Control Döngüsü              │
├─────────────────────────────────────────────────┤
│                                                 │
│ 1. Veri gönder ───→ 2. Paket kaybını bekle    │
│                     │                          │
│ 3. Pencereyi azalt  ←── Zaman aşımı?          │
│    (agresif)             Duplikat ACK mi?     │
│                                                 │
│ 4. Yavaş yavaş artır → (Recovery Phase)       │
│                                                 │
└─────────────────────────────────────────────────┘
```

## Algoritmaları Karşılaştırma

| Algoritma | Agresiflik | Uyumluluk | CPU | RTT Dayanıklılık | Best For |
|-----------|-----------|--------|(---|------------------|----------|
| Reno | Düşük | Çok iyi | Az | Kötü | Eski sistemler |
| CUBIC | Orta | İyi | Az | İyi | Linux varsayılan |
| BBR | Çok yüksek | Orta | Orta | Çok iyi | Mobil/uzun RTT |
| Vegas | Proaktif | Düşük | Yüksek | Çok iyi | Lab ortamları |

## 1) Reno (TCP Reno) - RFC 5681

En eski, halen yaygın kullanılan algoritma.

### Aşamalar

**Aşama 1: Slow Start**

```
Başlangıç penceresi (IW) = 10 × MSS
RTT başına ikiye katlan

Örnek (MSS=1500 byte):
RTT 1: 10 paket
RTT 2: 20 paket
RTT 3: 40 paket
RTT 4: 80 paket
RTT 5: 160 paket (kapasiteyi buldu → tıkanıklık)
```

İşe yaruyor mu?

```
RTT-1: 10 × 1500 = 15 KB gönder
RTT-2: 20 × 1500 = 30 KB gönder
RTT-3: 40 × 1500 = 60 KB göndir
...
```

**Aşama 2: Tıkanıklık Tespit**

```
Zaman Aşımı Oluyor:
- Pencereyi SSThresh'e (Slow start threshold) ayarla
- SSThresh = mevcut_pencere / 2

Örnek:
Mevcut pencere = 160 paket
Zaman aşımı! 
SSThresh = 160 / 2 = 80 paket
Pencereyi 10'a reset et (Slow Start başt)
```

**Aşama 3: Congestion Avoidance**

```
Pencere = SSThresh'i geçtikten sonra YANGIN ÜZERİ
Pencereyi her RTT başına 1 MSS artır (lineer)

Örnek (SSThresh=80):
RTT başına +1 paket

Pencere: 80 → 81 → 82 → 83 ...
```

### Linux'ta Aktivasyon

```bash
# Varsayılan olarak etkindir
cat /proc/sys/net/ipv4/tcp_congestion_control

# Eğer değilse:
sudo sysctl -w net.ipv4.tcp_congestion_control=reno
```

## 2) CUBIC - Varsayılan (Linux)

CUBIC, özellikle yüksek bant genişliğinde Reno'dan çok daha iyidir.

### Temel Fark

Reno: Pencere = Sabit hız ile artır (Linear)
CUBIC: Pencere = Matematiksel eğri ile artır (Cubic)

### Formül

```
Pencere(t) = C × (t - K)³ + W_max

t = Zaman (recovery'den sonra)
C = Sabit (CUBIC agresifliği)
K = Sabit (CUBIC dönüş noktası)
W_max = Önceki en yüksek pencere
```

Basit açıklamalar: Başlangıçta hızlı artır, sonra yavaşla.

### Davranış Karşılaştırması

```
Pencere boyutu gerçekleştirmesi (Tıkanıklık sonrası recovery)

RENO:
|     
|           ___─────────
|      ____╱
|   ╱╱
└─────────────────────
0    5    10   15   20 RTT

CUBIC:
|     
|           ────────
|      ╱╱╱╱╱
|  ╱
└─────────────────────
0    5    10   15   20 RTT

(CUBIC başlangıçta daha hızlı, sonra kontrollü)
```

### Linux'ta Aktivasyon

```bash
# CUBIC'e geç
sudo sysctl -w net.ipv4.tcp_congestion_control=cubic

# Kalıcı yap
echo "net.ipv4.tcp_congestion_control = cubic" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Kontrol et
cat /proc/sys/net/ipv4/tcp_congestion_control
```

### CUBIC Parametreleri

```bash
# CUBIC'in agresifliğini artır (daha hızlı recovery)
# Varsayılan C = 0.4

# /etc/sysctl.conf'a ekle (uyarı: test et)
net.ipv4.tcp_cubic_c = 0.4  # Aynen kalsın
net.ipv4.tcp_cubic_beta = 0.7  # Ağırlık faktörü
```

## 3) BBR (Bottleneck Bandwidth and Round-trip time) - Google

Hem bant genişliği hem de RTT'ye dayalı algoritma. Mobil ağlarda %25-300 iyileştirme sağlar.

### Mantığı

Reno/CUBIC: "Paket kaybını bekle"
BBR: "Band genişliğini ve RTT'yi ölç, dolu taşımadan tahmin et"

```
BBR Döngüsü:

1) GAIN (Kazanç) Aşaması: Pencereyi dinamik artır
   - Gain = 2.0 (çok agresif) → 1.0 (normal) → 0.75 (muhafazakar)

2) DRAIN (Drenaj) Aşaması: Ağdan gönderilen veriyi çıkart
   - Pencereyi küçült × 0.75

3) PROBE BW: En iyi bant genişliğini keşfet (her 10s)
   - Pencereyi momentlı +1 MSS yap → tepki gözle

4) PROBE RTT: Minimum RTT'yi keşfet (her 10s)
```

### Örnek Metric

```
Ölçülen:
- Bant = 100 Mbps = 12.5 MB/s
- RTT = 50 ms
- Pencere = 12.5 MB/s × 0.05s = 625 KB

BBR Pencere = 625 KB (optimal)

Reno/CUBIC = 64 KB (çok küçük!)
```

### Linux'ta Aktivasyon

BBR Linux 4.9+ içinde, kernel derlemesi gerekebilir.

```bash
# Kontrol et
cat /proc/sys/net/ipv4/tcp_available_congestion_control

# Çıktı örneği:
# reno cubic bbr

# BBR'ye geç
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# Kalıcı yap  
echo "net.ipv4.tcp_congestion_control = bbr" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Aktivasyon kontrol et
sysctl net.ipv4.tcp_congestion_control
```

### BBR Parametreleri

```bash
# BBR Min RTT filter window (varsayılan 10 saniye)
# TTL rotara bağlı değişken RTT'leri filtrelemek için
net.ipv4.tcp_bbr_min_rtt_wnd_sec = 10

# Probe yazılmasında min interval (ms)
net.ipv4.tcp_bbr_probe_rtt_win_ms = 10000
```

### BBR İstatistiği

```bash
# BBR'nin çalışmasını ss komutu ile kontrol et
ss -tin | grep -i bbr

# Örnek:
# cubic
# bbr

# Veya detailed:
ss -tin -e
```

## 4) Vegas (TCP Vegas) - RFC 3742

Ağda tıkanıklık **öncesinde** tepki veren proaktif algoritma. Production'da nadir.

### Mantığı

```
Vegas: "RTT'ye bakveri, azaltma yaparsam daha iyi olur"
Reno: "Paket kaybını bekliyorum..."

Vegas Temel Denklem:
Pencere = alpha, beta değerlerine dayalı
Farklı = (Beklenen_rate - Ölçülen_rate)

Algoritma:
if Fark < alpha: Pencereyi artır
if Fark > beta: Pencereyi azalt
else: Stabil tut
```

Problememleri: CPU yoğun, diğer algoritmalarla uyumsuz.

### Aktivasyon (Eski sistemler için)

```bash
sudo sysctl -w net.ipv4.tcp_congestion_control=vegas
```

------------------------------------------------------------------------

# 7) Pratik Uygulama ve Sistem Yapılandırması

## Tamamen Optimize Edilmiş Linux Sunucusu

### Adım 1: Kernel Parametreleri Ayarla

```bash
# Kök kullanıcısı ol
sudo -i

# /etc/sysctl.conf dosyasını aç
vim /etc/sysctl.conf

# Şu satırları ekle/düzenle:

# ===== TCP Window Optimization =====
net.ipv4.tcp_window_scaling = 1         # Window Scaling etkinleştir
net.core.rmem_max = 134217728           # Max 128 MB receive buffer
net.core.wmem_max = 134217728           # Max 128 MB write buffer

# ===== TCP Buffer Tuning =====
# Format: min default max (bytes)
net.ipv4.tcp_rmem = 4096 87380 67108864 # Receive memory (64 MB max)
net.ipv4.tcp_wmem = 4096 65536 67108864 # Write memory (64 MB max)

# ===== TCP Fast Open =====
net.ipv4.tcp_fastopen = 3               # Client + Server TFO

# ===== SACK =====
net.ipv4.tcp_sack = 1                   # Selective ACK etkinleştir
net.ipv4.tcp_dsack = 1                  # Duplicate SACK

# ===== Congestion Control =====
net.ipv4.tcp_congestion_control = bbr   # BBR algoritması
net.core.default_qdisc = fq             # Fair Queueing (BBR ile birlikte)

# ===== Backlog ve Connection =====
net.core.netdev_max_backlog = 5000      # NIC backlog
net.ipv4.tcp_max_syn_backlog = 5000     # SYN queue
net.ipv4.tcp_max_tw_buckets = 2000000   # Time-wait buckets

# ===== TCP Keepalive =====
net.ipv4.tcp_keepalive_time = 600       # 10 dakika
net.ipv4.tcp_keepalive_probes = 5       # 5 prob
net.ipv4.tcp_keepalive_intvl = 15       # 15 saniye aralıklar

# ===== Nagle'ın Algoritması =====
# NOT: TCP_NODELAY ile uygulama seviyesinde kontrol etmek daha iyi

# Kaydet ve çık (:wq)

# Ayarları yeniden yükle
sysctl -p
```

### Adım 2: İlgili Limit'leri Kontrol Et

```bash
# File descriptors limitini kontrol et
ulimit -n

# Çok küçükse (örneğin 1024), artır:
# /etc/security/limits.conf ekle:
*               soft    nofile          100000
*               hard    nofile          100000

# Sonra logout/login yap
```

### Adım 3: NIC Optimize Etme

```bash
# Ethtool ile RX/TX ring buffer'ı kontrol et
ethtool -g eth0

# Çıktı:
# Ring parameters for eth0:
# Pre-set maximums:
# RX:    4096
# RX Mini: 0
# RX Jumbo: 0
# TX:    4096

# Maksimuma ayarla
sudo ethtool -G eth0 rx 4096 tx 4096

# Kalıcı yap (ubuntu/debian):
# /etc/network/interfaces veya Netplan

# Veya systemd service oluştur:
sudo tee /etc/systemd/system/ethtool-optimize.service << EOF
[Unit]
Description=Optimize ethtool settings for high performance
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool -G eth0 rx 4096 tx 4096
ExecStart=/usr/sbin/ethtool -K eth0 gso on gro on tso on

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable ethtool-optimize
sudo systemctl start ethtool-optimize
```

### Adım 4: Monitoring ve Doğrulama

```bash
# Tüm TCP ayarlarını kontrol et
sysctl -a | grep tcp

# Belirli ayarları kontrol et
sudo ss -thin | head

# TCP istatistikleri (önemli metrikler)
netstat -s | grep -i tcp

# Çıktı örneği:
# Tcp: 123456 active connections openings
# 654321 passive connection openings
# 12345 failed connection attempts

# Kaybolan paketleri kontrol et
netstat -s | grep -i retrans

# BBR kullanılıyor mu?
cat /proc/sys/net/ipv4/tcp_congestion_control
```

## Windows Server Optimizasyonu

### PowerShell (Admin) ile Ayarlama

```powershell
# Tüm TCP parametreleri göster
Get-NetTCPSetting

# Global TCP ayarları göster
netsh int tcp show global

# AutoTuning'i kontrol et (Windows 7+)
# Values: RstrScenarios (varsayılan), Disabled, Normal, Aggressive
netsh int tcp set global autotuninglevel=normal

# SACK etkinleştir
netsh int tcp set global sack=enabled

# TCP Fast Open etkinleştir (Windows 10+)
netsh int tcp set global fastopen=enabled

# Başlangıç penceresi değişkeni (ISS)
netsh int tcp set global initialRto=2000

# Kalıcı olarak:
netsh int tcp set global autotuninglevel=normal
netsh int tcp set global sack=enabled
netsh int tcp set global fastopen=enabled
```

### Kayıt Defteri (Registry) Yöntemi

```powershell
# Administrator PowerShell aç

# TCP 1323 Extensions (Window Scaling)
New-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters' `
  -Name 'Tcp1323Opts' -Value 1 -PropertyType DWord -Force

# SACK
New-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters' `
  -Name 'SackOpts' -Value 1 -PropertyType DWord -Force

# Başlangıç pencere boyutu (Bytes)
New-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters' `
  -Name 'TcpWindowSize' -Value 67108864 -PropertyType DWord -Force

# TCP FastOpen (Windows 10 1607+)
New-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters' `
  -Name 'EnableTcpFastOpen' -Value 1 -PropertyType DWord -Force

# Değişiklikleri sınamak için reboot et
Restart-Computer -Force
```

------------------------------------------------------------------------

# 8) Performans Ölçümleri ve Analiz

## Benchmark Senaryoları

### Senaryo 1: Uzun Hasıl Bağlantı (Uydu/Intercontinental)

**Test Kurulumu:**
- Windows ve Linux arasında 200 ms RTT
- 1 Gbps link
- 0.1% paket kaybı

**Test Aracı:**

```bash
# Linux alıcısı
# Iperf3 kur
sudo apt install iperf3

# Server modu
iperf3 -s -p 5201

# Windows gönderici
# İndür: https://iperf.fr/iperf-download.php

# Command prompt'ta
iperf3.exe -c <linux_ip> -p 5201 -i 1 -t 60 -w 128M

# Çıktı örneği:
# [ ID] Interval      Transfer     Bandwidth
# [  4]   0.00-1.00 sec  115.2 MBytes   966 Mbps
# [  4]   1.00-2.00 sec  118.4 MBytes   994 Mbps
# ...
```

**Ölçüm Karşılaştırması:**

```
Belliament Test:
┌────────────────────────────────────────────┐
│ Metrik          │ Optimize Öncesi │ Sonrası │
├─────────────────┼─────────────────┼─────────┤
│ Throughput      │ 450 Mbps        │ 985 Mbps│
│ Retransmissions │ 2345            │ 12      │
│ Window Size     │ 64 KB (sabit)   │ 128 MB  │
│ RTT             │ 200 ms          │ 200 ms  │
└────────────────────────────────────────────┘
```

### Senaryo 2: Yüksek Hızlı LAN (100 Gbps simülasyonu)

```bash
# 100 Gbps üzerinden:

# Dosya kopyala (scp)
scp -C -p -r large_dir user@192.168.1.50:/dest

# SSH üzerinde test:
ssh user@192.168.1.50 "time cat filesize_10GB.img" | dd of=/dev/null bs=1M

# Direct TCP:
nc -l -p 5555 -q 1 < large_file.iso &
nc 192.168.1.50 5555 > /dev/null

# İstatistik:
time nc 192.168.1.50 5555 > output_file.iso
```

**Verileri Karşılaştırın:**

```
Test Sonuçları:
- Window Scaling AÇIK: 98 Gbps (teorik 100 Gbps)
- Window Scaling KAPALI: 12 Gbps (%12 verim)
```

## Tcpdump ile Gerçek Paket Analizi

### Pencere Boyutu Kontrolü

```bash
# Paketleri yakala (10 adet)
sudo tcpdump -i eth0 'tcp[13] & 0x02' -c 10 -vv -s 96

# SYN paketlerini ara (window scaling options)
sudo tcpdump -i eth0 'tcp[13] & 0x02' -A -vv

# Çıktı örneği:
# ...tcp-options=WS=7, TS val 123456789 ecr 987654321
# (WS = Window Scaling = 2^7 = 128 çarpan)
```

### SACK Paketleri

```bash
# ACK paketlerinde SACK blokları gözle
sudo tcpdump -i eth0 'tcp[13] & 0x10' -vv | grep -i sack

# Örnek:
# ack 1000 win 512 options [sack 1 {900:1000}]
```

### TCP Fast Open Tespiti

```bash
# SYN paketlerinde TFO seçeneği
sudo tcpdump -i eth0 'tcp[13] & 0x02' -A -vv | grep -i 'fastopen\|tfo'

# Veya
sudo tcpdump -i eth0 -vv | grep 'TCP.*Fast'
```

## ss Komutu ile Detaylı İstatistikler

```bash
# Detaylı TCP bağlantı bilgisi
ss -tin

# Çıktı:
# State    Recv-Q Send-Q
# ESTAB    0      0

# RTT ve Retransmit sayısını göster
ss -tin show

# BBR bilgisi
ss -tin -e | grep -i bbr

# Congestion control algoritması
ss -tni | awk '{print $1, $5, $7}'
```

## Wireshark ile GUI Analizi

```bash
# Pcap dosyası oluştur
sudo tcpdump -i eth0 -w tcp_analysis.pcap -c 1000

# Wireshark ile aç
wireshark tcp_analysis.pcap

# Filters (Wireshark'ta):
# - TCP Options: tcp.options.window_scale
# - SACK Blocks: tcp.options.sack
# - TFO: tcp.options.tfo
# - Retransmissions: tcp.analysis.retransmission
```

## Performance Test Ozeti

### test.sh Script'i

```bash
#!/bin/bash

# TCP Window Optimization Performance Test

echo "======== TCP Window Optimization Test ========"
echo "Test Tarihi: $(date)"
echo ""

# 1. Kernel Parametrelerini Kontrol Et
echo "1. Kernel Ayarları:"
echo "  Window Scaling: $(cat /proc/sys/net/ipv4/tcp_window_scaling)"
echo "  SACK: $(cat /proc/sys/net/ipv4/tcp_sack)"
echo "  Fast Open: $(cat /proc/sys/net/ipv4/tcp_fastopen)"
echo "  Congestion Control: $(cat /proc/sys/net/ipv4/tcp_congestion_control)"
echo ""

# 2. Buffer Boyutları
echo "2. Buffer Boyutları (bytes):"
echo "  TCP RMem: $(cat /proc/sys/net/ipv4/tcp_rmem)"
echo "  TCP WMem: $(cat /proc/sys/net/ipv4/tcp_wmem)"
echo ""

# 3. Ağ İstatistikleri
echo "3. TCP İstatistikleri:"
netstat -s | grep -i "tcp"
echo ""

# 4. Açık Bağlantılar
echo "4. Açık TCP Bağlantıları:"
ss -tan | grep ESTAB | wc -l
echo ""

# 5. Retransmission Oranı
echo "5. Retransmission Verisi:"
netstat -s | grep -i retrans
echo ""

echo "======== Test Tamamlandı ========"
```

Çalıştır:
```bash
chmod +x test.sh
./test.sh
```

------------------------------------------------------------------------

# Özet Rehberi

## Hızlı Kontrol Listesi (Quick Checklist)

```
☐ Window Scaling etkinleştirildi (tcp_window_scaling=1)
☐ Buffer boyutları ayarlandı (tcp_rmem/tcp_wmem max 128 MB)
☐ SACK etkinleştirildi (tcp_sack=1)
☐ D-SACK etkinleştirildi (tcp_dsack=1)
☐ TCP Fast Open etkinleştirildi (tcp_fastopen=3)
☐ Congestion control = BBR (yeni sistemler) veya CUBIC
☐ NIC ring buffer maksimize edildi
☐ Iperf3 veya benzer araçla test yapıldı
☐ Tcpdump/Wireshark ile doğrulandi
☐ Sysctl ayarları /etc/sysctl.conf'a yazıldı
```

## Seçim Kılavuzu

| Durumunuz | Önerilen Ayar |
|-----------|---------------|
| Eski sistem (Windows XP, Linux 2.4) | Window Scaling'siz TCP |
| Standart Linux server (2020+) | BBR, Window Scaling, SACK |
| Yüksek Hop Sayacı Ortam | CUBIC, yapılandırılmış RTT |
| Mobil/İnatçı Bağlantı | BBR, TFO, SACK |
| Veri Merkezi (LAN) | BBR, geniş buffer, TSO/GSO |
| Uydu/Uzun RTT | BBR, büyük pencere (100+ MB) |

------------------------------------------------------------------------

**Sonuç**: TCP Window Optimization, modern ağlarda verim %50-70 artırabilir. Sistem ve ağ özelliklerine göre yapılandırılan parametreler kritik öneme sahiptir.
