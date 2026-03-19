# Red Hat 9 High Availability Cluster — Ansible Playbook

**`ha_cluster_setup.yml`** · RHEL 9 · Pacemaker 2.1.x · Corosync 3.x · PostgreSQL 18

> Bu döküman `ha_cluster_setup.yml` playbook'unu hiç kullanmamış birinin sıfırdan kurulum yapabilmesi için hazırlanmıştır. Tüm adımlar, değişkenler, uyarılar ve sorun giderme yöntemleri eksiksiz olarak açıklanmıştır.

---

## İçindekiler

1. [Mimari Genel Bakış](#1-mimari-genel-bakış)
2. [Ön Koşullar](#2-ön-koşullar)
3. [Envanter ve Değişken Yapılandırması](#3-envanter-ve-değişken-yapılandırması)
4. [Kurulum Öncesi Yapılması Gerekenler](#4-kurulum-öncesi-yapılması-gerekenler)
5. [Kurulum Adımları (Play Akışı)](#5-kurulum-adımları-play-akışı)
6. [Cluster Kaynak Mimarisi](#6-cluster-kaynak-mimarisi)
7. [Kullanım](#7-kullanım)
8. [Doğrulama ve İzleme](#8-doğrulama-ve-izleme)
9. [Sorun Giderme](#9-sorun-giderme)
10. [Bakım İşlemleri](#10-bakım-işlemleri)
11. [Bilinen Sınırlamalar](#11-bilinen-sınırlamalar)

---

## 1. Mimari Genel Bakış

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Cluster: ha_webpgsql                          │
│                                                                      │
│  ┌──────────────────────┐          ┌──────────────────────┐          │
│  │       clstr01        │◄────────►│       clstr02        │          │
│  │   192.168.0.61       │ corosync │   192.168.0.62       │          │
│  │   172.16.16.61       │   ring   │   172.16.16.62       │          │
│  │  (enp1s0/enp2s0)     │          │  (enp1s0/enp2s0)     │          │
│  └──────────┬───────────┘          └───────────┬──────────┘          │
│             │                                  │                      │
│             └─────────────┬────────────────────┘                     │
│                           │ iSCSI (172.16.16.248:3260)               │
│                    ┌──────┴───────┐                                  │
│                    │  iSCSI LUN'lar│                                  │
│                    │              │                                   │
│  /dev/mapper/pv_ha_shared01 (100G)│ → shared_vg → gfs2_lv           │
│                    │   (GFS2)     │   /shared/webfs  [Aktif/Aktif]   │
│                    │              │                                   │
│  /dev/mapper/pv_ha_shared02 (200G)│ → db_vg → pgsql_lv              │
│                    │   (ext4)     │   /var/lib/pgsql [Aktif/Pasif]   │
│                    └──────────────┘                                  │
│                                                                      │
│  Virtual IP: 192.168.0.63/23  (Her zaman aktif node'da)             │
│                                                                      │
│  ┌──────────────────────┐                                            │
│  │    clstr-qrm01       │  Quorum Device (ffsplit, port 5403)        │
│  │    192.168.0.67      │  Cluster üyesi DEĞİL — sadece arbitratör   │
│  │   (qnetd arbiter)    │                                            │
│  └──────────────────────┘                                            │
└──────────────────────────────────────────────────────────────────────┘
```

### Bileşenler

| Bileşen | Değer | Açıklama |
|---|---|---|
| Cluster adı | `ha_webpgsql` | Pacemaker/Corosync cluster adı |
| Node 1 | `clstr01.lab.akyuz.tech` | 192.168.0.61 (public) / 172.16.16.61 (corosync) |
| Node 2 | `clstr02.lab.akyuz.tech` | 192.168.0.62 (public) / 172.16.16.62 (corosync) |
| Quorum device | `clstr-qrm01.lab.akyuz.tech` | 192.168.0.67 — ffsplit arbitratörü |
| Virtual IP | `192.168.0.63/23` | Aktif node üzerinde yaşar, otomatik failover |
| iSCSI target | `172.16.16.248` | Paylaşımlı depolama sunucusu |
| GFS2 diski | `/dev/mapper/pv_ha_shared01` → `shared_vg` | `/shared/webfs` — Aktif/Aktif (her iki node okur/yazar) |
| PostgreSQL diski | `/dev/mapper/pv_ha_shared02` → `db_vg` | `/var/lib/pgsql` — Aktif/Pasif (sadece VIP ile aynı node) |
| PostgreSQL | v18 (PGDG) | Pacemaker cluster resource olarak yönetilir |
| STONITH | `fence_ipmilan` | IPMI/iDRAC/iLO üzerinden out-of-band fencing |
| Multipath | `device-mapper-multipath` | iSCSI disk'lere sabit `/dev/mapper/` adı verir |

### Mimari Kararların Gerekçesi

**Neden iki ayrı iSCSI diski?**
- GFS2 diski (`shared_vg`) DLM + lvmlockd ile her iki node'da eşzamanlı okunur/yazılır. Bu disk web içerikleri gibi paylaşılan veriler için uygundur.
- PostgreSQL diski (`db_vg`) exclusive (tek node) erişim gerektirir. Ayrı disk kullanmak VG lock çakışmasını ve I/O izolasyon sorunlarını önler.

**Neden multipath (`/dev/mapper/pv_ha_shared0x`)?**
- iSCSI path'lerin her iki node'da farklı kernel aygıt adı alması (`sda`, `sdb`, `sdc`, `sdd`) sorununa karşı WWID bazlı sabit alias üretir.
- Path arızasında (örn. birden fazla HBA/path varsa) otomatik yük dengeleme ve geçiş sağlar.
- `hosts.yml`'deki `multipath_aliases` ile WWID → alias eşlemesi merkezi olarak yönetilir.

**Neden Quorum Device (qnetd)?**
- 2 node'lu cluster'da her node 1 oy alır. Bir node düşerse kalan node quorum elde edemez (`1/2 < yarı`). qnetd 3. tarafsız oy sağlar: `2/3 > yarı` → cluster çalışmaya devam eder.
- `ffsplit` algoritması: ağ bölünmesinde her iki taraf da qdevice'dan bağlantısını kaybederse küçük taraf kendini durdurur.

**Neden STONITH zorunlu?**
- Split-brain senaryosunda (ağ bölünmesi) her iki node da aktif olduğunu zanneder ve paylaşımlı diske aynı anda yazar → veri bozulması.
- STONITH (Shoot The Other Node In The Head): düşürülmesi gereken node IPMI/iDRAC/iLO üzerinden donanım düzeyinde yeniden başlatılır, böylece shared disk'e erişimi kesinlikle kesilir.

---

## 2. Ön Koşullar

### 2.1 Kontrol Makinesi (Ansible / Bastion Host)

Playbook'u çalıştıracak makine (bastion veya yönetim sunucusu):

```bash
# Ansible versiyonu kontrol et (>= 2.12 gerekli)
ansible --version

# Gerekli Red Hat System Roles koleksiyonu
ansible-galaxy collection install redhat.rhel_system_roles

# SSH anahtarı tüm node'lara dağıtılmış olmalı
ssh-copy-id root@192.168.0.61
ssh-copy-id root@192.168.0.62
ssh-copy-id root@192.168.0.67

# Bağlantı testi
ansible -i inventory/hosts.yml all -m ping
```

> **Not:** Bastion'da `psql` binary **gerekmez**. Parola ayarlama adımı (PLAY 13b) cluster node'u üzerinden çalışır.

### 2.2 Cluster Node'ları (clstr01, clstr02)

Her iki node için:

- **İşletim Sistemi:** RHEL 9.x (9.4, 9.5, 9.6, 9.7 test edilmiştir)
- **Kayıt:** Aktif RHSM (`subscription-manager status`) veya yerel repo
- **SSH:** `root` doğrudan erişimi veya `sudo NOPASSWD` yetkili kullanıcı
- **RAM:** Minimum 2 GB (playbook kontrol eder)
- **Disk (OS):** `/` üzerinde minimum 20 GB boş alan (playbook kontrol eder)
- **NIC sayısı:** En az 2 adet:
  - `enp1s0` → public/VIP ağı (192.168.0.0/23)
  - `enp2s0` → corosync ring ağı (172.16.16.0/24)
- **iSCSI LUN:** iSCSI target'a erişim (172.16.16.248:3260)
- **IPMI/iDRAC/iLO:** STONITH için **zorunlu**, kurulumdan önce doğrulanmalı

> **Kritik Uyarı:** STONITH (`fence_ipmilan`) çalışmadan Pacemaker split-brain durumunda veri bütünlüğünü koruyamaz. Kurulum IPMI erişimini doğrular; IPMI erişimi yoksa kurulum durur.

### 2.3 Quorum Device (clstr-qrm01)

- **İşletim Sistemi:** RHEL 9.x
- **Rol:** Cluster üyesi DEĞİLDİR, sadece `corosync-qnetd` çalıştırır
- **SSH:** `root` erişimi
- **Ağ:** Cluster node'larından TCP 5403 erişilebilir olmalı
- **Repo:** `corosync-qnetd` ve `pcs` paketleri için aktif repo

### 2.4 Ağ Gereksinimleri

| Kaynak | Hedef | Port | Protokol | Amaç |
|---|---|---|---|---|
| cluster nodes ↔ cluster nodes | — | 5405, 5406 | UDP | Corosync ring |
| cluster nodes ↔ cluster nodes | — | 2224, 2225 | TCP | pcsd |
| cluster nodes ↔ cluster nodes | — | 3121 | TCP | Pacemaker |
| cluster nodes ↔ cluster nodes | — | 21064 | TCP | DLM |
| cluster nodes → quorum device | 5403 | TCP | qnetd bağlantısı |
| cluster nodes → iSCSI target | 3260 | TCP | iSCSI |
| cluster nodes → IPMI | 623 (veya özel) | UDP/TCP | STONITH fencing |
| bastion → cluster nodes | 22 | TCP | SSH (Ansible) |
| bastion → quorum device | 22 | TCP | SSH (Ansible) |
| istemciler → VIP | 5432 | TCP | PostgreSQL erişimi |

### 2.5 iSCSI ve Multipath Ön Hazırlığı

Playbook çalışmadan önce iSCSI target'ın yapılandırılmış ve LUN'ların sunulmuş olması gerekir.

**WWID Tespiti** (kurulumdan önce yapılmalı):

```bash
# Önce iscsiadm ile bağlan
iscsiadm -m discovery -t st -p 172.16.16.248
iscsiadm -m node -T iqn.2025-01.com.sirket:storage01 -p 172.16.16.248 --login

# Multipath'i başlat
systemctl start multipathd
multipath -v2

# WWID'leri gör (parantez içindeki değer WWID'dir)
multipath -ll
```

Örnek çıktı:
```
mpatha (360014050b56232984714db8a4bdd0ea9) dm-2 LIO-ORG,lv_ha_shared01_
size=100G ...
mpathb (360014056d2949ca6a2b455d82d2c6b37) dm-3 LIO-ORG,lv_ha_shared02_
size=200G ...
```

Bu WWID değerleri `hosts.yml`'deki `multipath_aliases` bölümüne girilir (bkz. [Bölüm 3.3](#33-multipath-wwid-tanımları)).

---

## 3. Envanter ve Değişken Yapılandırması

### 3.1 Dosya Yapısı

```
proje-dizini/
├── ha_cluster_setup.yml          # Ana playbook — DEĞİŞTİRMEYİN
├── inventory/
│   ├── hosts.yml                 # ✏️  TÜM ayarlar buraya yapılır
│   └── group_vars/
│       └── all.yml               # Hesaplanmış değişkenler — DEĞİŞTİRMEYİN
```

> **Temel Kural:** Tüm yapılandırma `inventory/hosts.yml` dosyasından yapılır. `ha_cluster_setup.yml` ve `group_vars/all.yml` dosyalarında **hiçbir değişiklik yapılmaz.**

### 3.2 `hosts.yml` — Tam Referans

#### Cluster Node'ları (host-seviyesi değişkenler)

Her node için ayrı tanımlanır:

```yaml
all:
  children:
    cluster_nodes:
      hosts:
        clstr01.lab.akyuz.tech:
          ansible_host: 192.168.0.61          # SSH bağlantı IP'si
          ansible_user: root                   # SSH kullanıcısı
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
          cluster_ip: 172.16.16.61             # Corosync ring IP (enp2s0 interface)
          # ---- IPMI / iDRAC / iLO ----
          ipmi_ip: 192.168.1.248               # IPMI yönetim IP'si
          ipmi_port: 6330                      # IPMI port (standart: 623)
          ipmi_user: "admin"                   # IPMI kullanıcı adı
          ipmi_password: "DEGISTIRIN"          # !! MUTLAKA DEĞİŞTİRİN !!
          ipmi_lanplus: true                   # IPMI v2/iDRAC/iLO için true

        clstr02.lab.akyuz.tech:
          ansible_host: 192.168.0.62
          ansible_user: root
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
          cluster_ip: 172.16.16.62
          ipmi_ip: 192.168.1.248
          ipmi_port: 6331                      # clstr01'den farklı port (paylaşımlı IPMI gateway)
          ipmi_user: "admin"
          ipmi_password: "DEGISTIRIN"
          ipmi_lanplus: true

    quorum_device:
      hosts:
        clstr-qrm01.lab.akyuz.tech:
          ansible_host: 192.168.0.67
          ansible_user: root
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
          cluster_ip: 172.16.16.67             # Yönetim IP'si
```

#### Global Değişkenler (`vars` bölümü)

```yaml
  vars:
    # ---- ANSIBLE SSH -------------------------------------------------------
    ansible_ssh_common_args: '-o StrictHostKeyChecking=no -o ConnectTimeout=10'

    # ---- CLUSTER TEMEL AYARLAR ---------------------------------------------
    cluster_name: "ha_webpgsql"              # Corosync/GFS2 cluster adı
    cluster_password: "DEGISTIRIN"           # hacluster kullanıcısı şifresi !! MUTLAKA DEĞİŞTİRİN !!
    cluster_domain: "lab.akyuz.tech"         # FQDN domain kısmı

    # ---- AĞ AYARLARI -------------------------------------------------------
    public_nic: "enp1s0"                     # VIP ve public erişim interface'i
    cluster_nic: "enp2s0"                    # Corosync ring interface'i
    virtual_ip: "192.168.0.63"              # Cluster Virtual IP
    virtual_ip_cidr: "23"                    # Subnet mask (192.168.0.0/23)
    public_network: "192.168.0.0/23"         # Public ağ adresi

    # ---- iSCSI AYARLARI ----------------------------------------------------
    iscsi_target_ip: "172.16.16.248"         # iSCSI target sunucu IP'si
    iscsi_iqn: "iqn.2025-01.com.sirket:storage01"  # iSCSI IQN (target'tan alın)
    iscsi_initiator_base: "iqn.1994-05.lab.local"  # Initiator IQN prefix

    # ---- STORAGE: GFS2 Diski -----------------------------------------------
    shared_disk: "/dev/mapper/pv_ha_shared01"  # Multipath device adı
    shared_vg: "shared_vg"                   # GFS2 Volume Group adı
    gfs2_lv: "gfs2_lv"                       # GFS2 Logical Volume adı
    gfs2_lv_size: "50G"                      # GFS2 LV boyutu
    gfs2_mount: "/shared/webfs"              # GFS2 mount noktası (Aktif/Aktif)
    gfs2_journals: 2                         # Cluster node sayısına EŞİT OLMALI
    gfs2_lock_table: "webfs"                 # GFS2 lock table adı

    # ---- STORAGE: PostgreSQL Diski -----------------------------------------
    db_disk: "/dev/mapper/pv_ha_shared02"    # Multipath device adı
    db_vg: "db_vg"                           # PostgreSQL Volume Group adı
    pgsql_lv: "pgsql_lv"                     # PostgreSQL Logical Volume adı
    pgsql_lv_size: "40G"                     # PostgreSQL LV boyutu
    pgsql_mount: "/var/lib/pgsql"            # PostgreSQL mount noktası (Aktif/Pasif)

    # ---- MULTIPATH WWID TANIMLARI ------------------------------------------
    # WWID tespiti: multipath -ll | grep -E '^\S.*\('
    # !! Her ortam için WWID'leri güncelleyin !!
    multipath_aliases:
      - alias: "pv_ha_shared01"              # /dev/mapper/pv_ha_shared01
        wwid: "360014050b56232984714db8a4bdd0ea9"   # GFS2 disk WWID
      - alias: "pv_ha_shared02"              # /dev/mapper/pv_ha_shared02
        wwid: "360014056d2949ca6a2b455d82d2c6b37"   # PostgreSQL disk WWID

    # ---- POSTGRESQL AYARLARI -----------------------------------------------
    pgsql_version: "18"                      # PostgreSQL major version
    pgsql_port: 5432                         # PostgreSQL dinleme portu
    pgsql_data_dir: "/var/lib/pgsql/data"    # PostgreSQL data dizini
    pgsql_allowed_networks:                  # pg_hba.conf erişim listesi
      - "192.168.0.0/23"                     # Public ağdan erişim
      - "10.10.10.0/24"                      # Ek ağdan erişim
      - "127.0.0.1/32"                       # Localhost
    pgsql_password: "DEGISTIRIN"             # postgres kullanıcısı parolası !! MUTLAKA DEĞİŞTİRİN !!

    # ---- QUORUM DEVICE AYARLARI --------------------------------------------
    qdevice_port: 5403                       # corosync-qnetd dinleme portu
    qdevice_algorithm: "ffsplit"             # 2 node için ffsplit önerilir

    # ---- STONITH AYARLARI --------------------------------------------------
    stonith_enabled: true
    stonith_action: "reboot"                 # reboot (önerilen) | off

    # ---- CLUSTER RESOURCE TIMEOUT'LARI ------------------------------------
    resource_start_timeout: "60s"
    resource_stop_timeout: "60s"
    resource_monitor_interval: "30s"
    no_quorum_policy: "freeze"              # 2-node için "freeze" önerilir
    resource_stickiness: "100"             # Failback otomatik değil (manuel gerekir)
    migration_threshold: "3"               # 3 hata → node dışla

    # ---- FIREWALL PORT'LARI -----------------------------------------------
    ha_firewall_ports:
      - "2224/tcp"                          # pcsd
      - "3121/tcp"                          # pacemaker
      - "5405/udp"                          # corosync
      - "5406/udp"                          # corosync
      - "21064/tcp"                         # dlm
      - "9929/tcp"                          # corosync-qnetd
      - "9929/udp"                          # corosync-qnetd
      - "5432/tcp"                          # PostgreSQL
```

### 3.3 Multipath WWID Tanımları

`multipath_aliases` değişkeni her ortamda farklıdır. WWID tespiti için:

```bash
# 1. iSCSI oturumu başlatıldıktan sonra (ya da zaten bağlıysa)
# 2. multipathd çalışıyorsa:
multipath -ll

# Çıktı formatı:
# <isim> (<WWID>) dm-X <üretici,model>
# size=<boyut> ...
# |-+- policy=... status=active
# | `- 6:0:0:0 sda 8:0 active ready running
```

`alias` değeri `shared_disk` ve `db_disk` değerlerinin `basename`'i olmalıdır:
- `shared_disk: "/dev/mapper/pv_ha_shared01"` → `alias: "pv_ha_shared01"`
- `db_disk: "/dev/mapper/pv_ha_shared02"` → `alias: "pv_ha_shared02"`

### 3.4 `group_vars/all.yml` — Açıklama

Bu dosya `hosts.yml`'deki değerlerden otomatik türetilen değişkenleri içerir:

| Değişken | Açıklama |
|---|---|
| `ha_packages` | Cluster node'larına kurulacak paket listesi |
| `qdevice_packages` | Quorum device'a kurulacak paket listesi |
| `cluster_nodes_config` | `ha_cluster` rolü için node yapılandırması (Jinja2 ile türetilir) |
| `fence_resources` | Her node için STONITH resource tanımı (IPMI bilgileri inventory'den alınır) |
| `base_cluster_resources` | DLM, lvmlockd, VIP resource tanımları |
| `base_resource_clones` | DLM ve lvmlockd için clone yapılandırması |
| `base_constraints_order` | `dlm → lvmlockd` başlangıç sırası |
| `cluster_properties` | `stonith-enabled`, `no-quorum-policy` gibi cluster özellikleri |
| `fence_location_constraints` | Bir node'un kendi fence agent'ını başlatamaması kısıtlaması |

> **Bu dosyayı düzenlemeyin.** Değişiklik gerekiyorsa `hosts.yml`'deki kaynak değerleri güncelleyin.

### 3.5 Güvenlik — Hassas Değişkenler

Aşağıdaki değişkenler açık metin yerine **Ansible Vault** ile şifrelenmelidir:

```bash
# Tek değer şifreleme
ansible-vault encrypt_string 'gizli_sifre' --name 'cluster_password'
ansible-vault encrypt_string 'gizli_sifre' --name 'ipmi_password'
ansible-vault encrypt_string 'gizli_sifre' --name 'pgsql_password'

# Üretilen çıktıyı hosts.yml'e yapıştırın:
# cluster_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   ...

# Vault şifreli playbook çalıştırma
ansible-playbook ha_cluster_setup.yml --ask-vault-pass
# veya vault şifre dosyası ile
ansible-playbook ha_cluster_setup.yml --vault-password-file ~/.vault_pass
```

---

## 4. Kurulum Öncesi Yapılması Gerekenler

Playbook'u çalıştırmadan önce şu kontrol listesi tamamlanmalıdır:

### ✅ Kontrol Listesi

```
[ ] RHEL 9.x tüm node'larda kurulu ve güncel
[ ] Aktif RHSM veya yerel repo tüm node'larda erişilebilir
[ ] root SSH erişimi çalışıyor (tüm node'lar için test edildi)
[ ] Public NIC (enp1s0) ve Corosync NIC (enp2s0) her iki node'da yapılandırılmış
[ ] Corosync NIC'ler arası ping çalışıyor (172.16.16.61 ↔ 172.16.16.62)
[ ] iSCSI target erişilebilir (172.16.16.248:3260)
[ ] İki ayrı iSCSI LUN sunulmuş (GFS2 için, PostgreSQL için)
[ ] IPMI/iDRAC/iLO her iki node için erişilebilir ve test edildi
[ ] WWID'ler tespit edildi ve hosts.yml'e girildi
[ ] hosts.yml'deki tüm şifreler değiştirildi (cluster_password, ipmi_password, pgsql_password)
[ ] redhat.rhel_system_roles koleksiyonu kuruldu
```

### 4.1 IPMI Erişim Testi

```bash
# clstr01'in IPMI'sini test et
ipmitool -I lanplus \
  -H 192.168.1.248 \
  -p 6330 \
  -U admin \
  -P <parola> \
  chassis power status
# Beklenen: "Chassis Power is on"

# clstr02'nin IPMI'sini test et
ipmitool -I lanplus \
  -H 192.168.1.248 \
  -p 6331 \
  -U admin \
  -P <parola> \
  chassis power status
```

### 4.2 Corosync Ağ Testi

```bash
# clstr01'den clstr02'ye corosync ağı üzerinden ping
ping -I enp2s0 172.16.16.62 -c 4

# clstr02'den clstr01'e
ping -I enp2s0 172.16.16.61 -c 4
```

### 4.3 iSCSI Ön Test

```bash
# iSCSI target keşfi
iscsiadm -m discovery -t st -p 172.16.16.248

# Beklenen:
# 172.16.16.248:3260,1 iqn.2025-01.com.sirket:storage01
```

### 4.4 Ansible Bağlantı Testi

```bash
# Tüm node'lara ping
ansible -i inventory/hosts.yml all -m ping

# Beklenen:
# clstr01.lab.akyuz.tech | SUCCESS
# clstr02.lab.akyuz.tech | SUCCESS
# clstr-qrm01.lab.akyuz.tech | SUCCESS

# Syntax kontrolü
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml --syntax-check
```

---

## 5. Kurulum Adımları (Play Akışı)

Playbook **19 play**'den oluşur ve yaklaşık **8–12 dakika** sürer. Her play `any_errors_fatal: true` ile çalışır; bir hata tüm kurulumu durdurur.

### Play 1 — Önkoşul Kontrolü ve Hazırlık
**Hedef:** `all` (cluster nodes + quorum device)

Bu play kurulumun güvenli ilerleyebileceğini doğrular:

- **OS kontrolü:** RHEL 9.x doğrulanır; farklı sürümde durur
- **RAM kontrolü:** Minimum 2 GB kontrol edilir
- **Disk kontrolü:** `/` üzerinde 20 GB boş alan kontrol edilir
- **SELinux durumu:** Raporlanır (Enforcing önerilir ama zorunlu değil)
- **Hostname:** `systemd` ile FQDN olarak ayarlanır
- **`/etc/hosts` fix:** `::1` loopback satırından hostname çakışması temizlenir

  > Bu fix olmadan `pcs auth` ve corosync, node adını `::1`'e resolve ederek iletişim kuramaz.

- **`/etc/hosts` güncelleme:** Tüm cluster node'ları ve quorum device girişleri eklenir (inventory'den türetilir)
- **Hostname çözümleme testi:** `getent hosts` ile tüm node'lar doğrulanır

### Play 1b — IPMI Bağlantı Kontrolü
**Hedef:** `cluster_nodes`

- `ipmitool` kurulumu
- Her node'un IPMI IP:port'una TCP bağlantı testi
- `ipmitool chassis power status` ile kimlik doğrulama
- **Başarısız olursa kurulum durur** — STONITH olmadan cluster güvenli değildir

### Play 2 — Paket Kurulumu (Cluster Node'ları)
**Hedef:** `cluster_nodes`

1. Çakışan paketler kaldırılır (`python3-google-auth`, eski `postgresql` paketleri)
2. Sistem güncellemesi yapılır (`dnf update`)
3. PostgreSQL PGDG `dnf module` devre dışı bırakılır (sürüm çakışması önleme)
4. Tüm HA paketleri kurulur:

```
resource-agents     resource-agents-paf   pacemaker         pcs
fence-agents-all    corosync              corosync-qdevice  corosync-qnetd
dlm                 lvm2-lockd            gfs2-utils        ipmitool
iscsi-initiator-utils  lvm2              device-mapper-multipath
postgresql18-server postgresql18          postgresql18-contrib
```

### Play 3 — Paket Kurulumu (Quorum Device)
**Hedef:** `quorum_device`

- Quorum device paketleri kurulur: `corosync-qnetd`, `corosync-qdevice`, `pcs`
- `pcs` paketi zorunludur — `pcsd` servisi bu paketle gelir

### Play 4 — Firewall Yapılandırması
**Hedef:** `all`

| Node Grubu | Yapılandırma |
|---|---|
| Cluster nodes | `high-availability` firewalld servisi (2224, 3121/tcp; 5405, 5406/udp) |
| Cluster nodes | 5403/tcp (qdevice), 21064/tcp (DLM), 5432/tcp (PostgreSQL) |
| Quorum device | 5403/tcp (qnetd dinleme portu) |

### Play 5 — Quorum Device Servisi Kurulumu
**Hedef:** `quorum_device`

- NSS sertifika DB mevcut mu kontrol edilir
- Mevcut değilse: `pcs qdevice setup model net --enable --start`
- `corosync-qnetd` servis doğrulaması (aktif olana kadar retry)
- `force_reinstall=true` ile NSS DB ve tüm yapılandırma sıfırlanır

### Play 6 — iSCSI ve Multipath Yapılandırması
**Hedef:** `cluster_nodes`

Bu play iki kritik adımı birleştirir:

**iSCSI:**
1. Initiator adı yapılandırılır: `iqn.1994-05.lab.local:<hostname>`
2. `iscsid` servisi başlatılır/enable edilir
3. Target discovery: `iscsiadm -m discovery -t st -p 172.16.16.248`
4. LUN login: `iscsiadm -m node -T <iqn> --login`
5. Otomatik başlatma: `node.startup = automatic`

**Multipath:**
1. `device-mapper-multipath` kurulumu
2. `iscsiadm -m session --rescan` ile LUN'lar yeniden taranır
3. `/etc/multipath.conf` `hosts.yml → multipath_aliases`'ten WWID → alias eşlemesi ile yazılır:

```
multipaths {
    multipath {
        wwid  360014050b56232984714db8a4bdd0ea9
        alias pv_ha_shared01
    }
    multipath {
        wwid  360014056d2949ca6a2b455d82d2c6b37
        alias pv_ha_shared02
    }
}
```

4. `multipathd` restart → `multipath -v2` → `udevadm settle`
5. `/dev/mapper/pv_ha_shared01` ve `/dev/mapper/pv_ha_shared02` varlığı doğrulanır (120s timeout)

> **Kritik:** `multipath_aliases` içindeki WWID değerleri ortama özel olduğundan kurulum öncesi `multipath -ll` ile tespit edilmelidir (bkz. [Bölüm 4.3](#43-iscsi-ön-test)).

### Play 7 — hacluster Kullanıcısı ve PCSD
**Hedef:** `cluster_nodes`

- `hacluster` kullanıcısına `cluster_password` hash'lenerek atanır
- `pcsd` servisi başlatılır ve enable edilir
- Tüm cluster node'ları birbirini `pcs host auth` ile authenticate eder
- Quorum device de authenticate edilir (cluster setup için zorunlu)

### Play 7b — Corosync Ön Kontrol ve Kapsamlı Temizlik
**Hedef:** `cluster_nodes`

Corosync başlatma hatalarının %90'ını önleyen adımlar:

1. `cluster_ip`'nin ilgili interface'e atanmış olduğu doğrulanır
2. Node'lar arası cluster ağı ping testi
3. Eski `corosync.conf`, `authkey`, CIB, ring state dosyaları temizlenir
4. `pcsd` token cache temizlenir
5. Mevcut corosync/pacemaker servisleri durdurulur
6. `pcsd` yeniden başlatılır (temiz token cache ile)
7. Re-authentication yapılır

`force_reinstall=true` modu ek olarak:
- `pcs cluster destroy --all` ile cluster tamamen silir
- Tüm Pacemaker CIB ve state dosyalarını kaldırır
- Kritik dizinleri doğru sahiplik/izinlerle yeniden oluşturur

### Play 8 — HA Cluster Kurulumu (Red Hat System Roles)
**Hedef:** `cluster_nodes`

`redhat.rhel_system_roles.ha_cluster` rolü çalıştırılır:
- `corosync.conf` oluşturulur (Kronosnet transport, ring: `enp2s0`)
- Pacemaker başlatılır
- Cluster membership kurulur

**Post-task tanılama:** Corosync başarısız olursa otomatik teşhis raporu üretilir:
- `journalctl -u corosync` son 60 satır
- `corosync.conf` içeriği
- Cluster IP ring ping testi
- SELinux AVC denetimleri

### Play 8-diag — Corosync/Pacemaker Doğrulama
**Hedef:** `cluster_nodes`

- `systemctl is-active corosync pacemaker`
- `pcs status` çıktısı
- Corosync çalışmıyorsa tam teşhis:
  - `journalctl -u corosync -n 80`
  - `corosync.conf` içeriği
  - Cluster IP ping testi
  - SELinux AVC kayıtları
  - NetworkManager bağlantı durumu
  - `iptables -L INPUT` (firewall dışı kural kontrolü)

### Play 8b — Quorum Device Ekleme
**Hedef:** `cluster_nodes[0]` (clstr01)

```bash
pcs quorum device add model net \
  host=clstr-qrm01.lab.akyuz.tech \
  algorithm=ffsplit \
  port=5403
```

- Quorum device varlık kontrolü `pcs quorum device status` ile yapılır
- `corosync-qdevice` servisi her node'da başlatılır
- Quorum durumu doğrulanır (`Quorate: Yes`)

Beklenen quorum çıktısı:
```
Nodeid   Votes   Qdevice   Name
     1       1   A,V,NMW   clstr01.lab.akyuz.tech
     2       1   A,V,NMW   clstr02.lab.akyuz.tech
     0       1             Qdevice
```

> **Kritik:** `pcs quorum status` çıktısında tablo başlığında "Qdevice" geçer, bu nedenle qdevice varlığını doğrulamak için `pcs quorum device status` kullanılır.

### Play 8c — STONITH ve Temel Cluster Kaynakları
**Hedef:** `cluster_nodes[0]` (clstr01)

**Cluster özellikleri:**
```
stonith-enabled          = true
stonith-action           = reboot
no-quorum-policy         = freeze
start-failure-is-fatal   = false
resource-stickiness      = 100
migration-threshold      = 3
failure-timeout          = 3600s
```

**STONITH kaynakları (her node için ayrı):**
```bash
pcs stonith create fence_ipmi_clstr01 fence_ipmilan \
  pcmk_host_list=clstr01.lab.akyuz.tech \
  ip=192.168.1.248 ipport=6330 \
  pcmk_delay_base=0s \
  meta pcmk_reboot_action=reboot

pcs stonith create fence_ipmi_clstr02 fence_ipmilan \
  pcmk_host_list=clstr02.lab.akyuz.tech \
  ip=192.168.1.248 ipport=6331 \
  pcmk_delay_base=30s \
  meta pcmk_reboot_action=reboot
```

> **Asimetrik delay:** Split-brain'de her iki node birbirini fence etmeye çalışırsa clstr01'in delay'i 0s, clstr02'ninki 30s olduğundan clstr01 önce kazanır. Simetrik fencing döngüsü önlenir.

**Location constraint:** Her node kendi fence agent'ını çalıştıramaz:
```bash
pcs constraint location fence_ipmi_clstr01 avoids clstr01
pcs constraint location fence_ipmi_clstr02 avoids clstr02
```

**VIP resource:**
```bash
pcs resource create vip ocf:heartbeat:IPaddr2 \
  ip=192.168.0.63 cidr_netmask=23 nic=enp1s0
```

**DLM ve lvmlockd (her node'da çalışır — clone):**
```bash
pcs resource create dlm ocf:pacemaker:controld \
  op monitor interval=30s timeout=30s
pcs resource clone dlm interleave=true ordered=true

pcs resource create lvmlockd ocf:heartbeat:lvmlockd \
  op monitor interval=30s timeout=30s
pcs resource clone lvmlockd interleave=true ordered=true

pcs constraint order start dlm-clone then lvmlockd-clone
```

### Play 9 — LVM ve Depolama Yapılandırması
**Hedef:** `cluster_nodes`

Bu play'in adım sırası kritiktir:

```
1. lvm.conf → use_lvmlockd = 1 (lineinfile + regexp)
2. lvm2-lockd paketi kurulu mu doğrula
3. lvmlockd systemd birimi maskelenir (Pacemaker yönetecek)
4. DLM cluster resource başlamasını bekle (retry: 20 × 10s)
5. lvmlockd cluster resource başlamasını bekle (retry: 20 × 10s)
--- Sadece clstr01'de ---
6. pvcreate /dev/mapper/pv_ha_shared01
7. vgcreate --shared shared_vg /dev/mapper/pv_ha_shared01
8. vgchange --lock-start shared_vg   ← lock-start ÖNCE
9. vgs shared_vg                     ← verify SONRA
10. lvcreate -L 50G -n gfs2_lv shared_vg
11. pvcreate /dev/mapper/pv_ha_shared02
12. vgcreate --shared db_vg /dev/mapper/pv_ha_shared02
13. vgchange --lock-start db_vg
14. vgs db_vg
15. lvcreate -L 40G -n pgsql_lv db_vg
16. lvchange -aey --nolocking /dev/shared_vg/gfs2_lv   ← exclusive (mkfs için)
17. lvchange -aey --nolocking /dev/db_vg/pgsql_lv
18. mkfs.gfs2 -j2 -p lock_dlm -t ha_webpgsql:gfs2_lv /dev/shared_vg/gfs2_lv
19. lvchange -an --nolocking /dev/shared_vg/gfs2_lv    ← DEAKTIVE ET
20. mkfs.ext4 /dev/db_vg/pgsql_lv
21. lvchange -an --nolocking /dev/db_vg/pgsql_lv
--- Her node'da ---
22. pvscan --cache && vgscan
23. vgchange --lock-start shared_vg   ← her node
24. vgchange --lock-start db_vg       ← her node
25. vgs shared_vg doğrulama
```

**`use_lvmlockd = 1` nasıl ayarlanır?**

`lvm.conf` içindeki `use_lvmlockd` satırı genellikle yorum içindedir (`# use_lvmlockd = 0`). `lineinfile` modülü ile:

```yaml
- name: "LVM | lvm.conf - use_lvmlockd = 1 yap"
  lineinfile:
    path: /etc/lvm/lvm.conf
    regexp: '^\s*#?\s*use_lvmlockd\s*='
    line: "\tuse_lvmlockd = 1"
    backup: yes
```

`regexp` şu formatların hepsini yakalar:
- `# 	use_lvmlockd = 0` (standart yorum)
- `	use_lvmlockd = 0` (yorumsuz, 0 değerli)
- `use_lvmlockd = 0` (başında boşluk olmayan)

> **Kritik:** `vgcreate --shared` ile oluşturulan VG'ler (`vg_lock_type=dlm`) önce `vgchange --lock-start` çalıştırılmadan `vgs` ile görülemez. `--nolocking` bu sorunu çözmez.

> **Kritik:** `mkfs` sonrası LV `lvchange -an` ile deaktive edilmezse Pacemaker diğer node'da `activation_mode=shared` ile aktive ederken `LV locked by other host` hatası alır.

### Play 10 — Mount Noktaları
**Hedef:** `cluster_nodes`

- `/shared/webfs` dizini oluşturulur (`root:root`, 0755)
- `/var/lib/pgsql` dizini oluşturulur (`postgres:postgres`, 0700)
- `postgres` kullanıcısı kurulum sırasında oluşturulmuşsa sahiplik buna göre ayarlanır

### Play 11 — Cluster Kaynakları
**Hedef:** `cluster_nodes[0]` (clstr01)

Tüm Pacemaker resource'ları ve constraint'ler oluşturulur:

```bash
# GFS2 LV activate (her iki node'da — clone)
pcs resource create gfs2_lv_activate ocf:heartbeat:LVM-activate \
  vgname=shared_vg lvname=gfs2_lv \
  vg_access_mode=lvmlockd activation_mode=shared \
  op monitor interval=30s on-fail=fence
pcs resource clone gfs2_lv_activate interleave=true

# GFS2 Filesystem (her iki node'da — clone)
pcs resource create gfs2_fs ocf:heartbeat:Filesystem \
  device=/dev/shared_vg/gfs2_lv directory=/shared/webfs \
  fstype=gfs2 options=noatime,nodiratime \
  op monitor interval=30s on-fail=fence
pcs resource clone gfs2_fs interleave=true

# PostgreSQL LV activate (sadece aktif node — exclusive)
pcs resource create pgsql_lv_activate ocf:heartbeat:LVM-activate \
  vgname=db_vg lvname=pgsql_lv \
  vg_access_mode=lvmlockd activation_mode=exclusive \
  op monitor interval=30s on-fail=fence

# PostgreSQL Filesystem (sadece aktif node)
pcs resource create pgsql_fs ocf:heartbeat:Filesystem \
  device=/dev/db_vg/pgsql_lv directory=/var/lib/pgsql \
  fstype=ext4 \
  op stop on-fail=block meta failure-timeout=3600s

# Order constraints (başlangıç sırası):
pcs constraint order start dlm-clone         then lvmlockd-clone
pcs constraint order start lvmlockd-clone    then gfs2_lv_activate-clone
pcs constraint order start gfs2_lv_activate-clone then gfs2_fs-clone
pcs constraint order start lvmlockd-clone    then pgsql_lv_activate
pcs constraint order start pgsql_lv_activate then pgsql_fs

# Colocation constraints (aynı node zorunluluğu):
pcs constraint colocation add pgsql_lv_activate with vip INFINITY
pcs constraint colocation add pgsql_fs with pgsql_lv_activate INFINITY
```

### Play 12 — PostgreSQL Kurulumu ve Yapılandırması
**Hedef:** `cluster_nodes[0]` (clstr01)

1. `pgsql_fs`'in mount edilmesini bekler (retry: 18 × 10s)
2. `initdb` ile PostgreSQL başlatılır (zaten varsa atlanır)
3. `postgresql.conf` temel ayarları:
   - `listen_addresses = '*'`
   - `port = 5432`
   - `max_connections = 200`
   - `log_destination = 'stderr'`
   - `logging_collector = on`
4. `pg_hba.conf` network erişimleri (`hosts.yml → pgsql_allowed_networks`)
5. PostgreSQL systemd servisi **tüm node'larda** maskelenir (Pacemaker yönetecek)
6. PostgreSQL cluster resource `target-role=Stopped` ile oluşturulur
7. Constraint'ler eklenir (order: `pgsql_fs → postgresql`; colocation: `postgresql with pgsql_fs INFINITY`)
8. Stale probe kayıtları temizlenir (`pcs resource cleanup postgresql`)
9. `target-role=Started` yapılır
10. PostgreSQL'in başlaması beklenir (retry: 24 × 10s)

> **Neden `Stopped` ile oluşturulur?** `pcs resource create` sonrası Pacemaker tüm node'larda probe yapar. Constraint'ler henüz eklenmeden probe çalışırsa clstr02'de `pgsql_fs` mount'lu olmadığından `postgresql.conf` bulunamaz → `on-fail=block` tetiklenir → BLOCKED. `target-role=Stopped` → constraint ekle → cleanup → `Started` sırası bu sorunu önler.

### Play 13b — PostgreSQL Kullanıcı Parolası
**Hedef:** `cluster_nodes[0]` (clstr01)

Cluster kurulumu tamamlandıktan sonra `postgres` kullanıcısına parola atanır:

1. VIP üzerinden PostgreSQL portu erişilebilirliği beklenir (120s timeout)
2. `psql` binary yolu tespit edilir
3. **Local UNIX socket + peer auth** ile bağlanılır (`become_user: postgres`)
4. `ALTER ROLE postgres WITH PASSWORD '...'`

```bash
# Neden local socket?
# PostgreSQL ilk kurulduğunda postgres kullanıcısının parolası yoktur.
# pg_hba.conf: local all postgres peer → parola sorulmadan bağlanılır.
# Parola set edildikten sonra VIP üzerinden md5 ile bağlanılabilir.
```

> **Güvenlik:** `no_log: false` bırakıldığında Ansible çıktısında parola görünür. Üretim ortamı için `no_log: true` yapın ve Ansible Vault kullanın.

### Play 13 — Son Doğrulama ve Testler
**Hedef:** `cluster_nodes[0]` (clstr01)

- `pcs resource cleanup` (tüm hata sayaçları sıfırlanır)
- 45 saniye bekleme (tüm resource'ların stabilize olması için)
- `pcs status` — tam cluster durumu
- `pcs resource status` — kaynak durumu
- `pcs stonith` — STONITH durumu
- `pcs constraint show --full` — tüm kısıtlamalar
- Her iki node'da `mountpoint -q /shared/webfs` ve `mountpoint -q /var/lib/pgsql`
- VIP aktif mi? (`ip addr show`)
- `corosync-quorumtool -s` — quorum durumu
- `pcs resource failcount show` — başarısız kaynak sayısı
- Kurulum VIP tercihi kısıtlaması kaldırılır (`loc-vip-prefer-node0`)
- Kurulum tamamlama raporu (ASCII tablo)

---

## 6. Cluster Kaynak Mimarisi

### 6.1 Kaynak Hiyerarşisi

```
STONITH (Her node diğerini fence eder):
  fence_ipmi_clstr01  → Started: clstr02  (clstr01'i fence etmesi için clstr02'de çalışır)
  fence_ipmi_clstr02  → Started: clstr01

SHARED (Her iki node'da çalışır — clone):
  dlm-clone           → Started: [ clstr01, clstr02 ]
    └── lvmlockd-clone → Started: [ clstr01, clstr02 ]
          ├── gfs2_lv_activate-clone → Started: [ clstr01, clstr02 ]
          │     └── gfs2_fs-clone   → Started: [ clstr01, clstr02 ]
          │           /shared/webfs (GFS2, Aktif/Aktif)
          │
          └── pgsql_lv_activate  → Started: clstr01 (vip ile aynı node)
                └── pgsql_fs     → Started: clstr01
                      └── postgresql → Started: clstr01

VIP:
  vip (192.168.0.63)  → Started: clstr01
```

### 6.2 Order Constraints (Başlangıç Sırası)

```
dlm-clone → lvmlockd-clone → gfs2_lv_activate-clone → gfs2_fs-clone
                           ↘
                             pgsql_lv_activate → pgsql_fs → postgresql
```

### 6.3 Colocation Constraints (Aynı Node Zorunluluğu)

```
pgsql_lv_activate  must be on same node as  vip
pgsql_fs           must be on same node as  pgsql_lv_activate
postgresql         must be on same node as  pgsql_fs
```

Bu zincir: `postgresql`, `pgsql_fs`, `pgsql_lv_activate` ve `vip` her zaman aynı node'da çalışır.

### 6.4 on-fail Davranışları

| Resource | Operasyon | on-fail | Açıklama |
|---|---|---|---|
| DLM / lvmlockd | monitor | fence | Lock daemon erişim kaybı → node fence |
| gfs2_lv_activate | monitor | fence | Shared LV erişim kaybı → node fence |
| gfs2_fs | monitor | fence | GFS2 mount erişim kaybı → node fence |
| pgsql_fs | start | fence | Mount başlatma hatası → node fence |
| pgsql_fs | stop | **block** | Umount hatası → fence DEĞİL, bekler |
| postgresql | start | restart | Başlatma hatası → yeniden dene |
| postgresql | stop | **block** | Servis durdurma hatası → fence DEĞİL |
| postgresql | monitor | restart | Servis öldü → yeniden başlat |
| fence_ipmi | monitor | — | IPMI erişilemiyor → raporlanır |

> `on-fail=block` stratejisi fencing döngüsünü önler: stop fail edince node fence edilmez, operatör `pcs resource cleanup` ile müdahale eder.

---

## 7. Kullanım

### 7.1 Normal Kurulum

```bash
# Syntax kontrolü
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml --syntax-check

# Dry-run (değişiklik yapmadan simüle et)
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml --check

# Kurulum
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml

# Vault şifreli ortamda
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml --ask-vault-pass
```

### 7.2 Sıfırlama (Temiz Yeniden Kurulum)

Mevcut cluster tamamen silinip baştan kurulur:

```bash
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml -e force_reinstall=true
```

`force_reinstall=true` şunları yapar:
- `pcs cluster destroy --all`
- `corosync.conf`, `authkey`, CIB, ring state dosyalarını temizler
- `corosync-qnetd` NSS DB'yi siler ve yeniden oluşturur
- Mevcut qdevice yapılandırmasını kaldırır
- Pacemaker dizinlerini doğru sahiplik/izinlerle yeniden oluşturur
- Baştan kurulum yapar

### 7.3 Değişken Override

```bash
# Farklı PostgreSQL versiyonu ile test
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml -e pgsql_version=16

# Farklı stonith aksiyonu
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml -e stonith_action=off

# Belirli play'den başla
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml \
  --start-at-task "PLAY 9/13 | LVM ve Paylaşımlı Depolama Yapılandırması"
```

---

## 8. Doğrulama ve İzleme

### 8.1 Kurulum Sonrası Kontrol

```bash
# Cluster genel durumu
pcs status

# Beklenen başarılı çıktı:
# Cluster name: ha_webpgsql
# Status of pacemakerd: 'Pacemaker is running' (last seen: ...)
# Cluster Summary:
#   * Stack: corosync (Knet)
#   * Current DC: clstr01.lab.akyuz.tech (version ...) - partition with quorum
#   * Last updated: ...
#   * 2 nodes configured
#   * N resource instances configured
#
# Node List:
#   * Online: [ clstr01.lab.akyuz.tech clstr02.lab.akyuz.tech ]
#
# Full List of Resources:
#   * fence_ipmi_clstr01  (stonith:fence_ipmilan): Started clstr02.lab.akyuz.tech
#   * fence_ipmi_clstr02  (stonith:fence_ipmilan): Started clstr01.lab.akyuz.tech
#   * Clone Set: dlm-clone [dlm]:
#       Started: [ clstr01.lab.akyuz.tech clstr02.lab.akyuz.tech ]
#   * Clone Set: lvmlockd-clone [lvmlockd]:
#       Started: [ clstr01.lab.akyuz.tech clstr02.lab.akyuz.tech ]
#   * Clone Set: gfs2_lv_activate-clone [gfs2_lv_activate]:
#       Started: [ clstr01.lab.akyuz.tech clstr02.lab.akyuz.tech ]
#   * Clone Set: gfs2_fs-clone [gfs2_fs]:
#       Started: [ clstr01.lab.akyuz.tech clstr02.lab.akyuz.tech ]
#   * pgsql_lv_activate   Started clstr01.lab.akyuz.tech
#   * pgsql_fs             Started clstr01.lab.akyuz.tech
#   * vip                  Started clstr01.lab.akyuz.tech
#   * postgresql           Started clstr01.lab.akyuz.tech
```

```bash
# Constraint'ler
pcs constraint

# Quorum durumu
pcs quorum status
corosync-quorumtool -s

# Quorum device durumu
pcs quorum device status
# Algorithm: ffsplit  State: connected

# Yapılandırma doğrulama
crm_verify -L -V

# Mount noktaları
df -h /shared/webfs /var/lib/pgsql

# PostgreSQL bağlantısı (VIP üzerinden)
psql -h 192.168.0.63 -U postgres -c "SELECT version();"

# STONITH durumu
pcs stonith
```

### 8.2 Sürekli İzleme

```bash
# Gerçek zamanlı cluster durumu (5 saniyede bir)
watch -n 5 pcs status

# Pacemaker logları
journalctl -u pacemaker -f

# Corosync logları
journalctl -u corosync -f

# STONITH olayları
journalctl -u pacemaker | grep -i "fence\|stonith"

# Resource başlatma/durdurma olayları
journalctl -u pacemaker | grep "Result of"

# GFS2 durumu
gfs2_tool df /shared/webfs

# LVM lock durumu
lvmlockctl --dump
```

---

## 9. Sorun Giderme

### 9.1 Cluster Genel Sorunlar

#### Cluster başlamıyor / node'lar birbirini görmüyor

```bash
# Corosync ring kontrolü
corosync-cfgtool -s
# Beklenen: ring0: active with no faults

# Cluster ağı erişilebilir mi?
ping -I enp2s0 172.16.16.62   # clstr01'den clstr02'ye

# Corosync log
journalctl -u corosync -n 80

# authkey eşleşiyor mu?
md5sum /etc/corosync/authkey   # Her iki node'da aynı olmalı
```

**authkey farklıysa:**
```bash
corosync-keygen   # clstr01'de üret
scp /etc/corosync/authkey clstr02:/etc/corosync/
systemctl restart corosync   # Her iki node'da
```

---

#### Quorum yok, cluster freeze

```bash
pcs quorum status
# Quorate: YES görünmeli

# Qdevice bağlı mı?
pcs quorum device status
# State: connected görünmeli

# qnetd servisi çalışıyor mu? (quorum device sunucusunda)
ssh root@192.168.0.67 systemctl status corosync-qnetd

# corosync-qdevice çalışıyor mu? (her cluster node'da)
systemctl status corosync-qdevice
```

**Quorum device yeniden ekle:**
```bash
pcs quorum device remove
pcs host auth clstr-qrm01.lab.akyuz.tech addr=192.168.0.67 \
  -u hacluster -p <cluster_password>
pcs quorum device add model net \
  host=clstr-qrm01.lab.akyuz.tech \
  algorithm=ffsplit port=5403
systemctl enable --now corosync-qdevice
```

---

### 9.2 Multipath / iSCSI Sorunları

#### `/dev/mapper/pv_ha_shared01` bulunamıyor

```bash
# Multipath durumu
multipath -ll

# Eğer mpatha/mpathb görünüyor ama pv_ha_shared01 yoksa:
# multipath.conf'taki alias tanımı eksik veya yanlış WWID

# WWID'leri kontrol et
multipath -ll | grep -E '^\S.*\('

# multipath.conf'u kontrol et
cat /etc/multipath.conf

# multipathd'yi yeniden başlat
systemctl restart multipathd
multipath -v2
udevadm settle --timeout=30

# Device mevcut mu?
ls -la /dev/mapper/pv_ha_shared*
```

**WWID yanlışsa:**
1. `hosts.yml`'deki `multipath_aliases` bölümündeki `wwid` değerlerini `multipath -ll` çıktısından doğrulayın
2. Playbook'u yeniden çalıştırın

---

#### iSCSI login başarısız

```bash
# iSCSI oturumları
iscsiadm -m session

# Discovery yeniden çalıştır
iscsiadm -m discovery -t st -p 172.16.16.248

# Login
iscsiadm -m node -T iqn.2025-01.com.sirket:storage01 \
  -p 172.16.16.248 --login

# iscsid servisi
systemctl status iscsid
```

---

### 9.3 STONITH Sorunları

#### Sonsuz fencing döngüsü

```
reboot of clstr02 ... last failed at ...
reboot of clstr01 ... last failed at ...
```

**Neden:** Her iki node aynı anda birbirini fence etmeye çalışıyor. Asimetrik delay doğru ayarlanmamış.

```bash
# Delay değerlerini kontrol et
pcs stonith show fence_ipmi_clstr01.lab.akyuz.tech | grep delay
pcs stonith show fence_ipmi_clstr02.lab.akyuz.tech | grep delay

# clstr01 fence agent: pcmk_delay_base=0s
# clstr02 fence agent: pcmk_delay_base=30s

# Düzelt
pcs stonith update fence_ipmi_clstr01.lab.akyuz.tech pcmk_delay_base=0s
pcs stonith update fence_ipmi_clstr02.lab.akyuz.tech pcmk_delay_base=30s
```

---

#### STONITH başarısız: IPMI erişilemiyor

```bash
# IPMI erişim testi
ipmitool -I lanplus -H 192.168.1.248 -p 6330 \
  -U admin -P <ipmi_password> chassis power status

# STONITH agent testi
pcs stonith test fence_ipmi_clstr02.lab.akyuz.tech
```

---

### 9.4 LVM / GFS2 Sorunları

#### `use_lvmlockd = 1` etkili olmuyor

```bash
# Aktif değer kontrol
grep 'use_lvmlockd' /etc/lvm/lvm.conf | grep -v '^\s*#'
# Beklenen: use_lvmlockd = 1

# lvmlockd process çalışıyor mu? (Pacemaker başlatmış olmalı)
pgrep -a lvmlockd
pcs resource status lvmlockd-clone
```

---

#### `Volume group "shared_vg" not found`

**Neden:** DLM lockspace başlatılmadan `vgs` çalıştırıldı veya `lvmlockd` çalışmıyor.

```bash
# DLM çalışıyor mu?
pcs resource status dlm-clone

# lvmlockd çalışıyor mu?
pcs resource status lvmlockd-clone
pgrep -a lvmlockd

# lock-start
vgchange --lock-start shared_vg
vgchange --lock-start db_vg
vgs shared_vg
vgs db_vg

# Sorun devam ediyorsa
lvmlockctl --dump
journalctl -u pacemaker | grep lvmlockd | tail -30
```

---

#### GFS2 LV activate clone clstr02'de başlamıyor — `LV locked by other host`

```bash
# clstr01'de exclusive lock kalmış
journalctl -u pacemaker | grep gfs2_lv_activate | tail -20

# clstr01'de deaktive et
lvchange -an --nolocking /dev/shared_vg/gfs2_lv

# Cleanup
pcs resource cleanup gfs2_lv_activate-clone
```

---

#### GFS2 mount olmuyor

```bash
# GFS2 cluster name eşleşiyor mu?
blkid /dev/shared_vg/gfs2_lv
# TYPE="gfs2" LABEL="ha_webpgsql:webfs" görünmeli
# cluster_name:lock_table = ha_webpgsql:webfs

# DLM lock alanı
dlm_tool ls
dlm_tool status

# Journal sayısı yeterli mi? (node sayısı kadar olmalı)
gfs2_tool jindex /dev/shared_vg/gfs2_lv
```

---

### 9.5 PostgreSQL Sorunları

#### `postgresql FAILED clstr02 (blocked)` — `stop 'not installed'`

```
postgresql stop on clstr02 returned 'not installed'
(Configuration file /var/lib/pgsql/data/postgresql.conf doesn't exist)
```

**Neden:** `pgsql_fs` clstr02'de mount'lu değil, Pacemaker postgresql'i orada durdurmaya çalıştı.

```bash
# Hata sayaçlarını temizle
pcs resource cleanup postgresql

# Constraint'ler doğru mu?
pcs constraint | grep postgresql
# Beklenen:
# postgresql with pgsql_fs  INFINITY  (colo-postgresql-psqlfs)
# start pgsql_fs then postgresql (order-psqlfs-postgresql)

# Eksik constraint varsa ekle
pcs constraint colocation add postgresql with pgsql_fs INFINITY \
  id=colo-postgresql-psqlfs
pcs constraint order start pgsql_fs then postgresql \
  id=order-psqlfs-postgresql

pcs resource cleanup postgresql
```

---

#### PostgreSQL başlamıyor — `initdb` yapılmamış

```bash
# Data dizini boş mu?
ls -la /var/lib/pgsql/data/

# Manuel initdb
sudo -u postgres /usr/pgsql-18/bin/initdb -D /var/lib/pgsql/data

pcs resource cleanup postgresql
```

---

#### PostgreSQL başladı ama bağlanılamıyor

```bash
# Hangi node'da çalışıyor?
pcs resource status postgresql

# VIP ile aynı node'da mı?
pcs resource status vip

# Farklı node'lardaysa constraint sorunu
pcs constraint | grep postgresql

# pg_hba.conf erişim sorunu olabilir
grep -v '^#' /var/lib/pgsql/data/pg_hba.conf

# Bağlantı testi (VIP üzerinden)
psql -h 192.168.0.63 -U postgres -c "SELECT 1;"
```

---

#### postgres kullanıcısı parolası auth hatası

```
FATAL: password authentication failed for user "postgres"
```

**Neden:** İlk kurulumda postgres kullanıcısının parolası yoktur. VIP üzerinden `md5` auth için önce local socket ile parola atanmalıdır.

```bash
# Local socket üzerinden parola ata (peer auth, parola gerektirmez)
sudo -u postgres psql -p 5432 \
  -c "ALTER ROLE postgres WITH PASSWORD 'yeni_parola';"

# Sonra VIP üzerinden test
PGPASSWORD="yeni_parola" psql \
  -h 192.168.0.63 -p 5432 -U postgres -c "SELECT version();"
```

---

### 9.6 Resource Genel Sorunları

#### Resource başlamıyor, hata sayacı doldu

```bash
# Hata detayı
pcs resource failcount show postgresql

# Hata sayaçlarını sıfırla
pcs resource cleanup postgresql

# Tüm resource'lar
pcs resource cleanup

# Başarısız node'u devre dışı bırakıp temizle
pcs node standby clstr02.lab.akyuz.tech
pcs resource cleanup
pcs node unstandby clstr02.lab.akyuz.tech
```

---

#### Resource debug başlatma

```bash
# Resource'u debug modda başlat (detaylı log)
pcs resource debug-start postgresql

# OCF agent'ı doğrudan çalıştır
OCF_ROOT=/usr/lib/ocf \
OCF_RESKEY_pgctl=/usr/pgsql-18/bin/pg_ctl \
OCF_RESKEY_pgdata=/var/lib/pgsql/data \
/usr/lib/ocf/resource.d/heartbeat/pgsql monitor
```

---

### 9.7 Fencing ile İlgili Uyarılar

#### Node fenced sonrası geri dönmüyor

```bash
# IPMI ile node durumu
ipmitool -I lanplus -H 192.168.1.248 -p 6330 \
  -U admin -P <ipmi_password> chassis power status

# Güç ver
ipmitool -I lanplus -H 192.168.1.248 -p 6330 \
  -U admin -P <ipmi_password> chassis power on

# Node geri döndükten sonra standby'dan çıkar (gerekirse)
pcs node unstandby clstr01.lab.akyuz.tech
pcs resource cleanup
```

---

## 10. Bakım İşlemleri

### 10.1 Node Bakımı (Rolling Maintenance)

```bash
# Bakım yapılacak node'u standby'a al (resource'lar diğerine taşınır)
pcs node standby clstr01.lab.akyuz.tech

# Tüm resource'ların clstr02'ye taşındığını doğrula
pcs status

# Bakım yap (güncelleme, kernel reboot vb.)
# ...

# Node'u geri al
pcs node unstandby clstr01.lab.akyuz.tech

# resource-stickiness=100 olduğundan resource'lar otomatik geri dönmez
# İsteğe bağlı manuel geri taşıma:
pcs resource move postgresql clstr01.lab.akyuz.tech
pcs resource clear postgresql   # geçici location constraint kaldır
```

---

### 10.2 VIP Failover Testi

```bash
# Aktif node'u belirle
pcs resource status vip

# Failover tetikle
pcs resource move vip clstr02.lab.akyuz.tech

# Test sonrası constraint kaldır (tekrar otomatik failover için)
pcs resource clear vip
pcs status
```

---

### 10.3 PostgreSQL Manuel Failover Testi

```bash
# PostgreSQL'i diğer node'a taşı
pcs resource move postgresql clstr02.lab.akyuz.tech

# vip ve pgsql_fs de clstr02'ye taşınmalı (colocation)
pcs status

# Test sonrası constraint kaldır
pcs resource clear postgresql
```

---

### 10.4 Cluster Durdurma / Başlatma

```bash
# Tüm cluster'ı durdur (planlı bakım)
pcs cluster stop --all

# Tüm cluster'ı başlat
pcs cluster start --all

# Tek node
pcs cluster stop clstr01.lab.akyuz.tech
pcs cluster start clstr01.lab.akyuz.tech

# Cluster otomatik başlatma (boot'ta)
pcs cluster enable --all
```

---

### 10.5 Yapılandırma Yedekleme

```bash
# CIB (Cluster Information Base) yedeği
pcs cluster cib > cluster_cib_backup_$(date +%Y%m%d).xml

# Corosync yapılandırması
cp /etc/corosync/corosync.conf corosync_backup_$(date +%Y%m%d).conf

# multipath.conf
cp /etc/multipath.conf multipath_backup_$(date +%Y%m%d).conf

# CIB geri yükle
pcs cluster cib-push cluster_cib_backup.xml --config
```

---

### 10.6 PostgreSQL Verisi Yedekleme

```bash
# VIP üzerinden pg_dump (cluster'dan bağımsız çalışır)
pg_dump -h 192.168.0.63 -U postgres -Fc mydb > mydb_$(date +%Y%m%d).dump

# Tüm cluster yedekle
pg_dumpall -h 192.168.0.63 -U postgres > dumpall_$(date +%Y%m%d).sql

# Continuous archiving için postgresql.conf
# archive_mode = on
# archive_command = 'cp %p /backup/wal/%f'
```

---

### 10.7 Cluster'a Yeni Node Ekleme

> **Uyarı:** Bu playbook 2 node için optimize edilmiştir. Yeni node eklemek için `gfs2_journals` sayısı ve `qdevice_algorithm` güncellenmeli, `ha_cluster_setup.yml` yeniden çalıştırılmalıdır.

```bash
# Önce hosts.yml'e yeni node ekle
# gfs2_journals değerini 3'e çıkar
# Sonra playbook'u çalıştır (force_reinstall gerekmez, idempotent)
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml
```

---

## 11. Bilinen Sınırlamalar

| Konu | Detay |
|---|---|
| **2 node sabit** | `gfs2_journals=2` ve `qdevice_algorithm=ffsplit` 2 node için optimize edilmiştir. 3+ node için `gfs2_journals`, `qdevice_algorithm` ve resource constraint'leri güncellenmeli. |
| **Tek iSCSI target** | iSCSI target tekli noktadır. Target tarafında HA gerekiyorsa DRBD veya Ceph değerlendirilebilir. |
| **PostgreSQL replikasyon yok** | `rep_mode=none` — aktif/pasif yapılandırmada replikasyon yoktur. Yalnızca tek node veri yazar. Replikasyon için `rep_mode=sync` veya `async` ile `resource-agents-paf` değerlendirilebilir. |
| **IPMI zorunlu** | STONITH olmadan Pacemaker split-brain durumunda kümeyi koruyamaz. IPMI'siz ortamlarda `fence_vmware_soap` veya `fence_kdump` alternatifleri değerlendirilebilir. |
| **RHEL 9 / pcs 0.11+** | `pcs resource defaults update` (eski: `pcs resource defaults`), `meta pcmk_reboot_action` (eski: `action=`) gibi syntax farklılıkları nedeniyle RHEL 8 veya daha eski sürümlerle uyumlu değildir. |
| **Failback otomatik değil** | `resource-stickiness=100` nedeniyle node geri döndüğünde resource'lar otomatik taşınmaz. `pcs resource move` ile manuel taşıma gerekir. |
| **Ansible 2.14** | `ansible.posix` koleksiyonu Ansible 2.14.x ile `[WARNING]: Collection ansible.posix does not support Ansible version 2.14.18` uyarısı verebilir. Bu uyarı kritik değildir, kurulum tamamlanır. |
| **no_log ve hata ayıklama** | PLAY 13b'de `no_log: false` bırakılmıştır. Üretim ortamında `no_log: true` yapın ve parolaları Ansible Vault ile şifreleyin. |
| **bastion'da psql gerekmez** | PLAY 13b cluster node'u üzerinden çalıştığından bastion'da `psql` kurulu olmak zorunda değildir. |

---

*Playbook versiyonu: son güncelleme 19 Mart 2026 · RHEL 9.7 · Pacemaker 2.1.x · pcs 0.11.x · PostgreSQL 18 · Ansible 2.14.x*
