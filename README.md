# Wazuh Security Monitoring Lab

Hyper-V üzerinde kurduğum bu laboratuvarda Windows güvenlik olaylarını Wazuh ile merkezi olarak topladım ve inceledim. Başarısız oturum açma ve gerçek zamanlı dosya bütünlüğü izleme senaryolarını kontrollü testlerle doğruladım.

## Laboratuvar Ortamı

| Sistem | Görev | Laboratuvar IP adresi |
|---|---|---|
| WAZUH01 | Ubuntu Server 24.04 LTS, Wazuh merkezi bileşenleri | 192.168.50.30 |
| DC01 | Windows Server 2022, Active Directory ve DNS | 192.168.50.10 |
| CLIENT01 | Windows 11 Pro, Wazuh agent | 192.168.50.20 |

- **Domain:** lab.test
- **Sanallaştırma:** Hyper-V
- **Wazuh sürümü:** 4.14.10
- **İzlenen uç nokta:** CLIENT01

DC01 domain hizmetleri için kullanıldı; bu çalışmada DC01 üzerine Wazuh agent kurulmadı.

## Ağ Yapısı

WAZUH01 üzerinde iki ağ bağdaştırıcısı kullanıldı:

- **eth0 / Default Switch:** İnternet erişimi ve ana bilgisayardan yönetim bağlantısı.
- **eth1 / LAB-AG:** Windows laboratuvarına bağlantı; sabit IP 192.168.50.30/24.

LAB-AG tarafına varsayılan ağ geçidi eklenmedi. DC01 ve CLIENT01 mevcut laboratuvar ağında bırakıldı.

Default Switch üzerinden alınan adres DHCP nedeniyle değişebildiği için agent'ın yönetici sunucu adresinde sabit laboratuvar IP'si kullanıldı.

## Kurulum ve Doğrulama

- Ubuntu Server kuruldu ve güncellendi.
- OpenSSH üzerinden yönetim bağlantısı sağlandı.
- Wazuh manager, indexer ve dashboard aynı sunucuya kuruldu.
- CLIENT01 üzerine Windows agent kurularak manager adresi 192.168.50.30 olarak ayarlandı.
- Panelde CLIENT01'in 4.14.10 sürümüyle **Active** durumda olduğu doğrulandı.
- CLIENT01'den WAZUH01'in laboratuvar IP'sine ping testinde paket kaybı görülmedi.

## Senaryo 1: Başarısız Oturum Açma Analizi

CLIENT01 üzerinde LAB\mert.test hesabıyla iki kontrollü yanlış parola denemesi yapıldı.

Threat Hunting → Events bölümünde şu sorgu kullanıldı:

```text
agent.name:"CLIENT01" AND data.win.system.eventID:"4625"
```

### Gözlemlenen Olaylar

Saatler panelde görünen Türkiye saatidir.

| Tarih ve saat | Agent | Olay kimliği | Wazuh kuralı |
|---|---|---|---|
| 08.11.2026 23:33:55 | CLIENT01 | 4625 | 60122 |
| 08.11.2026 23:34:17 | CLIENT01 | 4625 | 60122 |

İncelenen olay ayrıntılarında:

| Alan | Değer |
|---|---|
| Hedef hesap | LAB\mert.test |
| Bilgisayar | CLIENT01.lab.test |
| Oturum türü | 2 - Etkileşimli giriş |
| Durum | 0xC000006D |
| Alt durum | 0xC000006A - Yanlış parola |
| Kaynak IP alanı | 127.0.0.1 |

Olayların hesap, zaman ve hata bilgileri kontrollü denemelerle karşılaştırıldı.

Bu senaryoda hazır Wazuh kuralları kullanıldı. Özel bir brute-force korelasyon kuralı oluşturulmadı; iki hata tek başına saldırı olarak değerlendirilmedi.

## Senaryo 2: Dosya Bütünlüğü İzleme

CLIENT01 üzerinde C:\FIM-Test klasörü oluşturuldu. Agent'ın ossec.conf dosyasındaki mevcut syscheck bölümüne şu satır eklendi:

```xml
<directories check_all="yes" realtime="yes">C:\FIM-Test</directories>
```

Yapılandırma yedeklendi, değişiklik kaydedildi ve WazuhSvc yeniden başlatıldı. Agent günlüğünden klasörün izlemeye alındığı, ilk taramanın tamamlandığı ve gerçek zamanlı izlemenin başladığı doğrulandı.

