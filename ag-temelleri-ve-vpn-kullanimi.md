# Ağ Teşhisi, Yetki Yönetimi ve Pratik VPN (VPNBook / OpenVPN) Kılavuzu

Bu dokümantasyon; ağ bağlantı testleri, Linux yönetici kabuğuna geçiş, DNS/VPN mekanizmaları ve ücretsiz VPN servisleri (VPNBook) üzerinden güvenli tünel yapılandırmasını içerir.

---

## 1. Ağ Erişimi ve Gecikme Testi: `ping`

`ping`, hedef IP adresi veya alan adı ile sistem arasındaki erişilebilirliği ve paket iletim süresini (gecikme/latency) test etmek için **ICMP (Internet Control Message Protocol)** yankı (Echo Request/Reply) paketlerini kullanır.

### Temel Kullanım ve Parametreler
* **Hedefe ping atma (Varsayılan olarak Linux'ta durdurulana kadar devam eder):**
  ```bash
  ping google.com
  ```
* **Belirli sayıda paket gönderip durdurma (`-c` parametresi):**
  ```bash
  ping -c 4 8.8.8.8
  ```
* **Paket aralığını değiştirme (`-i` parametresi):**
  ```bash
  ping -c 5 -i 0.5 192.168.0.1
  ```

### Çıktı Analizi
* **`ttl` (Time to Live):** Paketin hedefe ulaşana kadar kaç yönlendiriciden (hop) geçebileceğini gösterir; her geçişte 1 azalır.
* **`time=XX ms`:** Paketin gidip geri dönmesi arasında geçen gidiş-dönüş süresidir (RTT).
* **`packet loss`:** Gönderilen paketlerin kaybolma oranıdır (%0 olmalıdır; paket kaybı ağda sorun olduğuna işaret eder).

---

## 2. Yetki Yönetimi: `sudo su` Mantığı

Ağ yapılandırma dosyalarını değiştirmek (`/etc/resolv.conf`) ve tünel arayüzleri (`tun0`) oluşturmak doğrudan çekirdek seviyesinde yönetici yetkisi gerektirir.

* **`sudo <komut>`:** Tek bir komutu süper kullanıcı yetkisiyle yürütür.
* **`sudo su` (veya `sudo -i`):** Standart kullanıcı oturumundan doğrudan tam yetkili `root` kabuğuna geçiş yapar.
  ```bash
  sudo su
  # Komut satırı sembolü '$' yerine '#' haline gelir (root@kali:~#)
  ```
* **Güvenlik Notu:** Root kabuğundayken dikkatli olunmalıdır; hatalı çalıştırılan bir dosya silme veya yapılandırma komutu tüm işletim sistemini bozabilir. İşlem tamamlandığında standart kullanıcıya dönmek için `exit` komutu kullanılır.

---

## 3. IP, DNS ve Yönlendirici (Router) İlişkisi

1. **Yerel IP (LAN):** Yönlendiricinin (Router) yerel ağdaki cihazlara (telefon, bilgisayar vb.) dağıttığı `192.168.0.X` formatındaki adreslerdir.
2. **Ağ Geçidi (Gateway):** Cihazların internete çıkmak için başvurduğu yönlendirici IP'sidir (Genellikle `192.168.0.1`).
3. **Genel IP (WAN):** İnternet Servis Sağlayıcınızın (İSS) modeminize atadığı ve dış dünyada görünen tekil IP adresidir.
4. **DNS (Domain Name System):** `google.com` gibi alan adlarını IP adreslerine çeviren rehberdir.

### Linux'ta DNS Değiştirme
```bash
sudo nano /etc/resolv.conf
```
Dosya içine genel DNS sunucuları tanımlanır:
```text
nameserver 1.1.1.1
nameserver 8.8.8.8
```

---

## 4. OpenVPN Mimarisi

**OpenVPN**, istemci ile uzak bir sunucu arasında SSL/TLS şifrelemesi kullanarak sanal bir ağ tüneli oluşturan açık kaynaklı bir protokoldür.

* Bağlantı kurulduğunda Linux sisteminde sanal bir ağ arayüzü (**`tun0`** - Tunnel Interface) oluşturulur.
* Cihazın ürettiği tüm TCP/UDP veri paketleri bu arayüze yönlendirilir, paketler şifrelenir ve yönlendirici üzerinden VPN sunucusuna iletilir.
* Dış dünyadaki sunucular isteğin gerçek IP'den değil, VPN sunucusunun IP adresinden geldiğini görür.

---

## 5. Pratik VPN Uygulaması: VPNBook ile Kali Linux Bağlantısı

VPNBook; test, gizlilik ve eğitim amaçlı ücretsiz OpenVPN yapılandırma profilleri sunan bir servistir.

<Steps>
  <Step subtitle="1 dakika" title="Yapılandırma Dosyalarını İndir">
    Tarayıcıdan `vpnbook.com/freevpn` sayfasına gir:
    * **"OpenVPN"** sekmesindeki paketlerden birini (örneğin *PL OpenVPN Certificate Bundle* veya *FR/US Bundle*) `.zip` olarak indir.
    * Aynı sayfada o an geçerli olan **Username** (genellikle `vpnbook`) ve düzenli güncellenen **Password** bilgisini not al.
  </Step>

  <Step subtitle="30 saniye" title="Arşivi Çıkart">
    Terminali açıp indirilenler klasörüne git ve zip dosyasını aç:

```bash
cd ~/Downloads
unzip VPNBook.com-*.zip
```
  </Step>

  <Step subtitle="1 dakika" title="OpenVPN ile Bağlantıyı Başlat">
    Çıkan `.ovpn` dosyalarından birini (TCP 443 veya UDP 53/25000) seçip yönetici yetkisiyle başlat:

```bash
sudo openvpn --config vpnbook-pl-tcp443.ovpn
```
    *Terminalde sorulduğunda siteden aldığın kullanıcı adı ve şifreyi gir.*

    *Doğrulama:* Terminal akışının sonunda `Initialization Sequence Completed` mesajını gör.
  </Step>

  <Step subtitle="30 saniye" title="Yeni IP ve Tünel Durumunu Doğrula">
    Mevcut terminali kapatmadan yeni bir terminal sekmesi aç ve doğrulamayı yap:

```bash
# Sanal tünel adaptörünü (tun0) görüntüle
ip addr show tun0

# Dış dünyaya çıkan genel IP adresini kontrol et
curl [https://ifconfig.me](https://ifconfig.me)
```

    *Doğrulama:* `tun0` üzerinde yeni bir IP adresi görüldüğünü ve `curl` çıktısındaki IP'nin ev IP'sinden farklı (VPNBook sunucusunun IP'si) olduğunu doğrula.
  </Step>
</Steps>
