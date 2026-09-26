---
name: zigbee2mqtt-network-gateway
description: Поднять ещё один экземпляр Zigbee2MQTT в Home Assistant (Supervised) для сетевого Zigbee-шлюза в локальной сети (UZG-01, XZG, ZigStar, SLZB и т.п., Zigbee по TCP, обычно порт 6638) — когда оба официальных дополнения (stable и edge) уже заняты. Диагностика шлюза и координатора, локальное дополнение-копия, настройки, сторожевой таймер, боковая панель, автоматизация автозапуска после отключения света, проверки и разбор «почему экземпляр не запущен». Триггеры — «подними zigbee2mqtt для шлюза», «новый координатор», «ещё один z2m», «добавить zigbee шлюз», «z2m не запущен».
---

# Zigbee2MQTT для нового сетевого шлюза

Навык общий. Всё, что относится к конкретной установке (адреса, доступ,
существующие экземпляры, логины), — в локальных заметках проекта
`.claude/ha-local.md`. **Прочитай их первым делом**, если файл есть. Если нет —
находи значения на месте командами ниже и в конце предложи завести такой файл.

Примеры адресов ниже — документационные (`192.0.2.x`), подставляй реальные.

Шаги 4–6 не пропускай: без них экземпляр не переживёт отключение
электричества или перезагрузку роутера.

## 0. Что нужно

- Root-доступ по SSH к хосту Home Assistant Supervised с `docker`.
  (На HA OS пути другие: локальные дополнения кладутся в `/addons` через
  SSH-дополнение — адаптируй шаги.)
- Предупреждение о post-quantum обмене ключами в выводе ssh отфильтровывай:
  `| grep -viE "^\*\* WARNING|store now|openssh.com"`.
- Длинные проверки удобнее писать скриптом и подавать на вход
  `ssh <хост> bash -s < script.sh` — так не мучаешься с кавычками.

## 1. Разведка установки

```sh
ls /usr/share/hassio                    # каталог apps/ или addons/ — как называются дополнения
docker exec hassio_cli ha --help        # команда `apps` или `addons`
ls /usr/share/hassio/*/git/*/zigbee2mqtt/config.json   # где официальный репозиторий z2m
docker ps --format '{{.Names}}\t{{.Image}}' | grep -i zigbee
docker images | grep -i zigbee2mqtt     # какой тег образа уже скачан
```

Дальше `<A>` — `apps` или `addons`, `<R>` — каталог официального репозитория.

Существующие экземпляры, их папки, шлюзы, каналы, pan_id и socat-порты:

```sh
grep -h -A2 '^serial:' /usr/share/hassio/homeassistant/zigbee2mqtt*/configuration.yaml
grep -h -E 'pan_id|channel' /usr/share/hassio/homeassistant/zigbee2mqtt*/configuration.yaml
```

Проще всего — открыть `/usr/share/hassio/<A>.json`, раздел `user`: у каждого
z2m там `options.data_path`, `network`, `watchdog`, `ingress_panel`.
Канал, который z2m реально использует, — в MQTT-теме `<base_topic>/bridge/info`
(`config.advanced.channel`, `network.pan_id`).

Особенности API:
- В CLI может **не быть** `ha <A> options`. Тогда параметры меняются через API
  Supervisor изнутри `hassio_cli`. Токен не печатай и не вытаскивай наружу:
  ```sh
  docker exec hassio_cli sh -c 'curl -s -X POST -H "Authorization: Bearer $SUPERVISOR_TOKEN" \
    -H "Content-Type: application/json" -d "{...}" http://supervisor/addons/<slug>/options'
  ```
- Этот токен обычно **не** пускает в API самого HA (`/core/api/...` → 401).
  Тогда автоматизации после правки перечитывает пользователь
  (Инструменты разработчика → YAML → «Автоматизации»).

## 2. Диагностика шлюза

1. **Шлюз не занят**: его IP нет в `serial.port` ни одного
   `zigbee2mqtt*/configuration.yaml`, и ZHA не на нём
   (`.storage/core.config_entries`, `domain: zha`).
2. **Порт** Zigbee по TCP открыт (обычно 6638).
3. **Координатор живой** — запросы Z-Stack по TCP, все только на чтение:

   ```python
   import socket
   def znp(ip, frame, port=6638):
       s = socket.create_connection((ip, port), timeout=8); s.settimeout(6)
       s.sendall(bytes(frame)); d = s.recv(64); s.close(); return d
   znp("192.0.2.10", [0xFE,0x00,0x21,0x01,0x20])  # SYS_PING  -> fe 02 61 01 ...
   znp("192.0.2.10", [0xFE,0x00,0x21,0x02,0x23])  # SYS_VERSION: байты 6..8 версия, 9..12 сборка (LE)
   znp("192.0.2.10", [0xFE,0x00,0x27,0x00,0x27])  # UTIL_GET_DEVICE_INFO
   ```
   В ответе UTIL_GET_DEVICE_INFO байт состояния: 0 — сети нет (шлюз чистый),
   9 — координатор с поднятой сетью (чья-то сеть уже есть — выясни чья).

   Нет ответа на PING, а при сбросе чипа в логе шлюза проскакивает один
   «мусорный» байт — подозревай несовпадение скорости UART между ESP32 и
   радиочипом или пустую прошивку радиочипа.
