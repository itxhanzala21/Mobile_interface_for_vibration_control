# Mobile Interface for Vibration Control

A real-time wireless sensor data acquisition and visualization system built around an STM32 microcontroller. A custom Android app streams live accelerometer data over Bluetooth, the STM32 relays it to a PC over serial, and a Python script plots it live in 2D and 3D.

> Built for the Microprocessor & Interfacing course at PIEAS, Department of Electrical Engineering.

## Overview

This project demonstrates a complete sensor-to-visualization pipeline:

```
Smartphone (accelerometer) --Bluetooth--> HC-05 --USART1--> STM32 --USART2--> PC --> Python (live plot)
```

- A phone's built-in accelerometer is read by a custom **MIT App Inventor** app and streamed over Bluetooth.
- An **HC-05 Bluetooth module** receives that stream and feeds it to the STM32's USART1.
- The **STM32F103C8T6** receives the data using interrupt-driven UART (no polling, no blocking) and immediately forwards it out over USART2.
- A **PC** reads USART2 over serial and a **Python** script parses and plots the X/Y/Z acceleration values in real time.

The goal was to keep every stage interrupt-driven and non-blocking end-to-end, so the system stays responsive even under continuous data flow.

## Features

- Real-time wireless transmission of 3-axis accelerometer data
- Custom Android app (MIT App Inventor) for reading phone sensors and sending data over Bluetooth
- Interrupt-based UART reception and transmission on the STM32 (HAL library, no polling loops)
- Live 2D and 3D plotting of incoming data on a PC using Python
- Simple, reproducible hardware setup (breadboard-friendly, no custom PCB required)

## Hardware Used

| Component | Purpose |
|---|---|
| STM32F103C8T6 ("Blue Pill") | Receives Bluetooth data, forwards it to PC |
| HC-05 Bluetooth module | Wireless link between phone and STM32 |
| Smartphone (Android) | Runs the custom app, provides accelerometer data |
| ST-Link / debugger | Flashing and powering the STM32 |
| Breadboard + jumper wires | Wiring between HC-05 and STM32 |
| PC | Runs the serial monitor and Python plotting script |

## Software Used

- **STM32CubeMX** – peripheral configuration and code generation
- **Keil MDK** – firmware development and flashing
- **MIT App Inventor** – mobile app for sensor streaming
- **Python 3** with `pyserial` and `matplotlib` (or `plotly`) – serial reading and live plotting
- A serial terminal such as **PuTTY** or **Tera Term** – for raw data inspection

## How It Works

### 1. Hardware wiring

- HC-05 `TX` → STM32 `RX` (USART1)
- HC-05 `RX` → STM32 `TX` (USART1)
- STM32 `USART2` → PC (via ST-Link's virtual COM port or a USB-to-serial adapter)
- Pair the phone with the HC-05 module before running the app.

### 2. STM32 firmware

Two USART peripherals are configured in STM32CubeMX:

- **USART1** — receive-only, connected to the HC-05, 9600 baud. Configured for interrupt-based reception (`HAL_UART_Receive_IT`), so the MCU is never blocked waiting for data.
- **USART2** — transmit/receive, connected to the PC, 9600 baud. Used to forward whatever arrives on USART1 straight to the PC.

The reception callback (`HAL_UART_RxCpltCallback`) sets a flag when a full packet (13 bytes) has arrived; the main loop checks that flag and forwards the packet over USART2, then immediately re-arms USART1 to receive the next packet. An error callback re-arms reception if a framing/overrun error occurs, so the link self-recovers instead of stalling.

<details>
<summary>Core firmware logic (click to expand)</summary>

```c
#include "main.h"

UART_HandleTypeDef huart1;
UART_HandleTypeDef huart2;

uint8_t rx_data[13];
volatile uint8_t rx_complete = 0;

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
  if (huart->Instance == USART1)
  {
    rx_complete = 1;
    HAL_UART_Receive_IT(&huart1, rx_data, 13);
  }
}

void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart)
{
  if (huart->Instance == USART1)
  {
    HAL_UART_Receive_IT(&huart1, rx_data, 13);
  }
}

int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();
  MX_USART1_UART_Init();
  MX_USART2_UART_Init();

  HAL_UART_Receive_IT(&huart1, rx_data, 13);

  while (1)
  {
    if (rx_complete)
    {
      HAL_UART_Transmit(&huart2, rx_data, 13, HAL_MAX_DELAY);
      rx_complete = 0;
    }
  }
}
```

Full peripheral/clock/GPIO initialization code is in the firmware source under this repo's STM32 project folder.

</details>

### 3. Mobile app

A small MIT App Inventor app reads the phone's accelerometer sensor block and continuously sends the X, Y, Z values as a formatted string over the paired Bluetooth connection to the HC-05.

### 4. PC-side visualization

A Python script opens the PC's serial port with `pyserial`, reads and parses each incoming packet, and updates a live plot (2D line chart and/or 3D scatter) of the three acceleration axes as new data arrives.

## Getting Started

1. **Wire the hardware** as described above and connect the ST-Link debugger.
2. **Open the STM32CubeMX project**, generate code, and build/flash it from Keil MDK.
3. **Install the mobile app** on an Android phone and pair it with the HC-05 (default pairing code is usually `1234` or `0000`).
4. **Install Python dependencies** on the PC:
   ```bash
   pip install pyserial matplotlib
   ```
5. **Run the plotting script**, pointing it at the correct COM port / baud rate (9600):
   ```bash
   python plot_accelerometer.py --port COM5 --baud 9600
   ```
6. Move the phone around — you should see the live plot update in real time.

> Update the script name/arguments above to match whatever your plotting script is actually called in this repo.

## Results

The system successfully streamed accelerometer data end-to-end with low latency. Occasional minor delays or dropped packets were observed under heavy load, typically caused by UART buffer overruns or brief Bluetooth lag — mitigated by careful baud-rate tuning and robust interrupt handling rather than polling.

## Limitations & Possible Improvements

- Occasional packet loss under load — could be addressed with **DMA-based UART transfers** instead of interrupt-per-byte handling.
- Bluetooth (HC-05/SPP) range and reliability are limited — switching to **BLE or Wi-Fi** would improve robustness.
- No error-checking on the data itself — adding a **checksum or CRC** would catch corrupted packets.

## Authors

- Muhammad Fasih (BS-23-MB-100422)
- Muhammad Hanzala (BS-23-SB-197957)

Department of Electrical Engineering, Pakistan Institute of Engineering and Applied Sciences (PIEAS)

## References

- HC-05 Bluetooth Module Datasheet
- [`pyserial`](https://pyserial.readthedocs.io/) — Python Serial Port Extension documentation
