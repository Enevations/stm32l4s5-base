# STM32 B-L4S5I-IOT01A: CubeMX-Free C++17 + FreeRTOS Setup Guide

This guide documents the end-to-end development and debugging workflow for the **STMicroelectronics B-L4S5I-IOT01A** (`STM32L4S5VIT6`, ARM Cortex-M4F @ 120 MHz) on Windows without installing STM32CubeIDE or STM32CubeMX:

* **Build Environment:** Docker (`Ubuntu 24.04` + `arm-none-eabi-gcc` + `CMake` + `Ninja`)
* **Firmware Stack:** C++17 + FreeRTOS Kernel (`V11.3.3`) + STM32CubeL4 HAL (`v1.18.1`), fetched automatically via CMake `FetchContent`
* **Debugging & Flashing:** VS Code (`Cortex-Debug`) + Windows `arm-none-eabi-gdb.exe` + `xPack OpenOCD` + `WinUSB` (Zadig)

---

## 1. Project Structure

```text
b-l4s5i-standalone/
├── .devcontainer/
│   └── Dockerfile              # ARM GCC + CMake + Ninja build container
├── .vscode/
│   ├── launch.json             # OpenOCD + Cortex-Debug configuration (F5)
│   ├── settings.json           # Host paths to arm-none-eabi-gdb & openocd
│   └── tasks.json              # Process task triggering Docker CMake build
├── cmake/
│   └── arm-none-eabi.cmake     # Cortex-M4F cross-compilation toolchain file
├── config/
│   ├── FreeRTOSConfig.h        # FreeRTOS settings & CMSIS IRQ mappings
│   └── stm32l4xx_hal_conf.h    # Enabled STM32L4 HAL peripheral modules
├── src/
│   └── main.cpp                # 120 MHz clock, TIM6 HAL tick, FreeRTOS C++ task
├── CMakeLists.txt              # Root build script + FetchContent dependencies
├── SETUP_GUIDE.md              # This setup guide
└── STM32L4S5VITx_FLASH.ld      # Linker script (2048 KB Flash / 640 KB SRAM)
```

---

## 2. Docker Build Container Setup

### `.devcontainer/Dockerfile`

```dockerfile
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    cmake \
    gcc-arm-none-eabi \
    git \
    libnewlib-arm-none-eabi \
    libstdc++-arm-none-eabi-newlib \
    ninja-build \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /workspace
```

### Build & Verify the Docker Image

Run from your project root in Windows PowerShell:

```powershell
# Build the image
docker build -t stm32-builder .devcontainer/

# Verify toolchain inside container
docker run --rm stm32-builder bash -c "arm-none-eabi-g++ --version && cmake --version && ninja --version"
```

---

## 3. Cross-Compilation Toolchain & Linker Script

### `cmake/arm-none-eabi.cmake`

```cmake
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR arm)

set(CMAKE_C_COMPILER arm-none-eabi-gcc)
set(CMAKE_CXX_COMPILER arm-none-eabi-g++)
set(CMAKE_ASM_COMPILER arm-none-eabi-gcc)
set(CMAKE_OBJCOPY arm-none-eabi-objcopy)
set(CMAKE_SIZE arm-none-eabi-size)

# STM32L4S5 uses ARM Cortex-M4 with single-precision FPU (fpv4-sp-d16)
set(MCU_FLAGS "-mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard")

set(CMAKE_C_FLAGS_INIT "${MCU_FLAGS} -ffunction-sections -fdata-sections -Wall")
set(CMAKE_CXX_FLAGS_INIT "${MCU_FLAGS} -fno-exceptions -fno-rtti -fno-threadsafe-statics -ffunction-sections -fdata-sections -Wall")
set(CMAKE_ASM_FLAGS_INIT "${MCU_FLAGS} -x assembler-with-cpp")
set(CMAKE_EXE_LINKER_FLAGS_INIT "${MCU_FLAGS} --specs=nano.specs --specs=nosys.specs -Wl,--gc-sections")

set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)
```

> **Note on `-fno-threadsafe-statics`:** Prevents GCC from emitting `__cxa_guard_acquire` mutex locks around function-local `static` C++ objects, avoiding deadlocks inside FreeRTOS tasks.

### `STM32L4S5VITx_FLASH.ld`

Defines the **2048 KB Flash** (`0x08000000`) and **640 KB contiguous SRAM** (`0x20000000`) memory layout for the `STM32L4S5VIT6`:

