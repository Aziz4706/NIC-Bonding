# Linux'ta NIC Bonding: Yüksek Performans ve Yedeklilik Rehberi

> Bu yazı, hem bireysel geliştiriciler hem de sistem yöneticileri için NIC bonding kavramının ne olduğunu, neden kullanılması gerektiğini ve Ubuntu 22.04 ortamında nasıl uygulandığını adım adım anlatan teknik bir rehberdir.

---

## ✨ NIC Bonding Nedir?

NIC bonding, birden fazla fiziksel ağ arabirimini (Network Interface Card - NIC) tek bir mantıksal arabirim gibi kullanarak ağ trafiğini daha verimli hale getirmeye yarar. Bu, hem **performans artışı** (toplam bant genişliği) hem de **yedeklilik** (high availability) sağlar.

Diğer adlarıyla da bilinir:

* **Link Aggregation (LAG)**
* **LACP (802.3ad / 802.1AX)**
* **Interface Bonding**

---

## 🧠 Neden NIC Bonding Kullanılır?

### ✅ 1. Redundancy (Yedeklilik)

Eğer bir NIC ya da switch portu arızalanırsa, diğer NIC anında devreye girerek **bağlantının kesilmesini engeller.**

### ⚡ 2. Performans / Yük Dengeleme

Birden fazla NIC aynı anda veri taşıyabilir. Bu da **toplam bant genişliğini artırır.**

> Örnek: 2 adet 1 Gbps NIC → teorik 2 Gbps toplam bant genişliği (protokol overhead'leri dışında).

### 🚧 Ne Zaman Kullanılır?

* Web sunucuları
* Proxy/DLP/SIEM gibi güvenlik cihazları
* Switch uplink'leri
* Sunucu-sunucu yoğun veri transferleri

---

## 📊 NIC Bonding Modları (Linux'ta `mode` değerleri)

| Mod | Adı            | Amaç                      | Switch Gerekir mi? |
| --- | -------------- | ------------------------- | ------------------ |
| 0   | balance-rr     | Round-robin yük dengeleme | Hayır              |
| 1   | active-backup  | Yedeklilik                | Hayır              |
| 2   | balance-xor    | Hash tabanlı dengeleme    | Gerekebilir        |
| 3   | broadcast      | Hepsine aynı anda gönder  | Hayır              |
| 4   | 802.3ad (LACP) | LAG standardı             | Evet               |
| 5   | balance-tlb    | Dinamik gönderim dengesi  | Hayır              |
| 6   | balance-alb    | Tam uyarlamalı dengeleme  | Hayır              |

---

## ⚙ Ubuntu 22.04 Üzerinde NIC Bonding Uygulaması (Netplan ile)

### Gerekli Paketler

```bash
sudo apt update
sudo apt install ifenslave -y
```

### Kernel Modülünü Yükle

```bash
sudo modprobe bonding
```

### Netplan Yapılandırması

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33: {}
    ens37: {}
    ens38: {}
  bonds:
    bond0:
      interfaces:
        - ens33
        - ens37
        - ens38
      addresses:
        - 192.168.1.200/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
      parameters:
        mode: balance-rr
        mii-monitor-interval: 100
```

### Ayarı Uygula

```bash
sudo netplan apply
```

---

## 📈 Performans Testi: `iperf3` ile

### Sunucu tarafında (192.168.1.165):

```bash
iperf3 -s
```

### Bonding tarafında (192.168.1.200):

```bash
iperf3 -c 192.168.1.165 -t 10 -P 4
```

> ✅ Sonuç: Toplamda 1.3 Gbps'ye kadar bant genişliği elde edilebilir.

---

## 📊 Trafiğin Gerçekte Dağılıp Dağılmadığını Görmek

### 1. Anlık Bayt Takibi:

```bash
watch -n 1 cat /sys/class/net/ens33/statistics/tx_bytes
watch -n 1 cat /sys/class/net/ens37/statistics/tx_bytes
watch -n 1 cat /sys/class/net/ens38/statistics/tx_bytes
```

### 2. Alternatif olarak:

```bash
ethtool -S ens33 | grep tx
```

### 3. Toplu Takip Scripti:

```bash
#!/bin/bash
while true; do
  clear
  for i in ens33 ens37 ens38; do
    echo -n "$i: "; cat /sys/class/net/$i/statistics/tx_bytes
  done
  sleep 1
done
```

---

## 🌎 Sanal Ortamda (VMware) Ne Değişir?

* `mode=802.3ad` VMware Workstation gibi ortamlarda çalışmaz (switch desteği yok)
* En uyumlu mod: `balance-rr`
* Her sanal NIC "connected" durumda olmalı

---

## ✨ Sonuç

NIC Bonding, hem yüksek bant genişliği sağlamak hem de sistemin ağ yedekliliğini güvence altına almak için son derece etkili bir yöntemdir.

> 🔹 "Ağ performansı artırmak istiyorsan: `balance-rr` veya `802.3ad` (LACP) + layer3+4 hash policy."
> 🔹 "Yedeklilik istiyorsan: `active-backup`"

Sistemin ihtiyacına göre en uygun mod seçilmeli ve doğru şekilde test edilmelidir.

---

Hazırlandı ❤️

GitHub @azizortanc

---
