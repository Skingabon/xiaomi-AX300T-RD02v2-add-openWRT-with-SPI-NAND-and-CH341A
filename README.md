# Xiaomi AX3000T → OpenWrt + PassWall2 (VLESS-Reality): журнал

Цель: поставить OpenWrt на Xiaomi AX3000T и заменить им разом VPN-роутер
(MikroTik hEX RB750Gr3) и отдельную точку доступа — расчёт был на то, что
ARM-процессор с аппаратным ускорением шифрования решит проблему низкой
скорости VPN на слабом MIPS в hEX (подробности по MikroTik — в соседнем
репозитории [mikrotik-hEX-RB750Gr3-add-openWRT](https://github.com/Skingabon/mikrotik-hEX-RB750Gr3-add-openWRT)).

**Статус на 2026-09-28: успех.** Второй купленный роутер оказался
настоящим RD03 (MediaTek MT7981, не RD03v2) — полностью прошит,
настроен, заменил MikroTik в бою. Ниже — полный воспроизводимый журнал.

Первая попытка (2026-09-05, другой экземпляр роутера) не удалась — плата
оказалась RD03v2 (Qualcomm), софтверный root на неё не ставится. Журнал
той попытки сохранён в конце файла, раздел **«Попытка №1 (тупик)»** — он
всё ещё актуален как предупреждение: **разные копии одной модели в
рознице могут быть на разных платах**, обязательно проверяйте ревизию
перед началом (см. следующий раздел).

## ⚠️ Шаг 0, обязательный: проверить реальную ревизию платы

Штрихкод/коробка не гарантируют ревизию. Подключите роутер (заводская
прошивка MiWiFi) к ПК, узнайте его IP (обычно `192.168.31.1`) и
выполните:

```sh
curl http://192.168.31.1/cgi-bin/luci/api/xqsystem/init_info
```

Смотрите на поле `"hardware"`:
- **`RD03`** (без v2) → это MediaTek MT7981 (Filogic), Cortex-A53,
  **этот журнал подходит**, эксплойт `xmir-patcher` работает.
- **`RD03v2`** → Qualcomm IPQ5018, **другая, несовместимая плата**,
  софтверного root нет (проверено эмпирически, см. «Попытка №1» внизу).
  Нужен UART/программатор, это другая процедура.

Не начинайте прошивку, пока не убедитесь, что `hardware: RD03`.

## Что понадобится

- Инструмент [`openwrt-xiaomi/xmir-patcher`](https://github.com/openwrt-xiaomi/xmir-patcher)
  (склонировать, внутри портативный Python в `python/`, отдельно
  собирать ничего не нужно)
- Windows SSH-клиент (встроенный OpenSSH) и `scp`
- Два образа OpenWrt под `xiaomi_mi-router-ax3000t` (target
  `mediatek/filogic`) — стандартный (не ubootmod) вариант, т.к. на
  роутере остаётся штатный Xiaomi U-Boot:
  - `openwrt-XX.XX.X-mediatek-filogic-xiaomi_mi-router-ax3000t-initramfs-factory.ubi`
  - `openwrt-XX.XX.X-mediatek-filogic-xiaomi_mi-router-ax3000t-squashfs-sysupgrade.bin`

  Скачивать с `https://downloads.openwrt.org/releases/<версия>/targets/mediatek/filogic/`
  (использовали `24.10.3`).

## 1. Получение root через xmir-patcher

Роутер б/у почти наверняка уже инициализирован (`inited: 1` в
`init_info`, чужое имя сети) — проще всего **сбросить кнопкой Reset**
(зажать ~8-10 сек) и на первом запуске веб-мастера задать **свой**
пароль веб-интерфейса. Сброс не трогает версию прошивки, эксплойт
продолжает работать.

Дальше `!START.bat` (или `run.bat` без параметров) → меню:

```
Select: 2                                   # Connect to device (install exploit)
Enter device WEB password: <ваш пароль>
...
Exploit "start_binding" detected!
Run SSH server on port 22 ...
```

Это даёт **временный** SSH (слетает после ребута): `ssh root@<IP>`,
пароль `root` (эксплойт сам ставит его, см. `connect6.py`).

**Обязательно дальше — пункт `4` в меню (`Create full backup`)**: дампит
все разделы (`mtd0`…`mtd12`, ~128 МБ) в `backups/`. Не пропускать —
единственная страховка от кирпича.

## 2. Определение раздела для прошивки

```sh
cat /proc/cmdline
# ... firmware=1 mtd=ubi1   ← значит активен ubi1 (mtd9), пишем в НЕактивный ubi (mtd8)
# (если firmware=0 — наоборот, пишем в mtd9)
```

Таблица разделов (типовая для этой платы, проверяйте `cat /proc/mtd`
у себя — номера могут отличаться):

```
mtd0  spi0.0    128 МБ (весь чип)
mtd1  BL2         1 МБ
mtd2  Nvram     256 КБ
mtd3  Bdata     256 КБ
mtd4  Factory     2 МБ
mtd5  FIP         2 МБ
mtd6  crash     256 КБ
mtd7  crash_log 256 КБ
mtd8  ubi        34 МБ   ← слот firmware=0
mtd9  ubi1       34 МБ   ← слот firmware=1
mtd10 overlay    32 МБ  (используется стоком, не OpenWrt)
mtd11 data       12 МБ
mtd12 KF        256 КБ
```

## 3. Заливка initramfs-образа и переключение слота

С компьютера (замените `mtd8` на актуальный неактивный слот и путь к
файлу):

```sh
scp -O -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa \
  openwrt-24.10.3-mediatek-filogic-xiaomi_mi-router-ax3000t-initramfs-factory.ubi \
  root@<IP>:/tmp/
```

На роутере по SSH:

```sh
ubiformat /dev/mtd8 -y -f /tmp/openwrt-24.10.3-...-initramfs-factory.ubi

nvram set boot_wait=on
nvram set uart_en=1
nvram set flag_boot_rootfs=0      # 0, если только что писали в mtd8 (был активен firmware=1)
nvram set flag_last_success=0
nvram set flag_boot_success=1
nvram set flag_try_sys1_failed=0
nvram set flag_try_sys2_failed=0
nvram commit
reboot
```

После перезагрузки (~1.5-2 мин): **переткнуть кабель в другой физический
порт** (не тот, что использовался для стоковой прошивки — там обычно
WAN), IP станет дефолтным OpenWrt `192.168.1.1`. Проверить `ping`, зайти
`ssh root@192.168.1.1` (пароль ещё не задан — просто Enter). Это ещё
**временная** initramfs-система (в RAM).

## 4. Финальная установка (sysupgrade)

```sh
# с компьютера:
scp -O -oHostKeyAlgorithms=+ssh-rsa,ssh-ed25519 \
  openwrt-24.10.3-mediatek-filogic-xiaomi_mi-router-ax3000t-squashfs-sysupgrade.bin \
  root@192.168.1.1:/tmp/

# на роутере:
sysupgrade -n /tmp/openwrt-24.10.3-...-squashfs-sysupgrade.bin
```

Сессия оборвётся («Closing all shell sessions») — это нормально, идёт
запись во флеш. Роутер сам перезагрузится (~2-3 мин) уже с постоянной
системой. Проверить: `mount | grep overlay` должен показывать реальное
блочное устройство (`/dev/ubi0_X`), а не `tmpfs`.

**Сразу задать пароль root** (`passwd`) — на свежей установке его нет.

### Грабли: старый ключ хоста в known_hosts / алгоритм ssh-rsa

- Современный Windows OpenSSH по умолчанию не принимает `ssh-rsa` (его
  предлагает dropbear на временной прошивке) → добавлять
  `-oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa`.
- Если в вашей сети уже есть другое устройство на `192.168.1.1`
  (например, действующий MikroTik по Wi-Fi, пока Xiaomi ещё на проводе) —
  Windows будет ругаться `REMOTE HOST IDENTIFICATION HAS CHANGED`.
  Это не атака, просто два разных устройства на одном IP на разных
  интерфейсах: `ssh-keygen -R <IP>` и подключаться заново.

## 5. PassWall2 + Xray-core

Официальных фидов с PassWall2 нет, ставится сторонним установщиком
(тот же, что и на MikroTik):

```sh
cd /tmp
wget -q https://raw.githubusercontent.com/enxy0/passwall2_install/main/passwall2.sh -O passwall2.sh
chmod +x passwall2.sh
sh passwall2.sh
```

Скрипт сам определяет `opkg`/`apk`, ставит LuCI-часть и пытается
поставить движки (`xray-core`, `sing-box`).

### Грабли: не хватает места в overlay под движок

На стандартной (не ubootmod) прошивке overlay ~60 МБ (UBIFS,
компрессия). После установки зависимостей PassWall2 останется мало
места, `xray-core` (~30 МБ) может не влезть. Нам лишний `sing-box` (54
МБ) не нужен вообще, и часть runtime-инструментов, которые тянутся по
умолчанию для протоколов, которыми мы не пользуемся:

```sh
opkg remove chinadns-ng shadowsocks-rust-sslocal shadowsocks-rust-ssserver \
  shadowsocksr-libev-ssr-local shadowsocksr-libev-ssr-redir \
  shadowsocksr-libev-ssr-server v2ray-plugin
# geoview / v2ray-geoip / v2ray-geosite трогать не пришлось — их
# требует сама luci-app-passwall2, но освобождённого места хватило и так

opkg install xray-core
```

Если всё равно не хватает — вариант с бОльшим запасом места: прошивка
через **ubootmod**-образы (заменяют U-Boot, дают ~75-85 МБ overlay
вместо ~60 МБ), это отдельная, более сложная процедура (см. страницу
устройства на wiki.openwrt.org), в этот раз не понадобилось.

## 6. Нода VLESS-Reality

Секреты (UUID, ключи) — **никогда не в git**, только в локальном
`CREDENTIALS.local.md` (см. `.gitignore`). Если уже есть работающий
клиент (например, MikroTik из соседнего репо) — проще всего утащить
готовый конфиг оттуда:

```sh
ssh root@<mikrotik-ip> "uci show passwall2" | grep -A20 nodes
```

Создание ноды на новом роутере (подставить реальные значения из
`CREDENTIALS.local.md`):

```sh
uci -q batch <<EOF
add passwall2 nodes
rename passwall2.@nodes[-1]=RealityServer
set passwall2.RealityServer.remarks="reality-server"
set passwall2.RealityServer.type="Xray"
set passwall2.RealityServer.protocol="vless"
set passwall2.RealityServer.address="<SERVER_IP>"
set passwall2.RealityServer.port="<SERVER_PORT>"
set passwall2.RealityServer.uuid="<VLESS_UUID>"
set passwall2.RealityServer.encryption="none"
set passwall2.RealityServer.tls="1"
set passwall2.RealityServer.reality="1"
set passwall2.RealityServer.tls_serverName="www.cloudflare.com"
set passwall2.RealityServer.reality_publicKey="<REALITY_PUBLIC_KEY>"
set passwall2.RealityServer.reality_shortId="<REALITY_SHORT_ID>"
set passwall2.RealityServer.fingerprint="safari"
set passwall2.RealityServer.transport="raw"
set passwall2.RealityServer.tcp_guise="none"
set passwall2.RealityServer.tcp_fast_open="0"
set passwall2.RealityServer.tcpMptcp="0"
set passwall2.@global[0].node="RealityServer"
set passwall2.@global[0].enabled="1"
commit passwall2
EOF
/etc/init.d/passwall2 restart
```

Проверка (транспарентно, без ручного прокси на клиенте — PassWall2 сам
через nftables/tproxy заворачивает весь трафик LAN):

```sh
curl -s https://ipinfo.io/json    # должен вернуть страну/IP сервера
```

## 7. Wi-Fi

У Xiaomi (в отличие от MikroTik) есть встроенный Wi-Fi 6 (2.4+5 ГГц) —
отдельная точка доступа не нужна. По умолчанию оба радио выключены,
SSID `OpenWrt` без пароля:

```sh
uci set wireless.radio0.disabled='0'
uci set wireless.radio1.disabled='0'
uci set wireless.default_radio0.ssid='<SSID>'
uci set wireless.default_radio0.encryption='sae-mixed'   # WPA2/WPA3 mixed
uci set wireless.default_radio0.key='<ПАРОЛЬ>'
uci set wireless.default_radio1.ssid='<SSID>'            # тот же SSID на оба диапазона — band steering
uci set wireless.default_radio1.encryption='sae-mixed'
uci set wireless.default_radio1.key='<ПАРОЛЬ>'
uci commit wireless
wifi reload
```

## 8. Переключение в боевой режим (замена MikroTik)

1. LAN-адрес Xiaomi по умолчанию — временный, отличный от `192.168.1.1`,
   пока рядом ещё жив старый роутер на этом же адресе (иначе конфликт
   IP в момент смены). Меняйте на финальный **только после** того, как
   старый роутер выключен/отключён от сети:
   ```sh
   uci set network.lan.ipaddr='192.168.1.1'
   uci commit network
   /etc/init.d/network restart
   ```
2. Физически: кабель от провайдера — в **WAN-порт** Xiaomi (не в один
   из трёх LAN). Если у старого роутера был обычный DHCP на WAN без
   привязки MAC — просто переткнуть кабель, ничего донастраивать не
   нужно.
3. Проверить: `ip -4 addr show wan`, `curl ipinfo.io/json` (должен
   отдать IP VPN-сервера, не роутера).

Старый роутер (MikroTik) можно оставить выключенным как резерв — если
что-то пойдёт не так, воткнуть провод провайдера обратно в него, ничего
вручную менять не нужно (он никак не зависит от того, что настроено на
Xiaomi).

## 9. Сторож памяти (memguard)

Скопирован `memguard.sh` из соседнего репозитория (MikroTik) — пороги
**скорректированы под Xiaomi**: тут ещё работает Wi-Fi (`hostapd` +
`wpa_supplicant`, лишние ~7 МБ RSS), поэтому здоровый фон "доступно"
ниже, чем на MikroTik. Актуальная версия скрипта (уже с адаптированными
порогами) — файл [`memguard.sh`](./memguard.sh) в этом репозитории.

```sh
scp -O -oHostKeyAlgorithms=+ssh-rsa,ssh-ed25519 memguard.sh root@192.168.1.1:/root/memguard.sh
ssh root@192.168.1.1 "chmod +x /root/memguard.sh; (crontab -l 2>/dev/null; echo '* * * * * /root/memguard.sh') | sort -u | crontab -; /etc/init.d/cron restart"
```

Ручная проверка без изменений: `ssh root@192.168.1.1 /root/memguard.sh --check`.
Молчит (не пишет лог), пока всё в норме; при WARN/CRIT пишет снимок +
попытку лечения в `/root/memguard.log` на самом роутере.

**Примечание:** поле `dnsmasq=` в снимке на Xiaomi всегда показывает
`0` — шаблон поиска процесса (`dnsmasq_default`) заточен под имена
процессов MikroTik, на Xiaomi процесс называется иначе
(`dnsmasq_acl_default` / `dnsmasq.conf.cfg...`). На пороги CRIT/WARN
это не влияет, только на отображаемое число — не повод для тревоги.

---

## Попытка №1 (тупик): экземпляр оказался RD03v2

*Сохранено для истории — актуально как предупреждение, что разные
копии одной модели в рознице могут быть на разных платах.*

**Дата: 2026-09-05.** Софтверный путь к root закрыт на этой прошивке,
нужен паяльник + USB-UART переходник 3.3V.

### Главный урок: штрихкод на коробке ЛГАЛ

На коробке было написано `RD03` — вариант на **MediaTek MT7981** (ARM
Cortex-A53, есть рабочий эксплойт `xmir-patcher`, есть реальный
бенчмарк WireGuard 371 Мбит/с на этой платформе). Но сам роутер через
встроенный API честно показал другое:

```sh
curl http://192.168.31.1/cgi-bin/luci/api/xqsystem/init_info
# "hardware":"RD03v2", "model":"xiaomi.router.rd03v2", "romversion":"2.0.28 release"
```

`RD03v2` — это **совсем другая плата**: Qualcomm IPQ5018 + свитч AN8855
+ радио QCN6122, 256 МБ RAM. Разные ревизии несовместимы по прошивкам —
инструкция не под ту плату превращает роутер в кирпич.

### Софтверный root закрыт на прошивке 2.0.28 — проверено эмпирически

```
$ python/python.exe connect.py 192.168.31.1
device_name = RD03V2
rom_version = 2.0.28 release
hackCheck version = 3
WARN: Exploits "arn_switch/start_binding/set_mac_filter/datacenter7" not working!!!
WARN: Exploits "Smartcontroller" are not usable (hackCheck:3)
WARN: Exploit "get_icon" not working!!! (API not founded)
```

Все три известных публичных эксплойта пропатчены Xiaomi.

### Оставшиеся пути для RD03v2 (по данным ADCDS, не проверялось)

Проект [`ADCDS/openwrt-xiaomi-ax3000t-rd03v2`](https://github.com/ADCDS/openwrt-xiaomi-ax3000t-rd03v2)
даёт чистую (mainline-based) сборку OpenWrt для этой платы:

1. **UART-пайка** — USB-UART переходник 3.3V, площадки на плате
   *"top-left, red box"*: **Rx · Gnd · Tx**, 115200 8N1, **3.3V** (не
   5V!). UART в стоке доступен только на чтение, проект использует его
   для перехвата загрузчика напрямую, минуя пропатченную ОС.
2. **Внешний SPI-NAND программатор** — чип флеша ESMT F50D1G41LB или
   Winbond W25N01KW, клипса SOIC8 на CH341A. Родная программа CH341A
   не понимает SPI-NAND (другая адресация, ECC) — нужен **AsProgrammer**
   (профили под Winbond W25N есть).

Готовые `recovery.bin` образы для версии 2.0.28 у ADCDS есть,
anti-rollback не блокер.
