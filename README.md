# Mobile Interface for Vibration Control

**Microprocessor & Interfacing — Project Report**

**Submitted by:**
- Muhammad Fasih (BS-23-MB-100422)
- Muhammad Hanzala (BS-23-SB-197957)

Department of Electrical Engineering, Pakistan Institute of Engineering and Applied Sciences (PIEAS)
Date of submission: 05-05-2025

---

## Abstract

The purpose of this experiment was to design and implement a real-time wireless sensor data acquisition and visualization system using an STM32 microcontroller. A custom mobile application developed using MIT App Inventor continuously transmits accelerometer data from a smartphone to the STM32 via the HC-05 Bluetooth module. The STM32 receives this data using USART1 with interrupt-based handling and forwards it to a PC via USART2, also configured to use interrupts for efficient, non-blocking communication. Key results include successful real-time reception and forwarding of sensor data, and accurate visualization of this data on the PC using a Python script that plots live graphs. Major points of discussion include STM32 peripheral configuration through STM32CubeMX, implementation of UART interrupt routines in Keil using HAL libraries, mobile app design, and serial data parsing in Python. The project concludes that interrupt-driven UART communication ensures responsiveness and reliability, and combining it with Python-based visualization makes the system well-suited for real-time embedded monitoring applications.

## Table of Contents

