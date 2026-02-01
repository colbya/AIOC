# Settings System

This document explains how AIOC stores and manages configuration settings in FLASH memory.

## Table of Contents

1. [Overview](#overview)
2. [Register Map](#register-map)
3. [FLASH Persistence](#flash-persistence)
4. [Accessing Settings](#accessing-settings)
5. [Adding New Settings](#adding-new-settings)
6. [Debug Registers](#debug-registers)

---

## Overview

AIOC uses a register-based configuration system:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Settings Architecture                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │               RAM Cache (settingsRegMap[])              │   │
│  │                                                         │   │
│  │  Address 0x00-0xBF: Read/Write configuration           │   │
│  │  Address 0xC0-0xFF: Read-only debug/status info        │   │
│  │                                                         │   │
│  │  256 registers × 32 bits = 1024 bytes                  │   │
│  └───────────────────────────┬─────────────────────────────┘   │
│                              │                                  │
│           ┌──────────────────┼──────────────────┐              │
│           │                  │                  │              │
│           ▼                  ▼                  ▼              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │  USB HID    │    │   Code      │    │   FLASH     │        │
│  │  Feature    │    │   Access    │    │  Storage    │        │
│  │  Reports    │    │             │    │  (.eeprom)  │        │
│  └─────────────┘    └─────────────┘    └─────────────┘        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Key Features

- **256 registers** (32-bit each)
- **Persistent storage** in FLASH
- **USB HID access** for host configuration
- **Magic number** validates stored settings
- **Survives firmware updates** (stored at fixed FLASH address)

---

## Register Map

### Configuration Registers (0x00-0xBF)

| Address | Name | Default | Description |
|---------|------|---------|-------------|
| 0x00 | MAGIC | 'AIOC' | Validates settings integrity |
| 0x24 | AIOC_IOMUX0 | - | PTT1 source selection |
| 0x25 | AIOC_IOMUX1 | - | PTT2 source selection |
| 0x44 | CM108_IOMUX0 | - | HID button mapping |
| 0x45 | CM108_IOMUX1 | - | HID button mapping |
| 0x46 | CM108_IOMUX2 | - | HID button mapping |
| 0x47 | CM108_IOMUX3 | - | HID button mapping |
| 0x64 | SERIAL_IOMUX0 | - | Serial DTR routing |
| 0x65 | SERIAL_IOMUX1 | - | Serial RTS routing |
| 0x66 | SERIAL_IOMUX2 | - | Serial line state input |
| 0x67 | SERIAL_IOMUX3 | - | Serial line state input |
| 0x72 | RX_GAIN | 0 | ADC input gain (0-4) |
| 0x78 | TX_BOOST | 0 | DAC output level (0-1) |
| 0x82 | VPTT_THRESHOLD | 0 | Virtual PTT audio threshold |
| 0x84 | VPTT_TAIL_TIME | 0 | Virtual PTT hold time (12.4 ms) |
| 0x92 | VCOS_THRESHOLD | 0 | Virtual COS audio threshold |
| 0x94 | VCOS_TAIL_TIME | 0 | Virtual COS hold time (12.4 ms) |
| 0xA0 | FOXHUNT_CTRL | - | Fox hunt control (interval, WPM, volume) |
| 0xA2 | FOXHUNT_MSG0 | 0 | Message characters 0-3 |
| 0xA3 | FOXHUNT_MSG1 | 0 | Message characters 4-7 |
| 0xA4 | FOXHUNT_MSG2 | 0 | Message characters 8-11 |
| 0xA5 | FOXHUNT_MSG3 | 0 | Message characters 12-15 |

### IOMUX Bit Definitions

The IOMUX registers control signal routing:

```
AIOC_IOMUX0/1 (PTT1/PTT2 sources):
  Bit 0: SERIAL_DTR
  Bit 1: SERIAL_RTS
  Bit 2: SERIAL_DTR_AND_NOT_RTS
  Bit 3: SERIAL_NOT_DTR_AND_RTS
  Bit 4: HID_GPIO_0
  Bit 5: HID_GPIO_1
  Bit 6: HID_GPIO_2
  Bit 7: HID_GPIO_3
  Bit 8: VPTT (Virtual PTT from audio)
  Bit 9: VCOS (Virtual COS from audio)
```

### Gain/Boost Values

```
RX_GAIN values:
  0 = 1x gain (line level)
  1 = 2x gain
  2 = 4x gain
  3 = 8x gain
  4 = 16x gain (mic level)

TX_BOOST values:
  0 = Attenuated output (mic level)
  1 = Full output (line level)
```

---

## FLASH Persistence

Settings are stored in a dedicated FLASH region.

### Memory Layout

```
┌──────────────────────────────────────────────────────────┐
│                    FLASH Memory                          │
├──────────────────────────────────────────────────────────┤
│  0x08000000  ┌───────────────────────────────────────┐   │
│              │          Firmware Code                │   │
│              │                                       │   │
│              │                                       │   │
│              └───────────────────────────────────────┘   │
│              ┌───────────────────────────────────────┐   │
│  0x0801F000  │     .eeprom Section (4 KB)            │   │
│              │     Settings storage area             │   │
│              │     256 × 4 bytes = 1024 bytes used   │   │
│  0x08020000  └───────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

### Linker Script Declaration

From `stm32f30_flash.ld`:

```ld
MEMORY {
    FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 128K
    RAM (xrw)  : ORIGIN = 0x20000000, LENGTH = 40K
}

SECTIONS {
    /* ... other sections ... */

    .eeprom 0x0801F000 : {
        _eeprom = .;
        KEEP(*(.eeprom))
    } > FLASH
}
```

### Store Operation

```c
void Settings_Store(void) {
    // 1. Unlock FLASH
    HAL_FLASH_Unlock();

    // 2. Erase pages (2 KB each)
    FLASH_EraseInitTypeDef EraseInit = {
        .TypeErase = FLASH_TYPEERASE_PAGES,
        .PageAddress = (uint32_t)&_eeprom,
        .NbPages = (SETTINGS_REGMAP_SIZE * 4 + 2047) / 2048
    };
    uint32_t PageError;
    HAL_FLASHEx_Erase(&EraseInit, &PageError);

    // 3. Program word by word
    for (uint32_t i = 0; i < SETTINGS_REGMAP_SIZE; i++) {
        HAL_FLASH_Program(
            FLASH_TYPEPROGRAM_WORD,
            (uint32_t)&_eeprom + (i * 4),
            settingsRegMap[i]
        );
    }

    // 4. Lock FLASH
    HAL_FLASH_Lock();
}
```

### Recall Operation

```c
void Settings_Recall(void) {
    // Check magic number
    if (_eeprom[0] != SETTINGS_REG_MAGIC_DEFAULT) {
        // Invalid or unprogrammed - use defaults
        Settings_Default();
        return;
    }

    // Copy from FLASH to RAM
    for (uint32_t i = 0; i < SETTINGS_REGMAP_SIZE; i++) {
        settingsRegMap[i] = _eeprom[i];
    }
}
```

### FLASH Endurance

- **Write cycles**: ~10,000 per page
- **Retention**: 30+ years at room temperature
- **Page size**: 2 KB

Settings are only written when explicitly requested, so wear is minimal.

---

## Accessing Settings

### From Firmware Code

```c
// Read a setting
uint32_t gain = Settings_RegRead(SETTINGS_REG_RX_GAIN);

// Write a setting (to RAM cache only)
Settings_RegWrite(SETTINGS_REG_RX_GAIN, 2);  // Set 4x gain

// Persist to FLASH (call sparingly)
Settings_Store();
```

### Atomic Access

Settings access disables interrupts for thread safety:

```c
uint32_t Settings_RegRead(uint8_t address) {
    uint32_t data;
    __disable_irq();
    data = settingsRegMap[address];
    __enable_irq();
    return data;
}

void Settings_RegWrite(uint8_t address, uint32_t data) {
    if (address >= SETTINGS_REGMAP_RO_ADDRESS) {
        return;  // Read-only region
    }
    __disable_irq();
    settingsRegMap[address] = data;
    __enable_irq();
}
```

### From USB HID

The host can access settings via HID Feature Reports:

```
GET_FEATURE Report:
  Send: [0x00, address, 0, 0, 0, 0]
  Receive: [0x00, address, data0, data1, data2, data3]

SET_FEATURE Report:
  Send: [0x01, address, data0, data1, data2, data3]
  (0x01 = write flag)

Special control bits:
  0x10 = Reset to defaults
  0x20 = Reboot device
  0x40 = Recall from FLASH
  0x80 = Store to FLASH
```

### Command-Line Tool

The [aioc-util](https://github.com/hrafnkelle/aioc-util) provides easy access:

```bash
# Read a register
aioc-util read 0x72

# Write a register
aioc-util write 0x72 2

# Store to FLASH
aioc-util store

# Reset to defaults
aioc-util reset
```

---

## Adding New Settings

### Step 1: Define the Register

In `settings.h`:

```c
// Choose an unused address in range 0x01-0xBF
#define SETTINGS_REG_MY_FEATURE         0x50
#define SETTINGS_REG_MY_FEATURE_DEFAULT 100
```

### Step 2: Add Default Value

In `settings.c` `Settings_Default()`:

```c
void Settings_Default(void) {
    // ... existing defaults ...

    settingsRegMap[SETTINGS_REG_MY_FEATURE] = SETTINGS_REG_MY_FEATURE_DEFAULT;
}
```

### Step 3: Use the Setting

In your code:

```c
void MyFeature_Init(void) {
    uint32_t config = Settings_RegRead(SETTINGS_REG_MY_FEATURE);

    if (config > 50) {
        // Do something
    }
}
```

### Step 4: Document It

Add the register to this documentation and any user-facing docs.

---

## Debug Registers

Read-only registers (0xC0-0xFF) provide runtime status information.

### Available Debug Registers

| Address | Name | Description |
|---------|------|-------------|
| 0xC0 | INFO_PTT | Current PTT state (bit 0=PTT1, bit 1=PTT2) |
| 0xC1 | INFO_AUDIO0 | Audio state flags |
| 0xC2 | INFO_AUDIO1 | Current sample rate |
| 0xC3 | INFO_AUDIO2 | Speaker buffer level |
| 0xC4 | INFO_AUDIO3 | Mic buffer level |
| 0xC5 | INFO_FEEDBACK_MIN | Minimum feedback value |
| 0xC6 | INFO_FEEDBACK_MAX | Maximum feedback value |
| 0xC7 | INFO_FEEDBACK_AVG | Average feedback value |

### Audio State Flags (INFO_AUDIO0)

```
Bit 0: Speaker mute
Bit 1: Mic mute
Bit 2: Speaker streaming active
Bit 3: Mic streaming active
Bit 4: VPTT active
Bit 5: VCOS active
```

### Reading Debug Info

```c
// Check if PTT is active
uint32_t pttState = Settings_RegRead(SETTINGS_REG_INFO_PTT);
if (pttState & 0x01) {
    // PTT1 is asserted
}

// Get current sample rate
uint32_t sampleRate = Settings_RegRead(SETTINGS_REG_INFO_AUDIO1);
```

These registers are useful for:
- Debugging PTT issues
- Monitoring audio buffer health
- Verifying feedback endpoint operation

---

*[Back to main guide](../new_dev_intro.md)*
