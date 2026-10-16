# Proxmox: Samsung 970 EVO NVMe-ni RAID5 üçün **yalnız oxuma (read) keşi** kimi qurmaq

## Vəziyyət

| Cihaz | Quruluş | Rolu |
|---|---|---|
| 3 × 1.2 TB SAS | HW RAID5 | Proxmox burada qurulub. Write cache → RAID controller-də (write-back) |
| 1 × Samsung 970 EVO NVMe | — | **Yalnız read cache**, ancaq RAID5 üçün |
| 9 × 2 TB SAS | HW RAID6 | Toxunulmur, keşi controller-də qalır |

Məqsəd: yazmalar birbaşa RAID5-ə (controller-in write cache-i ilə) getsin, NVMe isə yalnız
tez-tez oxunan blokları saxlasın. NVMe sıradan çıxsa, **heç bir məlumat itməməlidir**.

Bunun üçün Proxmox-un istifadə etdiyi storage tipinə görə iki yol var:

- **Proxmox ext4/xfs (LVM / LVM-thin) üzərində qurulubsa** (default quraşdırma) → **LVM cache (dm-cache), `writethrough` rejimi**
- **Proxmox ZFS üzərində qurulubsa** → **ZFS L2ARC** (o, onsuz da yalnız read cache-dir)

> `bcache` tövsiyə olunmur: mövcud diski yenidən formatlamağı tələb edir, Proxmox isə həmin RAID üzərindədir.
> `dm-writecache` isə əksinə yalnız yazma keşidir — bizə lazım deyil.

---

## 0. Əvvəlcə yoxlama

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
pvs; vgs; lvs -a
zpool status 2>/dev/null
cat /etc/pve/storage.cfg
```

Nəyə baxırıq:

- RAID5 virtual diski (məs. `/dev/sda`) və RAID6 virtual diski (məs. `/dev/sdb`) hansıdır — **ölçüyə görə** ayırd edin
  (RAID5 ≈ 2.4 TB, RAID6 ≈ 14 TB).
- `pvs` çıxışında `pve` VG-si RAID5 üzərindədirsə → **Variant A (LVM)**.
- `zpool status` `rpool` göstərirsə → **Variant B (ZFS)**.
- NVMe-nin adı (adətən `/dev/nvme0n1`) və onun boş olduğu.

NVMe-nin sağlamlığı:

```bash
apt install -y nvme-cli smartmontools
nvme list
smartctl -a /dev/nvme0n1
```

> **Diqqət:** NVMe-nin daha əvvəl Proxmox storage kimi əlavə olunmadığından əmin olun
> (`/etc/pve/storage.cfg`). Aşağıdakı addımlar NVMe-dəki hər şeyi silir.

---

## Variant A — LVM cache (writethrough) — default Proxmox quraşdırması

Default quraşdırmada `pve` VG-sində bunlar var:

- `pve/root` — Proxmox sistemi
- `pve/swap`
- `pve/data` — LVM-thin pool, VM/CT diskləri burada (`local-lvm`)

Keşi **`pve/data`** thin pool-a qoşuruq (VM-lərin oxuması üçün). `root`-u keşləməyə ehtiyac yoxdur.

`writethrough` rejimində hər yazma **eyni anda** RAID5-ə yazılır və yazma yalnız RAID5 təsdiq
etdikdən sonra bitmiş sayılır. NVMe-də heç vaxt "dirty" (RAID-də olmayan) məlumat olmur →
NVMe xarab olsa belə məlumat itkisi yoxdur. Yazma sürəti = RAID controller-in write-back sürəti.

### A.1 Hazırlıq

```bash
# lazımi alətlər (Proxmox-da adətən artıq var)
apt install -y thin-provisioning-tools lvm2

# Mümkünsə VM/CT-ləri dayandırın (online da işləyir, amma ehtiyat üçün)
qm list; pct list
```

### A.2 NVMe-ni təmizləyib `pve` VG-sinə əlavə etmək

```bash
NVME=/dev/nvme0n1          # öz cihaz adınızı yazın!

wipefs -a $NVME
blkdiscard $NVME           # SSD-ni tam TRIM edir (opsional, tövsiyə olunur)

pvcreate $NVME
vgextend pve $NVME
pvs                        # nvme0n1 artıq pve VG-sindədir
```

> Keş LV-si mütləq keşlənən LV ilə **eyni VG-də** olmalıdır, ona görə NVMe `pve`-yə əlavə olunur.
> Bu, `root`-un NVMe-yə keçməsi demək deyil — yeni LV-ləri yalnız NVMe-yə yaradırıq (əmrin sonunda `$NVME` göstərilir).

### A.3 Cache pool yaratmaq

970 EVO-nun ~10%-ni boş saxlayın (SSD-nin ömrü və performansı üçün over-provisioning):

| 970 EVO ölçüsü | Tövsiyə olunan cache pool |
|---|---|
| 250 GB | `-L 210G` |
| 500 GB | `-L 420G` |
| 1 TB   | `-L 850G` |

```bash
lvcreate --type cache-pool -L 420G -n nvmecache pve $NVME
```

(Metadata LV-si avtomatik yaradılır.)

### A.4 Keşi `pve/data` thin pool-a qoşmaq — **writethrough**

```bash
lvconvert --type cache \
          --cachepool pve/nvmecache \
          --cachemode writethrough \
          pve/data
