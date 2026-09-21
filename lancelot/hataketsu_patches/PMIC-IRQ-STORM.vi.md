# Bao ngat (IRQ storm) tren IRQ 172 / mt6358-irq — Xiaomi Redmi 9 "lancelot", mainline 6.18

Trang thai: **CHUA SUA XONG — bao ngat van con tren may.**

Tom tat trong 5 dong:
1. Da tim va sua mot **loi that** cua mainline (`mt6358/core.h` thieu nhom XPP o bit 6,
   lam AUD/MISC lech 1 bit). Phan cung xac nhan (`TOP_INT_MASK_CON0 = 0x0040`).
2. Nhung **loi do khong phai nguyen nhan** cua bao ngat tren may nay. Da chung minh bang
   do dac: PMIC hoan toan khong keo duong ngat (`TOP_INT_STATUS0/RAW/INT_STA` = 0).
3. Nguyen nhan that nam o chan EINT 144 cua SoC bi giu o muc tich cuc.
4. Gia thuyet dan dau: `RG_INT_POLARITY` (PMIC `0x1a2` bit0, dang = 0) dat sai cuc tinh
   dau ra ngat cua PMIC.
5. Con mot phep do quyet dinh chua chay (module `mt6358_eintpol.ko`, muc 6.2) —
   phu thuoc agent chinh vi toi khong duoc dung toi may that.

---

## 1. Trieu chung

```
172:    3162669    0 0 0 0 0 0 0   mt-eint 144 Level     mt6358-irq
--- sau 3s ---
172:    3280466    0 0 0 0 0 0 0   mt-eint 144 Level     mt6358-irq
```

~39.000 ngat/giay. Tien trinh `irq/172-mt6358-irq` an ~14,3% CPU lien tuc.
Tat ca IRQ con cua PMIC (`mt6397-rtc`, `mtk-pmic-keys`) dem gan nhu bang 0.

### 1.1. Gia thuyet ban dau da bi BAC BO

Gia thuyet ban dau: "driver `generic_adc_battery` lam bao ngat tang gap 3 lan"
(13.721/s khi go module, 39.289/s khi nap lai), va nghi `mt6358_read_imp()` trong
`mt6359-auxadc.c` de lai co ngat chua clear.

Do lai nhieu lan tren may that (agent chinh thuc hien):

```
KHONG nap generic_adc_battery: 25621 /s
KHONG nap generic_adc_battery: 13940 /s
KHONG nap generic_adc_battery: 39097 /s

KHONG co module pin: 39607 /s
CO module pin:       39633 /s
```

=> Toc do dao dong 14k–39k/giay theo tung thoi diem va **khong phu thuoc driver pin**.
Con so "13.7k vs 39.3k" ban dau la nhieu do (do ngay sau `rmmod`, trung luc may lang).

**Ket luan: driver pin vo can. Duong `mt6358_read_imp()` cung vo can.**
Con so dao dong nhu vay dung voi kieu bao ngat "duong ngat bi keo lien tuc, toc do
chi phu thuoc CPU dang chay o tan so nao" (moi vong lap ~25–70us).

---

## 2. Do dac tren may that

### 2.1. Doc thanh ghi PMIC qua regmap debugfs

`/sys/kernel/debug/regmap/1000d000.pwrap/registers` (can `sudo`).

Ban dau tuong day khong phai khong gian thanh ghi MT6358, nhung **no dung**:

```
0000: 2800
0002: 1000
0004: 0000
0006: 0000
0008: 5820      <- HWCID  = 0x5820
000a: 5820      <- SWCID  = 0x5820  => chip_id 0x58 = MT6358
```

`mt6397-core.c` cua mainline doc dung `MT6358_SWCID` (0x0a) de nhan dien chip, va ra 0x58.
Regmap cua pwrap chinh la regmap cua PMIC (pwrap chi la bus; mfd `mt6397-core` lay regmap
cua device cha). `pwrap_regmap_config16`: `reg_bits=16, val_bits=16, reg_stride=2,
max_register=0xffff`, khong co cache — nen moi so doc ra deu la doc that tu PMIC.

### 2.2. Ket qua then chot (doc bang regmap debugfs, sau do xac nhan bang module chi-doc)

