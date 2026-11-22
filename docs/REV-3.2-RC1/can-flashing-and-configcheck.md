# Прошивка CAN-шины (Katapult + Klipper) и проверка конфига (REV.3.2-RC1)

## 1. Настройка интерфейса can0 на Linux-хосте (1 Мбит/с)
Создать файл `/etc/network/interfaces.d/can0`:
```bash
sudo tee /etc/network/interfaces.d/can0 << 'EOF'
allow-hotplug can0
iface can0 can static
    bitrate 1000000
    up ifconfig \$IFACE txqueuelen 1024
EOF
sudo ifup can0
ip -details link show can0
```

## 2. Прошивка BTT Octopus v1.1 (USB-to-CAN bridge)
В режиме `USB-to-CAN bridge` основная плата прошивается через USB DFU (`BOOT0` + `RESET`).
Параметры `make menuconfig` в `~/klipper`:
- **Micro-controller Architecture:** `STMicroelectronics STM32`
- **Processor model:** `STM32F446` (кварц `12 MHz`) или `STM32F429` (кварц `8 MHz` — сверить маркировку чипа на плате!)
- **Bootloader offset:** `32KiB bootloader`
- **Communication interface:** `USB to CAN bus bridge (USB on PA11/PA12)`
- **CAN bus interface:** `CAN bus (on PD0/PD1)` (разъём RJ12)
- **CAN bus speed:** `1000000`

Команда прошивки в режиме DFU (`0483:df11`):
```bash
make flash FLASH_DEVICE=0483:df11
```

## 3. Прошивка плат голов BTT EBB36 v1.2 (Katapult + Klipper)
1. **Сборка загрузчика Katapult (`STM32G0B1`, кварц `8 MHz`):**
   - **Communication interface:** `CAN bus (on PB0/PB1)`
   - **Application start offset:** `8KiB offset`
   - **CAN bus speed:** `1000000`
   - **Enable Status LED:** `PA13`
   - Прошивка по USB DFU (с установленной перемычкой `VUSB`, по одной плате за раз):
     ```bash
     sudo dfu-util -a 0 -D ~/katapult/out/katapult.bin --dfuse-address 0x08000000:force:mass-erase:leave -d 0483:df11
     ```
   - После прошивки **обязательно снять перемычку `VUSB`** перед подключением 24В по CAN!

2. **Сборка и заливка Klipper по шине CAN:**
   - **Micro-controller Architecture:** `STMicroelectronics STM32`
   - **Processor model:** `STM32G0B1`, **Bootloader offset:** `8KiB bootloader`, **Clock:** `8 MHz crystal`
   - **Communication interface:** `CAN bus (on PB0/PB1)`, **Speed:** `1000000`
   - Поиск UUID и прошивка каждой платы:
     ```bash
     ~/klippy-env/bin/python ~/klipper/scripts/canbus_query.py can0
     python3 ~/katapult/scripts/flashtool.py -i can0 -f ~/klipper/out/klipper.bin -u <UUID_ПЛАТЫ>
     ```

## 4. Проверка конфига и терминаторов
- Сопротивление между `CAN_H` и `CAN_L` при выключенном питании должно составлять **~60 Ом** (резистор 120 Ом у разъёма RJ12 Octopus + джампер `120R` на конечной EBB36; на промежуточной EBB36 джампер `120R` снят).
- Проверка парсера Klipper на Linux-хосте (при подключённых MCU или в режиме отладки):
  ```bash
  ~/klippy-env/bin/python ~/klipper/klippy/klippy.py ~/printer_data/config/printer.cfg -o /tmp/test.gcode -l /tmp/klippy.log
  ```