4. **Прошивка шлюза** (если нужно обновлять):
   - ZigStar и UZG-01 перешли на XZG (`xyzroe/XZG`). Последние выпуски старых
     линеек — мостики для обновления по сети.
   - Обновление по сети меняет только программу, но **не таблицу разделов и
     не загрузчик**. Если старая прошивка размечала память иначе, чем XZG
     (у XZG стандартный `default.csv`: app0 0x10000, app1 0x150000, spiffs
     0x290000), новый образ зальётся, «OK» вернётся, а после перезагрузки
     шлюз откатится на старую версию. Так ведёт себя ZigStar на
     TTGO T-Internet-POE (`min_spiffs`) — там только USB:
     `esptool.py --chip esp32 write_flash 0x0 XZG_<версия>.full.bin`, в XZG выбрать плату LilyZig.
   - UZG-01 обновляется по сети в две ступени: мостик `UZG-01.bin`
     (`mercenaruss/uzg-firmware` v1.0.0) → `XZG_<версия>.ota.bin`.
   - Контрольный опыт, если обновление «не встаёт»: залить соседнюю версию
     старой линейки. Сменилась — механизм исправен, проблема в образе.
   - Заливка: `curl -H "Expect:" -F "update=@файл.bin" http://<шлюз>/update`.
     Проверяй версию после перезагрузки, а не ответ «OK».
   - Сигнатура образа ESP32 — первый байт `e9`; версия IDF — в описателе
     приложения со смещения 0x20 (поле idf_ver в 0x90).
   - `/releases/latest` у XZG пропускает пред-релизы — спроси пользователя,
     стабильная или самая новая.

## 3. Локальное дополнение

N — номер нового экземпляра. Копия описания официального дополнения:

```sh
mkdir -p /usr/share/hassio/<A>/local/zigbee2mqttN
python3 - <<'PY'
import json
src = '<R>/zigbee2mqtt/config.json'
d = json.load(open(src))
port = <следующий свободный socat-порт, например 8487>
d.update(name='Zigbee2MQTT N', slug='zigbee2mqttN',
         description='Zigbee2MQTT N (шлюз <IP>)',
         version='<тег образа, который уже есть в docker images>')
d['ports'] = {'8485/tcp': port, '8099/tcp': None}
o = d['options']
o['data_path'] = '/config/zigbee2mqttN'
o['socat']['master'] = 'pty,raw,echo=0,link=/tmp/ttyZ2MN,mode=777'
o['socat']['slave'] = o['socat']['slave'].replace('8485', str(port))
json.dump(d, open('/usr/share/hassio/<A>/local/zigbee2mqttN/config.json', 'w'),
          ensure_ascii=False, indent=2)
PY
```

В описании есть `"image": "ghcr.io/zigbee2mqtt/zigbee2mqtt-{arch}"` — образ
скачивается готовым, сборки нет. Бери тег, который уже лежит локально, —
тогда и качать нечего.

`/usr/share/hassio/homeassistant/zigbee2mqttN/configuration.yaml`. Логин и
пароль MQTT возьми из соседнего экземпляра, не выдумывай:

```yaml
version: 4
homeassistant:
  enabled: true
mqtt:
  base_topic: zigbee2mqttN
  server: mqtt://<брокер>:1883
  user: <из соседнего экземпляра>
  password: <из соседнего экземпляра>
  keepalive: 60
  reject_unauthorized: true
  version: 4
serial:
  port: tcp://<IP шлюза>:6638
  adapter: zstack
advanced:
  channel: <свободный канал>
  network_key: GENERATE
  pan_id: GENERATE
  ext_pan_id: GENERATE
frontend:
  enabled: true
  port: 8099
device_options: {}
devices: {}
```

- **Канал выбери сразу**, пока в сети нет устройств. Не совпадай с другими
  экземплярами; рядом с Wi-Fi лучше 15, 20, 25. Позже смена канала =
  перепривязка всех устройств. В пустой сети z2m переводит координатор сам:
  в логе `Channel changed to 'NN'`.
