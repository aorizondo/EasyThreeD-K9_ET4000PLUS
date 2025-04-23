# EasyThreeD-K9_ET4000PLUS
Marlin firmware for 3D Printer brand EasyThreeD model K9 board ET4000PLUS

## This is a hard fork from https://github.com/schmttc/EasyThreeD-K7-STM32
After search and analize the public information about EasyThreeD printers I finally found a repo that has
all the info needed for customize the firmware of these kind of printers.
If you have the intention of easy-upgrade your printer this **IS NOT** a recommended repo for you. The
intent here primary is to upgrade to Marlin 2.x release and activate features for self-leveling, power-failure recovery
and resuming, and full USB control. In first instant the printer interface is dropped so do not expect LCD, wifi or
buttons to work.

# Installation
The board's bootloader is proprietary by MKS, which reads a binary firmware file mksLite.bin from the SD card on boot.
Firmware files are found in config/EasyThreeD/ET4000PLUS/ under the folder of the printer model
- Make a copy of mksLite.CUR from your SD card. This is the original firmware, and is required if you run into any issues
- Copy mksLite.bin to the SD card, and restart the printer
- After a short time (<30s) the firmware is written to the board, and mksLite.bin on the SD card is renamed to mksLite.CUR

## Overview
- Compile using PlatformIO, board "mks_robin_lite_maple"
- Physical buttons and LED currently are functional as per the standard manufacturer's behaviour
- Start button LED
    - LED blinks slowly when printing/processing
    - When paused blinks LED quickly
    - LED is on when job is cancelled or completed
- Home button and filament feed/retract slider working
- Long press print button to raise print head 10mm while not printing
- Short press print button to print most recent file on SD card
- Serial baud rate is set to 115200, matching the original firmware
- Onboard EEPROM is enabled, matching the original firmware

## Additional Files
Compiled binary - mksLite.bin
- Hotbed is enabled. If you do not have a hotbed, make sure the temp is set to 0 in your slicer
- Backlash correction is enabled
- Input Shaping: Disabled

## Notes on Marlin 2 Config
- Make sure 'VALIDATE_HOMING_ENDSTOPS' is disabled, as we do not have X and Y stoppers to provide feedback, and the printer will halt.
- Multiple calls in quick succession to queue.inject_P() will fail. Use a single call, with multiple commands seprated by "\n"
- Setting acceleration of around 100 or higher may result in layer shifting when backlash compensation is enabled (see https://github.com/schmttc/EasyThreeD-K7-STM32/issues/2 )

## License
Marlin Firmware: https://github.com/MarlinFirmware/Marlin

Marlin is published under the [GPL license](https://github.com/COPYING.md) because we believe in open development. The GPL comes with both rights
and obligations. Whether you use Marlin firmware as the driver for your open or closed-source product, you must keep
Marlin open, and you must provide your compatible Marlin source code to end users upon request. The most straightforward
way to comply with the Marlin license is to make a fork of Marlin on Github, perform your modifications, and direct
users to your modified fork.

While we can't prevent the use of this code in products (3D printers, CNC, etc.) that are closed source or crippled by a
patent, we would prefer that you choose another firmware or, better yet, make your own.