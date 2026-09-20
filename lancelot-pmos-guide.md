# postmarketOS на Xiaomi Redmi 9 (lancelot) — полный рабочий гайд

> **Проверено на реальном устройстве (сентябрь 2026).** Работают: загрузка,
> дисплей, тач, GPU, Plasma Mobile, SSH по USB. Ниже — точная процедура, по
> которой это было достигнуто, вместе со всеми граблями.

---

## 0. Итоговое состояние (что работает)

| Компонент | Статус | Как достигнуто |
|---|---|---|
| Ядро | ✅ mainline 6.16.0 | `linux-postmarketos-mediatek-mt6768` |
| Дисплей | ✅ 1080×2340 | dtb `huaxing-ktd` + `clk_ignore_unused pd_ignore_unused` |
| Тач | ✅ Focaltech FT8719 (10 пальцев) | портированный драйвер + прошивка |
| GUI | ✅ Plasma Mobile | systemd (не OpenRC) |
| GPU | ✅ panfrost (Mali-G52) | из коробки |
| SSH/USB | ✅ `172.16.42.1` | из коробки |
| Батарея/зарядка | ❌ | нет power_supply (нужны патчи hataketsu + SMB1351) |
| Wi-Fi/BT | ❌ | нет gen4m-драйвера |
| Аудио | ❌ | нет DAI-обвязки mt6358 |

---

## 1. ВАЖНО: этот аппарат — ревизия Huaxing/FT8719, а не Tianma/Novatek

Xiaomi Redmi 9 выпускался минимум с **двумя** вариантами панели и тача:

| Вариант | Панель | Тач | Драйвер в mainline |
|---|---|---|---|
| Tianma | NT36672A | Novatek NT36672A (SPI) | ✅ есть (`novatek-nvt-ts-spi`) |
| **Huaxing** | FT8719 | **Focaltech FT8719 (SPI)** | ❌ **нет** (его мы и портировали) |

Стоковый pmOS-порт (`device-xiaomi-lancelot`, tier testing) собран под
**Tianma**. На аппарате с Huaxing из коробки **ничего не работает**: ни дисплей
(не тот dtb), ни тач (нет драйвера FT8719).

**Как определить свою ревизию** (в LineageOS/Android):

```bash
adb shell getprop ro.boot.lcm    # или
adb shell cat /proc/cmdline | tr ' ' '\n' | grep LCM_name
```

- `LCM_name=...tianma...` → Tianma/Novatek (стоковый порт подойдёт).
- `LCM_name=ft8719_fhdp_dsi_vdo_huaxing_j19_lcm_drv` → **Huaxing/FT8719** (этот гайд).

---

## 2. Предварительные требования

- **Хост**: Ubuntu (проверено на 26.04), 4+ ядра, 8+ ГБ RAM, 40+ ГБ диска.
- **Телефон**: разблокированный загрузчик (Mi Unlock, до 7 дней ожидания).
- Установленные пакеты: `git python3 adb fastboot`, а также `simg2img`/`img2simg`
  (`apt install android-sdk-libsparse-utils`) и `dtc` (`apt install device-tree-compiler`).

---

## 3. Сборка (pmbootstrap)

```bash
# pmbootstrap из git (нужна свежая версия >= 3.11, apt-версия устарела)
git clone https://gitlab.postmarketos.org/postmarketOS/pmbootstrap.git
cd pmbootstrap
python3 pmbootstrap.py init
```

В `init`: `channel=edge`, `vendor=xiaomi`, `device=lancelot`, `ui=plasma-mobile`,
`user=<любой>`. Система будет **systemd** (plasma-mobile требует systemd в новых
pmaports — это нормально для mainline-ядра).

### 3.1. Правка 1 — dtb под Huaxing

Отредактируй `device/testing/device-xiaomi-lancelot/deviceinfo` в pmaports:

```
deviceinfo_dtb="mediatek/mt6769t-xiaomi-lancelot-huaxing-ktd"
```

(стоковое значение `tianma-ti` — для Tianma. Для Huaxing нужен `huaxing-ktd`.)

После правки обнови sha512 в `APKBUILD` этого device-пакета и подними `pkgrel`:

```bash
cd device/testing/device-xiaomi-lancelot
sha512sum deviceinfo   # вставь новое значение в APKBUILD
```

### 3.2. Правка 2 — драйвер FT8719 + touch-нода + конфиг ядра

Все изменения — в одном патче **`0001-ft8719-touch.patch`** (в этом репозитории).
Он делает три вещи:

1. Добавляет драйвер `drivers/input/touchscreen/focaltech_ft8719.c`
   (порт downstream-драйвера MTK `FT8719P`, ~900 строк, без MTK-зависимостей).
2. Регистрирует его в `Kconfig` (`TOUCHSCREEN_FOCALTECH_FT8719`) и `Makefile`.
3. Добавляет touch-ноду `focaltech,ft8719` в
   `arch/arm64/boot/dts/mediatek/mt6769t-xiaomi-lancelot-huaxing.dtsi`
   (irq = pio 1, reset = pio 92, falling edge).

