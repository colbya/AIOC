# I/O and Peripherals

This document explains GPIO configuration, LED control, external inputs, and the fox hunt morse beacon.

## Table of Contents

1. [GPIO Configuration](#gpio-configuration)
2. [LED PWM Control](#led-pwm-control)
3. [External Input Handling](#external-input-handling)
4. [PTT Routing via IOMUX](#ptt-routing-via-iomux)
5. [Fox Hunt Morse Beacon](#fox-hunt-morse-beacon)

---

## GPIO Configuration

AIOC uses several GPIO pins for different purposes.

### Pin Assignments

```
┌─────────────────────────────────────────────────────────────────┐
│                    STM32F302 Pin Usage                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Port A:                                                        │
│    PA0  - PTT2 output (push-pull, active high)                 │
│    PA1  - PTT1 output (push-pull, active high)                 │
│    PA3  - TX level control (output, attenuator select)         │
│    PA4  - DAC output (analog, audio to radio)                  │
│    PA9  - USART1 TX (alternate function)                       │
│    PA10 - USART1 RX (alternate function)                       │
│    PA11 - USB D- (alternate function)                          │
│    PA12 - USB D+ (alternate function)                          │
│                                                                 │
│  Port B:                                                        │
│    PB2  - ADC input (analog, audio from radio)                 │
│    PB3  - SWO debug output (alternate function)                │
│    PB6  - External input 1 (input with pull-up, EXTI)         │
│    PB7  - External input 2 (input with pull-up, EXTI)         │
│    PB8  - LED1 (TIM4_CH3, PWM output)                         │
│    PB9  - LED2 (TIM4_CH4, PWM output)                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### GPIO Initialization

From `io.h` (inline functions):

```c
static inline void IO_Init(void) {
    // Enable GPIO clocks
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();

    // Configure PTT outputs (PA0, PA1)
    GPIO_InitTypeDef GPIO_InitStruct = {
        .Pin = GPIO_PIN_0 | GPIO_PIN_1,
        .Mode = GPIO_MODE_OUTPUT_PP,
        .Pull = GPIO_PULLDOWN,
        .Speed = GPIO_SPEED_FREQ_LOW
    };
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    // Configure external inputs (PB6, PB7) with EXTI
    GPIO_InitStruct.Pin = GPIO_PIN_6 | GPIO_PIN_7;
    GPIO_InitStruct.Mode = GPIO_MODE_IT_RISING_FALLING;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

    // Enable EXTI interrupt
    NVIC_SetPriority(EXTI9_5_IRQn, AIOC_IRQ_PRIO_IO);
    NVIC_EnableIRQ(EXTI9_5_IRQn);
}
```

### PTT Control Functions

```c
// Assert PTT (set pin high)
static inline void IO_PTTAssert(uint8_t pttMask) {
    if (pttMask & 0x01) {
        GPIOA->BSRR = GPIO_PIN_1;  // PTT1
        LED_SET(0, 1);             // LED indication
    }
    if (pttMask & 0x02) {
        GPIOA->BSRR = GPIO_PIN_0;  // PTT2
        LED_SET(1, 1);
    }
}

// Deassert PTT (set pin low)
static inline void IO_PTTDeassert(uint8_t pttMask) {
    if (pttMask & 0x01) {
        GPIOA->BSRR = GPIO_PIN_1 << 16;  // Reset bit
        LED_SET(0, 0);
    }
    if (pttMask & 0x02) {
        GPIOA->BSRR = GPIO_PIN_0 << 16;
        LED_SET(1, 0);
    }
}
```

---

## LED PWM Control

The two LEDs use PWM for brightness control and animations.

### Hardware Configuration

```
┌─────────────────────────────────────────────────────────────────┐
│                    LED PWM Circuit                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TIM4 @ ~8 kHz (ARR = 500)                                     │
│  ┌──────────┐                                                   │
│  │   CH3    │──► PB8 ──┤>├── LED1 ──┬── GND                    │
│  │   CH4    │──► PB9 ──────────────┘    │                      │
│  └──────────┘              LED2 ◄──┤<├──┘                       │
│                                                                 │
│  Note: LEDs are anti-parallel (opposite polarity)              │
│        CH3 drives LED1 (inverted logic)                        │
│        CH4 drives LED2 (normal logic)                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Timer Configuration

From `led.c`:

```c
void LED_Init(void) {
    // Enable clocks
    __HAL_RCC_TIM4_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();

    // Configure PB8, PB9 as timer outputs
    GPIO_InitTypeDef GPIO_InitStruct = {
        .Pin = GPIO_PIN_8 | GPIO_PIN_9,
        .Mode = GPIO_MODE_AF_PP,
        .Pull = GPIO_NOPULL,
        .Speed = GPIO_SPEED_FREQ_LOW,
        .Alternate = GPIO_AF2_TIM4
    };
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

    // Configure TIM4 for PWM
    TIM4->PSC = (2 * HAL_RCC_GetPCLK1Freq() / 8000000) - 1;  // ~8 kHz
    TIM4->ARR = 500 - 1;  // 500 counts per period
    TIM4->CCMR2 = TIM_CCMR2_OC3M_1 | TIM_CCMR2_OC3M_2  // PWM mode 1
                | TIM_CCMR2_OC4M_1 | TIM_CCMR2_OC4M_2;
    TIM4->CCER = TIM_CCER_CC3E | TIM_CCER_CC4E;  // Enable outputs
    TIM4->CCR3 = 500;  // LED1 off (inverted)
    TIM4->CCR4 = 0;    // LED2 off
    TIM4->DIER = TIM_DIER_UIE;  // Enable update interrupt
    TIM4->CR1 = TIM_CR1_CEN;    // Start timer

    NVIC_SetPriority(TIM4_IRQn, AIOC_IRQ_PRIO_LED);
    NVIC_EnableIRQ(TIM4_IRQn);
}
```

### Brightness Levels

```c
#define LED_OFF_LEVEL   0     // Completely off
#define LED_IDLE_LEVEL  25    // Dim glow when not active
#define LED_FULL_LEVEL  249   // Maximum brightness
```

### LED Modes

```c
typedef enum {
    LED_MODE_SOLID = 0,       // Constant brightness
    LED_MODE_SLOWPULSE_4,     // 4 slow pulses then solid
    LED_MODE_SLOWPULSE_3,     // 3 slow pulses
    LED_MODE_SLOWPULSE_2,     // 2 slow pulses
    LED_MODE_SLOWPULSE_1,     // 1 slow pulse
    LED_MODE_FASTPULSE_4,     // 4 fast pulses then solid
    LED_MODE_FASTPULSE_3,     // 3 fast pulses
    LED_MODE_FASTPULSE_2,     // 2 fast pulses
    LED_MODE_FASTPULSE_1,     // 1 fast pulse
} LedMode_t;
```

### LED Animation ISR

```c
void TIM4_IRQHandler(void) {
    TIM4->SR = 0;  // Clear interrupt

    LedCounter++;  // 16-bit counter, wraps

    for (int i = 0; i < 2; i++) {
        uint16_t brightness;

        if (LedState[i]) {
            brightness = LED_FULL_LEVEL;
        } else if (LedMode[i] == LED_MODE_SOLID) {
            brightness = LED_IDLE_LEVEL;
        } else {
            // Pulse animation based on counter bits
            bool slowPulse = (LedCounter & 0x2000);  // ~1 Hz
            bool fastPulse = (LedCounter & 0x400);   // ~8 Hz

            if (/* pulse triggered */) {
                brightness = LED_FULL_LEVEL;
                LedMode[i]--;  // Decrement pulse count
            } else {
                brightness = LED_IDLE_LEVEL;
            }
        }

        // Set PWM (LED1 inverted, LED2 normal)
        if (i == 0) {
            TIM4->CCR3 = 500 - brightness;
        } else {
            TIM4->CCR4 = brightness;
        }
    }
}
```

### LED Control Macros

```c
// Set LED state (atomic)
#define LED_SET(led, on) do { \
    __disable_irq(); \
    LedState[led] = (on); \
    __enable_irq(); \
} while(0)

// Set LED mode (atomic)
#define LED_MODE(led, mode) do { \
    __disable_irq(); \
    LedMode[led] = (mode); \
    __enable_irq(); \
} while(0)
```

---

## External Input Handling

The AIOC has two external input pins for COS (Carrier Operated Switch) or buttons.

### EXTI Configuration

External inputs use edge-triggered interrupts:

```
PB6 (Input 1) ──► EXTI6 ──┐
                          ├──► EXTI9_5_IRQHandler
PB7 (Input 2) ──► EXTI7 ──┘
```

### Interrupt Handler

From `io.c`:

```c
void EXTI9_5_IRQHandler(void) {
    // Handle Input 1 (PB6)
    if (EXTI->PR & EXTI_PR_PR6) {
        EXTI->PR = EXTI_PR_PR6;  // Clear pending

        // Read inverted state (pull-up, active low)
        bool state = !(GPIOB->IDR & GPIO_PIN_6);

        // Route to configured destinations
        ProcessInputChange(0, state);
    }

    // Handle Input 2 (PB7)
    if (EXTI->PR & EXTI_PR_PR7) {
        EXTI->PR = EXTI_PR_PR7;

        bool state = !(GPIOB->IDR & GPIO_PIN_7);
        ProcessInputChange(1, state);
    }
}
```

### Input Routing

Inputs can be routed to multiple destinations:

```c
void ProcessInputChange(uint8_t input, bool state) {
    uint32_t iomux;

    // Route to HID buttons
    iomux = Settings_RegRead(SETTINGS_REG_CM108_IOMUX0 + input);
    if (iomux & CM108_INPUT_BTN1) {
        USB_HID_SetButton(0, state);
    }
    // ... more button mappings

    // Route to serial line states
    iomux = Settings_RegRead(SETTINGS_REG_SERIAL_IOMUX0 + input);
    if (iomux & SERIAL_INPUT_DCD) {
        USB_Serial_SetLineState(DCD, state);
    }
    // ... more line state mappings
}
```

### Input Use Cases

| Use Case | Configuration |
|----------|---------------|
| COS from radio | Route to HID button or serial DCD |
| External PTT button | Route to PTT via IOMUX |
| Squelch indicator | Route to serial RI |

---

## PTT Routing via IOMUX

The IOMUX system allows flexible routing of PTT sources to outputs.

### IOMUX Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    PTT IOMUX System                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Sources (bits in IOMUX register):                              │
│  ┌─────────────────┐                                            │
│  │ Serial DTR      │──┐                                         │
│  │ Serial RTS      │──┤                                         │
│  │ DTR && !RTS     │──┤      ┌─────────┐                       │
│  │ !DTR && RTS     │──┼─────►│  IOMUX0 │────► PA1 (PTT1)       │
│  │ HID GPIO 0      │──┤      │  (OR)   │                       │
│  │ HID GPIO 1      │──┤      └─────────┘                       │
│  │ HID GPIO 2      │──┤                                         │
│  │ HID GPIO 3      │──┤      ┌─────────┐                       │
│  │ VPTT            │──┼─────►│  IOMUX1 │────► PA0 (PTT2)       │
│  │ VCOS            │──┘      │  (OR)   │                       │
│  └─────────────────┘         └─────────┘                       │
│                                                                 │
│  Any source with its bit set in IOMUXn will assert PTTn        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### IOMUX Register Format

```
SETTINGS_REG_AIOC_IOMUX0 (PTT1) / IOMUX1 (PTT2):

Bit   Source                  Description
───   ──────                  ───────────
0     SERIAL_DTR              Serial DTR signal
1     SERIAL_RTS              Serial RTS signal
2     SERIAL_DTR_AND_NOT_RTS  DTR=1 AND RTS=0
3     SERIAL_NOT_DTR_AND_RTS  DTR=0 AND RTS=1
4     HID_GPIO_0              CM108 GPIO bit 0
5     HID_GPIO_1              CM108 GPIO bit 1
6     HID_GPIO_2              CM108 GPIO bit 2
7     HID_GPIO_3              CM108 GPIO bit 3
8     VPTT                    Virtual PTT (audio detect)
9     VCOS                    Virtual COS (carrier detect)
10    INPUT_1                 External input 1
11    INPUT_2                 External input 2
```

### Example Configurations

```c
// Default: PTT1 controlled by DTR && !RTS
Settings_RegWrite(SETTINGS_REG_AIOC_IOMUX0, (1 << 2));

// PTT1 from HID GPIO, PTT2 from VPTT
Settings_RegWrite(SETTINGS_REG_AIOC_IOMUX0, (1 << 4));  // HID GPIO 0
Settings_RegWrite(SETTINGS_REG_AIOC_IOMUX1, (1 << 8));  // VPTT

// Both PTTs from serial DTR
Settings_RegWrite(SETTINGS_REG_AIOC_IOMUX0, (1 << 0));
Settings_RegWrite(SETTINGS_REG_AIOC_IOMUX1, (1 << 0));
```

---

## Fox Hunt Morse Beacon

The fox hunt feature transmits a morse code beacon at configurable intervals.

### Configuration Registers

| Register | Purpose | Example |
|----------|---------|---------|
| FOXHUNT_INTERVAL | Seconds between transmissions (0=off) | 60 |
| FOXHUNT_WPM | Words per minute | 15 |
| FOXHUNT_VOLUME | Audio volume (0-0xFFFF) | 0x8000 |
| FOXHUNT_MSG0-3 | 16-character message | "DE K1ABC" |

### Morse Code Timing

Standard morse timing (PARIS = 50 units):

```
Element         Duration (units)
────────        ────────────────
Dot (dit)       1
Dash (dah)      3
Intra-character 1 (between dots/dashes)
Inter-character 3 (between letters)
Word space      7 (between words)
```

At 15 WPM: 1 unit = 80ms

### Morse Generation Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    Fox Hunt Pipeline                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. FoxHunt_Init():                                             │
│     ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│     │ Read message │───►│ Generate     │───►│ Store timing │   │
│     │ from settings│    │ morse timings│    │ array        │   │
│     └──────────────┘    └──────────────┘    └──────────────┘   │
│                                                                 │
│  2. FoxHunt_Tick() (called every second):                       │
│     ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│     │ Decrement    │───►│ Interval     │───►│ Start        │   │
│     │ interval     │    │ expired?     │Yes │ transmission │   │
│     │ counter      │    └──────────────┘    └──────────────┘   │
│     └──────────────┘                                            │
│                                                                 │
│  3. TIM15_IRQHandler() (48000 Hz):                              │
│     ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│     │ Keying?      │Yes │ Output sine  │───►│ Write to     │   │
│     │              │───►│ sample       │    │ DAC          │   │
│     └──────────────┘    └──────────────┘    └──────────────┘   │
│            │ No                                                 │
│            ▼                                                    │
│     ┌──────────────┐                                            │
│     │ Output       │                                            │
│     │ silence      │                                            │
│     └──────────────┘                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Sine Wave Generation

The beacon generates a 750 Hz tone:

```c
// 64-sample lookup table for one cycle at 48 kHz
// Frequency = 48000 / 64 = 750 Hz
static const int16_t carrierLUT[64] = {
    0, 3211, 6392, 9512, 12539, 15446, 18204, 20787,
    // ... 56 more samples ...
};

void TIM15_IRQHandler(void) {
    TIM15->SR = 0;  // Clear interrupt

    if (isIdentifying) {
        int16_t sample;

        if (isKeying) {
            sample = carrierLUT[sampleIndex++ & 63];
            sample = (sample * volume) >> 16;  // Apply volume
        } else {
            sample = 0;  // Silence
        }

        // Convert to unsigned for DAC
        DAC1->DHR12L1 = (sample + 32768) & 0xFFFF;

        // Advance timing
        if (--remainingCycles == 0) {
            isKeying = !isKeying;
            remainingCycles = timings[timingIndex++];

            if (timingIndex >= numTimings) {
                // Transmission complete
                isIdentifying = false;
                IO_PTTDeassert(PTT_MASK_1);
            }
        }
    }
}
```

### Timing Array Format

The `Morse_GenerateTimings()` function creates an array of sample counts:

```
Example: "CQ" at 15 WPM

Character   Morse    Elements
─────────   ─────    ────────
C           −.−.     KEY(3), GAP(1), KEY(1), GAP(1), KEY(3), GAP(1), KEY(1)
(space)              LETTER_GAP(3)
Q           −−.−     KEY(3), GAP(1), KEY(3), GAP(1), KEY(1), GAP(1), KEY(3)

Converted to sample counts at 48 kHz, 15 WPM:
  1 unit = 80ms = 3840 samples
  KEY(3) = 11520 samples
  KEY(1) = 3840 samples
  GAP(1) = 3840 samples
  etc.
```

### Integration with Audio System

Fox hunt reconfigures the DAC after USB audio init:

```c
void FoxHunt_Init(void) {
    // Only if fox hunt enabled
    if (Settings_RegRead(SETTINGS_REG_FOXHUNT_INTERVAL) == 0) {
        return;
    }

    // Reconfigure DAC for TIM15 trigger (instead of TIM6)
    DAC1->CR = DAC_CR_TSEL1_0 | DAC_CR_TSEL1_1 | DAC_CR_TSEL1_2  // TIM15
             | DAC_CR_TEN1
             | DAC_CR_EN1;

    // Configure TIM15 for 48 kHz
    // ... timer setup ...
}
```

**Note**: Fox hunt and USB audio use the same DAC, so they're mutually exclusive during beacon transmission.

---

*[Back to main guide](../new_dev_intro.md)*
