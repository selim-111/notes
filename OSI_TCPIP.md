# OSI ve TCP/IP Modelleri (Örnekler ve Kodlar ile)

## İçindekiler

1.  Katmanlı Mimari Mantığı
2.  OSI Modeli -- Derinlemesine İnceleme
3.  TCP/IP Modeli -- Gerçek Dünya
4.  OSI vs TCP/IP Eşleştirme
5.  Encapsulation / Decapsulation
6.  Gerçek Veri Akışı Senaryosu
7.  TCP vs UDP -- Derin Teknik Farklar
8.  HTTP Örneği (Gerçek Paket)
9.  Router & Switch Gerçek Senaryo

------------------------------------------------------------------------

# 1) Katmanlı Mimari Mantığı

Ağ iletişimi çok karmaşık bir süreçtir. Bir mesaj gönderildiğinde:

-   Uygulama veri üretir
-   Veri paketlenir
-   IP adresi eklenir
-   MAC adresi eklenir
-   Fiziksel sinyale çevrilir

Bu süreci yönetmek için sistem katmanlara bölünmüştür.

Her katman: - Belirli bir iş yapar - Üst katmandan veri alır - Alt
katmana iletir

Detaylı örnek akış (uygulama → fiziksel):

1) `Application` katmanında HTTP isteği oluşturulur.
2) `Transport` katmanı (TCP) isteği segmentlere böler ve sıra numarası verir.
3) `Network` katmanı (IP) her segmente IP başlığı ekler.
4) `Data Link` katmanı (Ethernet) frame oluşturur; kaynak/hedef MAC eklenir.
5) `Physical` katman elektriğe/optik/palpite çevirerek kabloya gönderir.

------------------------------------------------------------------------

# 2) OSI Modeli (7 Katman) -- Derinlemesine

## 7) Application Layer

Kullanıcının gördüğü katmandır.

### Protokoller

-   HTTP
-   FTP
-   SMTP
-   DNS

### Gerçek Örnek

Tarayıcıdan:

    https://google.com

Tarayıcı arka planda şunu yollar:

    GET / HTTP/1.1
    Host: google.com

------------------------------------------------------------------------

## 6) Presentation Layer

Veriyi dönüştürür.

### İşlevleri

-   Şifreleme (TLS)
-   Sıkıştırma
-   Karakter kodlama

### Örnek

-   UTF-8
-   JPEG
-   MP4

------------------------------------------------------------------------

## 5) Session Layer

İki cihaz arasındaki oturumu yönetir.

### Örnek

-   WhatsApp konuşması açık kalır
-   Web sitesinde login kalırsın

------------------------------------------------------------------------

## 4) Transport Layer

Veriyi küçük parçalara böler.

### TCP Özellikleri

-   Güvenli
-   Sıralı
-   Hata kontrolü

### UDP Özellikleri

-   Hızlı
-   Paket kaybı olabilir

### Port Kavramı

-   HTTP → 80
-   HTTPS → 443
-   MySQL → 3306

Transport katmanına kod örnekleri:

TCP (Python, bağlan-kapalı):

```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(('example.com', 80))
s.sendall(b'GET / HTTP/1.1\r\nHost: example.com\r\n\r\n')
print(s.recv(1024))
s.close()
```

UDP (Python, bağlanmadan gönderim):

```python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.sendto(b'hello', ('8.8.8.8', 53))  # DNS sunucusuna UDP paketi
```

------------------------------------------------------------------------

## 3) Network Layer

IP adresleme burada yapılır.

### Örnek

Bilgisayarın IP'si:

    192.168.1.10

Router bu IP'ye göre yönlendirir.

------------------------------------------------------------------------

## 2) Data Link Layer

MAC adresleri ile çalışır.

### Örnek MAC

    00:1A:2B:3C:4D:5E

Switch bu adres ile veri yollar.

Data Link örnekleri:

- Ethernet frame bileşenleri: preamble | dest MAC | src MAC | EtherType | payload | FCS
- ARP sorgusu (bash):

Scapy ile ARP sorgusu (Python):

```python
from scapy.all import ARP, Ether, srp
ans, _ = srp(Ether(dst='ff:ff:ff:ff:ff:ff')/ARP(pdst='192.168.1.0/24'), timeout=2)
for s,r in ans:
    print(r.psrc, r.hwsrc)
```

------------------------------------------------------------------------

## 1) Physical Layer

Gerçek sinyaller: - Elektrik - Fiber ışığı - WiFi dalgaları

Physical katman çeşitlerine örnek:

- Kablo tipleri ve hızlar: Cat5e (1Gbps), Cat6 (1-10Gbps), single-mode fiber (10G+)
- Wi-Fi frekansları: 2.4GHz vs 5GHz vs 6GHz

------------------------------------------------------------------------

# 3) TCP/IP Modeli (4 Katman)

## Application

OSI: Application + Presentation + Session birleşmiş.

## Transport

-   TCP
-   UDP

## Internet

-   IP

`Internet` katmanı: IP protokolünün rolleri — adresleme, en kısa yol seçimi ve paket yönlendirme.
## Network Access

-   MAC
-   Fiziksel iletim

------------------------------------------------------------------------

# 5) Encapsulation (Kapsülleme)

