# Polaris Flight Computer

An ESP32-S2-based rocket flight computer with onboard sensing, flight-data logging, main and drogue deployment outputs, and two-servo control. This repository includes the Arduino firmware, KiCad hardware design, and an OpenRocket model.

## Hardware

| Component | Purpose |
| --- | --- |
| ESP32-S2-SOLO-2-N4 | Microcontroller module on the custom PCB |
| BMI088 | Accelerometer and gyroscope |
| BMP280 | Pressure, temperature, and barometric altitude |
| microSD card | Flight logs and configuration file |
| Main and drogue circuits | Deployment control and continuity sensing |
| Servo outputs | Two-axis thrust-vector-control code |
| Buzzer | Startup and continuity feedback |

The KiCad PCB uses two copper layers. The firmware uses separate SPI buses for the sensors and SD card. Pin assignments are in [`src/pinout.h`](src/pinout.h).

Four servo pins are defined, but the current controller attaches two servos. Their center positions, deflection limits, and PWM settings are in `src/servo_controller.cpp`.

## Firmware

The application is written in C++ using the Arduino framework and PlatformIO. It reads the IMU at a nominal rate of about 100 Hz and the barometer at 20 Hz.

The main loop handles:

- Acceleration, angular-rate, pressure, and temperature readings.
- Orientation estimation with the included Fusion library, using the IMU without a magnetometer.
- Launch, post-apogee, and landed-state detection from sensor readings and altitude history.
- Main and drogue deployment triggers based on timers, apogee, or descent altitude.
- Buffered CSV logging to the SD card and serial output.
- Servo deflection calculated from estimated yaw and pitch.

The servo routine is called continuously from the main loop rather than being enabled only after launch. A separate Madgwick implementation is also included, but the active application uses Fusion.

## Data logging

Each startup creates a numbered file such as `log-0.csv` or `log-1.csv`. The log contains:

- Time in microseconds, pressure, temperature, and altitude.
- Yaw, pitch, roll, acceleration, and angular rate.
- Main and drogue continuity readings.
- Flight state and deployment timestamps.

The code reduces SD writes before launch and after landing, and drains buffered records more frequently during flight. Altitude is referenced to the launch ground level once launch is detected.

The CSV header labels acceleration as m/s², while the BMI088 driver returns values in milli-g. Account for that when analyzing the logs.

## Building and uploading

1. Install PlatformIO, either through its editor extension or command-line tools.
2. Open the repository folder containing `platformio.ini`.
3. Check the board selection and the wiring in `src/pinout.h` against your hardware.
4. Build and upload with:

   ```bash
   pio run
   pio run --target upload
   ```

5. Open the serial monitor at the baud rate used by the firmware:

   ```bash
   pio device monitor --baud 115200
   ```

The checked-in PlatformIO environment is `esp32-s2-saola-1`, with the Arduino framework and USB CDC enabled. The custom schematic uses an ESP32-S2-SOLO-2-N4 module, so verify the board and upload settings for the assembled PCB.

PlatformIO declares `ESP32Servo` version `^1.1.3` as a dependency. Fusion is included under `lib/Fusion/`.

## SD configuration

The firmware looks for `config.txt` in the SD card's root directory. The intended format is one `key=value` setting per line. Recognized settings include:

| Setting | Purpose |
| --- | --- |
| `launch_detection_threshold` | Altitude-rise threshold in meters |
| `launch_detection_min_g` | Minimum acceleration threshold in g |
| `main_timer_ms`, `drogue_timer_ms` | Deployment timers measured from launch |
| `main_fire_duration_ms`, `drogue_fire_duration_ms` | Output activation durations |
| `main_altitude_meters`, `drogue_altitude_meters` | Descent deployment altitudes |
| `main_on_apogee`, `drogue_on_apogee` | Enable deployment at detected apogee |

Defaults are defined near the top of `src/main.cpp`. No configuration file is included in the repository. The current parser uses fixed substring offsets; check the parsed values through serial output before relying on a configuration file.

## Repository layout

| Path | Contents |
| --- | --- |
| `src/` | Main application, sensor drivers, pin definitions, buffers, and servo control |
| `lib/Fusion/` | Orientation-estimation library |
| `chronos-polaris-hardware/` | KiCad project, schematic, PCB, and STEP model |
| `chronos-polaris-hardware/JLC2KiCad_lib/` | Custom symbols, footprints, and 3D models |
| `Polaris Program/Arcturus.ork` | OpenRocket model |
| `platformio.ini` | Firmware build configuration |

## Opening the hardware

Open `chronos-polaris-hardware/chronos-polaris-hardware.kicad_pro` in KiCad. The PCB was saved with KiCad 8.0. The project library tables use paths relative to the project directory, so keep the included `JLC2KiCad_lib` folder alongside the design files.

Open `Polaris Program/Arcturus.ork` in OpenRocket to inspect the included rocket model.
