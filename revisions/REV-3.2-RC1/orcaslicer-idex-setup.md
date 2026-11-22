# Настройка OrcaSlicer под IDEX 400×400×400 (REV.3.2-RC1)

## 1. Базовые настройки принтера в OrcaSlicer
- **Рабочая область (Printable area):** `400 × 400 мм`, максимальная высота `400 мм`.
- **Количество экструдеров (Extruders):** `2` (`T0` — левый, `T1` — правый).
- **Смещения экструдеров (Extruder offset):** оставить `X = 0, Y = 0` для обоих экструдеров (компенсация смещения `T1` выполняется на стороне Klipper через `_IDEX_OFFSET`).
- **Очистительная башня (Prime Tower):** **отключить** (не требуется при двух независимых головах с парковочными доками).
- **Поддержка нескольких материалов:** включить `Enable` в разделе `Multimaterial`, режим смены — без продувочной башни.

## 2. Стартовый G-code (`Machine start G-code`)
```gcode
M140 S[bed_temperature_initial_layer_single] ; Предварительный нагрев стола
G28                                          ; Базовый хоуминг всех осей (G28 Z выполняется в центре 200,200 головой T0)
IDEX_INIT                                    ; Парковка обеих голов в доки и активация safe_distance
M190 S[bed_temperature_initial_layer_single] ; Ожидание прогрева стола
Z_TILT_ADJUST                                ; Выравнивание плоскости стола по 3 винтам
G28 Z                                        ; Повторный хоуминг Z после выравнивания
BED_MESH_CALIBRATE                           ; Штатное построение сетки стола (для ADAPTIVE=1 требуется настройка [exclude_object])
IDEX_INIT                                    ; Возврат T0 в парковочный док после замера сетки перед нагревом сопел

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
Температура ожидания неактивной головы в доке (`idle_temperature`) подбирается индивидуально под тип пластика (для PLA/PETG/ABS снижение обычно составляет 30–40 °C, чтобы избежать деградации полимера в сопле).
```gcode
{if previous_extruder != -1}
M104 T[previous_extruder] S{idle_temperature[previous_extruder]}
{endif}

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
```