| Dia chi | Ten | Gia tri | Y nghia |
|---|---|---|---|
| 0x0008 / 0x000a | `HWCID` / `SWCID` | `0x5820` | xac nhan dung MT6358, dung khong gian thanh ghi |
| 0x0910 | `PSC_TOP_INT_CON0` | `0x000f` | 4 bit pwrkey/homekey press+release dang bat — dung voi `mtk-pmic-keys` |
| 0x052e | `SCK_TOP_INT_CON0` | `0x0001` | RTC dang bat — dung voi `mt6397-rtc` |
| **0x0198** | **`TOP_INT_MASK_CON0`** | **`0x0040`** | **bit 6 dang bi che — bit 6 chinh la XPP (xem muc 3)** |
| **0x019e** | **`TOP_INT_STATUS0`** | **`0x0000`** | khong nhom ngat nao pending |
| 0x01a0 | `TOP_INT_RAW_STATUS0` | `0x0000` | ke ca truoc khi che |
| 0x01a2 | `TOP_INT_CON0` = `RG_INT_POLARITY` (bit0) | `0x0000` | cuc tinh dau ra ngat cua PMIC |
| 0x0428 | `INT_TYPE_CON0` | `0x0000` | |
| **0x042e** | **`INT_STA`** (bit0 = `CPU_INT_STA`) | **`0x0000`** | **PMIC tu khai bao: khong he keo duong ngat ra CPU** |
| moi `*_INT_STATUS*` / `*_INT_RAW_STATUS*` cua ca 8 nhom | | `0x0000` | |

Va 32 lan doc lien tiep (module chan doan, doc lien tuc khong nghi):

```
[ 0] TOP_STA=0x0000 TOP_RAW=0x0000 INT_STA=0x0000
[ 1] ... [31] deu y het
```

Khong mot bit nao chop. **PMIC HOAN TOAN VO CAN — da chung minh, khong con la phong doan.**

Hai he qua phu rat co gia tri:

1. `TOP_INT_MASK_CON0 = 0x0040` (bit 6 bi che) la **xac nhan doc lap tu phan cung** rang
   bit 6 la mot nhom co that (XPP) va rang quy uoc `1 = bi che` la dung — nen ca hai gia
   dinh trong patch o muc 4 deu duoc phan cung xac nhan.
2. `INT_STA = 0x0000` khep lai huong dieu tra phia PMIC.

### 2.3. Vay chan EINT 144 dang o trang thai nao

Doc thanh ghi khoi EINT cua SoC tai `0x1000b000`, port 4 (EINT 144 = port 4, bit 16):

```
EINT_STA         port4 = 0x00000000 (bit16=0)   <- lan doc dau
EINT_MASK        port4 = 0xffffffff (bit16=1)
EINT_SENS        port4 = 0xffffffff (bit16=1)   <- level
EINT_SOFT        port4 = 0x00000000 (bit16=0)   <- KHONG phai ngat mem
EINT_POL         port4 = 0x00010000 (bit16=1)   <- rieng EINT144 active-high
EINT_DOM_EN      port4 = 0xffffffff (bit16=1)
EINT_DBNC_CTRL   port4 = 0x00000000

EINT_STA burst: 0x00000000, roi 0x00010000 x7 lien tuc
```

- `EINT_SOFT bit16 = 0` -> **loai tru** gia thuyet bootloader de lai ngat mem.
- `EINT_STA bit16` bat lai ngay sau khi handler ack va giu nguyen -> dung chan dung cua
  bao ngat muc (level) tu duy tri.
- `EINT_MASK = 0xffffffff` KHONG mau thuan: IRQ dang chay `IRQF_ONESHOT`, nen phan lon
  thoi gian EINT 144 bi che trong luc handler/thread chay. Module doc trung luc do.
  Dia chi dung: `mtk_generic_eint_regs` (`drivers/pinctrl/mediatek/mtk-eint.c:33`) ghi
  `.mask = 0x080`, `.mask_set = 0x0c0`, `.mask_clr = 0x100`.

### 2.4. Cuc tinh: bac bo o phia DTS, nhung mo ra huong o phia PMIC

Gia thuyet dau: DTS khai sai, phai la `IRQ_TYPE_LEVEL_LOW`.

**Bac bo o phia DTS** bang chinh mainline — moi board dung MT6358/MT6366 deu khai LEVEL_HIGH:

- `mt8183-evb.dts:379` — `interrupts-extended = <&pio 182 IRQ_TYPE_LEVEL_HIGH>` (MT6358)
- `mt8186-corsola.dtsi:1281` — `<&pio 201 IRQ_TYPE_LEVEL_HIGH>` (MT6366, cung ho)
- downstream Xiaomi cho chinh may nay — `<144 IRQ_TYPE_LEVEL_HIGH 144 0>`