Как применить — положи патч в каталог ядерного пакета и добавь в `APKBUILD`:

```bash
cp 0001-ft8719-touch.patch \
   device/testing/linux-postmarketos-mediatek-mt6768/
# в APKBUILD этого пакета:
#   source=" ... 0001-ft8719-touch.patch"
#   добавить sha512sum патча, поднять pkgrel
```

Затем включи в конфиге ядра
`config-postmarketos-mediatek-mt6768.aarch64` **три** обязательные опции:

```
CONFIG_TOUCHSCREEN_FOCALTECH_FT8719=y
CONFIG_INPUT_EVDEV=y                    # иначе нет /dev/input/eventX
CONFIG_MODULE_ALLOW_BTF_MISMATCH=y      # иначе не грузятся модули (BTF-ошибка)
```

### 3.3. Сборка и образы

```bash
python3 pmbootstrap.py build linux-postmarketos-mediatek-mt6768
python3 pmbootstrap.py install --password pmos
```

После `install`:
- `boot.img` (kernel+initramfs+dtb) — в `chroot_rootfs_xiaomi-lancelot/boot/`.
- rootfs-образ — в `chroot_native/home/pmos/rootfs/xiaomi-lancelot.img`
  (это **Android sparse** образ с GPT: p1=boot, p2=root).

---

## 4. Прошивка

### 4.1. Готовим boot.img (обязательные шаги)

1. **Добавь флаги в cmdline** — без них дисплей не работает:
   пересобери `boot.img`, добавив `clk_ignore_unused pd_ignore_unused` в cmdline
   (убери `quiet`, добавь `loglevel=8 ignore_loglevel` для отладки).

2. **Добавь AVB-footer** — иначе LK падает в `lk_crash`:

```bash
python3 avbtool.py add_hash_footer --image boot.img \
  --partition_name boot --partition_size 67108864 --algorithm NONE
```

(`avbtool.py` — из AOSP `external/avb`; в репозитории есть копия.)

### 4.2. rootfs в super (без SD-карты)

rootfs-образ содержит GPT (p1 boot + p2 root). Чтобы не возиться со вложенным
GPT, извлеки только корневой раздел и прошей его напрямую в `super`:

```bash
simg2img xiaomi-lancelot.img rootfs.raw      # sparse -> raw
fdisk -l rootfs.raw                          # найти смещение/размер p2
# p2 начинается на секторе 999424, длина 8425472 (секторов по 512)
dd if=rootfs.raw of=rootfs.ext4 bs=512 skip=999424 count=8425472
img2simg rootfs.ext4 rootfs.sparse           # обратно в sparse для fastboot
```

Затем (телефон в fastboot: Power+VolDown):

```bash
fastboot flash vbmeta vbmeta.img        # vbmeta с флагом verification-disabled
fastboot flash boot boot.img            # с AVB-footer
fastboot flash dtbo dtbo.img
fastboot flash super rootfs.sparse      # ~3-4 мин
fastboot reboot
```

> `vbmeta.img` с отключённой верификацией:
> `avbtool make_vbmeta_image --flags 2 --output vbmeta.img`

### 4.3. Правка fstab (иначе emergency mode)

Т.к. мы прошили только p2 (без p1=boot), в `/etc/fstab` раздела `/boot` нет.
До прошивки смонтируй `rootfs.ext4` на хосте и добавь `nofail`:

```
UUID=<boot-uuid> /boot ext2 nodev,nosuid,noexec,nofail 0 0
```

(или удали строку `/boot` совсем — ядро грузится из eMMC-раздела `boot`,
отдельный `/boot` для работы не нужен).

### 4.4. Прошивка тача

Скопируй на телефон (до первого касания) прошивку панели:

```bash
scp focaltech_ts_fw_huaxing.bin user@172.16.42.1:/tmp/
ssh user@172.16.42.1 "sudo cp /tmp/focaltech_ts_fw_huaxing.bin /lib/firmware/"
```

Файл берётся из `/vendor/firmware/` стоковой прошивки (нужен root/Magisk), либо
из LineageOS-образа `vendor`.

---

## 5. Пост-установка

### 5.1. Расширить rootfs (он всего 4.3 ГБ из 7 ГБ super)

```bash
ssh user@172.16.42.1 "sudo resize2fs /dev/mmcblk1p44"
```

### 5.2. Починить zram-swap (иначе сервис падает)

Ядро поддерживает только `lzo-rle`/`lzo`, а скрипт просит `zstd`:

```bash
echo 'deviceinfo_zram_swap_algo="lzo-rle"' \
  | ssh user@172.16.42.1 "sudo tee -a /usr/share/deviceinfo/deviceinfo"
ssh user@172.16.42.1 "sudo systemctl restart postmarketos-zram-swap"
```

---

## 6. ⚠️ Все грабли (что сэкономит часы)

1. **AVB-footer обязателен** даже при разблокированном загрузчике. Без него —
   `lk_crash`. Используй `--algorithm NONE` (как у стокового boot).
