# Настройка OrcaSlicer под IDEX 400×400×400 (REV.3.2-RC1)

## 1. Базовые настройки принтера в OrcaSlicer
- **Рабочая область (Printable area):** `400 × 400 мм`, максимальная высота `400 мм`.
- **Количество экструдеров (Extruders):** `2` (`T0` — левый, `T1` — правый).
- **Смещения экструдеров (Extruder offset):** оставить `X = 0, Y = 0` для обоих экструдеров (компенсация смещения `T1` выполняется на стороне Klipper через `_IDEX_OFFSET`).
- **Очистительная башня (Prime Tower):** **отключить** (не требуется при двух независимых головах с парковочными доками).
- **Поддержка нескольких материалов:** включить `Enable` в разделе `Multimaterial`, режим смены — без продувочной башни.

## 2. Стартовый G-code (`Machine start G-code`)
```gcode
M140 S[bed_temperature_initial_layer_single] ; Нагрев стола
G28                                          ; Базовый хоуминг всех осей
IDEX_INIT                                    ; Парковка обеих голов в доки и активация safe_distance
M190 S[bed_temperature_initial_layer_single] ; Ожидание прогрева стола
Z_TILT_ADJUST                                ; Выравнивание плоскости стола по 3 винтам
G28 Z                                        ; Повторный хоуминг Z после выравнивания
BED_MESH_CALIBRATE ADAPTIVE=1                ; Адаптивная сетка стола под зону печати

; Прогрев только используемых в детали экструдеров
{if is_extruder_used[0]}
M104 T0 S[nozzle_temperature_initial_layer]
{endif}
{if is_extruder_used[1]}
M104 T1 S[nozzle_temperature_initial_layer]
{endif}
{if is_extruder_used[0]}
M109 T0 S[nozzle_temperature_initial_layer]
{endif}
{if is_extruder_used[1]}
M109 T1 S[nozzle_temperature_initial_layer]
{endif}

T[initial_extruder]                          ; Активация стартового инструмента
G90
```

## 3. G-code смены инструмента (`Change filament G-code`)
Позволяет снижать температуру неактивного сопла в парковочном доке, предотвращая подтекание и перегрев пластика.
```gcode
; Снижение температуры уходящей головы в режиме ожидания (на 35 °C ниже рабочей)
{if previous_extruder != -1}
M104 T[previous_extruder] S{nozzle_temperature[previous_extruder] - 35}
{endif}

; Догрев следующей головы до рабочей температуры перед выездом из дока
M104 T[next_extruder] S[nozzle_temperature[next_extruder]]
T[next_extruder]
M109 T[next_extruder] S[nozzle_temperature[next_extruder]]
```

## 4. Завершающий G-code (`Machine end G-code`)
```gcode
G91
G1 E-2.0 F1800                               ; Короткий ретракт
G90
IDEX_INIT                                    ; Парковка обеих кареток в силиконовые доки
TURN_OFF_HEATERS                             ; Выключение стола и обоих хотэндов
M107                                         ; Выключение вентиляторов обдува
; Примечание: моторы Z остаются на удержании (управляется секцией [idle_timeout])
```