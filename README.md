# 28-Light Handheld Lux Meter Arduino Code

This repository contains the Arduino/ESP32 code for our **Handheld Lux Meter** proof-of-concept prototype.

Our device was designed as a low-cost system for measuring and documenting light trespass in urban environments. The prototype uses an **ESP32**, **VEML7700 ambient light sensor**, **SSD1306 OLED display**, **microSD card module**, and **Bluetooth Serial** to collect, store, view, and export lux data.

## Features

- Real-time lux measurement using the VEML7700 sensor
- OLED menu interface
- Data logging to microSD card
- Automatic log file creation (`LOG_001.TXT`, `LOG_002.TXT`, etc.)
- On-device file browser
- On-device log viewer
- Bluetooth log export
- SD card wipe confirmation
- Running average and percent deviation calculations

## Hardware Used

- ESP32
- VEML7700 ambient light sensor
- SSD1306 OLED display (128x32, I2C)
- microSD card module
- 4 push buttons
- USB power input

## Required Libraries

Make sure the following libraries are installed in the Arduino IDE:

- `Wire`
- `SPI`
- `SD`
- `Adafruit_GFX`
- `Adafruit_SSD1306`
- `SparkFun_VEML7700 Arduino Library`
- `BluetoothSerial`

## Pin Configuration

### Buttons
- **Back / Power**: GPIO 27
- **Select**: GPIO 26
- **Up**: GPIO 25
- **Down**: GPIO 33

### SD Card
- **Chip Select**: GPIO 5

### OLED Display
- **I2C Address**: `0x3C`

## Main Menu Options

The device menu currently includes:

1. **Record Data**
2. **Data Sets**
3. **Clear SD Card**
4. **Export via BT**

## Logging Format

Each log file is saved as a `.TXT` file in the format:

```text
Read_Num, Lux_Value, Pct_Deviation

Code by Marko Nguyen for 28-Light.