```ld
ENTRY(Reset_Handler)

_estack = ORIGIN(RAM) + LENGTH(RAM);
_Min_Heap_Size  = 0x1000; /* 4 KB */
_Min_Stack_Size = 0x1000; /* 4 KB MSP stack */

MEMORY
{
  FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 2048K
  RAM   (xrw) : ORIGIN = 0x20000000, LENGTH = 640K
}

SECTIONS
{
  .isr_vector :
  {
    . = ALIGN(4);
    KEEP(*(.isr_vector))
    . = ALIGN(4);
  } >FLASH

  .text :
  {
    . = ALIGN(4);
    *(.text)
    *(.text*)
    *(.glue_7)
    *(.glue_7t)
    KEEP(*(.init))
    KEEP(*(.fini))
    . = ALIGN(4);
    _etext = .;
  } >FLASH

  .rodata :
  {
    . = ALIGN(4);
    *(.rodata)
    *(.rodata*)
    . = ALIGN(4);
  } >FLASH

  .ARM.extab : { *(.ARM.extab* .gnu.linkonce.armextab.*) } >FLASH
  .ARM :
  {
    __exidx_start = .;
    *(.ARM.exidx*)
    __exidx_end = .;
  } >FLASH

  .preinit_array :
  {
    PROVIDE_HIDDEN(__preinit_array_start = .);
    KEEP(*(.preinit_array*))
    PROVIDE_HIDDEN(__preinit_array_end = .);
  } >FLASH

  .init_array :
  {
    PROVIDE_HIDDEN(__init_array_start = .);
    KEEP(*(SORT(.init_array.*)))
    KEEP(*(.init_array*))
    PROVIDE_HIDDEN(__init_array_end = .);
  } >FLASH

  .fini_array :
  {
    PROVIDE_HIDDEN(__fini_array_start = .);
    KEEP(*(SORT(.fini_array.*)))
    KEEP(*(.fini_array*))
    PROVIDE_HIDDEN(__fini_array_end = .);
  } >FLASH

  _sidata = LOADADDR(.data);

  .data :
  {
    . = ALIGN(4);
    _sdata = .;
    *(.data)
    *(.data*)
    . = ALIGN(4);
    _edata = .;
  } >RAM AT> FLASH

  . = ALIGN(4);
  .bss :
  {
    _sbss = .;
    __bss_start__ = _sbss;
    *(.bss)
    *(.bss*)
    *(COMMON)
    . = ALIGN(4);
    _ebss = .;
    __bss_end__ = _ebss;
  } >RAM

  ._user_heap_stack :
  {
    . = ALIGN(8);
    PROVIDE(end = .);
    PROVIDE(_end = .);
    . = . + _Min_Heap_Size;
    . = . + _Min_Stack_Size;
    . = ALIGN(8);
  } >RAM
}
```

---

## 4. Root `CMakeLists.txt`

Automatically fetches `STM32CubeL4` (selective submodules only) and `FreeRTOS-Kernel`, compiles `src/main.cpp`, and produces `.elf`, `.bin`, and `.hex` outputs:

```cmake
cmake_minimum_required(VERSION 3.22)

project(b-l4s5i-standalone LANGUAGES C CXX ASM)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

include(FetchContent)

# 1. Fetch ST's official STM32CubeL4 (HAL, CMSIS, and startup_stm32l4s5xx.s)
FetchContent_Declare(
    stm32cubel4
    GIT_REPOSITORY https://github.com/STMicroelectronics/STM32CubeL4.git
    GIT_TAG        v1.18.1
    GIT_SHALLOW    TRUE
    GIT_SUBMODULES "Drivers/CMSIS/Device/ST/STM32L4xx" "Drivers/STM32L4xx_HAL_Driver" "Drivers/CMSIS/Core"
)
FetchContent_MakeAvailable(stm32cubel4)

# 2. Fetch official FreeRTOS Kernel
FetchContent_Declare(
    freertos_kernel
    GIT_REPOSITORY https://github.com/FreeRTOS/FreeRTOS-Kernel.git
    GIT_TAG        V11.3.3
    GIT_SHALLOW    TRUE
)

add_library(freertos_config INTERFACE)
target_include_directories(freertos_config SYSTEM INTERFACE config)
target_compile_definitions(freertos_config INTERFACE projCOVERAGE_TEST=0)

set(FREERTOS_PORT GCC_ARM_CM4F CACHE STRING "" FORCE)
set(FREERTOS_HEAP 4 CACHE STRING "" FORCE)

FetchContent_MakeAvailable(freertos_kernel)

# 3. Collect required STM32L4 HAL & CMSIS source files
set(CUBE_DRIVERS "${stm32cubel4_SOURCE_DIR}/Drivers")

set(HAL_SOURCES
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_cortex.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_dma.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_flash.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_flash_ex.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_gpio.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_i2c.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_i2c_ex.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_pwr.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_pwr_ex.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_rcc.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_rcc_ex.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_tim.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_tim_ex.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_uart.c"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Src/stm32l4xx_hal_uart_ex.c"
)

# 4. Define the firmware executable
add_executable(${PROJECT_NAME}.elf
    src/main.cpp
    ${HAL_SOURCES}
    "${CUBE_DRIVERS}/CMSIS/Device/ST/STM32L4xx/Source/Templates/system_stm32l4xx.c"
    "${CUBE_DRIVERS}/CMSIS/Device/ST/STM32L4xx/Source/Templates/gcc/startup_stm32l4s5xx.s"
)

target_compile_definitions(${PROJECT_NAME}.elf PRIVATE
    USE_HAL_DRIVER
    STM32L4S5xx
)

target_include_directories(${PROJECT_NAME}.elf PRIVATE
    config
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Inc"
    "${CUBE_DRIVERS}/STM32L4xx_HAL_Driver/Inc/Legacy"
    "${CUBE_DRIVERS}/CMSIS/Device/ST/STM32L4xx/Include"
    "${CUBE_DRIVERS}/CMSIS/Core/Include"
)

target_link_libraries(${PROJECT_NAME}.elf PRIVATE
    freertos_kernel
)

target_link_options(${PROJECT_NAME}.elf PRIVATE
    -T${CMAKE_SOURCE_DIR}/STM32L4S5VITx_FLASH.ld
    -Wl,-Map=${PROJECT_BINARY_DIR}/${PROJECT_NAME}.map,--cref
)

# 5. Post-build: print Flash/RAM usage and generate .bin / .hex images
add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
    COMMAND ${CMAKE_SIZE} $<TARGET_FILE:${PROJECT_NAME}.elf>
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${PROJECT_NAME}.elf> ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.bin
    COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${PROJECT_NAME}.elf> ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.hex
)
```

---

## 5. Configuration Headers (`config/`)

### `config/stm32l4xx_hal_conf.h`

```c
#pragma once

#define HAL_MODULE_ENABLED
#define HAL_CORTEX_MODULE_ENABLED
#define HAL_DMA_MODULE_ENABLED
#define HAL_FLASH_MODULE_ENABLED
#define HAL_GPIO_MODULE_ENABLED
#define HAL_I2C_MODULE_ENABLED
#define HAL_PWR_MODULE_ENABLED
#define HAL_RCC_MODULE_ENABLED
#define HAL_TIM_MODULE_ENABLED
#define HAL_UART_MODULE_ENABLED

#define HSE_VALUE                 8000000U
#define HSE_STARTUP_TIMEOUT       100U
#define MSI_VALUE                 4000000U
#define HSI_VALUE                 16000000U
#define HSI48_VALUE               48000000U
#define LSI_VALUE                 32000U
#define LSE_VALUE                 32768U
#define LSE_STARTUP_TIMEOUT       5000U
#define EXTERNAL_SAI1_CLOCK_VALUE 48000U
#define EXTERNAL_SAI2_CLOCK_VALUE 48000U

#define VDD_VALUE                 3300U
#define TICK_INT_PRIORITY         0x0FU
#define USE_RTOS                  0U
#define PREFETCH_ENABLE           0U
#define INSTRUCTION_CACHE_ENABLE  1U
#define DATA_CACHE_ENABLE         1U

#include "stm32l4xx_hal_rcc.h"
#include "stm32l4xx_hal_gpio.h"
#include "stm32l4xx_hal_dma.h"
#include "stm32l4xx_hal_cortex.h"
#include "stm32l4xx_hal_flash.h"
#include "stm32l4xx_hal_i2c.h"
#include "stm32l4xx_hal_pwr.h"
#include "stm32l4xx_hal_tim.h"
#include "stm32l4xx_hal_uart.h"

#define assert_param(expr) ((void)0U)
```

### `config/FreeRTOSConfig.h`