- `GENERATE` даёт уникальные ключ и pan_id. После первого запуска z2m впишет
  реальные значения — сверь pan_id с соседями.
- Порт фронтенда 8099 у всех экземпляров одинаковый — это нормально, наружу он не выставлен.

## 4. Установка и параметры

```sh
docker exec hassio_cli ha store reload
docker exec hassio_cli ha <A> install local_zigbee2mqttN
```

Затем одним POST на `/addons/local_zigbee2mqttN/options`:

```json
{"options": {"data_path": "/config/zigbee2mqttN",
             "socat": {"...": "как в config.json"},
             "mqtt": {"base_topic": "zigbee2mqttN", "server": "mqtt://<брокер>:1883",
                      "user": "<...>", "password": "<...>"},
             "serial": {}},
 "watchdog": true,
 "ingress_panel": true}
```

- `watchdog: true` — сторожевой таймер Supervisor. По умолчанию **выключен**.
- `ingress_panel: true` — пункт в боковой панели. По умолчанию **выключен**,
  хотя ingress-адрес выдаётся сразу; не принимай адрес за признак панели.

Запуск: `ha <A> start local_zigbee2mqttN`. CLI отваливается по таймауту 30 с
(первое создание сети идёт ~35 с) — это не ошибка, смотри `ha <A> logs`.

## 5. Автоматизация автозапуска — обязательно

Zigbee2MQTT при потере связи с координатором сам завершается
(`restart=false, code=2`). Сторожевой таймер делает короткую серию попыток
(около 9 за 2–3 минуты) и **сдаётся**. После отключения электричества шлюзы
часто возвращаются позже Home Assistant — экземпляр останется лежать, пока
его не запустят руками. Поэтому каждому экземпляру нужна страховка.

Сделай копию `automations.yaml` и допиши в конец:

```yaml
- id: z2m_N_autorestart
  alias: Z2M_Автозапуск Zigbee2MQTT N
  description: Страховка на случай, когда watchdog Supervisor сдался. Проверка каждые
    10 минут.
  triggers:
  - trigger: time_pattern
    minutes: /10
  conditions:
  - condition: state
    entity_id: binary_sensor.zigbee2mqtt_bridge_connection_state_N
    state: 'off'
    for: 00:05:00
  actions:
  - action: hassio.addon_start
    data:
      addon: local_zigbee2mqttN
    continue_on_error: true
  mode: single
```

Имя датчика не угадывай — возьми из реестра после первого запуска:
`grep -o 'binary_sensor.zigbee2mqtt_bridge_connection_state[_0-9]*' .storage/core.entity_registry | sort -u`.
Проверь, что YAML читается, и попроси перечитать автоматизации, если сам не можешь.
Если у старых экземпляров такой автоматизации нет — предложи добавить и им.

## 6. Проверки

- Лог: `Coordinator firmware version`, `Connected to MQTT server`, `Zigbee2MQTT started!`.
- MQTT: `zigbee2mqttN/bridge/state` = online; в `zigbee2mqttN/bridge/info`
  канал и pan_id не совпадают с другими экземплярами.
- HA: в `.storage/core.entity_registry` появились сущности нового моста.
- `ha <A> info local_zigbee2mqttN`: `state: started`, `watchdog: true`, `ingress_panel: true`.
- Сторожевой таймер реально работает: `docker stop <контейнер>` → через ~15 с
  контейнер снова `running`.
- Автоматизация загружена: `automation.z2m_*` для нового экземпляра в состоянии `on`
  (`.storage/core.restore_state`).

## 7. После

- Добавь экземпляр в таблицу в `.claude/ha-local.md` (шлюз, канал, pan_id,
  socat-порт, автоматизация).
- Напомни пользователю: локальное дополнение не обновляется само. Новая
  версия — поменять `version` в его `config.json`, `ha store reload`, `ha <A> update`.

## Разбор «экземпляр оказался не запущен»

1. `ls /usr/share/hassio/homeassistant/zigbee2mqttN/log/` — серия сессий с шагом
   15–18 с означает, что сторожевой таймер крутил попытки и сдался. Хвост
   последней сессии покажет причину: `EHOSTUNREACH`/`ETIMEDOUT` — шлюз пропал из сети.
   Хранится только 10 последних сессий.
2. Системный журнал может храниться меньше суток — используй историю HA:
   `home-assistant_v2.db` (SQLite, открывай `mode=ro`), таблицы `states_meta` и
   `states`, датчики `binary_sensor.zigbee2mqtt_bridge_connection_state*`.
   Все мосты упали в одну минуту — значит, пропадала сеть или питание, а не
   конкретный шлюз. Сравни, кто и когда вернулся.
3. Проверь, что у экземпляра есть автоматизация из шага 5 и что она включена.