```

Təsdiq sualına `y` cavab verin.

### A.5 Yoxlama

```bash
lvs -a -o name,size,segtype,cache_mode,cache_policy,devices pve
```

Görməli olduğunuz: `[data_tdata]` → `segtype=cache`, `cache_mode=writethrough`, `cache_policy=smq`,
`[nvmecache_cdata]` / `[nvmecache_cmeta]` isə `/dev/nvme0n1` üzərində.

initramfs-i yeniləyin (dm-cache modulları daxil olsun):

```bash
update-initramfs -u -k all
reboot
```

Reboot-dan sonra yenidən `lvs -a ...` ilə keşin aktiv olduğunu yoxlayın, `pvesm status`-da `local-lvm` aktiv olmalıdır.

### A.6 Monitorinq

```bash
lvs -a -o name,cache_mode,cache_used_blocks,cache_total_blocks,\
cache_read_hits,cache_read_misses,cache_dirty_blocks pve
```

- `cache_read_hits` / `cache_read_misses` — oxumaların neçəsi NVMe-dən verilib.
- `cache_dirty_blocks` **həmişə 0** olmalıdır (writethrough-un sübutu).
- `smq` siyasəti blokları "isti" olduqca keşə qaldırır; keş bir neçə saat/gün işlədikdən sonra dolur.
  Böyük ardıcıl (sequential) oxumalar (backup və s.) keşi "zibilləmir" — bu normaldır.

Ətraflı statistika:

```bash
dmsetup status pve-data_tdata
```

### A.7 Keşi söndürmək / NVMe-ni çıxarmaq

```bash
lvconvert --uncache pve/data      # keşi ayırır və cache pool-u silir
vgreduce pve /dev/nvme0n1
pvremove /dev/nvme0n1
```

### A.8 NVMe xarab olarsa

Writethrough rejimində bütün məlumat RAID5-dədir. `root` NVMe-də olmadığı üçün sistem açılacaq,
yalnız `local-lvm` aktivləşməyəcək. Bərpa:

```bash
lvconvert --uncache --force pve/data
vgreduce --removemissing pve
vgchange -ay pve
```

Sonra yeni NVMe ilə A.2–A.5 addımlarını təkrarlayın.

---

## Variant B — ZFS L2ARC (əgər Proxmox ZFS üzərindədirsə)

L2ARC ZFS-in ikinci səviyyə **oxuma** keşidir — yazmalara heç vaxt istifadə olunmur, NVMe itsə pool zərər görmür.

> Qeyd: ZFS-i HW RAID virtual diski üzərində işlətmək ideal deyil, amma artıq belə qurulubsa, L2ARC işləyəcək.

```bash
ls -l /dev/disk/by-id/ | grep -i nvme       # stabil ad götürün
zpool add rpool cache /dev/disk/by-id/nvme-Samsung_SSD_970_EVO_XXXXXXXX
zpool status rpool                           # "cache" bölməsində NVMe görünməlidir
zfs get secondarycache rpool                 # "all" olmalıdır
```

Monitorinq:

```bash
zpool iostat -v rpool 5
arc_summary -s l2arc
```

Çıxarmaq: `zpool remove rpool /dev/disk/by-id/nvme-Samsung_SSD_970_EVO_XXXXXXXX`

OpenZFS 2.x-də L2ARC reboot-dan sonra da saxlanılır (persistent L2ARC, default aktivdir).
ZFS-də **SLOG (log) əlavə etməyin** — 970 EVO-da güc itkisindən qoruma (PLP) yoxdur, və bizə yazma keşi lazım deyil.

---

## RAID controller tərəfi (tövsiyələr)

**RAID5 virtual diski üçün:**

- Write policy: **Write Back** — yalnız BBU/CacheVault/flash-backed cache sağlamdırsa.
  BBU yoxdursa/xarabdırsa, controller-in avtomatik **Write Through**-a keçməsinə icazə verin
  ("Write Back with BBU", **"Always Write Back" yox**).
- Read policy: **No Read Ahead** (oxuma keşini NVMe edir; controller-in RAM-ı tamamilə yazmalara qalır).
  İstəsəniz "Read Ahead"-də də saxlaya bilərsiniz — ziddiyyət yaratmır.
- Disk (fiziki HDD) cache: **Disabled** (güc kəsilməsində qorunmur).

**RAID6 virtual diski:** dəyişiklik yoxdur. NVMe-ni RAID6 ilə bağlı heç bir VG-yə/pool-a əlavə etməyin.

---

## Qısa xülasə

| | LVM (default) | ZFS |
|---|---|---|
| Texnologiya | dm-cache, `writethrough` | L2ARC |
| Keşlənən | `pve/data` (VM/CT diskləri) | `rpool` |
| Yazma | birbaşa RAID5 (controller write-back) | birbaşa RAID5 |
| NVMe xarab olsa | məlumat itkisi yoxdur | məlumat itkisi yoxdur |
| Əsas əmr | `lvconvert --type cache --cachepool pve/nvmecache --cachemode writethrough pve/data` | `zpool add rpool cache <nvme>` |