İlk taramadan sonra aşağıdaki işlemler sırayla gerçekleştirildi. Her adımın panelde görülmesi beklendi.

### Dosya Oluşturma

```cmd
echo Wazuh dosya ekleme testi > C:\FIM-Test\test02.txt
```

### Dosya Değiştirme

```cmd
echo Ikinci satir eklendi >> C:\FIM-Test\test02.txt
```

### Test Dosyasını Silme

```cmd
del C:\FIM-Test\test02.txt
```

### Test Sonuçları

| Tarih ve saat | İşlem | Olay | Kural |
|---|---|---|---|
| 09.11.2026 00:05:49 | Dosya oluşturma | added | 554 |
| 09.11.2026 00:08:12 | İçeriğe satır ekleme | modified | 550 |
| 09.11.2026 00:10:33 | Dosya silme | deleted | 553 |

Üç olay da CLIENT01 üzerindeki C:\FIM-Test\test02.txt dosyasına aittir.

## Hash ve Dosya Boyutu İncelemesi

Modified olayında:

- İzleme modu realtime olarak görüldü.
- Dosya boyutu 27 bayttan 50 bayta çıktı.
- size, mtime, md5, sha1 ve sha256 alanlarının değiştiği kaydedildi.
- Önceki ve sonraki SHA-256 değerleri karşılaştırıldı.

| Değer | SHA-256 |
|---|---|
| Önce | 5c17603dda91e6a907d87979f656bca1a78f5b788d42504695e34bc739aeebefa |
| Sonra | 34903f8dc69d3623dd86b5cd4803b5c21b4533deec8dec96f610adca49d09941 |

Hash değerlerinin farklı olması, dosya içeriğinin değiştiğini doğruladı. Bu değişiklik laboratuvarda bilerek yapıldı.

Kuralın MITRE ATT&CK eşleştirmesi, tek başına kötü niyetli faaliyet kanıtı olarak yorumlanmadı.

## Karşılaştığım Sorunlar ve Çözümleri

### Yönetim IP'sinin değişmesi

Default Switch DHCP adresi yeniden başlatma ve ağ yapılandırması sırasında değişti. Güncel IP konsoldan kontrol edilerek SSH bağlantısı yeniden kuruldu. Agent iletişimi için LAB-AG tarafında sabit IP tanımlandı.

### Yetersiz RAM nedeniyle CLIENT01'in başlamaması

CLIENT01'in başlangıç belleği 6 GB'dan 4 GB'a düşürüldü. WAZUH01 için ayrılan 8 GB korundu.

### Windows'ta yönetici yetkisi gereksinimi

Servis başlatma ve agent yapılandırmasını düzenleme işlemleri yükseltilmiş yönetici oturumunda gerçekleştirildi.

### XML yapılandırma hatası

Agent günlüğünde ossec.conf için XML okuma hatası görüldü. Hatalı kapanış etiketi ve frequency satırı düzeltildi. Yeniden başlatma sonrasında güncel loglardan izleme durumunun çalıştığı doğrulandı.

### Panel sorgusunda sözdizimi hatası

Windows dosya yolunun doğrudan arama alanına yazılması sorgu hatasına yol açtı. Agent filtresi kullanılarak FIM olayları listelendi ve dosya yolu sonuçlardan incelendi.

## Kazandığım Beceriler

- SIEM kurulumu ve temel yönetimi
- Windows agent kurulumu ve bağlantı doğrulama
- Merkezi log toplama ve olay filtreleme
- Başarısız giriş olaylarını inceleme
- Gerçek zamanlı dosya bütünlüğü izleme
- Hash ve dosya boyutu değişikliklerini yorumlama
- Linux ağ yapılandırması ve SSH
- Log üzerinden yapılandırma hatası giderme
- Test sonuçlarını kanıtlarla belgeleme

## Kapsam

Bu çalışma kontrollü bir eğitim laboratuvarıdır. Bir Windows uç noktası izlenmiştir. Özel tespit kuralları, otomatik müdahale ve üretim ortamı güvenlik sertleştirmesi bu sürümün kapsamında değildir.

Parolalar, özel anahtarlar, agent kayıt anahtarları ve kurulumun ürettiği kimlik bilgisi arşivleri depoya dahil edilmez.

## İlgili Projeler

- [Active Directory Güvenlik Laboratuvarı](https://github.com/Raskeria/active-directory-security-lab)
- [Windows Log Analyzer](https://github.com/Raskeria/windows-log-analyzer)
