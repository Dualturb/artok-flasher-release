# ARTOK Flasher Runtime Library

The **ARTOK Flasher Runtime Library** (`libartok_flasher.a`) is a specialized static library designed to implement the secure communication protocol required to download HMI UI binaries from the **Artok Studio GUI Editor** and program them into an external flash chip on the target microcontroller board.

This library operates as a state machine, handling the entire command sequence, data transfer, and integrity checks from the host tool.

---

## 🎯 Target & Architecture

| Specification | Details |
| :--- | :--- |
| **Microcontroller** | ARM Cortex-M architecture (hardware-agnostic) |
| **Library Format** | Static Library (`.a`) |
| **Communication** | Abstracted via `Atk_FlasherInterface_t` (Requires implementation for UART, SPI, etc.) |
| **Flash Chip** | Abstracted via `Atk_FlasherDriver_t` (Built-in **W25Qxx** driver included) |

## 🏗️ Two-Layer Integration Architecture

The `artok-flasher` library requires the host firmware to implement **two distinct hardware abstraction layers (HALs)**: a communication layer and a flash layer.

### Layer 1: Communication Interface (`Atk_FlasherInterface_t`)

This interface abstracts the physical link between the microcontroller and the PC running Artok Studio.

| Function Pointer | Purpose | Example Protocol |
| :--- | :--- | :--- |
| `preinit` | Initializes the communication peripheral (e.g., configuring UART baud rate). | UART, USB CDC, SPI |
| `send_response_byte` | Sends a single status/acknowledgment byte back to the host tool. | UART, SPI |
| `device_select` / `device_deselect` | Controls the hardware select line (e.g., **Chip Select** for SPI) for the communication bus. | SPI, I2C |
| `transmit_data` / `receive_data` | Functions to send or receive blocks of protocol data. | UART, USB CDC, SPI |
| `delay_ms` | A simple blocking delay function required by the protocol timing. | SysTick or hardware timer. |

### Layer 2: Flash Driver Interface (`Atk_FlasherDriver_t`)

This interface abstracts the storage device itself, defining the low-level functions to manage the external flash chip.

| Function Pointer | Purpose |
| :--- | :--- |
| `init` | Initializes the flash chip (e.g., checks ID, sets modes). |
| `read` | Reads a block of data from a specified flash address. |
| `write` | Writes a block of data to a specified flash address. |
| `erase_sector` | Erases a sector of the flash at a given address (required for programming). |
| `erase_chip` | Erases the entire chip (used for factory reset/full update). |
| `get_size` | Returns the total capacity of the flash chip in bytes. |

#### Built-in Flash Driver

The library provides a pre-compiled driver for the common **W25Qxx** series flash chips:
```c
extern const Atk_FlasherDriver_t w25qxx_driver;
```

If using a W25Qxx chip, pass this symbol directly to flasher_init().

## ⚙️ Core API Functions

Your host firmware must integrate the flasher state machine using these functions:

| Function | Description | Required Call Location |
| :--- | :--- | :--- |
| `void flasher_init(...)` | Initializes the flasher library and registers both the Communication Interface and the Flash Driver. | Once, during application initialization. |
| `void flasher_process_byte(uint8_t byte)` | **Interrupt/Callback Handler:** Must be called immediately for every byte received from the host PC (e.g., inside the UART/SPI RX interrupt service routine). | High-priority context (ISR). |
| `void flasher_process(void)` | **State Machine Executor:** Runs the main flasher protocol logic, handling command parsing and flash operations. | Continuously, in the main `while(1)` loop. |


## 🛠️ STM32CubeIDE Integration Guide

This guide details the steps required to successfully integrate the `artok-flasher.a` static library into an **STM32CubeIDE** project.

### Step 1: Add the Library Files to the Project

1.  **Create a Folder:** In your STM32CubeIDE Project Explorer, create a new folder (e.g., **ArtokFlasher**) at the root level of your project.
2.  **Copy Files:** Copy `artok_flasher.h` and `libartok_flasher.a` into this new folder.
3.  **Refresh Project:** Right-click on your project name and select **Refresh** (or hit F5).

### Step 2: Configure the Compiler (Header Path)

1.  Right-click on the project and select **Properties**.
2.  Navigate to **C/C++ Build** > **Settings**.
3.  Under **Tool Settings**, select **MCU GCC Compiler** > **Include Paths**.
4.  Click the **Add** icon (`+`) and add the path to the folder you created (e.g., `ArtokFlasher`):
    ```
    "${workspace_loc:/${ProjName}/ArtokFlasher}"
    ```

### Step 3: Configure the Linker (Library and Library Path)

1.  Return to **Project Properties** > **C/C++ Build** > **Settings**.
2.  Under **Tool Settings**, select **MCU GCC Linker** > **Libraries**.
    * **Libraries (`-l`)**: Click **Add** (`+`) and enter the name of the library **without** `lib` or `.a`:
        ```
        artok_flasher
        ```
    * **Library search path (`-L`)**: Click **Add** (`+`) and add the path to the folder containing `libartok_flasher.a`:
        ```
        "${workspace_loc:/${ProjName}/ArtokFlasher}"
        ```
3.  Click **Apply and Close**.