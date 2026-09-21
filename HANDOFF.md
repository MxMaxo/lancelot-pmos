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

⚠️ **СНАЧАЛА определи, как зовётся eMMC!** Нумерация НЕСТАБИЛЬНА: в прошлой
сессии eMMC был `mmcblk1`, сейчас `mmcblk0`. Перед прошивкой проверь:
```bash
$SSH 'lsblk | grep -E "mmcblk[01] " ; ls -la /dev/mmcblk1* 2>/dev/null'
```
Boot-раздел — это тот, где rootfs `...p44` смонтирован как `/` (см. `mount`).
`boot` = тот же mmcblkX, раздел **p34** (64 МБ). Если rootfs на `mmcblk0p44`,
то boot = **mmcblk0p34** (НЕ слепо mmcblk1p34!).

```bash
SSH="ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i ~/.ssh/phone maxi@172.16.42.1"
scp -i ~/.ssh/phone flash/boot_touch.img maxi@172.16.42.1:/tmp/
# BOOT=... определи как выше (пример для mmcblk0):
$SSH 'sudo dd if=/tmp/boot_touch.img of=/dev/mmcblk0p34 bs=4096 oflag=direct conv=fsync; sudo systemctl reboot'
```

Проверка записи (минуя кеш): `$SSH 'sudo dd if=/dev/mmcblk0p34 iflag=direct bs=4096 count=16384 2>/dev/null | md5sum'`
Сравни с локальным `md5sum flash/boot_touch.img`.

SSH-ключ: `~/.ssh/phone` (ed25519), логин `maxi`, пароль `pmos`, passwordless sudo.

---

## 6. PMIC IRQ-шторм — ПРИЧИНА НАЙДЕНА, но фикс вскрывает зависание (см. §6b)

**Симптом:** `mt6358-irq` (IRQ 172 / EINT 144) генерирует ~25–39K прерываний/сек
→ 14–22% CPU.

**Разбор hataketsu (файлы в `lancelot/hataketsu_patches/`):**
- `pmic-irq-storm.diff` = 0001+0002 вместе (XPP-offset fix + маскировка). Уже было.
- `PMIC-IRQ-STORM.vi.md` (вьетнамский) — там настоящее расследование. Ключевые
  факты, подтверждённые им И мной на железе:
  1. Реальный баг mainline: в `mt6358/core.h` пропущена группа `XPP` (bit 6) →
     AUD/MISC сдвинуты на 1 бит. Это НЕ причина шторма, но баг настоящий
     (подтверждено железом: `TOP_INT_MASK_CON0 = 0x0040`). Оставлен в патче.
  2. **PMIC полностью невиновен**: все `TOP_INT_STATUS0/RAW/INT_STA = 0x0000`.
  3. Настоящая причина — **полярность линии EINT144**. Линия физически ПОКОИТСЯ
     В HIGH (PMIC работает active-low / open-drain), а DTS задавал
     `IRQ_TYPE_LEVEL_HIGH`. SoC видел HIGH = «активно» → вечный шторм.

**ЧТО ПРОВЕРЕНО (мои измерения на железе):**
- Гипотеза «RG_INT_POLARITY (0x1a2 bit0)» — **ОПРОВЕРГНУТА**. Записал 0→1
  (r17), шторм остался ~27K/с. Бит пишется, но уровень линии не меняет.
- Решающий тест: переключил EINT144 в active-low (`pol_clr`, POL bit16=0) прямо
  в рантайме через /dev/mem → **шторм упал в 0/с мгновенно**. Вернул active-high
  → шторм вернулся. Однозначно: линия в HIGH, нужен active-low.
- `mtk-eint.c`: `IRQ_TYPE_LEVEL_LOW → pol_clr` (ровно то, что делал вручную).

**ФИКС шторма (в патче, НО см. §6b):** в `mt6769t-xiaomi-common.dtsi`:
```
&pmic {
-	interrupts-extended = <&pio 144 IRQ_TYPE_LEVEL_HIGH>;
+	interrupts-extended = <&pio 144 IRQ_TYPE_LEVEL_LOW>;
};
```
Проверка: `cat /proc/interrupts | grep mt6358-irq` → счётчик 0/с (было ~28K/с).

---