```c
#pragma once

#include <stdint.h>
extern uint32_t SystemCoreClock;

#define configUSE_PREEMPTION                    1
#define configUSE_PORT_OPTIMISED_TASK_SELECTION 1
#define configUSE_TICKLESS_IDLE                 0
#define configCPU_CLOCK_HZ                      ( SystemCoreClock )
#define configTICK_RATE_HZ                      ( ( TickType_t ) 1000 )
#define configMAX_PRIORITIES                    ( 7 )
#define configMINIMAL_STACK_SIZE                ( ( uint16_t ) 128 )
#define configTOTAL_HEAP_SIZE                   ( ( size_t ) ( 32 * 1024 ) )
#define configMAX_TASK_NAME_LEN                 ( 16 )
#define configUSE_16_BIT_TICKS                  0
#define configIDLE_SHOULD_YIELD                 1
#define configUSE_TASK_NOTIFICATIONS            1
#define configUSE_MUTEXES                       1
#define configUSE_RECURSIVE_MUTEXES             1
#define configUSE_COUNTING_SEMAPHORES           1
#define configQUEUE_REGISTRY_SIZE               8
#define configUSE_QUEUE_SETS                    0
#define configUSE_TIME_SLICING                  1
#define configSUPPORT_STATIC_ALLOCATION         0
#define configSUPPORT_DYNAMIC_ALLOCATION        1

/* Hook functions */
#define configUSE_IDLE_HOOK                     0
#define configUSE_TICK_HOOK                     0
#define configCHECK_FOR_STACK_OVERFLOW          0
#define configUSE_MALLOC_FAILED_HOOK            0

/* Cortex-M4 interrupt priority configuration (4 priority bits on STM32L4) */
#define configPRIO_BITS                         4
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY         15
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY    5
#define configKERNEL_INTERRUPT_PRIORITY         ( configLIBRARY_LOWEST_INTERRUPT_PRIORITY << (8 - configPRIO_BITS) )
#define configMAX_SYSCALL_INTERRUPT_PRIORITY    ( configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY << (8 - configPRIO_BITS) )

/* Optional API functions */
#define INCLUDE_vTaskPrioritySet                1
#define INCLUDE_uxTaskPriorityGet               1
#define INCLUDE_vTaskDelete                     1
#define INCLUDE_vTaskSuspend                    1
#define INCLUDE_vTaskDelayUntil                 1
#define INCLUDE_vTaskDelay                      1
#define INCLUDE_xTaskGetSchedulerState          1

/* Map FreeRTOS handlers directly to CMSIS startup vector names */
#define vPortSVCHandler     SVC_Handler
#define xPortPendSVHandler  PendSV_Handler
#define xPortSysTickHandler SysTick_Handler
```

---

## 6. Application Entry Point (`src/main.cpp`)

Configures the **120 MHz** PLL clock (`16 MHz HSI / 2 * 30 / 2`), routes the STM32 HAL 1 ms tick to **`TIM6`** so FreeRTOS owns `SysTick`, initializes **User LED2 (`PB14`)** and **ST-LINK Virtual COM Port (`USART1` on `PB6`/`PB7` @ 115200 baud)**, and launches a C++17 FreeRTOS task:

```cpp
#include <array>
#include <cstdio>
#include <cstring>

extern "C" {
#include "stm32l4xx_hal.h"
#include "FreeRTOS.h"
#include "task.h"
}

namespace {

UART_HandleTypeDef huart1{};
TIM_HandleTypeDef  htim6{};

void SystemClock_Config() {
    RCC_OscInitTypeDef rcc_osc{};
    RCC_ClkInitTypeDef rcc_clk{};

    __HAL_RCC_PWR_CLK_ENABLE();
    HAL_PWREx_ControlVoltageScaling(PWR_REGULATOR_VOLTAGE_SCALE1_BOOST);

    // 16 MHz HSI -> PLL (M=2, N=30, R=2) = 120 MHz SYSCLK
    rcc_osc.OscillatorType      = RCC_OSCILLATORTYPE_HSI;
    rcc_osc.HSIState            = RCC_HSI_ON;
    rcc_osc.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;
    rcc_osc.PLL.PLLState        = RCC_PLL_ON;
    rcc_osc.PLL.PLLSource       = RCC_PLLSOURCE_HSI;
    rcc_osc.PLL.PLLM            = 2;
    rcc_osc.PLL.PLLN            = 30;
    rcc_osc.PLL.PLLP            = RCC_PLLP_DIV2;
    rcc_osc.PLL.PLLQ            = RCC_PLLQ_DIV2;
    rcc_osc.PLL.PLLR            = RCC_PLLR_DIV2;
    HAL_RCC_OscConfig(&rcc_osc);

    rcc_clk.ClockType      = RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_SYSCLK |
                             RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
    rcc_clk.SYSCLKSource   = RCC_SYSCLKSOURCE_PLLCLK;
    rcc_clk.AHBCLKDivider  = RCC_SYSCLK_DIV1;
    rcc_clk.APB1CLKDivider = RCC_HCLK_DIV1;
    rcc_clk.APB2CLKDivider = RCC_HCLK_DIV1;
    HAL_RCC_ClockConfig(&rcc_clk, FLASH_LATENCY_5);
}

void InitBoardPeripherals() {
    __HAL_RCC_GPIOB_CLK_ENABLE();
    __HAL_RCC_USART1_CLK_ENABLE();

    // User LED2 on PB14
    GPIO_InitTypeDef gpio{};
    gpio.Pin   = GPIO_PIN_14;
    gpio.Mode  = GPIO_MODE_OUTPUT_PP;
    gpio.Pull  = GPIO_NOPULL;
    gpio.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(GPIOB, &gpio);

    // ST-LINK Virtual COM Port: USART1 TX=PB6, RX=PB7 (AF7)
    gpio.Pin       = GPIO_PIN_6 | GPIO_PIN_7;
    gpio.Mode      = GPIO_MODE_AF_PP;
    gpio.Pull      = GPIO_NOPULL;
    gpio.Speed     = GPIO_SPEED_FREQ_VERY_HIGH;
    gpio.Alternate = GPIO_AF7_USART1;
    HAL_GPIO_Init(GPIOB, &gpio);

    huart1.Instance          = USART1;
    huart1.Init.BaudRate     = 115200;
    huart1.Init.WordLength   = UART_WORDLENGTH_8B;
    huart1.Init.StopBits     = UART_STOPBITS_1;
    huart1.Init.Parity       = UART_PARITY_NONE;
    huart1.Init.Mode         = UART_MODE_TX_RX;
    huart1.Init.HwFlowCtl    = UART_HWCONTROL_NONE;
    huart1.Init.OverSampling = UART_OVERSAMPLING_16;
    HAL_UART_Init(&huart1);
}

void SerialWrite(const char* str) {
    HAL_UART_Transmit(&huart1, reinterpret_cast<const uint8_t*>(str),
                      static_cast<uint16_t>(std::strlen(str)), 100);
}

void BlinkAndLogTask(void* /*pvParameters*/) {
    std::array<char, 80> line{};
    uint32_t count = 0;

    SerialWrite("\r\n[B-L4S5I-IOT01A] CubeMX-free FreeRTOS + C++17 running @ 120 MHz!\r\n");

    while (true) {
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_14);
        std::snprintf(line.data(), line.size(), "[%lu] FreeRTOS tick = %lu ms\r\n",
                      count++, static_cast<uint32_t>(xTaskGetTickCount()));
        SerialWrite(line.data());
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

} // namespace

// Override HAL's weak SysTick functions to use TIM6 (1 kHz) so FreeRTOS owns SysTick
extern "C" HAL_StatusTypeDef HAL_InitTick(uint32_t TickPriority) {
    RCC_ClkInitTypeDef clk_config{};
    uint32_t flash_latency = 0;

    HAL_NVIC_SetPriority(TIM6_DAC_IRQn, TickPriority, 0U);
    HAL_NVIC_EnableIRQ(TIM6_DAC_IRQn);

    __HAL_RCC_TIM6_CLK_ENABLE();
    HAL_RCC_GetClockConfig(&clk_config, &flash_latency);

    uint32_t uwTimclock = HAL_RCC_GetPCLK1Freq();
    if (clk_config.APB1CLKDivider != RCC_HCLK_DIV1) {
        uwTimclock *= 2U;
    }

    htim6.Instance               = TIM6;
    htim6.Init.Period            = (1000000U / 1000U) - 1U;
    htim6.Init.Prescaler         = (uwTimclock / 1000000U) - 1U;
    htim6.Init.ClockDivision     = 0;
    htim6.Init.CounterMode       = TIM_COUNTERMODE_UP;
    htim6.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_DISABLE;

    if (HAL_TIM_Base_Init(&htim6) == HAL_OK) {
        return HAL_TIM_Base_Start_IT(&htim6);
    }
    return HAL_ERROR;
}

extern "C" void HAL_SuspendTick(void) { __HAL_TIM_DISABLE_IT(&htim6, TIM_IT_UPDATE); }
extern "C" void HAL_ResumeTick(void)  { __HAL_TIM_ENABLE_IT(&htim6, TIM_IT_UPDATE); }

extern "C" void TIM6_DAC_IRQHandler(void) {
    if (__HAL_TIM_GET_FLAG(&htim6, TIM_FLAG_UPDATE) != RESET) {
        __HAL_TIM_CLEAR_IT(&htim6, TIM_IT_UPDATE);
        HAL_IncTick();
    }
}

int main() {
    HAL_Init();
    SystemClock_Config();
    InitBoardPeripherals();

    xTaskCreate(BlinkAndLogTask, "MainTask", 512, nullptr, tskIDLE_PRIORITY + 2, nullptr);
    vTaskStartScheduler();

    while (true) {}
}
```

