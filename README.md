# SSD1306 STM32 SPI Non-Blocking Drivers

This repository has code for SSD1306 display drivers for STM32. The driver uses non-blocking data transmission by leveraging interrupts and DMA.

The drivers will work for any SSD1306 based LCD or OLED display.

Note that these functions use the stm32 HAL. The HAL drivers for GPIO, SPI and DMA must be included in your project. 

# Tested Hardware

|     STM32      |        Display      |   Tested    |
| -------------- | ------------------- | ----------- |
|   STM32F446RE   | [Digilent PMOS OLED](https://store.digilentinc.com/pmod-oled-128-x-32-pixel-monochromatic-oled-display/) |  :heavy_check_mark:  |
|   STM32F446RE   | [Monochrome 128x32 SPI OLED graphic display](https://www.adafruit.com/product/661) |       :heavy_check_mark:      |

# Set-Up

Download the [Inc](./Inc) and [Src](./Src) folder contents. Include these files in your project's path. You can follow these next steps to use the drivers:

Note: You can view and test the simple program in [Example](./Example) where these steps are already implemented. The main program prints 'Hello World' across the screen.

### SPI with DMA

You must set up the SPI to work with DMA loading the Tx buffer. This can be done as follows:

1. In SPI init:
```
/* Associate DMA1 Stream 4 */
Spi_ssd1306Write.hdmatx = &hdma_spi2_tx;
```

2. Enable the HAL interrupt handlers
```
void DMA1_Stream4_IRQHandler(void)
{
  HAL_DMA_IRQHandler(&hdma_spi2_tx);
}

void SPI2_IRQHandler(void)
{
  HAL_SPI_IRQHandler(&Spi_ssd1306Write);
}
```

### Using the driver

1. Configure display dimensions in ssd1306.h
```
/* SSD1306 width in pixels */
#define SSD1306_WIDTH 128

/* SSD1306 OLED height in pixels */
#define SSD1306_HEIGHT 32
```

2. The handle to the SPI peripheral and SSD1306 data buffer must be initialized in main.c

In main.c
```
#include "ssd1306.h"

SSD1306_t SSD1306_Disp;

SPI_HandleTypeDef Spi_ssd1306Write;
```

The handle and buffer declared in main.c are used as extern in ssd1306.c and so the names must match

In ssd1306.c
```
/* Handle for SPI communication peripheral (declared in main) */
extern SPI_HandleTypeDef Spi_ssd1306Write;

/* Data buffer variable (declared in main) */
extern SSD1306_t SSD1306_Disp;
```

3. Set SPI complete callback to put SSD1306 strucuture in ready to transmit state
```
void HAL_SPI_TxCpltCallback(SPI_HandleTypeDef *pSpi2_oledWrite)
{
  /* Set the SSD1306 state to ready */
  SSD1306_Disp.state = SSD1306_STATE_READY;
}
```

### Avoiding Race Conditions

If you write to the SSD1306 buffer during data transmission, you will invoke a race condition! To avoid this, you can wrap any sections modifying the buffer in a condition checking SSD1306 state.

Example:
```
if (SSD1306_Disp.state == SSD1306_STATE_READY) {
  /* Write to the buffer here */
}
```

Similarly, when using an RTOS, semaphore or mutex can protect the data buffer.

# How it Works

These drivers offer the improvement of non-blocking functionality over [previously available drivers](https://github.com/afiskon/stm32-ssd1306). This is accomplished firstly by using the SSD1306's [horizontal addressing mode rather than the page addressing mode](https://cdn-shop.adafruit.com/datasheets/SSD1306.pdf). In this mode, the data is written to the SSD1306 to fill the entire screen without need for interruption. Moreover, using interrupts and DMA peripheral means the processor needs only to start data transfer and is then free for other work.