Chan INT cua MT6358 tich cuc muc CAO theo quy uoc chung. `EINT_POL bit16 = 1` la dung,
khong nen doi DTS sang LEVEL_LOW.

**Nhung cuc tinh do la CAU HINH DUOC o phia PMIC** — da tim ra thanh ghi:

```
/* .../mt-plat/mt6768/include/mach/upmu_hw.h:3391 */
#define PMIC_RG_INT_POLARITY_ADDR  MT6358_TOP_INT_CON0     /* = 0x1a2 */
#define PMIC_RG_INT_POLARITY_MASK  0x1
#define PMIC_RG_INT_POLARITY_SHIFT 0
```

Tuc `TOP_INT_CON0 @0x1a2` (do duoc `0x0000`) chinh la bit cuc tinh dau ra ngat cua PMIC,
va **no dang bang 0**.

Neu 0 nghia la "dau ra tich cuc muc THAP", thi chan INT cua PMIC ranh o muc CAO, ma SoC lai
dat LEVEL_HIGH => bao ngat vinh vien voi moi thanh ghi trang thai cua PMIC deu bang 0.
Khop chinh xac voi toan bo so lieu do duoc. **Day la gia thuyet dan dau hien nay.**

Da kiem tra: **khong mot driver nao trong downstream ghi vao thanh ghi nay** (chi co bang
thanh ghi codegen va muc debugfs). Nen gia tri 0 hoac la mac dinh khi reset, hoac do
preloader/LK dat. Neu la truong hop sau thi duong khoi dong khac nhau (Android LK so voi
u-boot/mainline) co the de PMIC o hai trang thai khac nhau — dieu do giai thich vi sao
downstream khong gap loi nay ma mainline gap.

### 2.5. Cau hoi con lai va phep do dang cho

Con dung mot cau hoi: **chan do that su o muc dien nao, va dao `RG_INT_POLARITY` co chua duoc khong?**

Khong doc duoc truc tiep: EINT 144 = GPIO180, ma cac bang thanh ghi trong
`pinctrl-mt6768.c` (`mt6768_pin_di_range` v.v.) chi phu den chan 179 — chan 180-185 la
chan chuyen dung, khong co thanh ghi DIN.

Nen phai do gian tiep, bang module `mt6358_eintpol.ko` (muc 6.2), hai phep thu:

1. **Dao cuc tinh phia SoC** (`mode=1`): dat rieng EINT 144 thanh active-low.
   - `EINT_STA bit16` sach va so ngat 172 dung yen => chan THAT SU o muc cao.
   - STA van bat va van bao ngat => chan o muc thap, tuc bit STA / khoi EINT bi ket,
     loi nam trong pinctrl/EINT.
2. **Dao cuc tinh phia PMIC** (`mode=3`): ghi `RG_INT_POLARITY = 1` (0x1a2 bit0).
   - So ngat 172 dung yen => **da tim ra cach chua**: driver chi can ghi
     `RG_INT_POLARITY = 1` luc khoi tao. Se la patch thu 3 va la patch that su chua benh.

### 2.6. Da loai tru phia SoC bang doc code

Doi chieu mainline voi downstream MTK (`~/ksrc/android_kernel_xiaomi_mt6768-.../`):

| Muc | Mainline | Downstream | Ket qua |
|---|---|---|---|
| So EINT cua PMIC | `&pio 144 IRQ_TYPE_LEVEL_HIGH` (`mt6769t-xiaomi-common.dtsi:104`) | `interrupts = <144 IRQ_TYPE_LEVEL_HIGH 144 0>` (`mt6768.dts:645`) | GIONG |
| Dia chi khoi EINT | `<0 0x1000b000 0 0x1000>` reg-name `eint` | `eint: apirq@1000b000` | GIONG |
| `mtk_eint_hw` | `port_mask=0x7, ports=6, ap_num=212, db_cnt=13` | y het | GIONG |
| EINT 144 thuoc chan nao | `GPIO180`, `MTK_EINT_FUNCTION(0, 144)` | y het | GIONG |

Nen **khong phai loi DTS/pinctrl hien nhien**.

