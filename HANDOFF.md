# HANDOFF — продолжение порта postmarketOS на Xiaomi Redmi 9 (lancelot)

> Вставь этот файл целиком в новый чат, чтобы продолжить работу.

---

## 1. Что это за проект

Порт **postmarketOS** на **Xiaomi Redmi 9** (codename `lancelot`, SoC MediaTek
Helio G80 / **MT6769T**). ВАЖНО: аппарат — ревизия **Huaxing / Focaltech FT8719**
(НЕ Tianma/Novatek, под которую рассчитан стоковый pmOS-порт `device-xiaomi-lancelot`).

**Рабочее состояние (проверено на железе, сентябрь 2026):**
- ✅ Загрузка (mainline ядро 6.16.0, пакет `linux-postmarketos-mediatek-mt6768`)
- ✅ Дисплей 1080×2340 (dtb `huaxing-ktd` + флаги cmdline)
- ✅ Тач Focaltech FT8719, 10 пальцев (портированный драйвер)
- ✅ GPU panfrost (Mali-G52), Plasma Mobile (systemd), SSH по USB (172.16.42.1)
- ✅ eMMC (4 ГБ RAM, карвеауты LK в порядке)

**НЕ работает:** батарея/зарядка (SMB1351), Wi-Fi/BT (gen4m), аудио (mt6358).

---

## 2. Где что лежит

Каталог проекта: **`/home/maxi/Документи/Default Project/`**

```
├── lancelot-pmos-guide.md          ← ПОЛНЫЙ гайд (переписан, актуальный)
├── README.md                       ← обзор репозитория + лицензии
├── .gitignore
├── lancelot/                       ← артефакты
│   ├── 0001-ft8719-touch.patch     ← ЕДИНЫЙ патч ядра (тач + backlight + PMIC-фикс)
│   ├── focaltech_ft8719.c          ← исходник портированного тач-драйвера
│   ├── config.fragment             ← 3 строки конфига ядра
│   ├── deviceinfo.patch            ← правка dtb в device-пакете
│   ├── hataketsu_patches/          ← референс-патчи hataketsu (battery/pmic-irq)
│   │   ├── 0001-mfd-mt6358-fix-top-level-irq-offsets.patch
│   │   ├── 0002-mfd-mt6358-no-unserviced-irq-storm.patch
│   │   └── (скачай pmic-irq-storm.diff — см. §6)
│   ├── ft8719_downstream/          ← downstream-драйвер MTK FT8719P (референс)
│   ├── extracted/firmware/         ← блобы (НЕ в git): тач + Wi-Fi прошивки
│   ├── flash/boot_touch.img        ← текущий рабочий boot.img (r15)
│   ├── tools/                      ← mkbootimg + unpack_bootimg
│   └── avbtool.py                  ← AVB-инструмент (AOSP)
├── linux-mt6768/                   ← КЛОН ядра mt6768-mainline/linux @ 781287a2
└── pmbootstrap/                    ← КЛОН pmbootstrap (v3.11.1)
```

**pmbootstrap work dir:** `/home/maxi/.local/var/pmbootstrap/`
**pmaports (ядерный пакет):**
`/home/maxi/.local/var/pmbootstrap/cache_git/pmaports/device/testing/linux-postmarketos-mediatek-mt6768/`

**GitHub:** https://github.com/MxMaxo/lancelot-pmos (ветка `main`, 2 коммита).
В git — только код (патч, драйвер, гайд, README, config.fragment, deviceinfo.patch).
Блобы и крупные файлы в `.gitignore`.

---

## 3. Ключевые факты о железе и решении

| Факт | Деталь |
|---|---|
| Ревизия панели/тача | Huaxing / FT8719 (определяется `adb shell getprop ro.boot.lcm` → `LCM_name=...huaxing...`) |
| Дисплей | dtb `huaxing-ktd` + cmdline `clk_ignore_unused pd_ignore_unused` |
| Тач | SPI0, irq = pio 1 (falling), reset = pio 92 |
| Прошивка тача | `focaltech_ts_fw_huaxing.bin` (131 КБ) в `/lib/firmware` |
| rootfs | на `super` (eMMC), НЕ на SD |
| Разделы в pmOS | eMMC = **mmcblk1** (не mmcblk0!), `boot`=mmcblk1p34, `super`=mmcblk1p44 |

---

## 4. Как всё собиралось (кратко, детали в гайде)

