# Audio Subsystem

This document explains how audio flows through the AIOC, including ADC/DAC configuration, timer-driven sampling, and automatic PTT detection.

## Table of Contents

1. [Audio Signal Flow](#audio-signal-flow)
2. [ADC Configuration (Microphone Input)](#adc-configuration-microphone-input)
3. [DAC Configuration (Speaker Output)](#dac-configuration-speaker-output)
4. [Timer-Driven Sample Generation](#timer-driven-sample-generation)
5. [Sample Rate Handling](#sample-rate-handling)
6. [Volume and Gain Control](#volume-and-gain-control)
7. [Virtual PTT/COS Detection](#virtual-pttcos-detection)

---

## Audio Signal Flow

Audio passes through the AIOC in both directions:

```
                            AIOC Audio Path
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  MICROPHONE PATH (Radio → Computer)                                 │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────────────┐  │
│  │  Radio  │───►│  OPAMP  │───►│  ADC1   │───►│  USB Audio IN   │  │
│  │  Audio  │    │  (PGA)  │    │ 12-bit  │    │  Endpoint 0x81  │  │
│  │  Out    │    │ Gain:   │    │         │    │                 │  │
│  │         │    │ 1-16x   │    │         │    │                 │  │
│  └─────────┘    └─────────┘    └─────────┘    └─────────────────┘  │
│      │                              ▲                               │
│      │         TIM3 TRGO triggers   │                               │
│      │         at sample rate ──────┘                               │
│                                                                     │
│  SPEAKER PATH (Computer → Radio)                                    │
│  ┌─────────────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐  │
│  │  USB Audio OUT  │───►│  DAC1   │───►│ Output  │───►│  Radio  │  │
│  │  Endpoint 0x02  │    │ 12-bit  │    │ Atten   │    │  Audio  │  │
│  │                 │    │         │    │ (PA3)   │    │  In     │  │
│  └─────────────────┘    └─────────┘    └─────────┘    └─────────┘  │
│                              ▲                                      │
│         TIM6 TRGO triggers   │                                      │
│         at sample rate ──────┘                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Purpose | Location |
|-----------|---------|----------|
| OPAMP | Programmable gain amplifier for mic input | Internal to STM32 |
| ADC1 | Analog-to-digital conversion (12-bit) | Internal |
| DAC1 | Digital-to-analog conversion (12-bit) | Output on PA4 |
| TIM3 | Triggers ADC at sample rate | Timer peripheral |
| TIM6 | Triggers DAC at sample rate | Timer peripheral |
| PA3 | Output level control (mic/line select) | GPIO |
| PB2 | ADC analog input | GPIO |

---

## ADC Configuration (Microphone Input)

The ADC captures audio from the radio's speaker output.

### ADC Setup

```c
// From usb_audio.c
static void ADC_Init(void) {
    // Enable clocks
    __HAL_RCC_ADC1_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();

    // Configure PB2 as analog input
    GPIO_InitTypeDef GPIO_InitStruct = {
        .Pin = GPIO_PIN_2,
        .Mode = GPIO_MODE_ANALOG,
        .Pull = GPIO_NOPULL
    };
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

    // ADC configuration
    ADC1->CR = 0;
    ADC1->CR = (0x01 << ADC_CR_ADVREGEN_Pos);  // Enable voltage regulator
    // ... wait for regulator ...

    ADC1->CFGR = ADC_CFGR_EXTEN_0        // Rising edge trigger
               | ADC_CFGR_EXTSEL_TIM3    // TIM3_TRGO
               | ADC_CFGR_ALIGN          // Left-aligned (12-bit in upper bits)
               | ADC_CFGR_RES_0;         // 12-bit resolution

    ADC1->SMPR1 = (0x07 << ADC_SMPR1_SMP3_Pos);  // Max sample time
}
```

### ADC Interrupt Handler

```c
void ADC1_2_IRQHandler(void) {
    // Read sample (12-bit left-aligned in 16-bit word)
    uint16_t sample = ADC1->DR;

    // Apply volume scaling
    sample = ApplyVolume(sample, micVolume);

    // Check for virtual COS (carrier detect)
    if (sample > vcosThreshold) {
        TIM17->EGR = TIM_EGR_UG;  // Restart COS timeout
        vcosActive = true;
    }

    // Send to USB
    tud_audio_write(&sample, sizeof(sample));

    // Clear interrupt flag
    ADC1->ISR = ADC_ISR_EOS;
}
```

### OPAMP for Gain Control

The internal OPAMP provides programmable gain:

```
Gain Settings (from settings register):
  RX_GAIN = 0: 1x   (line level input)
  RX_GAIN = 1: 2x
  RX_GAIN = 2: 4x
  RX_GAIN = 3: 8x
  RX_GAIN = 4: 16x  (mic level input)
```

---

## DAC Configuration (Speaker Output)

The DAC sends audio to the radio's microphone input.

### DAC Setup

```c
static void DAC_Init(void) {
    __HAL_RCC_DAC1_CLK_ENABLE();

    // Configure PA4 as DAC output
    GPIO_InitTypeDef GPIO_InitStruct = {
        .Pin = GPIO_PIN_4,
        .Mode = GPIO_MODE_ANALOG,
        .Pull = GPIO_NOPULL
    };
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    // DAC configuration
    DAC1->CR = DAC_CR_TSEL1_0    // TIM6_TRGO trigger
             | DAC_CR_TEN1       // Trigger enable
             | DAC_CR_EN1;       // DAC enable
}
```

### DAC Interrupt Handler

```c
void TIM6_DAC_IRQHandler(void) {
    // Clear interrupt flag
    TIM6->SR = 0;

    // Read sample from USB
    uint16_t sample;
    if (tud_audio_read(&sample, sizeof(sample)) == 0) {
        sample = 0x8000;  // Silence (mid-scale)
    }

    // Apply volume scaling
    sample = ApplyVolume(sample, speakerVolume);

    // Check for virtual PTT (voice detect)
    if (AbsoluteValue(sample - 0x8000) > vpttThreshold) {
        TIM16->EGR = TIM_EGR_UG;  // Restart PTT timeout
        vpttActive = true;
    }

    // Write to DAC (left-aligned 12-bit)
    DAC1->DHR12L1 = sample;
}
```

### Output Level Control

PA3 GPIO controls output attenuation:

```
TX_BOOST setting:
  0: Mic level output (attenuated)
  1: Line level output (full amplitude)
```

---

## Timer-Driven Sample Generation

Audio timing is critical. Timers generate precise interrupts at the sample rate.

### Timer Configuration

```
┌─────────────────────────────────────────────────────────────────┐
│                    Audio Timing Chain                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  72 MHz System Clock                                            │
│        │                                                        │
│        ├────────────────────────┬───────────────────────┐       │
│        ▼                        ▼                       ▼       │
│  ┌──────────┐            ┌──────────┐            ┌──────────┐   │
│  │   TIM3   │            │   TIM6   │            │  TIM15   │   │
│  │          │            │          │            │          │   │
│  │ PSC=0    │            │ PSC=0    │            │ PSC=0    │   │
│  │ ARR=1499 │            │ ARR=1499 │            │ ARR=1499 │   │
│  │ (48kHz)  │            │ (48kHz)  │            │ (48kHz)  │   │
│  └────┬─────┘            └────┬─────┘            └────┬─────┘   │
│       │                       │                       │         │
│       ▼                       ▼                       ▼         │
│   TIM3_TRGO              TIM6_TRGO               TIM15_TRGO     │
│   (triggers ADC)         (triggers DAC)          (fox hunt DAC) │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Calculating Timer Values

For a desired sample rate:

```c
// 72 MHz / 48000 Hz = 1500 cycles per sample
uint32_t divider = 72000000 / sampleRate;
TIMx->PSC = 0;              // No prescaler
TIMx->ARR = divider - 1;    // Auto-reload value
```

### Sample Rate Table

| Sample Rate | Divider (ARR+1) | Actual Rate | Error |
|-------------|-----------------|-------------|-------|
| 48000 Hz | 1500 | 48000.00 | 0 ppm |
| 32000 Hz | 2250 | 32000.00 | 0 ppm |
| 24000 Hz | 3000 | 24000.00 | 0 ppm |
| 22050 Hz | 3265 | 22051.99 | ~90 ppm |
| 16000 Hz | 4500 | 16000.00 | 0 ppm |
| 12000 Hz | 6000 | 12000.00 | 0 ppm |
| 11025 Hz | 6531 | 11024.96 | ~4 ppm |
| 8000 Hz | 9000 | 8000.00 | 0 ppm |

---

## Sample Rate Handling

The host can request different sample rates via USB Audio Class controls.

### Sample Rate Change Flow

```
Host sends SET_CUR to Clock Source
         │
         ▼
tud_audio_set_req_entity_cb()
         │
         ▼
Check if clock entity (ID=0x08 or 0x18)
         │
         ▼
Extract 32-bit sample frequency
         │
         ▼
Call Timer_ADC_Init(freq) or Timer_DAC_Init(freq)
         │
         ▼
Update TIMx->ARR for new rate
```

### Supported Rates Descriptor

The device advertises supported rates in the clock source descriptor:

```c
// Sample frequency ranges in descriptor
const uint32_t sampleRates[] = {
    48000, 32000, 24000, 22050, 16000, 12000, 11025, 8000
};
```

---

## Volume and Gain Control

Audio levels are controlled through USB Audio Class feature units.

### Volume Format

USB Audio uses a logarithmic scale:

```
Volume register: 16-bit signed, 7.8 fixed-point dB
  0x0000 = 0 dB (unity gain)
  0x8000 = -128 dB (effectively mute)
  0x7FFF = +127.99 dB (max gain)

AIOC supported range:
  Minimum: -96 dB (0xA000)
  Maximum: 0 dB (0x0000)
  Resolution: 1 dB (0x0100)
```

### Volume Conversion

```c
// Convert dB to linear scaling factor (0.16 fixed-point)
uint16_t dBToLinear(int16_t dB_7_8) {
    // dB_7_8 is in 7.8 format (1 unit = 1/256 dB)
    // Linear = 10^(dB/20)
    // Using lookup table or approximation
}
```

### Mute Control

Each feature unit has an independent mute:

```c
bool speakerMute = false;
bool micMute = false;

// In ISR:
if (!speakerMute) {
    sample = ApplyVolume(sample, speakerVolume);
} else {
    sample = 0x8000;  // Silence
}
```

---

## Virtual PTT/COS Detection

AIOC can automatically control PTT based on audio levels.

### Virtual PTT (VPTT)

Activates PTT when audio is being sent to the radio:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Virtual PTT State Machine                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Audio from Host                                                │
│        │                                                        │
│        ▼                                                        │
│  ┌───────────────┐      Level > Threshold?                      │
│  │ Check Level   │──────────Yes──────────┐                      │
│  └───────────────┘                       │                      │
│        │ No                              ▼                      │
│        │                          ┌──────────────┐              │
│        │                          │ Assert PTT   │              │
│        │                          │ Restart TIM16│              │
│        │                          └──────────────┘              │
│        │                                 │                      │
│        ▼                                 │                      │
│  ┌───────────────┐                       │                      │
│  │ TIM16 Expired?│◄──────────────────────┘                      │
│  └───────────────┘                                              │
│        │ Yes                                                    │
│        ▼                                                        │
│  ┌───────────────┐                                              │
│  │ Deassert PTT  │                                              │
│  └───────────────┘                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Virtual COS (VCOS)

Carrier Operated Switch - detects when radio is receiving:

```c
// In ADC interrupt:
int16_t level = (int16_t)sample - 0x8000;  // Convert to signed
if (abs(level) > vcosThreshold) {
    // Audio detected from radio
    TIM17->EGR = TIM_EGR_UG;  // Restart timeout
    if (!vcosActive) {
        vcosActive = true;
        // Optionally trigger actions (e.g., HID button)
    }
}

// TIM17 timeout interrupt:
void TIM17_IRQHandler(void) {
    vcosActive = false;
    // Optionally deassert actions
}
```

### Configuration Registers

| Register | Purpose | Format |
|----------|---------|--------|
| `VPTT_THRESHOLD` | Audio level to trigger PTT | 16-bit unsigned |
| `VPTT_TAIL_TIME` | Hold time after audio stops | 12.4 fixed-point ms |
| `VCOS_THRESHOLD` | Audio level to detect carrier | 16-bit unsigned |
| `VCOS_TAIL_TIME` | Hold time after carrier lost | 12.4 fixed-point ms |

### PTT Routing

VPTT/VCOS can be routed to physical PTT outputs via IOMUX:

```
SETTINGS_REG_AIOC_IOMUX0 (PTT1 sources):
  Bit 0: Serial DTR
  Bit 1: Serial RTS
  Bit 2: HID GPIO 0
  Bit 3: HID GPIO 1
  Bit 4: VPTT
  Bit 5: VCOS

SETTINGS_REG_AIOC_IOMUX1 (PTT2 sources):
  Same bit mapping
```

---

## Audio Processing Pipeline

Complete audio path with all processing stages:

```
MICROPHONE (ADC → USB):
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│  Radio  │──►│  OPAMP  │──►│   ADC   │──►│ Volume  │──►│   USB   │
│  Audio  │   │  Gain   │   │ Sample  │   │ Scale   │   │  FIFO   │
│  Out    │   │ (1-16x) │   │         │   │         │   │         │
└─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
                                 ▲              │
                          TIM3 TRGO            │
                                               ▼
                                         ┌─────────┐
                                         │  VCOS   │
                                         │ Detect  │
                                         └─────────┘

SPEAKER (USB → DAC):
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│   USB   │──►│ Volume  │──►│   DAC   │──►│ Output  │──►│  Radio  │
│  FIFO   │   │ Scale   │   │ Sample  │   │ Atten   │   │  Audio  │
│         │   │         │   │         │   │ (PA3)   │   │  In     │
└─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
                  │              ▲
                  │         TIM6 TRGO
                  ▼
            ┌─────────┐
            │  VPTT   │
            │ Detect  │
            └─────────┘
```

---

*[Back to main guide](../new_dev_intro.md)*