Mot diem dang nghi con lai o phia SoC: `mtk_eint_hw_init()` (mainline
`drivers/pinctrl/mediatek/mtk-eint.c:313`) chi ghi `dom_en = 0xffffffff` va
`mask_set = 0xffffffff`. No **khong xoa `EINT_SOFT`** va **khong xoa STA dang pending**.
Neu preloader/LK de lai bit `EINT_SOFT` cua EINT 144, thi ngay khi driver unmask,
duong ngat se bi keo mai mai. Downstream cung khong xoa — nhung downstream co the khong
gap vi luong khoi dong khac. Module chan doan doc `EINT_SOFT` de kiem chung diem nay.

---

## 3. Loi THAT cua mainline da tim ra (doc lap voi bao ngat nay)

MT6358 gom cac nguon ngat thanh 9 nhom cap cao, moi nhom 1 bit trong `TOP_INT_STATUS0`.
Theo `upmu_hw.h` cua downstream (`.../mt-plat/mt6768/include/mach/upmu_hw.h:3333-3357`):

```
BUCK=0  LDO=1  PSC=2  SCK=3  BM=4  HK=5  XPP=6  AUD=7  MISC=8
(bit 9..15 = TOP_RSV, du phong)
```

Nhung `include/linux/mfd/mt6358/core.h` cua mainline **thieu XPP**:

```c
enum mt6358_irq_top_status_shift {
	MT6358_BUCK_TOP = 0,
	...
	MT6358_HK_TOP,
	MT6358_AUD_TOP,    /* = 6, phan cung la 7 */
	MT6358_MISC_TOP,   /* = 7, phan cung la 8 */
};
```

Hai PMIC anh em trong CHINH mainline deu lam dung:

- `include/linux/mfd/mt6357/core.h`: co `MT6357_XPP_TOP` giua HK va AUD.
- `include/linux/mfd/mt6359/core.h`: ghi ro `MT6359_AUD_TOP = 7,` de nhay qua bit 6.

Chi rieng MT6358 sai. Hau qua trong `mt6358_irq_handler()`:

- Nhom AUD that (bit 7) noi len -> driver tuong la MISC -> doc/clear nham
  `MISC_TOP_INT_STATUS0` (0x194) thay vi `AUD_TOP_INT_STATUS0` (0x2234). Doc ra 0 ->
  `continue` -> **khong clear gi** -> IRQ cha level -> bao ngat vinh vien.
- Nhom XPP (bit 6) noi len -> driver tuong la AUD -> doc nham -> nhu tren.
- Nhom MISC that (bit 8, `SPI_CMD_ALERT`) **khong bao gio duoc nhin toi**.

Day la loi that, can sua du no co phai nguyen nhan cua bao ngat nay hay khong.

### Da tim tren Internet — chua ai bao cao

Da tim: linux-mediatek mailing list / patchwork / lore, postmarketOS.
Cac patch lien quan chi co:
- `mfd: Add support for the MediaTek MT6358 PMIC` (Hsin-Hsiung Wang, 2019) — chinh la
  cho enum sai duoc dua vao.