1. **pmbootstrap** (из git, v3.11.1+), `init`: edge, xiaomi, lancelot, plasma-mobile, systemd.
2. **dtb**: в `device-xiaomi-lancelot/deviceinfo` → `deviceinfo_dtb="...huaxing-ktd"`.
3. **Конфиг ядра** (`config-postmarketos-mediatek-mt6768.aarch64`):
   ```
   CONFIG_TOUCHSCREEN_FOCALTECH_FT8719=y
   CONFIG_INPUT_EVDEV=y
   CONFIG_MODULE_ALLOW_BTF_MISMATCH=y
   ```
4. **Патч ядра** `0001-ft8719-touch.patch` (тач + backlight + PMIC-фикс) в APKBUILD
   ядерного пакета (`source=` + sha512sum + pkgrel).
5. **Сборка**: `pmbootstrap build linux-postmarketos-mediatek-mt6768`, потом `install --password pmos`.

---

## 5. Как пересобрать и прошить boot.img (ВАЖНО)

Извлекаешь `vmlinuz` + dtb из собранного пакета (`packages/edge/aarch64/...rN.apk`),
пересобираешь boot.img со СТАРЫМ ramdisk и СТАРЫМ cmdline (чтобы не менять UUID rootfs):

```bash
cd ~/Документи/Default\ Project/lancelot
# распаковать пакет: tar -xzf .../rN.apk -C kernel_rN
# ramdisk из старого boot.img: tools/usr/bin/unpack_bootimg --boot_img flash/boot_touch.img --out /tmp/bt

NEW_CMDLINE="bootopt=64S3,32N2,64N2 buildvariant=user clk_ignore_unused pd_ignore_unused loglevel=8 ignore_loglevel pmos_boot_uuid=ce0b898c-71a0-480d-88d2-cb3510f2d99a pmos_root_uuid=e3304883-9097-45f4-8066-edebfc458b1b pmos_rootfsopts=defaults"

python3 tools/usr/share/android-tools/mkbootimg/mkbootimg.py \
  --kernel kernel_rN/boot/vmlinuz --ramdisk /tmp/bt/ramdisk \
  --dtb kernel_rN/boot/dtbs/mediatek/mt6769t-xiaomi-lancelot-huaxing-ktd.dtb \
  --cmdline "$NEW_CMDLINE" --header_version 2 --pagesize 2048 \
  --base 0x40078000 --kernel_offset 0x00008000 --ramdisk_offset 0x07c08000 \
  --second_offset 0xbff88000 --tags_offset 0x0bc08000 --dtb_offset 0x0bc08000 \
  -o flash/boot_touch.img

python3 avbtool.py add_hash_footer --image flash/boot_touch.img \
  --partition_name boot --partition_size 67108864 --algorithm NONE
```

**Прошивка (ОБЯЗАТЕЛЬНО через oflag=direct — иначе page-cache не сбросится!):**

```bash
SSH="ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i ~/.ssh/phone maxi@172.16.42.1"
scp -i ~/.ssh/phone flash/boot_touch.img maxi@172.16.42.1:/tmp/
$SSH 'sudo dd if=/tmp/boot_touch.img of=/dev/mmcblk1p34 bs=4096 oflag=direct conv=fsync; sudo systemctl reboot'
```

Проверка записи (минуя кеш): `$SSH 'sudo dd if=/dev/mmcblk1p34 iflag=direct bs=4096 count=16384 2>/dev/null | md5sum'`

SSH-ключ: `~/.ssh/phone` (ed25519), логин `maxi`, пароль `pmos`, passwordless sudo.

---

## 6. ⚠️ ТЕКУЩИЙ БЛОКЕР — PMIC IRQ-шторм (НЕ РЕШЁН)

**Симптом:** `mt6358-irq` (IRQ 172 / EINT 144) генерирует ~25 000 прерываний/сек
(`cat /proc/interrupts | grep mt6358-irq` → счётчик растёт на ~25K/с). Это даёт
14–22% CPU и вызывает **случайные зависания/перезагрузки**.

**Что уже сделано (в патче r15):** применил оба патча hataketsu
(`hataketsu/mt6768-mainline-notes` → `kernel-patches/pmic-irq/`):
- `0001-mfd-mt6358-fix-top-level-irq-offsets.patch` (offset fix — применяется чисто)
- `0002-mfd-mt6358-no-unserviced-irq-storm.patch` (маскировка unserviced групп —
  пришлось **вручную адаптировать** хунк `mt6358_irq_init` под 6.16:
  `irq_domain_create_linear(of_fwnode_handle(chip->dev->of_node), ...)` вместо
  `dev_fwnode(chip->dev)` из 6.18).

