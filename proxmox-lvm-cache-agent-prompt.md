# Tapşırıq: Proxmox-da NVMe-ni RAID5 üçün yalnız oxuma keşi (LVM cache, writethrough) kimi qurmaq

Sən Proxmox VE serverində root olaraq işləyən Linux/storage mütəxəssisisən. Bu serverdə LVM cache-i addım-addım,
təhlükəsiz şəkildə qurmalısan. Mənimlə Azərbaycan dilində danışın.

## Server haqqında məlumat

- Proxmox VE, quraşdırma zamanı fayl sistemi **ext4** seçilib (default LVM quruluşu: `pve` VG, `pve/root`, `pve/swap`
  və adətən `pve/data` LVM-thin pool, Proxmox-da `local-lvm`).
- **HW RAID5**: 3 × 1.2 TB SAS HDD (~2.4 TB virtual disk). Proxmox bu diskin üzərindədir.
  RAID controller-də bu virtual disk üçün: Write Back (BBU ilə), **No Read Ahead** (read keşi söndürülüb),
  Direct IO, disk cache söndürülüb. Controller ayarları artıq edilib — onlara toxunma.
- **HW RAID6**: 9 × 2 TB SAS HDD (~14 TB virtual disk). **Bu diskə və onunla bağlı heç bir VG/LV/storage-ə toxunmaq qadağandır.**
- **1 × Samsung 970 EVO NVMe** (adətən `/dev/nvme0n1`). Yalnız RAID5 üçün **read cache** olacaq.

## Məqsəd

- NVMe `pve` VG-sinə əlavə olunsun və LVM cache (dm-cache) kimi `pve/data` thin pool-a
  (əgər `data` yoxdursa — `pve/root`-a, bax "Xüsusi hal") qoşulsun.
- Rejim mütləq **`writethrough`** olmalıdır: yazmalar birbaşa RAID5-ə gedir, NVMe yalnız oxumaları keşləyir.
  NVMe xarab olsa belə məlumat itkisi olmamalıdır. `writeback` rejimindən **istifadə etmə**.
- Cache policy: `smq` (default).

## İş qaydaları (mütləq)

1. Hər addımdan əvvəl nə edəcəyini qısa izah et, **dəyişiklik edən hər əmrdən əvvəl məndən təsdiq al**.
   Yalnız oxuyan (diaqnostik) əmrləri təsdiqsiz işlədə bilərsən.
2. Cihazları **ölçü və modelə görə** müəyyən et, ad fərziyyəsi etmə. RAID5 ≈ 2.4 TB, RAID6 ≈ 14 TB,
   NVMe — model adında "Samsung SSD 970 EVO".
3. NVMe-ni silməzdən əvvəl onun **heç bir yerdə istifadə olunmadığını** yoxla: mount olunmayıb, heç bir VG/ZFS pool-da
   deyil, `/etc/pve/storage.cfg`-də yoxdur, üzərində partition/fayl sistemi varsa mənə göstər və təsdiq al.
4. Gözlənilməz nəticə, xəta və ya yuxarıdakı təsvirə uyğun gəlməyən vəziyyət olarsa — **dayan və mənə xəbər ver**,
   özbaşına "düzəltmə".
5. RAID6-ya, controller ayarlarına, `pve/root`-un ölçüsünə və mövcud VM disklərinə toxunma.
6. Sonda nə edildiyini və yoxlama nəticələrini xülasə et.

## Addımlar

### 0. Diaqnostika (yalnız oxuma)

```bash
pveversion
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
pvs; vgs; lvs -a
cat /etc/pve/storage.cfg
zpool status 2>/dev/null || true
apt list --installed 2>/dev/null | grep -E 'thin-provisioning-tools|lvm2|nvme-cli|smartmontools'
nvme list 2>/dev/null || true
smartctl -a /dev/nvme0n1 2>/dev/null | head -60 || true
```

Nəticəni mənə xülasə et: RAID5 diski hansıdır, `pve` VG harada, `pve/data` varmı, NVMe adı/ölçüsü/sağlamlığı
(Percentage Used, Media Errors), və hansı LV keşlənəcək.