2. **`clk_ignore_unused pd_ignore_unused`** в cmdline — без них дисплей спамит
   `mtk-iommu ... fault` и не доходит до systemd.
3. **systemd, а не OpenRC** — plasma-mobile в новых pmaports только под systemd.
   На mainline-ядре это работает (проблема OpenRC/systemd была у downstream 4.14).
4. **`CONFIG_INPUT_EVDEV=y`** — иначе нет `/dev/input/eventX` (evdev=m не
   грузится из-за BTF-mismatch).
5. **`CONFIG_MODULE_ALLOW_BTF_MISMATCH=y`** — иначе модули падают
   `failed to validate module BTF: -22`.
6. **Запись в eMMC через `dd` требует `oflag=direct`** — иначе page-cache не
   сбрасывается до reboot и прошивка не применяется:
   `sudo dd if=boot.img of=/dev/mmcblk1p34 bs=4096 oflag=direct conv=fsync`.
7. **Нумерация разделов меняется** между Android и pmOS: на pmOS eMMC = `mmcblk1`
   (не `mmcblk0`). `boot` = `mmcblk1p34`, `super` = `mmcblk1p44`. Проверяй через
   `cat /proc/partitions`.
8. **Прошивка тача запрашивается ДО монтирования rootfs** — драйвер делает
   `request_firmware` с ретраем (10×2с), поэтому файл в `/lib/firmware` обязателен.
9. **CRC в SPI-протоколе FT8719** — только когда послан command-package. При
   чтении touch-данных (cmd=NULL) CRC нет; драйвер проверяет буфер напрямую.
10. **`pmbootstrap install` каждый раз меняет UUID rootfs** — поэтому лучше
    пересобирать только `boot.img` (mkbootimg) со старым initramfs+cmdline, а
    rootfs перепрошивать только когда реально менялся.

---

## 7. Артефакты (в этом репозитории)

| Файл | Что это |
|---|---|
| `lancelot/0001-ft8719-touch.patch` | патч: драйвер + Kconfig + Makefile + touch-нода в huaxing dtsi |
| `lancelot/focaltech_ft8719.c` | исходник портированного драйвера (для справки) |
| `lancelot/extracted/firmware/focaltech_ts_fw_huaxing.bin` | прошивка тача Huaxing (131 КБ) |
| `lancelot/avbtool.py` | инструмент AVB (из AOSP external/avb) |

### 7.1. Сводка правок (для upstream/pmaports)

Для встраивания «в апстрим» достаточно отправить в pmaports:
1. Патч `0001-ft8719-touch.patch` в пакет `linux-postmarketos-mediatek-mt6768`.
2. `CONFIG_TOUCHSCREEN_FOCALTECH_FT8719=y` + `CONFIG_INPUT_EVDEV=y` +
   `CONFIG_MODULE_ALLOW_BTF_MISMATCH=y` в конфиг ядра.
3. `deviceinfo_dtb="...huaxing-ktd"` в device-пакет (или автоопределение
   ревизии по `LCM_name` через cmdline).
4. Прошивку `focaltech_ts_fw_huaxing.bin` — в `firmware-focaltech`-пакет
   (лицензия: blob из vendor, распространять нельзя, но можно добавить в
   `device-xiaomi-lancelot-firmware` с инструкцией извлечения, как `nvt_tm_fw.bin`).

---

## 8. Что осталось (не сделано)

1. **Батарея/зарядка** — перенести патчи `hataketsu/mt6768-mainline-notes`
   (`kernel-patches/battery/*.patch`) + написать mainline-драйвер зарядки SMB1351
   (его нет в mainline).
2. **Wi-Fi/BT** — порт gen4m (`hataketsu/redmi9-lancelot-mainline`, там Wi-Fi уже
   работает на 6.18; прошивки `WIFI_RAM_CODE_soc1_0_1a_1.bin` и `soc1_0_ram_*`
   уже извлечены в `lancelot/extracted/firmware/`).
3. **Подсветка не гаснет** при выключении экрана (PM-баг `ktd3137-backlight`).
4. **Аудио** — mt6358 DAI-обвязка.

---

## 9. Восстановить контекст для ИИ на новом ПК

```
Контекст: порт postmarketOS на Xiaomi Redmi 9 (lancelot, MediaTek Helio G80 /
MT6769T). Аппарат — ревизия Huaxing/FT8719 (НЕ Tianma/Novatek). Ядро
linux-postmarketos-mediatek-mt6768 (mainline 6.16). Работает: загрузка, дисплей
(1080x2340, dtb huaxing-ktd + clk_ignore_unused pd_ignore_unused), тач
(портированный драйвер focaltech_ft8719.c + прошивка focaltech_ts_fw_huaxing.bin),
GPU panfrost, Plasma Mobile, SSH 172.16.42.1. Не работает: батарея/зарядка
(SMB1351), Wi-Fi/BT (gen4m), аудио (mt6358), подсветка не гаснет (PM).
Полный гайд и патч — в lancelot-pmos-guide.md и lancelot/0001-ft8719-touch.patch.
Прочитай и продолжи.
```
