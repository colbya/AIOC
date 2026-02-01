# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AIOC (All-In-One Cable) is embedded C firmware for an STM32F302 microcontroller that implements a multi-interface USB device for ham radio applications. The device acts as a USB-C adapter providing:
- USB Audio Class (soundcard) for digital modes like APRS
- CDC Serial (virtual COM port) for radio programming and PTT control
- HID interface (CM108-compatible) for alternative PTT control
- DFU runtime for firmware updates

## Build Commands

**Prerequisites**: STM32CubeIDE, git with submodules initialized

```bash
# Clone and initialize
git clone <repository-url>
git submodule update --init

# Build
# Open STM32CubeIDE, create workspace at <project-root>/stm32
# File -> Import -> aioc-fw project (don't copy)
# Project -> Build All (use Release build for production)
```

**Programming**:
```bash
# Initial programming (short bootloader pins first)
dfu-util -a 0 -s 0x08000000:leave -D aioc-fw-x-y-z.bin

# Firmware update (v1.2.0+, auto-reboots to bootloader)
dfu-util -d 1209:7388 -a 0 -s 0x08000000:leave -D aioc-fw-x-y-z.bin
```

## Architecture

### Firmware Structure (`stm32/aioc-fw/`)
- `Src/` - Application source files
  - `main.c` - Entry point, system init, DFU reboot support
  - `usb_audio.c` - USB Audio Class implementation (~1270 lines), DMA-driven ADC/DAC streaming
  - `usb_serial.c` - CDC serial bridge with PTT control (DTR=1 && RTS=0 asserts PTT)
  - `usb_hid.c` - CM108-compatible HID for PTT
  - `usb_descriptors.c` - Composite USB device descriptors
  - `settings.c` - Persistent FLASH-based configuration storage
  - `fox_hunt.c` / `morse.c` - Morse code beacon mode
  - `led.c` / `io.c` - GPIO/LED control with PWM
- `Inc/` - Headers including `aioc.h` (IRQ priorities), `tusb_config.h` (USB class config)
- `Drivers/` - ST HAL and CMSIS libraries
- `Middlewares/Third-Party/tinyusb/` - TinyUSB stack (git submodule)

### Key Design Patterns

**USB Composite Device**: Four interfaces run simultaneously - audio streaming uses DMA with feedback endpoint for clock sync, CDC handles serial bridging, HID provides CM108 PTT compatibility.

**Interrupt Priority Hierarchy** (defined in `aioc.h`):
- Audio: Priority 2 (highest)
- Serial: Priority 3
- USB: Priority 4
- I/O (EXTI): Priority 5
- SysTick: Priority 6
- LED: Priority 7 (lowest)

**Settings System**: Register-based persistent storage in STM32 embedded FLASH, survives firmware updates. Configures audio levels, I/O multiplexing, PTT sources.

**Audio Sample Rates**: 48kHz (preferred), 32kHz, 24kHz, 22.05kHz (APRSdroid), 16kHz, 12kHz, 11.025kHz, 8kHz

### Hardware Design (`kicad/k1-aioc/`)
- KiCAD schematic and PCB layout
- JLCPCB fabrication files in `jlcpcb/` subdirectory
- Current hardware version: 1.2 (requires firmware >= 1.4.0)

### Dependencies
- **TinyUSB** (git submodule) - USB device stack
- **ST HAL** - Hardware abstraction layer (included)
- **ARM CMSIS** - Cortex-M core support (included)

## Hardware Specifications
- MCU: STM32F302CBTx (ARM Cortex-M4, 72MHz, 128KB Flash, 16KB RAM as configured)
- USB: Full-Speed (12 Mbps), USB-C connector
- Audio: Mono in/out via internal 12-bit ADC/DAC
