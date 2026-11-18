# CG2028 — ElderCare Wearable Safety Companion

Bare-metal C and ARM Thumb assembly firmware for the **B-L4S5I-IOT01A** Discovery board (STM32L4S5VIT6, Cortex-M4). Detects falls and prolonged stillness using the onboard LSM6DSL IMU, signals alarms locally, and uploads event telemetry to ThingsBoard.

## Technical features

- **Six-axis sensing:** accelerometer ±8 g and gyroscope ±2000 °/s, both configured at 208 Hz. A single 12-byte I2C2 burst reads all axes using auto-increment, block-data update, and cached sensitivity factors; readings are converted from mg/mdps to g/°/s.
- **Assembly EWMA filter:** `ewma_filter(new, previous, alpha)` computes `(alpha × new + (100 − alpha) × previous) / 100` with signed integer arithmetic and truncation toward zero. Uses `MUL`, `MLA`, and `SDIV`, only R0–R2, and no stack or floating-point operations. Independent C reference filters check all six outputs every loop and report mismatches over UART. Acceleration uses α = 50%; gyro uses α = 15%.
- **Fall detection:** a four-state machine uses filtered vector magnitudes and independent persistence timers for low acceleration and rapid rotation. A 500 ms settling period suppresses triggers after startup or acknowledgement.
- **Local alarm:** SysTick updates LED2 every 1 ms; its toggle interval is 1000 ms in `NORMAL`/`CONFIRMING` and 150 ms in either alarm state. Blinking continues during blocking UART/Wi-Fi calls. Hold the active-low USER button for more than 2 seconds to acknowledge an alarm.
- **Diagnostics:** USART1 over ST-LINK outputs state, magnitudes, and XYZ readings every 20 loops. Optional timing instrumentation reports average loop period/rate over 1000 normal-state loops, excluding interrupted batches and the timing summary itself.

## Detection rules

All thresholds apply to filtered magnitudes.

| Transition | Condition |
| --- | --- |
| `NORMAL → CONFIRMING` | Acceleration < 0.60 g **or** rotation > 100 °/s, continuously observed for ≥70 ms. |
| `CONFIRMING → FALL_CONFIRMED` | Acceleration ≥3.00 g before the 1000 ms confirmation window expires. |
| `CONFIRMING → NORMAL` | Confirmation window expires without an impact. |
| `FALL_CONFIRMED → LONG_LIE` | Acceleration 0.80–1.20 g and rotation <50 °/s continuously observed for ≥30 seconds; movement restarts the timer. |
| Either alarm → `NORMAL` | USER held for >2 seconds; alarms otherwise remain latched. |

## ThingsBoard telemetry

The onboard ISM43362 Wi-Fi module uses SPI3 with an EXTI1 data-ready callback. After Wi-Fi connection and DNS resolution, each fall, long-lie, or acknowledgement sends one JSON HTTP POST to `/api/v1/<token>/telemetry` over a fresh TCP connection.

- Fall: `state`, `acceleration_g`, `angular_velocity_dps`, `detected_at_uptime_ms` (milliseconds since boot).
- Long lie / acknowledgement: `state` only (`LONG_LIE` / `NORMAL`), preserving the previous fall snapshot.
- Delivery requires HTTP 200; the client handles fragmented status lines, checks buffer lengths and bytes sent, and closes each opened socket. Failed setup/delivery triggers network reinitialization on the next event; failed events are not queued for replay.

Local detection remains available without a network. Uploads block sampling and button polling; acknowledgement/stillness timers restart after uploads so those gaps do not count as observed holds. Transport is plain HTTP on port 80, without TLS. Server-side dashboards and alarm rules require separate ThingsBoard configuration.

## Build and run

1. Import the `FallDetector` folder as an existing project in **STM32CubeIDE**. If an older project is already imported, remove its workspace entry without deleting its files, then import this renamed folder. HAL, CMSIS, and BSP sources are included.
2. For telemetry, set Wi-Fi credentials and the device token/host in `Core/Inc/thingsboard.h`. An empty SSID or token skips networking; keep credentials private.
3. Select the **Debug build configuration** and build. The tracked `.project`/`.cproject` files include the Wi-Fi headers and floating-point `printf` support. Connect the board through its ST-LINK USB port and create the local debug launch below to flash/run.
4. Open the ST-LINK virtual COM port at **115200 baud, 8N1, no flow control**. Observe readings and LED2; hold USER to clear a latched alarm.

### Local debug configuration

The `.launch` file and two generated OpenOCD `.cfg` files are intentionally ignored. On a fresh clone, open **Run → Debug Configurations → STM32 C/C++ Application**, create a configuration, and use:

| Setting | Value |
| --- | --- |
| Project / build | `FallDetector` / `Debug`; build before launch. |
| Application | `Debug/FallDetector.elf`. |
| Target | B-L4S5I-IOT01A / STM32L4S5VITx. |
| Debug probe / interface | **ST-LINK (ST-LINK GDB server)** / **SWD**, automatic/default frequency. |
| Server / debugger | Start local server; GDB port `61234`; `arm-none-eabi-gdb`. |
| Reset / startup | Software system reset; download executable and symbols; stop at `main`. |

Click **Debug** to flash and halt at `main`, then **Resume** to run. OpenOCD is an alternative: select **ST-LINK (OpenOCD)** and automatic configuration generation for this target (SWD, 8000 kHz, software system reset, GDB port `3333`); CubeIDE generates the `.cfg` locally. The previous scripts enabled low-power debugging and stopped watchdogs while halted. No copied machine-specific paths are required.

Tune thresholds, EWMA weights, and reporting in `Core/Src/main.c`. Set `REPORT_DISABLE=1` to suppress routine readings and `SAMPLE_TIMING_ENABLE=1` to measure loop timing. The 2 ms pacing delay does **not** establish a fixed sample rate: sensor ODR is 208 Hz, but acquisition is polling-based and UART/network work adds latency. I2C read failures retain the previous readings.

## Source map

| Path | Purpose |
| --- | --- |
| `Core/Src/main.c` | Acquisition, C reference filter, state machine, button, UART, and timing. |
| `Core/Src/mov_avg.s` | ARM assembly EWMA implementation. |
| `Core/Src/thingsboard.c`, `Core/Inc/thingsboard.h` | HTTP telemetry and network configuration. |
| `Core/Src/stm32l4xx_it.c` | SysTick LED service and Wi-Fi interrupt handling. |
| `Drivers/BSP/` | Board support, modified LSM6DSL acquisition, and Wi-Fi drivers. |
| `Core/Startup/`, `STM32L4S5VITX_*.ld` | Startup code and Flash/RAM linker scripts. |
