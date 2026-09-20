# postmarketOS на Xiaomi Redmi 9 (lancelot) — порт для ревизии Huaxing/FT8719

Рабочий порт postmarketOS на Xiaomi Redmi 9 (codename `lancelot`, MediaTek Helio
G80 / MT6769T) с панелью **Huaxing** и тачем **Focaltech FT8719**.

> Стоковый порт `device-xiaomi-lancelot` в pmaports (tier testing) рассчитан на
> ревизию **Tianma/Novatek NT36672A**. Этот репозиторий закрывает пробел для
> ревизии **Huaxing/FT8719**, которой нет ни драйвера тача, ни нужного dtb.

## Что работает

Загрузка, дисплей (1080×2340), тач (10 пальцев), GPU (panfrost), Plasma Mobile,
SSH по USB. Подробный статус и пошаговая инструкция — в
[`lancelot-pmos-guide.md`](lancelot-pmos-guide.md).

## Содержимое

| Путь | Что это |
|---|---|
| `lancelot-pmos-guide.md` | Полный гайд: от разблокировки до рабочего устройства + все грабли |
| `lancelot/0001-ft8719-touch.patch` | **Патч ядра**: драйвер + Kconfig + Makefile + touch-нода в DTS |
| `lancelot/focaltech_ft8719.c` | Исходник портированного драйвера (для справки) |
| `lancelot/README.md` | Описание артефактов |

## Быстрый старт

1. Применить патч к ядру `mt6768-mainline/linux` @ `781287a2` (6.16.0).
2. Включить в конфиге:
   ```
   CONFIG_TOUCHSCREEN_FOCALTECH_FT8719=y
   CONFIG_INPUT_EVDEV=y
   CONFIG_MODULE_ALLOW_BTF_MISMATCH=y
   ```
3. В device-пакете: `deviceinfo_dtb="...huaxing-ktd"`.
4. Извлечь прошивку тача `focaltech_ts_fw_huaxing.bin` из `/vendor/firmware`
   своего устройства (нужен root) и положить в `/lib/firmware/`.

Полная процедура (включая прошивку, AVB-footer, rootfs в super и пр.) — в гайде.

## Лицензии

- Код (драйвер, патч) — **GPL-2.0-only** (производное от downstream-драйвера
  FocalTech/Xiaomi `android_kernel_xiaomi_mt6768`).
- Документация — **MIT**.
- Прошивки (`focaltech_ts_fw_huaxing.bin`, `WIFI_RAM_CODE_*.bin`) — проприетарные
  блобы, **в репозиторий не включены** (извлекаются из собственного устройства).

## См. также

- Стоковый порт: https://wiki.postmarketos.org/wiki/Xiaomi_Redmi_9_(xiaomi-lancelot)
- Mainline-порт (референс, Wi-Fi/батарея работают): https://github.com/hataketsu/redmi9-lancelot-mainline
- Заметки mainline: https://github.com/hataketsu/mt6768-mainline-notes