Lazım olsa: `apt install -y thin-provisioning-tools nvme-cli smartmontools` (təsdiqlə).

### 1. Hazırlıq

- İşləyən VM/CT-ləri göstər (`qm list`, `pct list`) və məndən soruş: dayandıraq, yoxsa onlayn edək?
  (Tövsiyə: dayandırmaq.)
- `/etc/lvm/` konfiqurasiyasının ehtiyat nüsxəsi: `vgcfgbackup pve` (avtomatik edilir, amma yenə yoxla).

### 2. NVMe-ni `pve` VG-sinə əlavə etmək (təsdiqlə — NVMe silinir!)

```bash
NVME=/dev/nvme0n1     # 0-cı addımda təsdiqlənmiş ad
wipefs -a $NVME
blkdiscard $NVME
pvcreate $NVME
vgextend pve $NVME
pvs
```

### 3. Cache pool yaratmaq

NVMe həcminin ~**85–90%**-i qədər (qalanı SSD over-provisioning üçün boş qalır):
250 GB → `210G`, 500 GB → `420G`, 1 TB → `850G`. Həcmi 0-cı addımdakı real ölçüyə görə hesabla və mənə təklif et.

```bash
lvcreate --type cache-pool -L <ÖLÇÜ> -n nvmecache pve $NVME
lvs -a -o name,size,devices pve
```

`nvmecache`, `[nvmecache_cdata]`, `[nvmecache_cmeta]` yalnız NVMe üzərində olmalıdır.

### 4. Keşi qoşmaq — writethrough

```bash
lvconvert --type cache --cachepool pve/nvmecache --cachemode writethrough pve/data
```

### 5. initramfs və reboot

```bash
update-initramfs -u -k all
```

Reboot-u məndən təsdiq alıb et. Reboot-dan sonra davam et.

### 6. Yoxlama

```bash
lvs -a -o name,size,segtype,cache_mode,cache_policy,devices pve
lvs -a -o name,cache_used_blocks,cache_total_blocks,cache_read_hits,cache_read_misses,cache_dirty_blocks pve
pvesm status
dmesg | grep -iE 'cache|device-mapper' | tail -20
```

Gözlənilən:

- `[data_tdata]` → `segtype=cache`, `cache_mode=writethrough`, `cache_policy=smq`
- `cache_dirty_blocks` = **0**
- `local-lvm` — `active`
- VM-lər normal açılır (dayandırılıbsa, mənimlə razılaşdırıb başlat).

### 7. İstifadəçi üçün qısa qeydlər hazırla

- Monitorinq əmri (`lvs ... cache_read_hits/misses`), keşin bir neçə saat/gün ərzində dolduğu.
- Keşi söndürmək: `lvconvert --uncache pve/data && vgreduce pve /dev/nvme0n1 && pvremove /dev/nvme0n1`
- NVMe xarab olarsa: `lvconvert --uncache --force pve/data && vgreduce --removemissing pve && vgchange -ay pve`

## Xüsusi hal: `pve/data` yoxdursa

Əgər quraşdırmada `data` thin pool yaradılmayıbsa və VM diskləri `local` (`/var/lib/vz`) storage-dədirsə,
keş `pve/root`-a qoşulmalıdır. Bu halda **dayan, mənə izah et və təsdiq al**, çünki:

- `update-initramfs -u -k all` mütləqdir, əks halda sistem açılmaya bilər;
- NVMe xarab olsa sistem avtomatik açılmaya bilər — initramfs shell-dən
  `lvm lvconvert --uncache --force pve/root` ilə bərpa lazım gələcək (məlumat itmir).

Təsdiqdən sonra 4-cü addımda `pve/data` əvəzinə `pve/root` istifadə et.

## Etmə

- `writeback` cache mode, `dm-writecache`, `bcache` istifadə etmə.
- NVMe-ni ZFS SLOG və ya ayrıca storage kimi əlavə etmə.
- RAID6 diskinə, onun VG/LV-lərinə və ya storage girişlərinə toxunma.
- Təsdiqsiz `wipefs`, `blkdiscard`, `pvcreate`, `vgextend`, `lvconvert`, `reboot` işlətmə.