Veri aşağı inerken başlık eklenir:

    HTTP Data
    ↓
    TCP Header + Data
    ↓
    IP Header + TCP + Data
    ↓
    MAC Header + IP + TCP + Data

Byte-level örnek (basitleştirilmiş):

- Uygulama verisi: 100 oktet
- TCP header: 20 oktet (varsayılan)
- IP header: 20 oktet
- Ethernet header: 14 oktet

Toplam = 154 oktet (ek VLAN, options, FCS vb. eklenebilir).

------------------------------------------------------------------------

# 6) Gerçek Hayat Senaryosu

Google'a girildiğinde:

1)  HTTP isteği oluşur\
2)  TCP segmentlere böler\
3)  IP adresi ekler\
4)  MAC adresi ekler\
5)  Fiziksel sinyal olarak gider

Gözlemlenebilir adımlar ve komut örnekleri:

1) DNS: `dig google.com`
2) TCP el sıkışma: `sudo tcpdump -n -i any tcp and host google.com and port 443 -c 3`
3) TLS el sıkışması sonrası HTTPS trafik: `openssl s_client -connect google.com:443` (handshake detayları için)

Basitleştirilmiş zaman çizelgesi (paket bazlı):

- t0: DNS sorgusu gönderildi
- t1: DNS yanıtı alındı (IP)
- t2: TCP SYN gönderildi
- t3: TCP SYN-ACK alındı
- t4: TCP ACK gönderildi
- t5: TLS ClientHello gönderildi
- t6: TLS ServerHello + sertifika geldi
- t7: şifreli HTTP/2 akışı başladı

------------------------------------------------------------------------

# 7) TCP vs UDP -- Teknik

## TCP 3-Way Handshake

TCP 3-Way Handshake: bağlantı kurulurken kullanılan üç temel adım ve ilgili bayraklar:

1) Client → SYN (seq = x)
    - Client bağlantı isteği gönderir, SYN bayrağı set edilir, bir başlangıç sıra numarası (`seq`) atanır.
2) Server → SYN,ACK (seq = y, ack = x+1)
    - Server SYN+ACK ile cevap verir; hem kendi SYN'ini hem de client sırasını onaylamak için ACK gönderir.
3) Client → ACK (seq = x+1, ack = y+1)
    - Client son bir ACK gönderir; artık iki taraf veri göndermeye hazırdır.

Detaylar:
- `seq` (sequence number): gönderilen oktetlerin sıra numarası.
- `ack` (acknowledgment number): karşı tarafın gönderdiği son oktetten bir sonraki beklenen sıra numarası.
- Bayraklar: SYN, ACK, FIN, RST, PSH, URG.

Örnek `tcpdump` çıktısı (basitleştirilmiş):

     10:00:00.000000 IP 192.168.1.10.50000 > 93.184.216.34.80: Flags [S], seq 1000, win 65535
     10:00:00.010000 IP 93.184.216.34.80 > 192.168.1.10.50000: Flags [S.], seq 2000, ack 1001, win 64240
     10:00:00.020000 IP 192.168.1.10.50000 > 93.184.216.34.80: Flags [.], ack 2001, win 65535

Bu çıktıda ilk satır client'ın SYN, ikinci satır server'in SYN-ACK, üçüncü satır client'ın ACK'idir.

Neden önemli?
- Handshake, her iki tarafın da başlangıç sıra numaralarını senkronize eder; bu sıra numaraları güvenli, sıralı ve hataya dayanıklı veri aktarımı sağlar.

------------------------------------------------------------------------

# 8) HTTP Paket Örneği

Gerçek bir POST isteği:

    POST /login HTTP/1.1
    Host: example.com
    Content-Type: application/json

    {
     "username": "selim",
     "password": "123456"
    }

Gerçek sunucudan ham HTTP yanıtı görmek için:

```bash
curl -v -X POST https://httpbin.org/post -H 'Content-Type: application/json' -d '{"user":"selim"}'
```

Ham TCP paketleriyle HTTP akışı yakalamak için `tcpdump` veya `wireshark` kullanılabilir.

-------------------------------------------------------------------------

Not: `socket` örnekleri, gerçek uygulamalarda hata yönetimi, zaman aşımları, paket boyutu kontrolü ve güvenlik (TLS) gerektirir.

# 9) Router & Switch Gerçek Senaryo

Aynı evde:

Laptop → Switch → Modem → Router → Internet

Switch: - MAC ile çalışır

Router: - IP ile çalışır

Router örnekleri ve CLI komutları:

- Linux yol/route tabelesi görüntüleme:

```bash
ip route show
```

- Basit routing tablosu örneği:


Destination     Gateway         Genmask         Iface
0.0.0.0         192.168.1.1     0.0.0.0         eth0
192.168.1.0     0.0.0.0         255.255.255.0   eth0

------------------------------------------------------------------------

# 10) Veri Katmanlarda Nasıl İsim Değiştirir?

  Katman        Veri Adı
  ------------- ----------
  Application   Data
  Transport     Segment
  Network       Packet
  Data Link     Frame
  Physical      Bit

Örnek: Bir HTTP GET isteğinin isim dönüşümleri:

- Application: "GET /index.html"
- Transport: TCP segment (seq/ack)
- Network: IP paket (src/dst IP)
- Data Link: Ethernet frame (src/dst MAC)
- Physical: Bit dizisi

------------------------------------------------------------------------
