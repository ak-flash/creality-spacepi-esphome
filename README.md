# 🌀 SpacePi-ESPHome — управление сушилкой Creality SpacePi SE через эмуляцию энкодера

[![ESPHome](https://img.shields.io/badge/Made%20with-ESPHome-000000.svg?logo=esphome)](https://esphome.io/)
[![Home Assistant](https://img.shields.io/badge/Integrates%20with-Home%20Assistant-41BDF5.svg?logo=homeassistant)](https://www.home-assistant.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

*EN: Control a Creality SpacePi SE (DryerBox 2.0) filament dryer from Home Assistant by emulating its rotary encoder with an ESP32 and three optocouplers — no stock PCB replacement, no vendor firmware hacking.*

Управление температурой и временем сушилки филамента **Creality SpacePi SE** (плата `CREALITY DryerBox_2.0`) из Home Assistant / ESPHome **без замены родной платы и без вмешательства в её прошивку**: ESP генерирует сигналы энкодера и кнопки через оптопары, подключённые параллельно штатным контактам.

---

## ⚠️ Дисклеймер

- Проект неофициальный, с Creality не связан.
- Вскрытие корпуса и пайка = потеря гарантии.
- На плате сушилки присутствует **сетевое напряжение 220 В** (клеммник, X2-конденсатор, трансформатор). Все работы — только на полностью обесточенном устройстве, с разрядом конденсаторов.
- Вы всё делаете на свой страх и риск. Автор не отвечает за повреждение устройства, пожар и прочие последствия.

---

## ✨ Возможности

- поворот «энкодера» на N шагов в обе стороны из ESPHome / Home Assistant;
- эмуляция нажатия и длинного нажатия кнопки;
- макросы установки температуры и времени через меню сушилки;
- штатная ручка продолжает работать вручную (оптопары в покое = обрыв);
- полная гальваноразвязка: земли ESP и платы сушилки не соединяются.

---

## 🔧 Как это работает

Штатный энкодер сушилки — обычный EC11-тип с кнопкой:

| Выводы энкодера | Назначение | Уровни на плате |
|---|---|---|
| 3 в ряд, крайние | каналы `A` и `B` | 3.3 В в покое (подтяжка), замыкание на `C` = активный низкий |
| 3 в ряд, средний | общий `C` (GND) | 0 В |
| 2 напротив | кнопка `SW1`/`SW2` | ~2.6 В в покое (подтяжка через делитель/RC) |

Вместо того чтобы эмулировать логические уровни, мы **дублируем механические контакты** оптопарами: фототранзистор оптопары подключается параллельно контакту энкодера и замыкает те же две точки. Полярность, уровни и номиналы подтяжек при этом не важны.

Квадратура одного «клика» (замыкание = `turn_on`):

```
CW:  замкнуть A → замкнуть B → отпустить A → отпустить B
CCW: замкнуть B → замкнуть A → отпустить B → отпустить A
```

---

## 📦 Спецификация

| Позиция | Номинал / тип | Кол-во | Примечание |
|---|---|---|---|
| Микроконтроллер | ESP32 DevKit (любой с 3 свободными GPIO) | 1 | подойдёт и ESP8266 |
| Оптопара | PC817C (EL817, LTV817, TLP291) | 3 | или 1× PC847 / LTV-847 / TLP281-4 |
| Резистор | 1.0–1.2 кОм, 0.25 Вт | 3 | маркировка 1k: коричн-чёрн-красн; 1.2k: коричн-красн-красн |
| Питание ESP | 5 В (buck от площадки «+12V FAN» или отдельный БП) | 1 | AMS1117 от 12 В не рекомендуется (греется) |
| Провод монтажный | 0.2–0.3 мм | ~1 м | + термоклей / каптоновая лента |

Резисторы нужны **только на входной (светодиодной) стороне** оптопар. На выходной стороне резисторов нет.

---

## 🔌 Схема подключения

Полная векторная схема: [`docs/schema.svg`](docs/schema.svg)

```
  ESP32                 PC817 (точка = pin1)          Плата сушилки
  ─────                 ────────────────────          ─────────────
  GPIO25 ─[R1 1.2k]─ U1 pin1   U1 pin4 ────────────── A  (крайний из трёх)
  GPIO26 ─[R2 1.2k]─ U2 pin1   U2 pin4 ────────────── B  (другой крайний)
  GPIO27 ─[R3 1.2k]─ U3 pin1   U3 pin4 ────────────── SW1 (кнопка, вывод 1)

  ESP GND ─┬─ U1 pin2          U1 pin3 ─┐
           ├─ U2 pin2          U2 pin3 ─┼──────────── C  (средний из трёх)
           └─ U3 pin2          U3 pin3 ────────────── SW2 (кнопка, вывод 2)
```

### Нетлист (всего 9 цепей, других соединений нет)

| № | Цепь |
|---|---|
| 1 | GPIO25 — R1 — U1 pin 1 |
| 2 | GPIO26 — R2 — U2 pin 1 |
| 3 | GPIO27 — R3 — U3 pin 1 |
| 4 | ESP GND — U1 pin 2 — U2 pin 2 — U3 pin 2 |
| 5 | U1 pin 4 — вывод энкодера A |
| 6 | U2 pin 4 — вывод энкодера B |
| 7 | U3 pin 4 — вывод кнопки SW1 |
| 8 | U1 pin 3 — U2 pin 3 — вывод энкодера C |
| 9 | U3 pin 3 — вывод кнопки SW2 |

**Земли ESP и платы сушилки НЕ соединяются.**

### Как найти выводы на плате (проверка перед пайкой, обесточено!)

1. Прозвонка двух выводов кнопки: нажатие на ручку → КЗ.
2. Прозвонка среднего вывода из трёх с крайними: медленное вращение → прерывистое КЗ.
3. Прозвонка среднего вывода с GND платы (клемма «12V-», минус конденсаторов) → КЗ: значит `C = GND`.
4. Под напряжением (осторожно!): крайние из трёх = 3.3 В в покое, «скачут» при вращении (это норма, мультиметр не успевает за импульсами); кнопка в покое ~2.6 В, при нажатии падает.

Паяться тонким проводом к паяным точкам выводов энкодера с нижней стороны платы; провода зафиксировать термоклеем и увести от сетевой части.

---

## 🧪 Проверка приставки до установки

Без питания сушилки подайте 3.3 В на GPIO25 через R1: мультиметр в режиме прозвонки между выводами A и C на стороне платы должен показать КЗ; без подачи — обрыв. Повторить для каналов B и кнопки.

---

## 📟 Прошивка

Готовый конфиг: [`firmware/spacepi-se-encoder.yaml`](firmware/spacepi-se-encoder.yaml)

```yaml
esphome:
  name: spacepi-se-encoder
  friendly_name: SpacePi SE Encoder Control

esp32:
  board: esp32dev
  framework:
    type: arduino

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

logger:
api:
  encryption:
    key: !secret api_key
  services:
    - service: rotate_cw
      variables:
        num_steps: int
      then:
        - script.execute: { id: rotate_cw, steps: !lambda 'return num_steps;' }
    - service: rotate_ccw
      variables:
        num_steps: int
      then:
        - script.execute: { id: rotate_ccw, steps: !lambda 'return num_steps;' }
    - service: press_button
      then:
        - script.execute: button_click

ota:
  password: !secret ota_password

button:
  - platform: template
    name: "Encoder CW 1 step"
    on_press: { script.execute: { id: rotate_cw, steps: 1 } }
  - platform: template
    name: "Encoder CCW 1 step"
    on_press: { script.execute: { id: rotate_ccw, steps: 1 } }
  - platform: template
    name: "Press button"
    on_press: { script.execute: button_click }

output:
  - platform: gpio
    id: enc_a          # оптопара между A и C
    pin: GPIO25
  - platform: gpio
    id: enc_b          # оптопара между B и C
    pin: GPIO26
  - platform: gpio
    id: btn            # оптопара между SW1 и SW2
    pin: GPIO27

script:
  - id: rotate_cw
    mode: queued
    parameters:
      steps: int
    then:
      - repeat:
          count: !lambda 'return steps;'
          then:
            - output.turn_on:  enc_a
            - delay: 15ms
            - output.turn_on:  enc_b
            - delay: 15ms
            - output.turn_off: enc_a
            - delay: 15ms
            - output.turn_off: enc_b
            - delay: 30ms

  - id: rotate_ccw
    mode: queued
    parameters:
      steps: int
    then:
      - repeat:
          count: !lambda 'return steps;'
          then:
            - output.turn_on:  enc_b
            - delay: 15ms
            - output.turn_on:  enc_a
            - delay: 15ms
            - output.turn_off: enc_b
            - delay: 15ms
            - output.turn_off: enc_a
            - delay: 30ms

  - id: button_click
    mode: queued
    then:
      - output.turn_on: btn
      - delay: 150ms
      - output.turn_off: btn
      - delay: 400ms
```

### Первичная проверка

1. Прошить, открыть веб-интерфейс / HA.
2. `Encoder CW 1 step` → значение на экране сушилки меняется на один шаг.
3. Если меняется в минус — использовать `rotate_ccw` как «вперёд» или поменять местами провода A/B.
4. `Press button` → вход в меню / подтверждение.

---

## 🏠 Home Assistant

Сервисы устройства:

```yaml
service: esphome.spacepi_se_encoder_rotate_cw
data:
  num_steps: 4

service: esphome.spacepi_se_encoder_press_button
```

Пример макроса установки параметров (последовательность меню снимается вручную для вашей ревизии прошивки и зашивается сюда):

```yaml
script:
  - id: macro_set_temp
    mode: queued
    parameters:
      clicks: int
    then:
      - script.execute: button_click     # вход в меню
      - delay: 700ms
      - script.execute: { id: rotate_cw, steps: !lambda 'return clicks;' }
      - delay: 300ms
      - script.execute: button_click     # подтверждение
      - delay: 500ms
```

> 💡 У эмуляции энкодера нет обратной связи от дисплея. Перед каждым макросом рекомендуется приводить меню в известное состояние (например, длинным нажатием / выходом в главный экран), иначе автоматика может начать менять не тот параметр.

---

## 🐛 Troubleshooting

| Симптом | Причина / решение |
|---|---|
| Значение не меняется | Увеличить `delay` до 30–50 мс; проверить прозвонкой каналы; проверить, что эмиттер/коллектор припаяны к нужной паре контактов |
| Значение меняется в обратную сторону | Поменять `rotate_cw`/`rotate_ccw` местами или провода A/B |
| Значение прыгает через шаг | Энкодер даёт полцикла на детент: уменьшить последовательность или увеличить задержки |
| Кнопка не срабатывает | Увеличить длительность до 200–300 мс |
| Работает через раз | Плохая пайка, холодные контакты оптопары, слишком длинные провода рядом с сетевой частью |
| Сушилка «сама» меняет значения | Оптопара припаяна не к той паре контактов (например, эмиттер на C, коллектор на соседний вывод) |

---

## 🗂 Структура репозитория

```
.
├── README.md
├── LICENSE
├── docs/
│   └── schema.svg              # точная схема подключения
├── firmware/
│   └── spacepi-se-encoder.yaml # конфиг ESPHome
└── images/
    ├── board_bottom.jpg        # нижняя сторона платы, точки пайки
    └── board_top.jpg           # верхняя сторона (сетевая часть — не трогать!)
```

---

##  TODO / идеи

- [ ] Обратная связь: чтение дисплея или sniff UART для контроля реальных значений
- [ ] Компонент ESPHome `number` с пересчётом «градусы ↔ клики» и автосбросом меню
- [ ] Датчик температуры/влажности внутри камеры для автоматических профилей сушки
- [ ] Англоязычная версия README

---

## 🤝 Contribution

PR и issues приветствуются. Особенно ценны: карты меню других ревизий прошивки (`Dryer2_0`, `DryerBox_2.0`), осциллограммы энкодера, фото точек пайки.

---

## 📄 License

MIT — см. [LICENSE](LICENSE).