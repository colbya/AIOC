# USB Architecture

This document explains the USB implementation in AIOC firmware, including how the composite device is structured and how each interface works.

## Table of Contents

1. [Composite Device Overview](#composite-device-overview)
2. [USB Descriptors](#usb-descriptors)
3. [USB Audio Class (UAC2)](#usb-audio-class-uac2)
4. [CDC Serial Interface](#cdc-serial-interface)
5. [HID Interface (CM108)](#hid-interface-cm108)
6. [DFU Runtime](#dfu-runtime)
7. [TinyUSB Configuration](#tinyusb-configuration)

---

## Composite Device Overview

AIOC presents itself to the host as a **composite USB device** - a single physical device that contains multiple logical devices:

```
┌─────────────────────────────────────────────────────────────────┐
│                    AIOC USB Device                              │
│                    VID: 0x1209  PID: 0x7388                     │
├─────────────────────────────────────────────────────────────────┤
│  Configuration 1 (100mA)                                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  IAD: Audio Function                                      │  │
│  │  ├── Interface 0: Audio Control                           │  │
│  │  ├── Interface 1: Audio Streaming OUT (Speaker)           │  │
│  │  └── Interface 2: Audio Streaming IN (Microphone)         │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │  Interface 3: HID (CM108 compatible)                      │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │  IAD: CDC Function                                        │  │
│  │  ├── Interface 4: CDC Control                             │  │
│  │  └── Interface 5: CDC Data                                │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │  Interface 6: DFU Runtime                                 │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Endpoint Allocation

```
Endpoint   Direction   Type          Used By
────────   ─────────   ────          ───────
0x00/0x80  OUT/IN      Control       Default control pipe
0x02       OUT         Isochronous   Audio OUT (speaker)
0x81       IN          Isochronous   Audio IN (microphone)
0x82       IN          Isochronous   Audio feedback
0x03       OUT         Interrupt     HID OUT
0x83       IN          Interrupt     HID IN
0x04       OUT         Bulk          CDC Data OUT
0x84       IN          Bulk          CDC Data IN
0x85       IN          Interrupt     CDC Notifications
```

---

## USB Descriptors

USB descriptors tell the host what the device is and what it can do.

### Device Descriptor

From `usb_descriptors.c`:

```c
tusb_desc_device_t const desc_device = {
    .bLength            = sizeof(tusb_desc_device_t),
    .bDescriptorType    = TUSB_DESC_DEVICE,
    .bcdUSB             = 0x0200,           // USB 2.0
    .bDeviceClass       = TUSB_CLASS_MISC,  // Composite device
    .bDeviceSubClass    = MISC_SUBCLASS_COMMON,
    .bDeviceProtocol    = MISC_PROTOCOL_IAD,
    .bMaxPacketSize0    = CFG_TUD_ENDPOINT0_SIZE,  // 8 bytes
    .idVendor           = 0x1209,           // pid.codes VID
    .idProduct          = 0x7388,           // AIOC PID
    .bcdDevice          = 0x0100,           // Version 1.0
    .iManufacturer      = 0x01,
    .iProduct           = 0x02,
    .iSerialNumber      = 0x03,
    .bNumConfigurations = 0x01
};
```

### Interface Association Descriptors (IAD)

IADs group multiple interfaces into a single function:

```
Audio IAD (interfaces 0-2):
  - bFirstInterface = 0
  - bInterfaceCount = 3
  - bFunctionClass = AUDIO

CDC IAD (interfaces 4-5):
  - bFirstInterface = 4
  - bInterfaceCount = 2
  - bFunctionClass = CDC
```

### String Descriptors

String index 3 (serial number) is dynamically generated from the STM32's unique ID:

```c
case 3:  // Serial number
    uint32_t uid_crc = HAL_CRC_Calculate(...);
    // Convert to hex string
```

---

## USB Audio Class (UAC2)

The audio interface provides soundcard functionality for digital modes.

### Audio Topology

```
                        AIOC Audio Topology
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Speaker Path (Host → Device):                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │  USB IN  │───►│ Feature  │───►│  Output  │───►│  Radio   │  │
│  │ Terminal │    │  Unit    │    │ Terminal │    │  (TRS)   │  │
│  │ ID=0x01  │    │ ID=0x02  │    │ ID=0x03  │    │          │  │
│  └──────────┘    │Vol/Mute  │    └──────────┘    └──────────┘  │
│                  └──────────┘                                   │
│                                                                 │
│  Microphone Path (Device → Host):                               │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │  Radio   │───►│  Input   │───►│ Feature  │───►│ USB OUT  │  │
│  │  (TRS)   │    │ Terminal │    │  Unit    │    │ Terminal │  │
│  │          │    │ ID=0x11  │    │ ID=0x12  │    │ ID=0x13  │  │
│  └──────────┘    └──────────┘    │Vol/Mute  │    └──────────┘  │
│                                  └──────────┘                   │
│                                                                 │
│  Clock Sources:                                                 │
│  ┌──────────┐              ┌──────────┐                        │
│  │  Speaker │              │   Mic    │                        │
│  │  Clock   │              │  Clock   │                        │
│  │ ID=0x08  │              │ ID=0x18  │                        │
│  └──────────┘              └──────────┘                        │
└─────────────────────────────────────────────────────────────────┘
```

### Supported Sample Rates

The firmware supports multiple sample rates for compatibility:

| Rate | Use Case | Notes |
|------|----------|-------|
| 48000 Hz | Default, Direwolf | Preferred |
| 32000 Hz | General | |
| 24000 Hz | General | |
| 22050 Hz | APRSdroid | ~90ppm error due to crystal |
| 16000 Hz | General | |
| 12000 Hz | General | |
| 11025 Hz | Legacy | ~90ppm error |
| 8000 Hz | Minimum | |

### Audio Streaming

Audio uses **isochronous endpoints** for guaranteed bandwidth:

```
Speaker (EP 0x02 OUT):
  - 48 samples × 2 bytes × 1 channel = 96 bytes per USB frame
  - USB frame = 1ms
  - Asynchronous mode with feedback endpoint

Microphone (EP 0x81 IN):
  - Same format as speaker
  - Device paces the data

Feedback (EP 0x82 IN):
  - Reports actual sample rate to host
  - Prevents buffer over/underrun
```

### Feedback Mechanism

The feedback endpoint tells the host the exact sample rate:

```c
void tud_audio_feedback_interval_isr(uint8_t func_id, ...) {
    // Measure time since last SOF using TIM2
    uint32_t cycles = TIM2->CNT - lastCount;

    // Calculate feedback: samples per USB frame in 16.16 format
    // feedback = (cycles × sampleFreq) / masterClock
    uint64_t fb64 = ((uint64_t)cycles * sampleFreq) << 16;
    fb64 /= 72000000;  // 72 MHz master clock

    // Adjust based on buffer level to prevent drift
    int32_t bufferError = bufferLevel - targetLevel;
    feedback -= bufferError;

    tud_audio_n_fb_set(func_id, feedback);
}
```

### Audio Callbacks

Key TinyUSB callbacks in `usb_audio.c`:

| Callback | Purpose |
|----------|---------|
| `tud_audio_set_itf_cb` | Interface activated (alt setting changed) |
| `tud_audio_set_itf_close_EP_cb` | Interface deactivated |
| `tud_audio_set_req_entity_cb` | Host sets volume/mute/sample rate |
| `tud_audio_get_req_entity_cb` | Host queries audio properties |
| `tud_audio_tx_done_pre_load_cb` | Ready to send mic data |
| `tud_audio_rx_done_post_read_cb` | Speaker data received |
| `tud_audio_feedback_interval_isr` | Time to send feedback |

---

## CDC Serial Interface

The CDC (Communications Device Class) interface provides a virtual serial port.

### Purpose

- **Radio programming**: CHIRP uses this to program radios
- **PTT control**: Asserted via DTR/RTS signals
- **Data bridge**: UART passthrough to radio

### UART Configuration

```c
// Default: 9600 8N1
USART1 on PA9 (TX) and PA10 (RX)

// Host can change via SET_LINE_CODING:
- Baud rate: Any
- Data bits: 8 only
- Parity: None, Odd, Even
- Stop bits: 1, 1.5, 2
```

### PTT Control via DTR/RTS

The serial port controls PTT through modem control signals:

```c
// From usb_serial.c tud_cdc_line_state_cb()
void tud_cdc_line_state_cb(uint8_t itf, bool dtr, bool rts) {
    // Default behavior: PTT = DTR && !RTS
    // This allows CHIRP to work (sets DTR during programming)
    // while still supporting PTT for digital modes

    if (dtr && !rts) {
        IO_PTTAssert(PTT_MASK_1);  // Assert PTT
    } else {
        IO_PTTDeassert(PTT_MASK_1);
    }
}
```

Configurable via settings:
- `SERIAL_DTR`
- `SERIAL_RTS`
- `SERIAL_DTR_AND_NOT_RTS` (default)
- `SERIAL_NOT_DTR_AND_RTS`

### Data Flow

```
Host → AIOC:
  tud_cdc_rx_cb() → Enable USART1 TX → TXE interrupt → Send byte

AIOC → Host:
  USART1 RXNE interrupt → tud_cdc_n_write() → USB IN endpoint
```

---

## HID Interface (CM108)

The HID interface emulates a CM108 audio chip for legacy PTT control.

### Why CM108?

Many ham radio applications (Direwolf, fldigi) support the CM108 chip for PTT control. AIOC emulates this for compatibility.

### HID Report Format

```
INPUT Report (4 bytes, device → host):
  [0] Buttons: VolUp(1) | VolDn(1) | PlayMute(1) | RecMute(1) | 0000
  [1] GPIO state
  [2-3] Reserved

OUTPUT Report (4 bytes, host → device):
  [0] Control bits
  [1] GPIO outputs (bits 0-3 control PTT)
  [2-3] Reserved

FEATURE Report (6 bytes, bidirectional):
  [0] Control: Write(1) | Default(4) | Reboot(5) | Recall(6) | Store(7)
  [1] Settings register address
  [2-5] 32-bit data value
```

### Settings Access via HID

The feature report provides direct access to the settings register map:

```c
// Read setting
GET_FEATURE:
  [0] = 0x00           // No control bits
  [1] = address        // Register to read
  Response:
  [2-5] = value        // 32-bit register value

// Write setting
SET_FEATURE:
  [0] = 0x01           // Write bit set
  [1] = address        // Register to write
  [2-5] = value        // New value
```

### Special Control Bits

| Bit | Action |
|-----|--------|
| 0x01 | Write data to register |
| 0x10 | Reset to defaults |
| 0x20 | Reboot device |
| 0x40 | Recall settings from FLASH |
| 0x80 | Store settings to FLASH |

---

## DFU Runtime

Device Firmware Update allows reprogramming without physical bootloader pins.

### How It Works

1. Host sends DFU_DETACH request to interface 6
2. Firmware enables watchdog timer
3. Watchdog triggers reset
4. On reset, firmware detects watchdog reset and jumps to bootloader
5. Host can now program new firmware

```c
void tud_dfu_runtime_reboot_to_dfu_cb(void) {
    // Enable window watchdog with short timeout
    WWDG_HandleTypeDef hwwdg = {
        .Instance = WWDG,
        .Init.Prescaler = WWDG_PRESCALER_1,
        .Init.Window = 0x7F,
        .Init.Counter = 0x50,
    };
    HAL_WWDG_Init(&hwwdg);
    // Device will reset before returning
}
```

### Programming After DFU_DETACH

```bash
# This command triggers DFU_DETACH automatically
dfu-util -d 1209:7388 -a 0 -s 0x08000000:leave -D firmware.bin
```

---

## TinyUSB Configuration

TinyUSB is the USB stack used by AIOC. Configuration is in `tusb_config.h`.

### Key Settings

```c
// Device mode only
#define CFG_TUD_ENABLED       1
#define CFG_TUD_MAX_SPEED     OPT_MODE_FULL_SPEED

// Classes enabled
#define CFG_TUD_AUDIO         1
#define CFG_TUD_CDC           1
#define CFG_TUD_HID           1
#define CFG_TUD_DFU_RUNTIME   1

// Buffer sizes
#define CFG_TUD_AUDIO_FUNC_1_EP_IN_SW_BUF_SZ   1024
#define CFG_TUD_AUDIO_FUNC_1_EP_OUT_SW_BUF_SZ  1024
#define CFG_TUD_CDC_RX_BUFSIZE  512
#define CFG_TUD_CDC_TX_BUFSIZE  512
#define CFG_TUD_HID_EP_BUFSIZE  8

// Audio format
#define CFG_TUD_AUDIO_FUNC_1_N_CHANNELS_RX  1  // Mono
#define CFG_TUD_AUDIO_FUNC_1_N_CHANNELS_TX  1
#define CFG_TUD_AUDIO_FUNC_1_N_BYTES_PER_SAMPLE_RX  2  // 16-bit
#define CFG_TUD_AUDIO_FUNC_1_N_BYTES_PER_SAMPLE_TX  2
```

### TinyUSB Callbacks

The firmware implements these TinyUSB callbacks:

```c
// Device callbacks (usb.c)
void tud_mount_cb(void);       // Device configured
void tud_umount_cb(void);      // Device removed
void tud_suspend_cb(bool);     // Bus suspended
void tud_resume_cb(void);      // Bus resumed

// Descriptor callbacks (usb_descriptors.c)
uint8_t const* tud_descriptor_device_cb(void);
uint8_t const* tud_descriptor_configuration_cb(uint8_t index);
uint16_t const* tud_descriptor_string_cb(uint8_t index, uint16_t langid);

// Audio callbacks (usb_audio.c)
// CDC callbacks (usb_serial.c)
// HID callbacks (usb_hid.c)
// DFU callbacks (usb_dfu.c)
```

### OS Detection Quirk

Windows requires a different feedback endpoint format than macOS/Linux:

```c
// From usb_descriptors.c
// Detect OS based on descriptor request pattern
// macOS: STRING → BOS/CONFIG (standard order)
// Windows: Different pattern

if (isWindows) {
    // Use 4-byte feedback (16.16 format)
} else {
    // Use 3-byte feedback (10.14 format)
}
```

---

*[Back to main guide](../new_dev_intro.md)*