## 6b. ⚠️ ЗАВИСАНИЕ (виснет на ~45–100с) — ОТКРЫТО, НЕ РЕШЕНО

**Что выяснилось при проверке фикса:**
- **LEVEL_LOW (r18/r19) убирает шторм, но устройство НАДЁЖНО виснет через
  ~45–100с** (несколько бутов подряд). Симптом: `NETDEV WATCHDOG: transmit queue
  timed out` → USB отваливается → ребут. В журнале перед зависанием — флуд
  `panfrost ... _set_opp: switching OPP` (GPU DVFS скачет).
- **LEVEL_HIGH (r17, со штормом) — НЕ виснет так быстро**: держался 5+ мин
  (несколько раз), т.е. шторм НЕ является причиной зависаний (вопреки старой
  формулировке §6). Похоже, шторм «маскировал» зависание: постоянная активность
  pwrap/IRQ не даёт системе дойти до проблемного состояния.
- `IdleAction=ignore` в logind → это НЕ авто-suspend (suspend отключён).

**Гипотезы зависания (не проверены):**
1. GPU/panfrost DVFS-гонка (флуд `_set_opp` перед hang) — вероятный кандидат.
2. Питание/PMIC (SMB1351/зарядка не портированы) — pwrap уходит в idle без шторма.
3. Некоторый nested-PMIC IRQ (RTC/keys) теперь реально доставляется с LEVEL_LOW.

**Что дальше (по приоритету):**
1. Разобраться с зависанием (GPU DVFS? `CONFIG_DEVFREQ`? пауза между OPP?).
2. Потом вернуться к шторму — LEVEL_LOW-фикс готов и корректен, но его нельзя
   включать, пока висит устройство.

**Текущее состояние устройства: прошит r17 (LEVEL_HIGH + шторм)** — это самый
стабильный наблюдаемый вариант. Патч в репозитории содержит LEVEL_LOW-фикс
(с комментарием «см. §6b»), чтобы его было легко включить после решения зависания.

---

## 7. Второй баг (подсветка) — исправлен, проверь

В `huaxing.dtsi` panel-нода не имела `backlight = <&backlight>;`. Добавил строку
(есть в r15). Проверь: при выключении экрана подсветка должна гаснуть
(`/sys/class/backlight/ktd3137-backlight/brightness` → 0).

---

## 8. Следующие шаги (приоритет)

1. **Разобраться с зависанием (~45–100с)** — главный блокер, см. §6b. Кандидат —
   GPU/panfrost DVFS (флуд `_set_opp` перед hang). Проверить: отключить GPU devfreq,
   зафиксировать частоту GPU, или ловить точный момент зависания через persistent-лог.
2. **Вернуть LEVEL_LOW-фикс шторма** (уже в патче) после решения зависания.
3. **Батарея/зарядка** — патчи `hataketsu/mt6768-mainline-notes` →
   `kernel-patches/battery/*.patch` + драйвер SMB1351 (его нет в mainline).
4. **Wi-Fi/BT** — порт gen4m (`hataketsu/redmi9-lancelot-mainline`); прошивки уже
   извлечены в `lancelot/extracted/firmware/` (WIFI_RAM_CODE_soc1_0_1a_1.bin и др.).
5. **Аудио** — mt6358 DAI-обвязка.

---

## 9. Все грабли (кратко)

1. AVB-footer обязателен (`--algorithm NONE`), иначе `lk_crash`.
2. `clk_ignore_unused pd_ignore_unused` в cmdline — без них дисплей не работает.
3. systemd (не OpenRC) — plasma-mobile в новых pmaports только systemd.
4. `CONFIG_INPUT_EVDEV=y` — иначе нет /dev/input/eventX (evdev=m падает из-за BTF).
5. `CONFIG_MODULE_ALLOW_BTF_MISMATCH=y` — иначе модули падают `BTF: -22`.
6. **`dd` в eMMC — `oflag=direct`**, иначе page-cache не сбрасывается до reboot.
7. **Нумерация разделов НЕСТАБИЛЬНА**: eMMC бывает и `mmcblk0`, и `mmcblk1`
   (зависит от наличия SD-карты/порядка probe). ВСЕГДА проверяй `lsblk` + `mount`
   перед прошивкой boot (см. §5). В прошлой сессии был mmcblk1, сейчас mmcblk0.
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
