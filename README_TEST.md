# Red Hat 9 HA Cluster — Test Playbook

**`ha_cluster_test.yml`** · RHEL 9 · Pacemaker 2.1.x · Corosync 3.x · PostgreSQL 18

> Bu döküman `ha_cluster_test.yml` playbook'unu daha önce hiç kullanmamış birinin testleri başarıyla çalıştırabilmesi için hazırlanmıştır. Her testin ne yaptığı, nasıl çalıştırıldığı, hangi koşulları doğruladığı ve sonuçların nasıl yorumlandığı eksiksiz olarak açıklanmıştır.
>
> Test playbook'u **yalnızca** `ha_cluster_setup.yml` ile başarıyla tamamlanmış bir cluster kurulumundan sonra çalıştırılmalıdır.

---

## İçindekiler

1. [Genel Bakış](#1-genel-bakış)
2. [Ön Koşullar](#2-ön-koşullar)
3. [Dizin Yapısı](#3-dizin-yapısı)
4. [Kullanım](#4-kullanım)
5. [PREP — Test Öncesi Hazırlık](#5-prep--test-öncesi-hazırlık)
6. [Test Detayları](#6-test-detayları)
7. [RAPOR — Son Doğrulama ve Ortam Restore](#7-rapor--son-doğrulama-ve-ortam-restore)
8. [Markdown Rapor Dosyası](#8-markdown-rapor-dosyası)
9. [Güvenlik ve Temizlik Mekanizması](#9-güvenlik-ve-temizlik-mekanizması)
10. [Test Hatalarını Giderme](#10-test-hatalarını-giderme)

---

## 1. Genel Bakış

Test playbook'u cluster'ın tüm başarısızlık senaryolarında doğru davranıp davranmadığını otomatik olarak doğrular. Testler iki gruba ayrılır:

### Güvenli Testler (her zaman çalışır)

| Test | Açıklama |
|---|---|
| T01 | Temel cluster sağlık kontrolü |
| T02 | VIP failover (node bakımı simülasyonu) |
| T06 | PostgreSQL servis çöküşü (kill -9) |
| T07 | GFS2 eşzamanlı yazma (her iki node) |
| T08 | LVM lock doğrulaması |
| T11 | Yük altında failover |
| T12 | Çoklu ardışık failover (dayanıklılık) |

### Yıkıcı Testler (`-e run_destructive=true` gerekli)

| Test | Açıklama | Risk |
|---|---|---|
| T03 | Aktif node ani kapatma | Node durdurulup yeniden başlatılır |
| T04 | Corosync ring ağ kesintisi | iptables kuralları uygulanır |
| T05 | Split-brain önleme (qdevice engelleme) | iptables kuralları uygulanır |
| T10 | iSCSI bağlantı kesintisi | iptables kuralları uygulanır |
| T13 | Tam cluster yeniden başlatma | Tüm node'lar durdurulup başlatılır |

### STONITH Testi (`-e run_stonith_test=true` gerekli)

| Test | Açıklama | Risk |
|---|---|---|
| T09 | STONITH fence agent testi | **Node IPMI üzerinden REBOOT edilir!** |

---

## 2. Ön Koşullar

- `ha_cluster_setup.yml` ile cluster kurulumu başarıyla tamamlanmış olmalı
- Her iki node online: `pcs status` çıktısında OFFLINE veya FAILED resource yok
- PostgreSQL VIP üzerinden erişilebilir: `psql -h 192.168.0.63 -U postgres -c "SELECT 1;"`
- `pgsql_password` değişkeni `inventory/hosts.yml`'de doğru tanımlı
- `tasks/failover_cycle.yml` dosyası proje dizininde mevcut (T12 için)

### Test Öncesi Cluster Sağlık Kontrolü

```bash
# Cluster temiz mi?
pcs status
# Beklenen: tüm resource'lar Started, FAILED veya OFFLINE yok

# Stale failure counter'ları temizle
pcs resource cleanup

# PostgreSQL erişilebilir mi?
psql -h 192.168.0.63 -U postgres -c "SELECT version();"

# pgsql_password doğru mu?
PGPASSWORD="hosts.yml_deki_parola" psql -h 192.168.0.63 -U postgres -c "SELECT 1;"
```

---

## 3. Dizin Yapısı

```
proje-dizini/
├── ha_cluster_test.yml           # Ana test playbook'u
└── tasks/
    └── failover_cycle.yml        # T12 failover döngüsü task dosyası
```

`tasks/failover_cycle.yml` dosyası T12 tarafından `include_tasks` ile çağrılır. `ha_cluster_test.yml` ile aynı dizindeki `tasks/` alt klasöründe bulunmalıdır.

---

## 4. Kullanım

### Tüm Testleri Çalıştır (Güvenli Mod)

```bash
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml
```

Yıkıcı testler (T03, T04, T05, T10, T13) ve STONITH testi (T09) bu modda **atlanır**.

### Yıkıcı Testler Dahil

```bash
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml \
  -e run_destructive=true
```

### STONITH Testi Dahil (Node Reboot Olur!)

```bash
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml \
  -e run_destructive=true \
  -e run_stonith_test=true
```

### Tek Test Çalıştır

```bash
# Sadece T01
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml -t test_T01

# Sadece T06
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml -t test_T06

# Sadece T09 (STONITH — node reboot!)
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml \
  -t test_T09 -e run_stonith_test=true
```

### Belirli Test Seti Çalıştır

```bash
# T01 ve T02
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml \
  -t "test_T01,test_T02"

# Sadece güvenli testler
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml \
  -t "test_T01,test_T02,test_T06,test_T07,test_T08,test_T11,test_T12"

# Yıkıcı testler hariç tümü (T09 dahil değil)
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml \
  -t "test_T03,test_T04,test_T05,test_T10,test_T13" \
  -e run_destructive=true
```

### Değişken Referansı

| Değişken | Varsayılan | Açıklama |
|---|---|---|
| `run_destructive` | `false` | Yıkıcı testleri etkinleştir (T03, T04, T05, T10, T13) |
| `run_stonith_test` | `false` | STONITH testini etkinleştir (T09) — node reboot olur! |
| `failover_wait` | `60` | Failover tamamlanması için bekleme süresi (saniye) |
| `settle_wait` | `30` | Resource stabilizasyonu için bekleme süresi (saniye) |
| `pgsql_password` | (hosts.yml'den) | PostgreSQL postgres kullanıcısı parolası — ayrı tanım gerekmez |

> **Not:** `pgsql_password` değişkeni `inventory/hosts.yml`'deki değerden otomatik alınır. Tüm play'lerde inventory group_vars üzerinden erişilir; ayrıca tanımlanmasına gerek yoktur.

---

## 5. PREP — Test Öncesi Hazırlık

Her test çalışmadan önce PREP play'i şu temizlik ve doğrulama adımlarını otomatik olarak gerçekleştirir:

### Önceki Test Artıklarını Temizleme

1. **iptables temizliği:** Tüm cluster node'larından önceki test çalışmalarına ait iptables kuralları kaldırılır (corosync 5405, qdevice 5403, iSCSI 3260 port kuralları)
2. **PostgreSQL tablo temizliği:** Kalan test tabloları drop edilir: `ha_test_t02`, `ha_test_t03`, `ha_test_t06`, `ha_load_t11`, `ha_test_t12`
3. **Location constraint temizliği:** VIP üzerindeki tüm geçici kısıtlamalar kaldırılır (`pcs resource clear vip`)
4. **Resource cleanup:** `pcs resource cleanup` ile stale failure counter'lar sıfırlanır
5. **Stabilizasyon bekleme:** 15 saniye beklenir

### Cluster Sağlık Doğrulaması

6. **`pcs status` kontrolü:** OFFLINE veya FAILED resource varsa test durur
7. **PostgreSQL bağlantı testi:** `pgsql_password` ile VIP üzerinden bağlantı test edilir — parola yanlışsa T01'de değil, burada net hata verir
8. **Aktif/pasif node belirleme:** VIP'in hangi node'da olduğu tespit edilir ve ekrana yazdırılır

### Test Ortamı Özeti

PREP sonunda şu bilgiler ekrana basılır:

```
╔══════════════════════════════════════════════════════════════╗
║          TEST ORTAMI HAZIR                                   ║
╠══════════════════════════════════════════════════════════════╣
║  Cluster     : ha_webpgsql
║  Aktif Node  : clstr01.lab.akyuz.tech
║  Pasif Node  : clstr02.lab.akyuz.tech
║  VIP         : 192.168.0.63
║  PostgreSQL  : 192.168.0.63:5432
║  GFS2 Mount  : /shared/webfs
║  pgsql Mount : /var/lib/pgsql
║  Yıkıcı test : DEVRE DIŞI (güvenli mod)
║  STONITH test: DEVRE DIŞI
╚══════════════════════════════════════════════════════════════╝
```

---

## 6. Test Detayları

### T01 — Temel Cluster Sağlık Kontrolü

**Tag:** `test_T01` | **Tür:** Güvenli | **Her zaman çalışır:** Evet

Bu test cluster'ın bütüncül sağlığını doğrular. Diğer testler çalıştırılmadan önce bu testin geçmesi beklenir.

**Yapılan kontroller:**

| Kontrol | Komut | Beklenen Sonuç |
|---|---|---|
| Corosync ring | `corosync-cfgtool -s` | Tüm peer'ler connected, disconnected/lost yok |
| Quorum device | `pcs quorum device status` | State: connected |
| Cluster quorate | `pcs quorum status` | Quorate: Yes |
| Tüm resource'lar | `pcs resource status` | FAILED veya Stopped yok |
| GFS2 her iki node'da | `pcs resource status gfs2_fs-clone` | Her iki node adı çıktıda var |
| GFS2 mount (her node) | `mountpoint -q /shared/webfs` | rc=0 her iki node'da |
| PostgreSQL mount | `mountpoint -q /var/lib/pgsql` | rc=0 aktif node'da |
| PostgreSQL bağlantısı | `psql -h VIP -c "SELECT version()"` | Başarılı |
| STONITH etkin | `pcs property config stonith-enabled` | stonith-enabled=true |
| IPMI erişilebilirliği | `ipmitool chassis power status` | Her node için rc=0 |
| Yapılandırma doğrulaması | `crm_verify -L -V` | rc=0 |
| Servis durumu | `systemctl is-active` | corosync, pacemaker, corosync-qdevice, multipathd, iscsid hepsi active |

**Hata durumunda:** Temel cluster sorunu çözülmeden diğer testlere geçilmemelidir.

---

### T02 — VIP Failover (Node Bakımı Simülasyonu)

**Tag:** `test_T02` | **Tür:** Güvenli | **Her zaman çalışır:** Evet

**Ne yapar:**

1. Hangi node'un VIP'i tuttuğunu kaydeder
2. PostgreSQL'e test kaydı yazar (`ha_test_t02` tablosu)
3. VIP'i diğer node'a taşır: `pcs resource move vip <hedef>`
4. VIP'in hedef node'a geçmesini bekler (en fazla 60s, 5s aralıklarla)
5. PostgreSQL'in yeni node'da başlamasını bekler
6. Failover sonrası ikinci kayıt ekler
7. Failover öncesi kaydın korunduğunu doğrular (`count >= 1`)
8. Location constraint kaldırır (`pcs resource clear vip`)
9. Test tablosunu drop eder

**Güvenlik:** `block/rescue/always` yapısı ile sarılmıştır. Test başarısız olsa bile location constraint kaldırılır ve test tablosu temizlenir.

**Kanıtladığı şeyler:**
- VIP failover yapılandırılmış timeout içinde tamamlanır
- PostgreSQL, failover boyunca VIP üzerinden erişilebilir kalır
- Failover öncesi yazılan veri, failover sonrasında korunur

---

### T03 — Aktif Node Ani Kapatma

**Tag:** `test_T03` | **Tür:** Yıkıcı | **Gereksinim:** `-e run_destructive=true`

**Ne yapar:**

1. Aktif node (VIP sahibi) ve pasif node'u belirler
2. PostgreSQL'e test kaydı yazar (`ha_test_t03` tablosu)
3. Aktif node'u durdurur: `pcs cluster stop <aktif_node> --force` (asenkron, 10s timeout)
4. Pasif node üzerinde PostgreSQL'in başlamasını bekler (en fazla 120s)
5. VIP'in pasif node'a geçtiğini doğrular
6. Pasif node üzerinden VIP'e bağlanarak veri tutarlılığını test eder
7. Failover öncesi kaydın korunduğunu doğrular (`count >= 1`)

**always bloğu (her durumda çalışır):**
- Durdurulan node yeniden başlatılır: `pcs cluster start <aktif_node>`
- Node'un cluster'a geri gelmesi beklenir (24 × 5s)
- `pcs resource cleanup`
- Test tablosu drop edilir

**Kanıtladığı şeyler:**
- Cluster node arızasını doğru tespit eder ve otomatik failover yapar
- Pasif node PostgreSQL'i devralır
- Ani node kapatma sonrası veri korunur
- Durdurulmuş node cluster'a başarıyla geri döner

---

### T04 — Corosync Ring Ağ Kesintisi

**Tag:** `test_T04` | **Tür:** Yıkıcı | **Gereksinim:** `-e run_destructive=true`

**Ne yapar:**

1. Aktif ve pasif node'u belirler
2. Pasif node üzerinde corosync trafiğini keser: iptables ile UDP/TCP 5405 portunu 45s boyunca engeller (asenkron)
3. Kesinti sırasında aktif node'dan cluster durumunu izler
4. Ağın geri gelmesini bekler (45 + 10s)
5. Cluster stabilizasyonunu bekler (en fazla 120s)
6. `pcs resource cleanup` çalıştırır
7. VIP üzerinden PostgreSQL erişilebilirliğini test eder
8. Corosync ring'in tamamen kurtardığını doğrular (`corosync-cfgtool -s`)

**Neden 45 saniye?** Bu süre STONITH delay değerinden (clstr02 için 30s) büyük olmalıdır. Quorum/fencing mantığını tetiklemek için yeterli süre sağlar.

**always bloğu:** iptables kuralları her durumda pasif node'dan kaldırılır.

**Kanıtladığı şeyler:**
- Cluster geçici corosync ring kesintisini yönetir
- Ağ kurtarma sonrası corosync ring yeniden bağlanır
- VIP ve PostgreSQL kesinti boyunca erişilebilir kalır

---

### T05 — Split-Brain Önleme (Quorum Device Engelleme)

**Tag:** `test_T05` | **Tür:** Yıkıcı | **Gereksinim:** `-e run_destructive=true`

**Ne yapar:**

1. Her iki cluster node'unda qdevice portunu (TCP 5403) iptables ile 30s boyunca engeller (asenkron)
2. Kesinti sırasında quorum durumunu izler: `corosync-quorumtool -s`
3. Beklenen davranış analizi ekrana basılır: `no-quorum-policy=freeze` ile resource'lar dondurulmalı
4. Kurtarma beklenir (30 + 15s)
5. Qdevice'ın yeniden bağlandığını doğrular (`pcs quorum device status`)

**always bloğu:** iptables kuralları her iki node'dan kaldırılır.

**Kanıtladığı şeyler:**
- Qdevice bağlantısı kesildiğinde `no-quorum-policy=freeze` devreye girer (resource'lar durdurulmaz, dondurulur)
- Qdevice bağlantısı yeniden kurulduktan sonra cluster fully quorate duruma döner
- ffsplit algoritması qdevice kaybını split-brain oluşturmadan yönetir

---

### T06 — PostgreSQL Servis Çöküşü (kill -9)

**Tag:** `test_T06` | **Tür:** Güvenli | **Her zaman çalışır:** Evet

**Ne yapar:**

1. PostgreSQL'in hangi node'da çalıştığını tespit eder
2. PostgreSQL'e test kaydı yazar (`ha_test_t06` tablosu)
3. Postmaster PID'ini tespit eder: `pgrep -o -f 'postgres.*postmaster'`
4. Postmaster'a `kill -9` gönderir
5. 5 saniye bekler (Pacemaker'ın arızayı fark etmesi için)
6. Pacemaker'ın PostgreSQL'i yeniden başlatmasını bekler (en fazla 60s, 5s aralıklarla)
7. Failcount kontrol eder — migration threshold (3) aşılmamış olmalı
8. Yeniden başlatma sonrası test kaydı ekler
9. Kill öncesi kaydın korunduğunu doğrular (`count >= 1`)

**always bloğu:** `pcs resource cleanup postgresql`, test tablosu drop edilir.

**Kanıtladığı şeyler:**
- Pacemaker'ın `on-fail=restart` monitor aksiyonu çöken PostgreSQL'i doğru yeniden başlatır
- Tek çöküşte migration threshold (3) aşılmaz
- Çöküş öncesi yazılan veri yeniden başlatma sonrasında korunur

---

### T07 — GFS2 Eşzamanlı Yazma (Her İki Node)

**Tag:** `test_T07` | **Tür:** Güvenli | **Her zaman çalışır:** Evet | **Hedef:** `cluster_nodes` (her ikisi)

**Ne yapar:**

1. Her iki node'da GFS2 mount durumunu doğrular
2. `/shared/webfs` disk alanını kontrol eder
3. Her iki node'dan eşzamanlı olarak `/shared/webfs`'e 100 satır yazar (asenkron, 60s timeout):
   - Dosya adı: `ha_test_t07_<hostname>.txt`
4. Her iki yazma işleminin tamamlanmasını bekler
5. Her iki node'dan diğer node'un yazdığı dosyanın görünebildiğini doğrular (çapraz okuma)
6. Her dosyada 100+ satır olduğunu doğrular
7. GFS2'nin `gfs2` tipiyle read-write mount edildiğini doğrular

**always bloğu:** GFS2'deki tüm test dosyaları kaldırılır (`rm -f ha_test_t07_*.txt`), run_once=true.

**Kanıtladığı şeyler:**
- GFS2 her iki node'dan eşzamanlı yazmaları bozulmadan yönetir
- Bir node'un yazdığı dosya diğer node'dan hemen görünür
- DLM üzerinden GFS2 dağıtık kilitleme doğru çalışır

---

### T08 — LVM Lock Doğrulaması

**Tag:** `test_T08` | **Tür:** Güvenli | **Her zaman çalışır:** Evet

**Ne yapar:**

1. `lvmlockd` process'inin çalışıp çalışmadığını doğrular (`pgrep -x lvmlockd`), PID'i ekrana basar
2. Başlangıç lockspace durumunu alır: `lvmlockctl --dump`
3. `shared_vg` lockspace'in aktif olduğunu doğrular
4. `db_vg` lockspace'in aktif olduğunu doğrular
5. `pcs resource failcount show lvmlockd` ile failcount raporunu gösterir
6. `lvm.conf`'taki `use_lvmlockd` değerini okur ve gösterir
7. Her iki VG için `vgchange --lock-start` idempotency testi yapar (tekrar çalıştırma hata vermemeli)
8. Her iki VG'ye `vgs` ile erişim testi yapar
9. `lvs shared_vg` ve `lvs db_vg` ile LV aktivasyon durumunu raporlar

**Kanıtladığı şeyler:**
- `lvmlockd` çalışıyor ve dağıtık lock'ları doğru yönetiyor
- Her iki paylaşımlı VG aktif lockspace'e sahip
- lock-start idempotent (birden fazla çalıştırılabilir, hata vermez)
- `lvm.conf`'ta `use_lvmlockd = 1` aktif

---

### T09 — STONITH Fence Agent Testi

**Tag:** `test_T09` | **Tür:** STONITH | **Gereksinim:** `-e run_stonith_test=true`

> **UYARI:** Bu test pasif cluster node'unu IPMI üzerinden fiziksel olarak REBOOT EDER. Ortamınızda buna izin verildiğinden emin olun.

**Ne yapar:**

1. Pasif node'u (VIP'i tutmayan) belirler
2. Pasif node'un IPMI'sini test eder: `ipmitool chassis power status`
3. Pasif node'u fence eder: `pcs stonith fence <pasif_node> --off`
4. Node'un OFFLINE durumuna geçmesini bekler (12 × 5s)
5. Node'u IPMI ile geri açar: `ipmitool chassis power on`
6. SSH'ın erişilebilir olmasını bekler (30s delay + en fazla 180s timeout)
7. Cluster servisini başlatır: `pcs cluster start`
8. Node'un cluster'a döndüğünü doğrular (24 × 10s)

**rescue bloğu:** Hata durumunda node'u IPMI ile açmayı dener.

**always bloğu:** `pcs resource cleanup` çalışır.

**Kanıtladığı şeyler:**
- STONITH agent (`fence_ipmilan`) node'u başarıyla fence edebilir
- IPMI kimlik bilgileri ve bağlantısı doğrudur
- Fenced node kurtarılıp cluster'a yeniden katılabilir

---

### T10 — iSCSI Bağlantı Kesintisi

**Tag:** `test_T10` | **Tür:** Yıkıcı | **Gereksinim:** `-e run_destructive=true`

**Ne yapar:**

1. Aktif iSCSI session'larını ve multipath durumunu kaydeder (kesinti öncesi)
2. Her iki cluster node'undan iSCSI hedefini 30s boyunca iptables ile engeller (asenkron):
   - `iptables -I OUTPUT -p tcp -d <iscsi_target_ip> --dport 3260 -j DROP`
   - `iptables -I INPUT  -p tcp -s <iscsi_target_ip> --dport 3260 -j DROP`
3. Kesinti sırasında cluster davranışını izler
4. İSCSI kurtarmasını bekler (30 + 15s)
5. `pcs resource cleanup` çalıştırır
6. Cluster stabilizasyonunu bekler (Failed Resource Actions yok olana kadar, 24 × 5s)
7. Her iki node'da GFS2 mount durumunu doğrular
8. Multipath kurtarma durumunu raporlar

**always bloğu:** iptables kuralları her iki node'dan kaldırılır.

**Kanıtladığı şeyler:**
- Cluster geçici iSCSI yolu kesintisini yönetir
- Multipath bağlantı yeniden kurulunca path recovery gerçekleştirir
- GFS2 mount'ları iSCSI kurtarma sonrası düzelir

---

### T11 — Yük Altında Failover

**Tag:** `test_T11` | **Tür:** Güvenli | **Her zaman çalışır:** Evet

**Ne yapar:**

1. `ha_load_t11` tablosunu oluşturur
2. Arka planda yük üretir: 1000 INSERT işlemi, 30ms aralıklarla (asenkron, 90s timeout):
   - Her INSERT: `repeat('x', 1000)` — 1KB'lık veri
3. 10 saniye bekler (yükün birikmesi için)
4. Yük devam ederken VIP failover tetikler:
   - `pcs resource move vip <hedef>` ile hedef node'u dinamik belirler
5. Failover tamamlanmasını bekler: PostgreSQL hedef node'da `Started` olana kadar (18 × 5s)
6. Yük bitince VIP üzerinden `SELECT count(*) FROM ha_load_t11;` sorgular
7. Kaydedilen kayıt sayısını raporlar

**always bloğu:** Location constraint kaldırılır, test tablosu drop edilir.

**Kanıtladığı şeyler:**
- Cluster PostgreSQL yük altındayken VIP failover gerçekleştirebilir
- Pacemaker yük koşullarında resource geçişini doğru yönetir
- Failover öncesi commit edilen kayıtlar korunur

---

### T12 — Çoklu Ardışık Failover (Dayanıklılık)

**Tag:** `test_T12` | **Tür:** Güvenli | **Her zaman çalışır:** Evet

**Ne yapar:**

`ha_test_t12` tablosunu oluşturur, ardından `tasks/failover_cycle.yml` dosyasını **3 kez** döngüyle çalıştırır (`failover_cycles: 3`).

**Her döngüde (`failover_cycle.yml`):**

1. VIP'in hangi node'da olduğunu belirler
2. `ha_test_t12` tablosuna kayıt ekler: `(cycle, node)` değerleriyle
3. VIP'i diğer node'a taşır: `pcs resource move vip <hedef>`
4. Failover tamamlanmasını bekler (en fazla 60s, 5s aralıklarla)
5. VIP'in hedef node'a geçtiğini doğrular
6. PostgreSQL'in yeni node'da başlamasını bekler (12 × 5s)
7. VIP üzerinden bağlanarak o döngünün kaydının var olduğunu doğrular (`count = 1`)
8. Location constraint kaldırır (`pcs resource clear vip`)
9. Stabilizasyon bekler (min 30s)

Tüm döngüler tamamlandıktan sonra toplam kayıt sayısı döngü sayısına eşit mi doğrular.

**always bloğu:** Location constraint kaldırılır, test tablosu drop edilir.

**Kanıtladığı şeyler:**
- Cluster birden fazla ardışık failover sonrasında bozulmadan çalışmaya devam eder
- Her failover başarıyla tamamlanır
- Her döngüde yazılan veri tüm sonraki failover'lardan sonra da korunur

---

### T13 — Tam Cluster Yeniden Başlatma

**Tag:** `test_T13` | **Tür:** Yıkıcı | **Gereksinim:** `-e run_destructive=true`

**Ne yapar:**

1. Tüm cluster'ı durdurur: `pcs cluster stop --all --wait=120`
2. Her iki node'da corosync'in durduğunu doğrular
3. 10 saniye bekler
4. Tüm cluster'ı başlatır: `pcs cluster start --all --wait=120`
5. Her iki node'un online olmasını ve tüm resource'ların başlamasını bekler (18 × 10s)
6. `pcs resource cleanup` çalıştırır
7. GFS2'nin her iki node'da başladığını doğrular (12 × 10s)
8. PostgreSQL'in başladığını doğrular (12 × 10s)
9. VIP üzerinden PostgreSQL bağlantısı test eder (6 × 5s)

**rescue bloğu:** Hata durumunda `pcs cluster start --all` ile cluster başlatılmaya çalışılır.

**always bloğu:** `pcs resource cleanup` çalışır.

**Kanıtladığı şeyler:**
- Cluster kontrollü tam durdurma ve yeniden başlatma işlemini gerçekleştirebilir
- Tam restart sonrası tüm resource'lar doğru sırayla başlar
- VIP üzerinden PostgreSQL tam restart sonrası erişilebilir durumdadır

---

## 7. RAPOR — Son Doğrulama ve Ortam Restore

Tüm testler tamamlandıktan sonra RAPOR play'i `tags: always` ile her zaman çalışır.

### Kapsamlı Temizlik

1. **iptables temizliği:** Tüm test kaynaklı kurallar tüm node'lardan kaldırılır:
   - corosync (UDP/TCP 5405)
   - qdevice (TCP 5403)
   - iSCSI (TCP 3260)
2. **PostgreSQL tablo temizliği:** Tüm test tabloları drop edilir
3. **GFS2 dosya temizliği:** `/shared/webfs/ha_test_t07_*.txt` silinir
4. **Location constraint temizliği:** `pcs resource clear vip`
5. **Son resource cleanup:** `pcs resource cleanup`
6. **Stabilizasyon bekleme:** 20 saniye

### Son Durum Kontrolleri

7. `pcs status` — son cluster durumu
8. `pcs constraint` — son constraint durumu
9. `pcs quorum status` — son quorum durumu
10. Her iki node'da GFS2 mount durumu: `df -h /shared/webfs`
11. Her iki node'da disk kullanımı: `df -h /shared/webfs /var/lib/pgsql`
12. Son PostgreSQL bağlantı testi: `SELECT 'cluster OK' AS durum, now() AS zaman, version() AS versiyon;`

### Konsol Özet Raporu

```
╔══════════════════════════════════════════════════════════════════════╗
║                  HA CLUSTER TEST RAPORU                              ║
╠══════════════════════════════════════════════════════════════════════╣
║  Tarih    : 2026-03-19 15:30:00
║  Cluster  : ha_webpgsql
║  VIP      : 192.168.0.63:5432
╠══════════════════════════════════════════════════════════════════════╣
║  TEST SONUÇLARI:
║
║  T01 Temel Sağlık Kontrolü          ✅ (her zaman çalışır)
║  T02 VIP Failover                   ✅ (her zaman çalışır)
║  T03 Aktif Node Kapatma             ✅ (çalıştı) / ⏭ (atlandı)
║  ...
╠══════════════════════════════════════════════════════════════════════╣
║  SON CLUSTER DURUMU:
║  [pcs status çıktısı]
╚══════════════════════════════════════════════════════════════════════╝
```

---

## 8. Markdown Rapor Dosyası

Test tamamlandıktan sonra aşağıdaki konuma detaylı bir Markdown rapor dosyası yazılır:

```
/tmp/ha_cluster_test/ha_cluster_test_report_<ZAMAN_DAMGASI>.md
```

### Rapor İçeriği

| Bölüm | İçerik |
|---|---|
| **Genel Bilgiler** | Tarih, cluster adı, node'lar, VIP, mount noktaları, test modu |
| **Sistem Bilgileri** | Kernel versiyonu, OS, pcs/corosync/psql versiyonları |
| **Test Sonuçları Özeti** | Her test için ✅/⏭ durumu tablo formatında |
| **pcs status** | Son cluster durumu tam çıktı |
| **Quorum Durumu** | `corosync-quorumtool -s` tam çıktı |
| **Corosync Ring** | `corosync-cfgtool -s` ring bağlantı durumu |
| **Constraint'ler** | `pcs constraint show --full` tam çıktı |
| **iSCSI Session'ları** | `iscsiadm -m session` aktif session listesi |
| **Multipath Cihazlar** | `multipath -ll` tam çıktı |
| **LVM Lock Durumu** | `lvmlockctl --dump` tam çıktı |
| **Disk / Mount Kullanımı** | Her iki node'da `df -h` çıktısı |
| **PostgreSQL** | VIP bağlantı durumu ve versiyon |
| **Resource Failcount** | `pcs resource failcount show` tam çıktı |
| **Test Detayları** | Her testin ne yaptığının açıklaması |

### Raporu İndirme

```bash
# Cluster node'undan yerel makineye kopyala
scp root@192.168.0.61:/tmp/ha_cluster_test/ha_cluster_test_report_*.md ./ha_cluster_raporu.md

# Veya tüm rapor dizinini kopyala
scp -r root@192.168.0.61:/tmp/ha_cluster_test/ ./cluster_test_raporlari/
```

---

## 9. Güvenlik ve Temizlik Mekanizması

### block / rescue / always Yapısı

T02, T03, T04, T05, T06, T07, T09, T10, T11, T12, T13 testlerinin tamamı `block/rescue/always` ile sarılmıştır:

```yaml
block:
  # Test adımları
rescue:
  # Hata durumunda güvenli çıkış:
  #   - iptables kurallarını kaldır
  #   - Durdurulmuş node'u yeniden başlat
  #   - fail: ile hata mesajı ver
always:
  # Her durumda (başarılı veya başarısız) çalışır:
  #   - iptables kurallarını kaldır
  #   - Durdurulmuş node'u yeniden başlat
  #   - pcs resource cleanup
  #   - Test tablolarını drop et
  #   - pcs resource clear vip
```

Bu yapı sayesinde test yarıda kesilse bile iptables kuralları temizlenir, durdurulmuş node'lar yeniden başlatılır ve test tabloları silinir. **Cluster her koşulda kullanılabilir durumda kalır.**

### Otomatik Ön Temizlik

PREP play'i her test çalışması öncesinde önceki çalışmalardan kalan artıkları temizler:

- Tüm test iptables kuralları tüm node'lardan silinir
- Tüm test tabloları drop edilir
- VIP location constraint'leri temizlenir
- `pcs resource cleanup` çalıştırılır
- 15 saniye stabilizasyon beklenir

**Test playbook'u art arda birden fazla kez güvenle çalıştırılabilir.**

### Temizlenen Kaynaklar Özeti

| Kaynak | Nerede Temizlenir |
|---|---|
| iptables kuralları (5405, 5403, 3260) | PREP + Her test `always` bloğu + RAPOR |
| VIP location constraint'leri | Her test `always` bloğu + RAPOR |
| PostgreSQL test tabloları | Her test `always` bloğu + RAPOR |
| GFS2 test dosyaları (`ha_test_t07_*.txt`) | T07 `always` bloğu + RAPOR |
| Resource failure counter'ları | Her test + RAPOR son cleanup |
| Durdurulmuş node'lar | T03 ve T09 `always` blokları |

---

## 10. Test Hatalarını Giderme

### T01 Hataları

#### `Corosync ring hatası`
```bash
corosync-cfgtool -s
# disconnected veya lost girişlerine bakın

journalctl -u corosync -n 50

# Cluster ağı erişilebilir mi?
ping -I enp2s0 172.16.16.62
```

#### `Quorum device bağlı değil`
```bash
pcs quorum device status
systemctl status corosync-qdevice

# Qdevice yeniden ekle
pcs quorum device remove
pcs quorum device add model net \
  host=clstr-qrm01.lab.akyuz.tech \
  algorithm=ffsplit port=5403
systemctl enable --now corosync-qdevice
```

#### `Başarısız veya durmuş resource var`
```bash
pcs status
pcs resource cleanup
journalctl -u pacemaker -n 50
```

#### `PostgreSQL bağlantı hatası (PREP aşamasında)`
```bash
# pgsql_password doğru mu?
PGPASSWORD="hosts.yml_parola" psql -h 192.168.0.63 -U postgres -c "SELECT 1;"

# PostgreSQL çalışıyor mu?
pcs resource status postgresql

# Bağlantı testi
psql -h 192.168.0.63 -p 5432 -U postgres
# "Password for user postgres:" geliyorsa parola yanlış
```

---

### T02 / T11 / T12 Hataları

#### `VIP timeout içinde taşınamadı`
```bash
# Engelleyici constraint var mı?
pcs constraint

# Resource durumu
pcs resource status vip

# Constraint kaldır ve manuel dene
pcs resource clear vip
pcs resource move vip clstr02.lab.akyuz.tech
```

#### `Failover sonrası PostgreSQL erişilemiyor`
```bash
# PostgreSQL ve VIP aynı node'da mı?
pcs resource status postgresql
pcs resource status vip

# Colocation constraint eksikse ekle
pcs constraint colocation add postgresql with pgsql_fs INFINITY
pcs resource cleanup
```

---

### T03 Hataları

#### `Failover tamamlanamadı`
```bash
# Pasif node'dan kontrol et
pcs status

# Cleanup
pcs resource cleanup

# Gerekirse node'u zorla başlat
pcs cluster start <durdurulmuş_node>
```

#### `Veri tutarlılığı doğrulaması başarısız`
```bash
# Direkt sorgu
psql -h 192.168.0.63 -U postgres \
  -c "SELECT * FROM ha_test_t03;"
```

---

### T06 Hataları

#### `PostgreSQL kill -9 sonrası yeniden başlatılamadı`
```bash
# Resource durumu
pcs resource status postgresql

# Failcount kontrol (< 3 olmalı)
pcs resource failcount show postgresql

# Cleanup ve retry
pcs resource cleanup postgresql
```

#### `Postmaster PID tespit edilemedi`
```bash
# PostgreSQL çalışıyor mu?
pcs resource status postgresql

# Process bul
pgrep -a postgres
ps aux | grep postgres
```

---

### T09 Hataları

#### `IPMI erişilemiyor`
```bash
# Manuel test
ipmitool -I lanplus \
  -H <ipmi_ip> -p <ipmi_port> \
  -U <ipmi_user> -P <ipmi_pass> \
  chassis power status

# hosts.yml: ipmi_ip, ipmi_port, ipmi_user, ipmi_password kontrol et
```

#### `Fence edilen node geri dönmedi`
```bash
# Güç durumu kontrol
ipmitool -I lanplus \
  -H <ipmi_ip> -p <ipmi_port> \
  -U <ipmi_user> -P <ipmi_pass> \
  chassis power status

# Güç ver
ipmitool -I lanplus ... chassis power on

# Boot bekleme sonrası cluster başlat
pcs cluster start <node>
```

---

### Genel Temizlik Komutları

Herhangi bir test başarısız olduktan sonra cluster'ı temiz hale getirmek için:

```bash
# iptables kurallarını manuel kaldır (her iki node'da)
iptables -D INPUT  -p udp --dport 5405 -j DROP 2>/dev/null
iptables -D OUTPUT -p udp --dport 5405 -j DROP 2>/dev/null
iptables -D INPUT  -p tcp --dport 5405 -j DROP 2>/dev/null
iptables -D OUTPUT -p tcp --dport 5405 -j DROP 2>/dev/null
iptables -D OUTPUT -p tcp --dport 5403 -j DROP 2>/dev/null
iptables -D INPUT  -p tcp --dport 5403 -j DROP 2>/dev/null
iptables -D OUTPUT -p tcp -d 172.16.16.248 --dport 3260 -j DROP 2>/dev/null
iptables -D INPUT  -p tcp -s 172.16.16.248 --dport 3260 -j DROP 2>/dev/null

# VIP location constraint kaldır
pcs resource clear vip
pcs constraint | grep vip

# Test tablolarını temizle
psql -h 192.168.0.63 -U postgres -c \
  "DROP TABLE IF EXISTS ha_test_t02, ha_test_t03, ha_test_t06,
   ha_load_t11, ha_test_t12 CASCADE;"

# Resource cleanup
pcs resource cleanup

# GFS2 test dosyaları
rm -f /shared/webfs/ha_test_t07_*.txt
```

> Temizlik komutlarını tek tek çalıştırmak yerine test playbook'unu yeniden başlatmak da yeterlidir — PREP play'i her durumda temizliği otomatik yapar.

---

*Test playbook versiyonu: son güncelleme 19 Mart 2026 · RHEL 9.7 · Pacemaker 2.1.x · pcs 0.11.x · PostgreSQL 18 · Ansible 2.14.x*
