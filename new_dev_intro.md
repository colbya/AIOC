# AIOC Firmware: New Developer Introduction

Welcome to the AIOC (All-In-One Cable) project! This guide will help you understand the firmware from the ground up, even if you've never worked with embedded systems before.

## Table of Contents

1. [What is AIOC?](#what-is-aioc)
2. [Prerequisites & Development Environment](#prerequisites--development-environment)
3. [Firmware Concepts for Beginners](#firmware-concepts-for-beginners)
4. [Project Architecture Overview](#project-architecture-overview)
5. [Core Subsystems](#core-subsystems)
6. [Making Your First Change](#making-your-first-change)
7. [Building, Flashing, and Debugging](#building-flashing-and-debugging)
8. [Resources & Further Reading](#resources--further-reading)

---

## What is AIOC?

AIOC is a small USB-C adapter that connects a ham radio to a computer. When you plug it in, your computer sees it as **four devices simultaneously**:

```
┌─────────────────────────────────────────────────────────────┐
│                         AIOC Device                         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────┐│
│  │  Soundcard  │  │  Serial Port│  │  HID Device │  │ DFU ││
│  │  (Audio)    │  │  (COM/tty)  │  │  (CM108)    │  │     ││
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────┘│
│        │                │                │             │    │
│   For digital     For radio        For PTT      For firmware│
│   modes (APRS,    programming      control      updates     │
│   Direwolf)       (CHIRP)          (legacy)                 │
└─────────────────────────────────────────────────────────────┘
```

**The firmware you'll be working with** runs on an STM32F302 microcontroller (a small computer on a chip) and makes all of this possible.

---

## Prerequisites & Development Environment

### Required Software

| Tool | Purpose | Download |
|------|---------|----------|
| **STM32CubeIDE** | IDE for editing, building, and debugging | [st.com/stm32cubeide](https://www.st.com/en/development-tools/stm32cubeide.html) |
| **Git** | Version control | [git-scm.com](https://git-scm.com/) |
| **dfu-util** | Programming the device via USB | [dfu-util.sourceforge.net](http://dfu-util.sourceforge.net/) |

### Hardware for Development

- **AIOC board** - The device itself (can be ordered from JLCPCB, see main README)
- **USB-C cable** - To connect the AIOC to your computer
- **Optional: ST-LINK/V2** - Hardware debugger for step-by-step debugging

### Getting the Code

```bash
# Clone the repository
git clone https://github.com/skuep/AIOC.git
cd AIOC

# Initialize submodules (downloads TinyUSB library)
git submodule update --init
```

### Building for the First Time

1. Open **STM32CubeIDE**
2. Create a new workspace at `<project-root>/stm32`
3. Go to **File → Import → Existing Projects into Workspace**
4. Browse to `stm32/aioc-fw` and import (don't copy)
5. Select **Project → Build All** (use Release for production, Debug for development)

---

## Firmware Concepts for Beginners

If you're new to embedded development, this section explains the key concepts you'll encounter.

### What is a Microcontroller?

A microcontroller (MCU) is a tiny computer on a single chip. Unlike your desktop PC:

```
Desktop Computer                    Microcontroller (STM32F302)
─────────────────                   ──────────────────────────
CPU: Multi-GHz, multiple cores      CPU: 72 MHz, single core
RAM: 8-64 GB                        RAM: 16 KB (linker configured)
Storage: 256+ GB SSD                Storage: 128 KB Flash
OS: Windows/Linux/macOS             OS: None (bare-metal)
```

The AIOC runs on an **STM32F302** which has:
- **ARM Cortex-M4** processor at 72 MHz
- **128 KB Flash** (where your code lives permanently)
- **16 KB RAM** (for variables while running, as configured in linker script)
- Built-in **USB**, **ADC**, **DAC**, **Timers**, and more

### Bare-Metal Programming

Unlike desktop applications that run on an operating system, AIOC firmware is **bare-metal** - your code runs directly on the hardware with no OS underneath.

This means:
- **No `malloc`/`free`** - All memory is allocated at compile time
- **No threads** - Only one thing runs at a time (except interrupts)
- **Direct hardware access** - You read/write hardware registers directly

### The "Super Loop" Architecture

The firmware follows a simple pattern:

```c
int main(void) {
    // 1. Initialize everything once
    SystemClock_Config();
    Settings_Init();
    LED_Init();
    IO_Init();
    USB_Init();

    // 2. Loop forever
    while (1) {
        USB_Task();        // Handle USB events
        FoxHunt_Tick();    // Check morse beacon
        // ... other periodic tasks
    }
}
```

This is called a **super loop** - an infinite loop that runs tasks repeatedly.

### What are Peripherals?

Peripherals are hardware modules built into the microcontroller:

| Peripheral | Purpose in AIOC |
|------------|-----------------|
| **GPIO** | General Purpose I/O - controls PTT pins, reads buttons |
| **ADC** | Analog-to-Digital Converter - reads audio from radio |
| **DAC** | Digital-to-Analog Converter - sends audio to radio |
| **Timer** | Generates precise timing for audio sample rates |
| **UART** | Serial communication - bridges to virtual COM port |
| **USB** | Handles all USB communication with the host PC |

### Interrupts and Priority Levels

An **interrupt** is a signal that pauses your main code to handle something urgent. When audio needs to be processed at exactly 48,000 times per second, an interrupt ensures it happens on time.

```
Priority   Interrupt              What it does
────────   ─────────              ────────────
2 (High)   TIM6/TIM15, ADC        Audio sample processing
3          Serial                  USB serial data
4          USB                     USB core events
5          EXTI                    Button/input changes
6          SysTick                 System timing
7 (Low)    TIM4                    LED animations
```

Lower number = higher priority = runs first if multiple interrupts occur.

### Memory Layout

The microcontroller's memory is divided into regions:

```
┌────────────────────────────────────────────┐ 0x08020000
│                  FLASH (128 KB)            │
│  ┌──────────────────────────────────────┐  │
│  │  .eeprom section (settings storage)  │  │ 0x0801F000
│  ├──────────────────────────────────────┤  │
│  │  .rodata (constant strings, tables)  │  │
│  ├──────────────────────────────────────┤  │
│  │  .text (your code)                   │  │
│  ├──────────────────────────────────────┤  │
│  │  .isr_vector (interrupt handlers)    │  │ 0x08000000
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐ 0x20004000
│                   RAM (16 KB)              │
│  ┌──────────────────────────────────────┐  │
│  │  Stack (grows downward)         ↓    │  │ _estack
│  ├──────────────────────────────────────┤  │
│  │  .bss (zero-initialized globals)     │  │
│  ├──────────────────────────────────────┤  │
│  │  .data (initialized globals)         │  │ 0x20000000
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

**FLASH** is like an SSD - data survives power off but is slow to write.
**RAM** is fast but loses data when power is removed.

For more detail, see [docs/firmware_concepts.md](docs/firmware_concepts.md).

---

## Project Architecture Overview

### Directory Structure

```
AIOC/
├── stm32/
│   └── aioc-fw/                    # Firmware project
│       ├── Src/                    # Source files (your code)
│       │   ├── main.c              # Entry point
│       │   ├── usb_audio.c         # Audio streaming
│       │   ├── usb_serial.c        # Serial port + PTT
│       │   ├── usb_hid.c           # HID interface
│       │   ├── usb_descriptors.c   # USB device description
│       │   ├── settings.c          # Configuration storage
│       │   ├── led.c               # LED control
│       │   ├── io.c                # GPIO/button handling
│       │   └── fox_hunt.c          # Morse beacon
│       ├── Inc/                    # Header files
│       │   ├── aioc.h              # Interrupt priorities
│       │   └── tusb_config.h       # USB stack config
│       ├── Drivers/                # ST HAL library
│       └── Middlewares/
│           └── Third-Party/
│               └── tinyusb/        # USB stack (submodule)
├── kicad/                          # PCB design files
├── 3d/                             # Enclosure models
└── doc/                            # Datasheets
```

### Key Source Files

| File | Lines | Purpose |
|------|-------|---------|
| `main.c` | ~280 | Boot sequence, clock config, main loop |
| `usb_audio.c` | ~1270 | USB Audio Class, ADC/DAC streaming |
| `usb_serial.c` | ~230 | Virtual COM port, PTT via DTR/RTS |
| `usb_hid.c` | ~200 | CM108 emulation, settings access |
| `usb_descriptors.c` | ~500 | USB device/config/string descriptors |
| `settings.c` | ~120 | FLASH-based configuration storage |
| `led.c` | ~110 | PWM-based LED control |
| `io.c` | ~85 | GPIO and external interrupt handling |
| `fox_hunt.c` | ~150 | Morse code beacon generator |

### Boot Sequence

```
Power On
    │
    ▼
┌─────────────────┐
│  Reset_Handler  │  Assembly startup code
│  (startup_*.s)  │  Sets up stack, copies data to RAM
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  SystemReset()  │  Check if watchdog triggered (→ bootloader)
└────────┬────────┘
         │
         ▼
┌─────────────────────┐
│ SystemClock_Config()│  Set up 72 MHz from 8 MHz crystal
└────────┬────────────┘
         │
         ▼
┌─────────────────┐
│  Settings_Init  │  Load configuration from FLASH
├─────────────────┤
│  LED_Init       │  Set up PWM timer for LEDs
├─────────────────┤
│  IO_Init        │  Configure GPIO pins, enable interrupts
├─────────────────┤
│  USB_Init       │  Start TinyUSB stack, enable USB interrupt
├─────────────────┤
│  FoxHunt_Init   │  Set up morse beacon (if enabled)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Main Loop     │  ← Runs forever
│  ┌───────────┐  │
│  │ USB_Task  │  │  Handle USB events
│  │ FoxHunt   │  │  Check beacon timer
│  │ Watchdog  │  │  Prevent lockup
│  └───────────┘  │
└─────────────────┘
```

### Interrupt Priority Diagram

```
                    Highest Priority
                          │
    ┌─────────────────────┴─────────────────────┐
    │  AUDIO (Priority 2)                       │
    │  TIM6_DAC_IRQHandler - Speaker samples    │
    │  ADC1_2_IRQHandler   - Microphone samples │
    │  TIM15_IRQHandler    - Morse tone gen     │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────┴─────────────────────┐
    │  SERIAL (Priority 3)                      │
    │  USART1_IRQHandler - UART data            │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────┴─────────────────────┐
    │  USB (Priority 4)                         │
    │  USB_LP_IRQHandler  - USB events          │
    │  USB_HP_IRQHandler  - Isochronous xfers   │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────┴─────────────────────┐
    │  I/O (Priority 5)                         │
    │  EXTI9_5_IRQHandler - Button edges        │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────┴─────────────────────┐
    │  SYSTICK (Priority 6)                     │
    │  SysTick_Handler - 1ms system tick        │
    └─────────────────────┬─────────────────────┘
                          │
    ┌─────────────────────┴─────────────────────┐
    │  LED (Priority 7)                         │
    │  TIM4_IRQHandler - LED PWM animation      │
    └─────────────────────┬─────────────────────┘
                          │
                    Lowest Priority
```

---

## Core Subsystems

The firmware is organized into several subsystems. Here's a brief overview with links to detailed documentation.

### USB Stack

The AIOC presents itself as a **composite USB device** with 7 interfaces:

| Interface | Class | Purpose |
|-----------|-------|---------|
| 0 | Audio Control | Master for audio settings |
| 1 | Audio Streaming | Speaker (host → device) |
| 2 | Audio Streaming | Microphone (device → host) |
| 3 | HID | CM108 button emulation |
| 4-5 | CDC | Serial port control + data |
| 6 | DFU Runtime | Firmware update trigger |

**Key files**: `usb.c`, `usb_descriptors.c`, `tusb_config.h`

For details, see [docs/usb_architecture.md](docs/usb_architecture.md).

### Audio Processing

Audio flows through the device in both directions:

```
Radio                        AIOC                         Computer
─────                        ────                         ────────
       ┌──────────────────────────────────────────┐
 MIC ──┤ TRS Jack → ADC → USB Audio IN endpoint  ├──→ Soundcard IN
       │           (TIM3 triggers at sample rate) │
       │                                          │
 SPK ←─┤ TRS Jack ← DAC ← USB Audio OUT endpoint ├──← Soundcard OUT
       │           (TIM6 triggers at sample rate) │
       └──────────────────────────────────────────┘
```

Supports sample rates: 48kHz, 32kHz, 24kHz, 22.05kHz, 16kHz, 12kHz, 11.025kHz, 8kHz

**Key file**: `usb_audio.c`

For details, see [docs/audio_subsystem.md](docs/audio_subsystem.md).

### Settings & Configuration

Configuration is stored in FLASH memory and survives power cycles:

- **256 registers** (32-bit each) organized as a register map
- Addresses 0x00-0xBF are read/write
- Addresses 0xC0-0xFF are read-only debug info
- Accessed via USB HID feature reports

**Key files**: `settings.c`, `settings.h`

For details, see [docs/settings_system.md](docs/settings_system.md).

### I/O and PTT Control

PTT (Push-To-Talk) can be controlled from multiple sources:

```
┌─────────────────────────────────────────────────────────────┐
│                     PTT Control Sources                     │
├─────────────────────────────────────────────────────────────┤
│  Serial DTR/RTS  ──┐                                        │
│  HID GPIO bits   ──┼──→ IOMUX Settings ──→ PA1 (PTT1)      │
│  Audio Voice PTT ──┤                   └──→ PA0 (PTT2)      │
│  Audio Voice COS ──┘                                        │
└─────────────────────────────────────────────────────────────┘
```

**Key files**: `io.c`, `io.h`, `usb_serial.c`, `usb_hid.c`

For details, see [docs/io_and_peripherals.md](docs/io_and_peripherals.md).

---

## Making Your First Change

Ready to modify the firmware? Here are some examples to get you started.

### Example 1: Change LED Behavior

The LEDs pulse when USB is active. To change the idle brightness:

1. Open `stm32/aioc-fw/Src/led.c`
2. Find these constants near the top:
   ```c
   #define LED_OFF_LEVEL   0
   #define LED_IDLE_LEVEL  25    // ← Change this (0-255)
   #define LED_FULL_LEVEL  249
   ```
3. Rebuild and flash

### Example 2: Add a New Setting Register

To add a configurable value that persists across reboots:

1. Open `stm32/aioc-fw/Inc/settings.h`
2. Add a new register definition:
   ```c
   #define SETTINGS_REG_MY_FEATURE     0x50  // Pick unused address
   #define SETTINGS_REG_MY_FEATURE_DEFAULT 100
   ```
3. Open `stm32/aioc-fw/Src/settings.c`
4. In `Settings_Default()`, add:
   ```c
   settingsRegMap[SETTINGS_REG_MY_FEATURE] = SETTINGS_REG_MY_FEATURE_DEFAULT;
   ```
5. Use it in your code:
   ```c
   uint32_t myValue = Settings_RegRead(SETTINGS_REG_MY_FEATURE);
   ```

### Where to Look for Different Changes

| I want to... | Look in... |
|--------------|------------|
| Change USB behavior | `usb_*.c`, `tusb_config.h` |
| Modify audio processing | `usb_audio.c` |
| Change PTT logic | `usb_serial.c`, `io.c` |
| Add a new setting | `settings.h`, `settings.c` |
| Change LED patterns | `led.c` |
| Modify morse beacon | `fox_hunt.c`, `morse.c` |
| Change clock/timing | `main.c` (SystemClock_Config) |

---

## Building, Flashing, and Debugging

### Build Configurations

| Configuration | Optimization | Debug Symbols | Use For |
|---------------|--------------|---------------|---------|
| **Debug** | `-Og` | Yes | Development, debugging |
| **Release** | `-Os` | Minimal | Production firmware |

### Programming via DFU

**First time (device is unprogrammed):**
1. Short the bootloader pins on the programming header
2. Connect USB
3. Run: `dfu-util -a 0 -s 0x08000000:leave -D aioc-fw.bin`
4. Remove short and reconnect

**Updating existing firmware (v1.2.0+):**
```bash
dfu-util -d 1209:7388 -a 0 -s 0x08000000:leave -D aioc-fw.bin
```
This automatically reboots into bootloader and programs.

### Using a Hardware Debugger (ST-LINK)

For step-by-step debugging:

1. Connect ST-LINK to the SWD header (SWDIO, SWCLK, GND)
2. In STM32CubeIDE: **Run → Debug Configurations**
3. Create new **STM32 C/C++ Application**
4. Select your ST-LINK in the Debugger tab
5. Click **Debug** to flash and start debugging

### Reading Debug Output (ITM/SWO)

The firmware can output debug messages via SWO (Serial Wire Output):

1. Connect SWO pin (PB3) to your debugger
2. In STM32CubeIDE: **Window → Show View → SWV → SWV ITM Data Console**
3. Configure for 72 MHz core clock
4. Enable port 0
5. Use `printf()` in code - output appears in console

---

## Resources & Further Reading

### Official Documentation

- **STM32F302 Reference Manual** (RM0365) - `doc/PDFs/rm0365-stm32f302xbcde.pdf`
- **STM32F302 Datasheet** - `doc/PDFs/stm32f302cb.pdf`
- **TinyUSB Documentation** - [docs.tinyusb.org](https://docs.tinyusb.org/)

### Detailed Subsystem Guides

- [Firmware Concepts Deep Dive](docs/firmware_concepts.md)
- [USB Architecture](docs/usb_architecture.md)
- [Audio Subsystem](docs/audio_subsystem.md)
- [Settings System](docs/settings_system.md)
- [I/O and Peripherals](docs/io_and_peripherals.md)

### Ham Radio Background

- **APRS** - Automatic Packet Reporting System
- **Direwolf** - Software TNC (modem) for APRS
- **CHIRP** - Radio programming software

### Community

- [AIOC Discord](https://discord.gg/wCbXu9R95C)
- [GitHub Issues](https://github.com/skuep/AIOC/issues)

---

*Last updated: February 2025*
