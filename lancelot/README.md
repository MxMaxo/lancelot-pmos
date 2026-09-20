# lancelot/ — артефакты порта postmarketOS (Huaxing/FT8719)

Рабочие артефакты порта postmarketOS на Xiaomi Redmi 9 (lancelot) ревизии
**Huaxing/FT8719**. Полная процедура — в соседнем файле
`../lancelot-pmos-guide.md`.

## Содержимое

```
lancelot/
├── 0001-ft8719-touch.patch       # ПАТЧ (главное): драйвер + Kconfig + Makefile + DTS
├── config.fragment               # 3 строки конфига ядра (тач + evdev + BTF-mismatch)
├── deviceinfo.patch              # правка dtb в device-пакете (tianma-ti → huaxing-ktd)
├── focaltech_ft8719.c            # исходник портированного драйвера (для справки)
├── avbtool.py                    # AVB-инструмент (из AOSP external/avb)
├── ft8719_downstream/            # downstream-драйвер MTK FT8719P (референс, GPL-2.0)
├── tools/                        # mkbootimg + unpack_bootimg (из android-tools)
├── flash/
│   ├── boot_touch.img            # итоговый рабочий boot.img (с AVB-footer, с тачем)
│   ├── dtbo.img                  # dtbo
│   └── vbmeta.img                # vbmeta с отключённой верификацией
└── extracted/
    ├── firmware/                 # блобы из /vendor/firmware (нужен root/Magisk)
    │   ├── focaltech_ts_fw_huaxing.bin     # прошивка тача Huaxing (131 КБ)
    │   ├── WIFI_RAM_CODE_soc1_0_1a_1.bin   # Wi-Fi (для будущего порта gen4m)
    │   ├── soc1_0_ram_*_hdr.bin            # Wi-Fi/BT патчи
    │   └── ...
    └── stock_*.img               # бэкапы стоковых разделов (boot/dtbo/vbmeta*)
```

## Как применить патч

Патч `0001-ft8719-touch.patch` применяется к ядру
`linux-postmarketos-mediatek-mt6768` (источник `mt6768-mainline/linux`, коммит
`781287a2e573dc9bb85a1bc90b86a94f58671853`, версия 6.16.0).

В pmaports: положить патч в `device/testing/linux-postmarketos-mediatek-mt6768/`,
добавить в `source=` APKBUILD (+sha512sum, +pkgrel), включить в конфиге:

```
CONFIG_TOUCHSCREEN_FOCALTECH_FT8719=y
CONFIG_INPUT_EVDEV=y
CONFIG_MODULE_ALLOW_BTF_MISMATCH=y
```

Плюс в device-пакете `device-xiaomi-lancelot`:
`deviceinfo_dtb="mediatek/mt6769t-xiaomi-lancelot-huaxing-ktd"`.

## Лицензии

- `focaltech_ft8719.c` — GPL-2.0-only (производное от downstream-драйвера FocalTech/Xiaomi).
- `ft8719_downstream/` — GPL-2.0 (исходный downstream-код MTK, для референса).
- `avbtool.py` — Apache-2.0 (AOSP `external/avb`).
- Прошивки в `extracted/firmware/` — проприетарные блобы, **не распространять**;
  извлекаются из собственного устройства.