---

## 7. Windows Host Tools & USB Driver Setup

1. **Physical Connection:** Plug a micro-USB data cable into **CN7 (`USB ST-LINK`)** at the top edge of the board (not CN5 `USB OTG`).
2. **ARM GNU Toolchain (for Windows GDB):** Install via `winget install Arm.GnuArmEmbeddedToolchain` (gives `arm-none-eabi-gdb.exe`).
3. **xPack OpenOCD:** Extract or install `winget install openocd-xpack` on Windows (e.g., `D:/xpack-openocd-0.12.0-7/bin/openocd.exe`).
4. **ST-LINK USB Driver (`WinUSB`):**
   * If OpenOCD fails with `Error: libusb_open() failed with LIBUSB_ERROR_NOT_FOUND`.
   * Install Zadig utility `https://zadig.akeo.ie/#` and run **Zadig**.
   * Select **Options → List All Devices**, choose **`STM32 STLink (Interface 0)`** (the Debug endpoint, leaving the Virtual COM Port and Mass Storage interfaces untouched), and install **`WinUSB`**.
5. **VS Code Extension:** Install **Cortex-Debug** (`marus25.cortex-debug`).

---

## 8. VS Code Configuration (`.vscode/`)

### `.vscode/settings.json`

```json
{
  "cortex-debug.armToolchainPath": "C:/Program Files (x86)/Arm GNU Toolchain arm-none-eabi/14.2 rel1/bin",
  "cortex-debug.openocdPath": "D:/xpack-openocd-0.12.0-7/bin/openocd.exe"
}
```

### `.vscode/tasks.json`

Uses `"type": "process"` so Windows PowerShell 5.1 does not choke on `&&` or strip quotes passed to `bash -c`:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Docker: Build STM32 Firmware",
      "type": "process",
      "command": "docker",
      "args": [
        "run",
        "--rm",
        "-v",
        "${workspaceFolder}:/workspace",
        "stm32-builder",
        "bash",
        "-c",
        "cmake -B build -G Ninja -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake && cmake --build build"
      ],
      "group": {
        "kind": "build",
        "isDefault": true
      },
      "problemMatcher": ["$gcc"]
    }
  ]
}
```

### `.vscode/launch.json`

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug B-L4S5I-IOT01A",
      "cwd": "${workspaceFolder}",
      "executable": "${workspaceFolder}/build/b-l4s5i-standalone.elf",
      "request": "launch",
      "type": "cortex-debug",
      "runToEntryPoint": "main",
      "preLaunchTask": "Docker: Build STM32 Firmware",
      "servertype": "openocd",
      "configFiles": [
        "interface/stlink.cfg",
        "target/stm32l4x.cfg"
      ]
    }
  ]
}
```

---

## 9. Daily Workflow

* **Build & Debug (`F5`):** Press **F5** in VS Code. Docker incrementally compiles `build/b-l4s5i-standalone.elf` (and `.bin`/`.hex`), OpenOCD flashes the MCU over ST-LINK, and GDB halts at `main()`.
* **Serial Monitor:** Open VS Code's **Serial Monitor** tab on **STMicroelectronics STLink Virtual COM Port** at **115200 baud** (`8N1`) to view real-time UART logs while **LED2 (`PB14`)** blinks at 1 Hz.
* **Drag-and-Drop Alternative:** Copy `build/b-l4s5i-standalone.bin` onto the **`DIS_L4S5VI`** USB mass-storage drive in Windows File Explorer to flash without launching a debug session.
