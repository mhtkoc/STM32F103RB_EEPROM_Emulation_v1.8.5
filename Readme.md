# STM32F103RB EEPROM Emulation

This repository implements **EEPROM Emulation** using the internal Flash memory of the **STM32F103RB** microcontroller (specifically tailored for the **NUCLEO-F103RB** board). 

It is based on the STMicroelectronics Application Note **AN2594** ("EEPROM emulation in STM32F10x microcontrollers") and adapted to utilize STM32Cube HAL drivers.

---

## 📌 Table of Contents
1. [Overview](#-overview)
2. [Hardware & Software Prerequisites](#-hardware--software-prerequisites)
3. [Memory Configuration](#%EF%B8%8F-memory-configuration)
4. [How EEPROM Emulation Works](#-how-eeprom-emulation-works)
5. [File Structure](#%EF%B8%8F-file-structure)
6. [API Reference](#-api-reference)
7. [Usage and Example Application](#-usage-and-example-application)
8. [Original References](#-original-references)

---

## 🔍 Overview

Microcontrollers like the STM32F103RB do not feature an internal, dedicated EEPROM. To store non-volatile user configuration data, calibration parameters, or state information, developers can either:
1. Connect an external EEPROM chip (e.g., via I2C or SPI).
2. **Emulate EEPROM using internal Flash memory**, which is the cost-effective and space-saving method implemented in this repository.

Because Flash memory must be erased in pages (typically 1 KB on STM32F1 Medium-Density devices) and has a limited write/erase lifecycle (typically ~10,000 cycles), writing directly to a fixed Flash location on every change would quickly wear out the Flash. 

This project implements a **wear-leveling and double-page transfer algorithm** to safely and efficiently emulate EEPROM.

---

## 🛠️ Hardware & Software Prerequisites

* **Microcontroller:** STM32F103RBT6 (ARM Cortex-M3, 128 KB Flash, 20 KB SRAM).
* **Board:** NUCLEO-F103RB (or any custom board containing an STM32F103xB series MCU).
* **Software Environment:** 
  * STM32CubeF1 Firmware Library (v1.8.5 or compatible).
  * STM32Cube HAL drivers (`stm32f1xx_hal.h` must be available in your build path).
  * Toolchains: STM32CubeIDE, Keil MDK, IAR, or GCC ARM Embedded.

---

## 🖥️ Memory Configuration

The STM32F103RB contains **128 KB of Flash memory**, divided into **128 pages of 1 KB each** (Page 0 to Page 127).

This emulation driver uses **two non-contiguous pages** to store data and handle page swap operations:

| Resource | Page Index | Base Address | Size | Description |
|---|---|---|---|---|
| **PAGE0** | Page 32 | `0x08008000` | 1 KB | First Flash page reserved for EEPROM emulation. |
| **PAGE1** | Page 96 | `0x08018000` | 1 KB | Second Flash page reserved for EEPROM emulation. |

> [!NOTE]
> The spacing of 64 KB between Page 32 and Page 96 is designed to prevent overlapping with application code, assuming application code is configured to fit in the lower portion of the Flash memory.

---

## ⚙️ How EEPROM Emulation Works

The emulation algorithm uses two pages (an **Active** page and a **Transfer** page) to store 16-bit virtual variables linked to 16-bit data.

### 1. Structure of a Flash Page
Each 1 KB page is formatted as follows:
* **Page Header (First 4 Bytes):** Stores the Page Status.
  * `0xFFFF` (**ERASED**): Page is blank.
  * `0xEEEE` (**RECEIVE_DATA**): Page is prepared to receive valid variables from the other page during a cleanup transfer.
  * `0x0000` (**VALID_PAGE**): Page contains active/valid data.
* **Data Field:** Pairs of 32-bit values containing:
  * **Virtual Address** (16-bit): Identifier for the variable.
  * **Data** (16-bit): Current value of the variable.

### 2. Page Life Cycle and Swapping (Page Transfer)
When you write a variable, the driver searches for the next free space on the current **VALID_PAGE** and appends the new value. The old value of that virtual variable remains on Flash but is ignored (only the last written instance is read).

When the active page becomes **full**:
1. The secondary page status is set to `RECEIVE_DATA`.
2. The driver reads the latest values for all registered virtual variables from the active page and writes them to the secondary page.
3. The secondary page status is updated to `VALID_PAGE`.
4. The old active page is erased to `ERASED` status.
5. The secondary page becomes the new active page.

This wear-leveling dramatically increases the lifetime of the Flash memory by avoiding constant page-erase operations.

---

## 🗂️ File Structure

The project has a modular layout containing driver files and the demo application:

```
STM32F103RB_EEPROM_EEPROM_Emulation_v1.8.5/
│
├── Readma.md                          # Repository documentation (this file)
├── main.c                             # Application entry point & demo test-bench
│
└── eepromEmulationDrivers/
    ├── Inc/
    │   └── eeprom.h                   # Driver configuration, macros, and API declarations
    └── Src/
        └── eeprom.c                   # Implementation of EEPROM emulation & Flash management
```

---

## 🔌 API Reference

The driver provides three primary functions declared in [eeprom.h](STM32F103RB_EEPROM_EEPROM_Emulation_v1.8.5/eepromEmulationDrivers/Inc/eeprom.h):

### 1. Initialize EEPROM
```c
uint16_t EE_Init(void);
```
* **Description:** Restores pages to a known valid state on boot. It checks for header corruptions (e.g. from an abrupt power loss during page swap) and repairs them.
* **Return Value:** `HAL_StatusTypeDef` status or Flash error code.

### 2. Read Variable
```c
uint16_t EE_ReadVariable(uint16_t VirtAddress, uint16_t* Data);
```
* **Description:** Reads the most recently written value associated with `VirtAddress`.
* **Parameters:**
  * `VirtAddress`: Virtual address (16-bit, cannot be `0xFFFF`).
  * `Data`: Pointer to a 16-bit variable to store the retrieved data.
* **Return Value:** 
  * `0`: Variable was successfully found.
  * `1`: Variable does not exist (not written yet).
  * Flash read error code.

### 3. Write/Update Variable
```c
uint16_t EE_WriteVariable(uint16_t VirtAddress, uint16_t Data);
```
* **Description:** Writes or updates a variable. If the current active page is full, it triggers a page swap automatically.
* **Parameters:**
  * `VirtAddress`: Virtual address (16-bit, cannot be `0xFFFF`).
  * `Data`: 16-bit data value to store.
* **Return Value:** `HAL_STATUS` or Flash status (e.g., `PAGE_FULL` or write error codes).

---

## 💻 Usage and Example Application

A demonstration of writing and reading variables is implemented in [main.c](STM32F103RB_EEPROM_EEPROM_Emulation_v1.8.5/main.c).

### Virtual Variable Definition
```c
/* Declaring the list of allowed virtual addresses */
uint16_t VirtAddVarTab[NB_OF_VAR] = {0x5555, 0x6666, 0x7777};
```

### Main Write Cycle
The demo performs sequential intensive writes to test the page swap threshold:
1. **Initialize System & EEPROM Emulation:**
   ```c
   HAL_Init();
   SystemClock_Config();
   EE_Init(); /* Internally unlocks and locks the Flash automatically */
   ```
2. **Variable 1 Writes:** Writes `0x1000` values to Virtual Variable `0x5555`.
3. **Variable 2 Writes:** Writes `0x2000` values to Virtual Variable `0x6666`.
4. **Variable 3 Writes:** Writes `0x3000` values to Virtual Variable `0x7777`.
5. **Read Verification:** Validates that the latest value is successfully read back and stored in SRAM `VarDataTab`:
   ```c
   EE_ReadVariable(VirtAddVarTab[0], &VarDataTab[0]);
   EE_ReadVariable(VirtAddVarTab[1], &VarDataTab[1]);
   EE_ReadVariable(VirtAddVarTab[2], &VarDataTab[2]);
   ```

---

## 🔗 Original References

* **Original STMicroelectronics Project Template:** [STM32CubeF1 Projects](https://github.com/STMicroelectronics/STM32CubeF1/tree/master/Projects/STM32F103RB-Nucleo/Applications/EEPROM/EEPROM_Emulation)
* **Application Note:** [AN2594 on ST.com](https://www.st.com/resource/en/application_note/cd00165692-eeprom-emulation-in-stm32f10x-microcontrollers-stmicroelectronics.pdf)