**НО шторм ВСЁ ЕЩЁ ЕСТЬ (~26K/с после r15).** Причина не устранена. По записям
hataketsu: «найден и исправлен реальный баг mainline — но он НЕ причина шторма».
В его репозитории есть ТРЕТИЙ файл **`pmic-irq-storm.diff`** (5988 байт) и док
**`docs/PMIC-IRQ-STORM.vi.md`** (вьетнамский) — это, вероятно, ключ к настоящей
причине. Скачай их:
```
https://raw.githubusercontent.com/hataketsu/mt6768-mainline-notes/master/kernel-patches/pmic-irq/pmic-irq-storm.diff
https://raw.githubusercontent.com/hataketsu/mt6768-mainline-notes/master/docs/PMIC-IRQ-STORM.vi.md
```
Изучи `pmic-irq-storm.diff` и проверь, маскируются ли/чистятся ли нужные биты.
Гипотеза: какой-то источник (например, от загрузчика оставлен включённым) держит
уровень, и его надо маскировать в TOP_INT_MASK или отключить в en_reg конкретной группы.

---

## 7. Второй баг (подсветка) — исправлен, проверь

В `huaxing.dtsi` panel-нода не имела `backlight = <&backlight>;`. Добавил строку
(есть в r15). Проверь: при выключении экрана подсветка должна гаснуть
(`/sys/class/backlight/ktd3137-backlight/brightness` → 0).

---

## 8. Следующие шаги (приоритет)

1. **Добить PMIC IRQ-шторм** (главное — иначе случайные ребуты) — см. §6.
2. **Батарея/зарядка** — патчи `hataketsu/mt6768-mainline-notes` →
   `kernel-patches/battery/*.patch` + драйвер SMB1351 (его нет в mainline).
3. **Wi-Fi/BT** — порт gen4m (`hataketsu/redmi9-lancelot-mainline`); прошивки уже
   извлечены в `lancelot/extracted/firmware/` (WIFI_RAM_CODE_soc1_0_1a_1.bin и др.).
4. **Аудио** — mt6358 DAI-обвязка.

---

## 9. Все грабли (кратко)

1. AVB-footer обязателен (`--algorithm NONE`), иначе `lk_crash`.
2. `clk_ignore_unused pd_ignore_unused` в cmdline — без них дисплей не работает.
3. systemd (не OpenRC) — plasma-mobile в новых pmaports только systemd.
4. `CONFIG_INPUT_EVDEV=y` — иначе нет /dev/input/eventX (evdev=m падает из-за BTF).
5. `CONFIG_MODULE_ALLOW_BTF_MISMATCH=y` — иначе модули падают `BTF: -22`.
6. **`dd` в eMMC — `oflag=direct`**, иначе page-cache не сбрасывается до reboot.
7. Нумерация разделов: в pmOS eMMC = mmcblk1 (не mmcblk0).
8. Прошивка тача запрашивается ДО монтирования rootfs → нужен ретрай в драйвере
   (уже есть) + файл в /lib/firmware.
9. CRC в SPI FT8719 — только когда послан command-package; при чтении touch-данных
   (cmd=NULL) CRC нет.
10. rootfs на super: прошивать только p2 (корневой ext4), `/boot` в fstab — nofail.
11. `pmbootstrap install` меняет UUID rootfs → пересобирай только boot.img со старым cmdline.

---

## 10. Контекст для нового чата (вставь это)

```
Продолжаем порт postmarketOS на Xiaomi Redmi 9 (lancelot, MT6769T, ревизия
Huaxing/Focaltech FT8719). Работает: загрузка, дисплей 1080x2340, тач, GPU,
Plasma Mobile, SSH 172.16.42.1 (логин maxi, ключ ~/.ssh/phone, sudo без пароля).
Ядро mainline 6.16 (linux-postmarketos-mediatek-mt6768, коммит 781287a2).
ГЛАВНЫЙ НЕРЕШЁННЫЙ БАГ: PMIC IRQ-шторм (~25K IRQ/с от mt6358-irq) → случайные
зависания/ребуты. Применён патч hataketsu 0001+0002, но шторм остался — см.
файл HANDOFF.md §6. Второй баг (подсветка не гаснет) исправлен добавлением
backlight = <&backlight> в huaxing.dtsi — проверить.
Проект: ~/Документи/Default Project/ (гайд lancelot-pmos-guide.md, патч
lancelot/0001-ft8719-touch.patch). GitHub: github.com/MxMaxo/lancelot-pmos.
Прочитай HANDOFF.md и продолжи.
```