- `[v4,1/9] mfd: mt6358: refine interrupt code`
  (https://patchwork.kernel.org/project/linux-mediatek/patch/1608104827-7937-2-git-send-email-hsin-hsiung.wang@mediatek.com/)

**Khong tim thay bao cao nao ve "mt6358 interrupt storm" / "mt6358-irq flood".**
=> Chua co patch san, phai tu soan.

---

## 4. Da sua gi

3 thay doi trong `~/linux-618` (KHONG dung vao `mt6359-auxadc.c` hay
`generic-adc-battery.c` — giu nguyen 3 fix cua agent pin):

### (a) `include/linux/mfd/mt6358/core.h` — sua loi lech bit

Them `MT6358_XPP_TOP,` giua `MT6358_HK_TOP` va `MT6358_AUD_TOP`.

### (b) `drivers/mfd/mt6358-irq.c` — che cac nhom cap cao khong phuc vu duoc

Uu tien (a) trong yeu cau goc: che nhung gi khong co handler.

- Luc probe: tinh mask cac bit trong `TOP_INT_STATUS0` khong ung voi nhom nao trong
  `pmic_ints[]` (voi MT6358 sau khi sua: bit 6 XPP + bit 9..15 du phong), roi ghi vao
  `TOP_INT_MASK_CON0_SET` (0x19a — cung offset o ca MT6357/MT6358/MT6359, da doi chieu
  downstream). Dung thanh ghi `_SET` nen khong can doc-sua-ghi.
- Trong handler: neu sau vong lap van con bit thua, `dev_warn_ratelimited()` roi che luon.
  Dong log nay cung la cong cu chan doan: neu bao ngat con ma **khong** co dong
  `Masking unserviced top level IRQ groups` thi PMIC chac chan vo can.

### (c) `drivers/mfd/mt6358-irq.c` — tat nguon ngat khong ai nhan

Trong `mt6358_irq_sp_handler()`, khi `irq_find_mapping()` tra ve 0 (khong driver nao
dang ky), truoc day chi ghi W1C ack. Neu dieu kien nguon van con dung (nguon kieu muc,
vi du CHRDET) thi ack khong co tac dung lau dai -> bao ngat. Nay them: tat luon bit do
trong `en_reg`.

An toan ve cache: `virq == 0` nghia la khong ai map -> `enable_hwirq[]`/`cache_hwirq[]`
deu `false` -> ghi 0 vao phan cung khong lam lech cache. Chi co tac dung that khi bit do
bi bootloader bat san.

`include/linux/mfd/mt6358/registers.h` va `mt6359/registers.h`: them
`TOP_INT_MASK_CON0_SET = 0x19a`. `mt6357/registers.h` da co san.

### Diff

```
 drivers/mfd/mt6358-irq.c             | 49 ++++++++++++++++++++++++++++++++++--
 include/linux/mfd/mt6358/core.h      |  2 ++
 include/linux/mfd/mt6358/registers.h |  1 +
 include/linux/mfd/mt6359/registers.h |  1 +
```

Ban day du: `patches-pmic-irq/pmic-irq-storm.diff`

---

## 5. Patch gui upstream

Da soan 2 patch rieng, da kiem tra `patch -p1 --dry-run` ap sach len ban goc:

- `patches-pmic-irq/0001-mfd-mt6358-fix-top-level-irq-offsets.patch`
  `mfd: mt6358: Fix the AUD and MISC top level interrupt offsets`
  (co `Fixes:` tro ve commit them MT6358 PMIC)
- `patches-pmic-irq/0002-mfd-mt6358-no-unserviced-irq-storm.patch`
  `mfd: mt6358: Do not let unserviced interrupts storm`

Nguoi nhan: `linux-mediatek@lists.infradead.org`, maintainer MFD (Lee Jones).

---

## 6. Module chan doan

Hai module chi dung de do, khong phai san pham. Ca hai cung vermagic
`6.18.0-gc75a6bdd2a48-dirty`, ca hai deu **tu go** (`module_init` tra ve `-EINVAL`, nen
`insmod` bao "Invalid argument" — do la co y, khong can `rmmod`).

### 6.1. `pmos/mt6358_irqdbg.ko` — CHI DOC, da chay xong

```
sudo insmod ./mt6358_irqdbg.ko
sudo dmesg | grep irqdbg
```

In: toan bo thanh ghi ngat cua MT6358 (ke ca `0x1a2`, `0x428`, `0x42e` truoc do chua ai
doc), 32 lan doc lien tiep `TOP_INT_STATUS0`/`TOP_INT_RAW_STATUS0`/`INT_STA`, va thanh ghi
EINT cua SoC cho EINT 144. Ket qua o muc 2.2 va 2.3.

Nguon: `~/pmicdbg/mt6358_irqdbg.c` tren `ctdagent`.

### 6.2. `pmos/mt6358_eintpol.ko` — do muc dien cua chan (dang cho chay)

Co tham so `mode`:

| mode | Lam gi |
|---|---|
| 0 (mac dinh) | chi doc: STA/MASK/SENS/SOFT/POL/DOM_EN cua CA 7 port + 16 lan lay mau STA cach nhau 1ms |
| 1 | dat EINT 144 thanh **active-low** phia SoC (`pol_clr`) roi `ack`, giu nguyen de do |
| 2 | tra EINT 144 ve **active-high** phia SoC (`pol_set`) roi `ack` |
| 3 | ghi **`RG_INT_POLARITY = 1`** phia PMIC (`0x1a2` bit0) roi `ack` |
| 4 | tra `RG_INT_POLARITY` ve 0 |

Quy trinh do:

```
sudo dmesg -C
sudo insmod ./mt6358_eintpol.ko                 # B1: chi doc, ca 7 port

grep "^ *172:" /proc/interrupts                 # B2
sudo insmod ./mt6358_eintpol.ko mode=1
sleep 3; grep "^ *172:" /proc/interrupts
sudo insmod ./mt6358_eintpol.ko mode=2          # tra lai

grep "^ *172:" /proc/interrupts                 # B3
sudo insmod ./mt6358_eintpol.ko mode=3
sleep 3; grep "^ *172:" /proc/interrupts
sudo insmod ./mt6358_eintpol.ko mode=4          # tra lai

sudo dmesg | grep eintpol
```

`mode=1/2` chi ghi 1 bit POL cua rieng EINT 144. `mode=3/4` chi ghi 1 bit trong PMIC.
Ca hai dao nguoc duoc, khoi dong lai cung tra ve nguyen trang. Khong flash.

Rui ro duy nhat: trong luc thu, phim nguon co the tam khong nhan. `mode=2`/`mode=4` hoac
reboot la het.

Nguon: `~/pmicdbg/mt6358_eintpol.c` tren `ctdagent`.

---

## 7. Artefact da build

Build sach, khong loi (`~/irqbuild.log`).

| File | Ghi chu |
|---|---|
| `pmos/ml618-Image-irq` | Image co 3 sua o muc 4 |
| `pmos/ml618-lancelot-irq.dtb` | tu `mt6769t-xiaomi-lancelot-tianma-ktd.dts` |
| `pmos/lancelot-modules-irq.tar.gz` | goc `usr/lib/modules/6.18.0-gc75a6bdd2a48-dirty`, 60 module, co `generic-adc-battery.ko` |
| `pmos/mt6358_irqdbg.ko` | module do, chi doc |
| `pmos/mt6358_eintpol.ko` | module do cuc tinh EINT |

Cac thay doi cua agent khac trong `~/linux-618` (`mt6359-auxadc.c`,
`generic-adc-battery.c`, DTS) **duoc giu nguyen**, khong hoan tac.
Khong sua gi trong `~/ksrc/*` hay `~/rom/*`.

**Chua flash `ml618-Image-irq`** — thong nhat voi agent chinh: so lieu da chung minh PMIC
vo can nen flash luc nay chi ton mot chu ky va lam mat mach do GPU dang chay tren may.

---

## 8. Danh gia trung thuc theo tieu chi thanh cong

Tieu chi: sau khi sua, IRQ 172 phai giam ve vai chuc/vai tram ngat moi giay,
ke ca khi da nap module pin.

**CHUA DAT.** Noi thang:

1. Ba sua o muc 4 sua mot **loi that** cua mainline (da duoc phan cung xac nhan qua
   `TOP_INT_MASK_CON0 = 0x0040`), nhung do dac chung minh **loi do khong phai nguyen nhan
   cua bao ngat tren may nay**. PMIC hoan toan khong keo duong ngat.
2. Nguyen nhan that nam o phia SoC: chan EINT 144 lien tuc thoa dieu kien muc tich cuc.
   Chua biet vi sao. Chua co phep do cuoi cung (muc 6.2) nen chua the ket luan.
3. Do do bao ngat **van con** tren may. Chua co ban sua nao chua duoc trieu chung.

Ba huong dieu tra da bi loai tru bang bang chung, khong phai bang suy doan:
- driver pin / `mt6358_read_imp()` — loai (do lai, khong phu thuoc)
- nguon ngat PMIC pending khong co handler — loai (`TOP_INT_STATUS0`/`RAW`/`INT_STA` = 0)
- ngat mem `EINT_SOFT` do bootloader de lai — loai (`EINT_SOFT bit16 = 0`)
- lech cuc tinh DTS — loai (moi board MT6358 trong mainline deu LEVEL_HIGH)
- sai so EINT / sai dia chi khoi EINT / sai thong so `mtk_eint_hw` — loai (giong downstream)

Buoc tiep theo, theo thu tu:
1. Chay `mt6358_eintpol.ko mode=0` roi `mode=1` (muc 6.2). Day la phep do quyet dinh.
2. Neu chan that su o muc cao: tim vi sao PMIC (hoac mach) keo chan len trong khi
   `INT_STA = 0`. Huong nghi tiep: `RG_INT_POLARITY` (0x1a2 bit0) va `INT_TYPE_CON0`
   (0x428) — so sanh gia tri tren may voi gia tri sau khi Android/downstream khoi dong,
   vi preloader/LK co the dat khac.
3. Neu chan o muc thap ma STA van ket: loi trong khoi EINT / `pinctrl-mt6768`.
   Doc them `EINT_STA` cua ca 7 port de xem con EINT nao khac cung ket
   (`mode=0` da in san).
4. Rieng 2 patch o muc 5: van dung, van nen gui upstream, doc lap voi ket qua tren.