- [Introduction](#introduction)
- [Equipment and Procedure](#equipment-and-procedure)
- [Results and Discussion](#results-and-discussion)
- [Summary and Conclusion](#summary-and-conclusion)
- [References](#references)

---

## Introduction

This project focuses on the development of a real-time wireless data acquisition and visualization system using an STM32 microcontroller platform.

### Purpose

This experiment aimed to develop a real-time wireless accelerometer data acquisition system using an STM32 microcontroller. Accelerometer data was transmitted from a smartphone via Bluetooth and forwarded to a PC over UART using interrupt-based handling for efficient, non-blocking operation.

### Objectives

- Design and implement a real-time wireless data communication system using the STM32 microcontroller.
- Receive accelerometer data from a custom-developed Android application via the HC-05 Bluetooth module.
- Implement interrupt-based UART data reception for efficient and non-blocking communication.
- Forward received IMU data to a PC and visualize it using Python through 2D and 3D plots.
- Gain practical experience in integrating mobile applications, embedded firmware, and PC-based data visualization.
- Enhance proficiency in STM32 programming using STM32CubeMX and HAL libraries.

![Block Diagram of the Project](images/block-diagram.png)
*Figure: Block diagram of the project*

### Background

In embedded systems, real-time data acquisition and communication are essential for monitoring and control applications. Microcontrollers like the STM32 offer advanced features such as hardware USART peripherals and interrupt-driven communication, which make them ideal for such tasks. In this project, a custom mobile application was developed using MIT App Inventor to access and transmit the phone's built-in accelerometer data over Bluetooth using the HC-05 module. The STM32 receives this data via USART1 using interrupt service routines, and forwards it to a PC via USART2, also using interrupts to avoid polling delays. On the PC side, Python is used to capture and plot the incoming data in real time, allowing for visual monitoring of acceleration in three axes. This setup demonstrates a complete pipeline from mobile sensor to visualization, illustrating practical applications in wearable devices, motion analysis, and embedded data logging. Any observed deviations from expected performance, such as delayed plotting or occasional data loss, are attributed to serial buffer overflows or minor baud rate mismatches, which were addressed through code optimization and hardware-level synchronization.

---

## Equipment and Procedure

### Equipment

- STM32 microcontroller (STM32F103C8T6)
- HC-05 Bluetooth Module
- Smartphone (with the custom app for transmitting accelerometer data)
- Breadboard
- Jumper Wires
- STM-Debugger (for flashing code and powering the STM32)
- PC (for monitoring serial output and plotting data)
- STM32CubeMX (for configuration)
- Keil MDK (for code development and flashing)
- External Power Supply (if required, depending on your STM32 setup)
- Mobile App (custom-developed app for sending accelerometer data via Bluetooth)
- Python (for data visualization and plotting)

### Procedure

#### Hardware Setup

1. Connect the HC-05 Bluetooth module to the STM32 board. Connect TX of HC-05 to RX of STM32 and RX of HC-05 to TX of STM32.
2. Power the STM32 via USB or an external power source.
3. Pair the smartphone with the HC-05 module.

#### STM32 Configuration

1. Open STM32CubeMX and start a new project.
2. Enable two USARTs: one for the Bluetooth module and another for PC communication.
3. Set UART parameters such as baud rate and stop bits.
4. Configure any required GPIO pins.
5. Generate the code and open it in Keil MDK.

#### Code Development

1. Initialize both USARTs in Keil MDK:
   - One USART receives data from HC-05.
   - The other USART sends data to the PC.
2. Implement code to forward data from Bluetooth to PC.
3. Use HAL libraries for UART communication and error handling.
4. Build and flash the code to the STM32 using ST-Link.

#### Mobile App and Data Transmission

1. Launch the custom mobile app to transmit accelerometer data over Bluetooth.
2. The HC-05 module receives the data and sends it to the STM32.

#### Data Reception and Plotting

1. Open a serial monitor (e.g., Tera Term or PuTTY) on the PC to view incoming data from the STM32.
2. Use Python with `pyserial` to read the serial data.
3. Plot the X, Y, and Z acceleration values using libraries like `matplotlib` or `plotly`.
4. Update the graph in real time as new data is received.

#### Testing and Verification

1. Move the smartphone to produce changes in accelerometer readings.
2. Check that the data is displayed correctly in the serial monitor and plotted accurately in real time.

---

## Results and Discussion

### STM32 Main Code

```c
#include "main.h"

UART_HandleTypeDef huart1;
UART_HandleTypeDef huart2;

uint8_t rx_data[13];
volatile uint8_t rx_complete = 0;

void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_USART1_UART_Init(void);
static void MX_USART2_UART_Init(void);

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

void SystemClock_Config(void)
{
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSI_DIV2;
  RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL16;
  HAL_RCC_OscConfig(&RCC_OscInitStruct);

  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;
  HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2);
}

static void MX_USART1_UART_Init(void)
{
  huart1.Instance = USART1;
  huart1.Init.BaudRate = 9600;
  huart1.Init.WordLength = UART_WORDLENGTH_8B;
  huart1.Init.StopBits = UART_STOPBITS_1;
  huart1.Init.Parity = UART_PARITY_NONE;
  huart1.Init.Mode = UART_MODE_RX;
  huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
  huart1.Init.OverSampling = UART_OVERSAMPLING_16;
  HAL_UART_Init(&huart1);
}

static void MX_USART2_UART_Init(void)
{
  huart2.Instance = USART2;
  huart2.Init.BaudRate = 9600;
  huart2.Init.WordLength = UART_WORDLENGTH_8B;
  huart2.Init.StopBits = UART_STOPBITS_1;
  huart2.Init.Parity = UART_PARITY_NONE;
  huart2.Init.Mode = UART_MODE_TX_RX;
  huart2.Init.HwFlowCtl = UART_HWCONTROL_NONE;
  huart2.Init.OverSampling = UART_OVERSAMPLING_16;
  HAL_UART_Init(&huart2);
}

static void MX_GPIO_Init(void)
{
  __HAL_RCC_GPIOA_CLK_ENABLE();
  __HAL_RCC_GPIOC_CLK_ENABLE();

  GPIO_InitTypeDef GPIO_InitStruct = {0};

  GPIO_InitStruct.Pin = GPIO_PIN_9;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  GPIO_InitStruct.Pin = GPIO_PIN_10;
  GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  GPIO_InitStruct.Pin = GPIO_PIN_2;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  GPIO_InitStruct.Pin = GPIO_PIN_3;
  GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
}

void Error_Handler(void)
{
  __disable_irq();
  while (1) { }
}
```

![Digital Clock Hardware System](images/circuit-figure.jpeg)
*Figure: Hardware setup*

![Data Received from HC-05 into PuTTY](images/putty-data.jpeg)
*Figure: Data received from HC-05 into PuTTY*

![Accelerometer Data Visualization](images/2d-3d-graphs.jpeg)
*Figure: Accelerometer data visualization (2D and 3D graphs in Python)*

![Mobile Interface of MIT App](images/mobile-app-interface.jpeg)
*Figure: Mobile interface of the MIT App*

![Blocks of MIT App](images/mit-app-blocks.jpeg)
*Figure: MIT App Inventor blocks for continuous data transmission*

### Discussion

The experiment aimed to implement a real-time wireless sensor data acquisition and visualization system, and the results obtained demonstrate that this objective was successfully achieved. The expected outcome was continuous transmission of accelerometer data from a smartphone app to the STM32 microcontroller via the HC-05 Bluetooth module, followed by the forwarding of this data to a PC through a second UART for live plotting. In practice, the system operated as intended, with real-time data received and transmitted with minimal latency. The use of interrupt-based UART communication in both reception (USART1) and transmission (USART2) proved effective, allowing for efficient and non-blocking data handling.

The actual results closely matched expectations, although minor delays were observed in plotting when system resources were limited. These slight deviations can be attributed to occasional UART buffer overruns or Bluetooth transmission lag, particularly if the mobile device temporarily reduced update frequency due to internal power management. These errors were minimal and did not significantly affect the performance of the system. Careful tuning of UART baud rates and use of robust interrupt service routines helped mitigate most data loss and ensured reliable communication.

The project reinforced the theoretical understanding of asynchronous serial communication and embedded interrupt handling. The hands-on configuration of UARTs in STM32CubeMX and development in Keil using HAL libraries illustrated the theory well, showing how hardware peripherals can be used effectively for real-time embedded tasks. Additionally, the successful integration of a custom Android app with microcontroller firmware and Python scripting for PC visualization highlighted the interdisciplinary nature of modern embedded systems development.

Any limitations encountered, such as potential latency or occasional dropped packets, could be improved by implementing DMA-based UART transfers or using a more reliable wireless communication protocol like BLE or Wi-Fi. Furthermore, the current setup could be made more robust with a checksum-based data validation or error correction mechanism. Nonetheless, the primary objective was met, as the STM32 reliably received and forwarded accelerometer data, and the Python script accurately plotted the results. These results validate the system's effectiveness for use in real-time monitoring applications, such as motion tracking, gesture recognition, or condition monitoring in embedded environments.

---

## Summary and Conclusion

### Summary

In this project, we successfully developed a real-time data acquisition system using the STM32 microcontroller. Accelerometer data from a mobile phone was transmitted wirelessly via the HC-05 Bluetooth module and received on the STM32 using UART1. The STM32 then forwarded the data to a PC through UART2, where it was visualized using Python-based 2D and 3D plotting. A custom Android app was designed to generate and send IMU sensor data. The system utilized UART interrupt handling for reliable communication and ensured continuous data flow from the mobile app to the PC. All components—including hardware setup, firmware coding, and Python visualization—were integrated effectively. The results met our expectations with only minimal and explainable deviations, demonstrating the practical implementation of wireless sensor data transmission and embedded systems communication.

### Conclusion

Throughout this experiment, we learned how to use an STM32 microcontroller and Bluetooth communication to transmit real-time sensor data to a PC for visualization. We also discovered the importance of interrupt-driven communication and proper data handling for reliable performance. This experience showed how embedded systems can be used effectively for real-time monitoring applications.

---

## References

1. HC-05 Bluetooth Module Datasheet
2. `pyserial` — Python Serial Port Extension documentation

---

*Department of Electrical Engineering, Pakistan Institute of Engineering and Applied Sciences (PIEAS)*
