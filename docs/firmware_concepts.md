# Firmware Concepts Deep Dive

This document provides in-depth explanations of embedded systems concepts used in the AIOC firmware.

## Table of Contents

1. [Microcontroller Architecture](#microcontroller-architecture)
2. [Bare-Metal Programming](#bare-metal-programming)
3. [Interrupt Handling](#interrupt-handling)
4. [Memory Regions](#memory-regions)
5. [Clock Configuration](#clock-configuration)
6. [HAL vs Direct Register Access](#hal-vs-direct-register-access)

---

## Microcontroller Architecture

### The STM32F302

The AIOC uses an **STM32F302CBTx** microcontroller from STMicroelectronics:

```
┌─────────────────────────────────────────────────────────────────┐
│                      STM32F302CBTx                              │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐    ┌─────────────────────────────────────┐│
│  │  ARM Cortex-M4  │    │          Peripherals                ││
│  │  @ 72 MHz       │    │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐   ││
│  │                 │    │  │GPIO │ │ ADC │ │ DAC │ │Timer│   ││
│  │  - 32-bit       │◄──►│  └─────┘ └─────┘ └─────┘ └─────┘   ││
│  │  - FPU          │    │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐   ││
│  │  - DSP          │    │  │UART │ │ USB │ │ DMA │ │OPAMP│   ││
│  └─────────────────┘    │  └─────┘ └─────┘ └─────┘ └─────┘   ││
│           │             └─────────────────────────────────────┘│
│           ▼                                                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                    Bus Matrix                            │  │
│  └───────────┬─────────────────────────────┬───────────────┘  │
│              │                             │                   │
│  ┌───────────▼───────────┐     ┌───────────▼───────────┐      │
│  │   FLASH (128 KB)      │     │    SRAM (16 KB)       │      │
│  │   0x0800_0000         │     │    0x2000_0000        │      │
│  └───────────────────────┘     └───────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### Key Specifications

| Feature | Value | Notes |
|---------|-------|-------|
| Core | ARM Cortex-M4F | Has floating-point unit (FPU) |
| Clock | Up to 72 MHz | AIOC runs at max speed |
| Flash | 128 KB | Stores code and settings |
| SRAM | 16 KB (configured) | Runtime variables and stack |
| USB | Full-speed (12 Mbps) | Native USB peripheral |
| ADC | 12-bit, 5 Msps | Used for audio input |
| DAC | 12-bit | Used for audio output |
| Timers | Multiple | TIM2, TIM3, TIM4, TIM6, TIM15, TIM16, TIM17 |

### Memory-Mapped Peripherals

In a microcontroller, peripherals are controlled by reading and writing to specific memory addresses:

```c
// Example: Setting GPIO pin PA1 high
// GPIOA base address: 0x48000000
// BSRR (Bit Set/Reset Register) offset: 0x18
// Bit 1 sets PA1

*((volatile uint32_t*)(0x48000000 + 0x18)) = (1 << 1);

// Or using ST's definitions:
GPIOA->BSRR = GPIO_PIN_1;
```

This is why you'll see code like `GPIOA->ODR` or `TIM6->ARR` - these are pointers to hardware registers.

---

## Bare-Metal Programming

### What "Bare-Metal" Means

AIOC firmware runs without an operating system. This has important implications:

```
Desktop Application                 Bare-Metal Firmware
───────────────────                 ───────────────────
┌─────────────────┐                 ┌─────────────────┐
│  Application    │                 │   Your Code     │
├─────────────────┤                 ├─────────────────┤
│     OS          │                 │   HAL Library   │
├─────────────────┤                 ├─────────────────┤
│   Drivers       │                 │    Hardware     │
├─────────────────┤                 └─────────────────┘
│   Hardware      │
└─────────────────┘                 No OS layer!
```

### Implications

1. **No dynamic memory allocation** - Avoid `malloc()` and `free()`. All variables are either:
   - Global/static (allocated at compile time)
   - Local (on the stack)

2. **No threads** - Only one execution context, plus interrupt handlers

3. **No protection** - A bug can corrupt any memory or crash the system

4. **You control everything** - Complete control over timing and hardware

### The Super Loop Pattern

The main structure of bare-metal firmware:

```c
int main(void) {
    // Phase 1: Initialization (runs once)
    Init_Clocks();
    Init_GPIO();
    Init_Peripherals();

    // Phase 2: Main loop (runs forever)
    while (1) {
        // Poll for work
        if (data_available) {
            Process_Data();
        }

        // Perform periodic tasks
        if (timer_expired) {
            Do_Periodic_Task();
        }

        // Feed watchdog to prevent reset
        IWDG_Refresh();
    }
}
```

In AIOC, the main loop calls `USB_Task()` and `FoxHunt_Tick()` repeatedly.

---

## Interrupt Handling

### What is an Interrupt?

An interrupt temporarily pauses your main code to handle an urgent event:

```
Main Loop Running
        │
        │  ← Interrupt signal (e.g., timer overflow)
        │
        ├──────────────────────────────────┐
        │                                  │
        │                           ┌──────▼──────┐
        │                           │  Save state │
        │                           │  (registers)│
        │                           └──────┬──────┘
        │                                  │
        │                           ┌──────▼──────┐
        │                           │   Run ISR   │
        │                           │  (handler)  │
        │                           └──────┬──────┘
        │                                  │
        │                           ┌──────▼──────┐
        │                           │Restore state│
        │                           └──────┬──────┘
        │                                  │
        │◄─────────────────────────────────┘
        │
        ▼
Main Loop Continues
```

### NVIC (Nested Vectored Interrupt Controller)

The Cortex-M4 has a sophisticated interrupt controller:

```c
// Enable an interrupt
NVIC_EnableIRQ(TIM6_DAC_IRQn);

// Set priority (0-15, lower = higher priority)
NVIC_SetPriority(TIM6_DAC_IRQn, 2);

// Disable an interrupt
NVIC_DisableIRQ(TIM6_DAC_IRQn);
```

### Interrupt Priority in AIOC

AIOC uses 8 priority levels (0-7). Critical rules:

1. **Lower number = higher priority**
2. **Higher priority can interrupt lower priority** (preemption)
3. **Same priority never interrupts each other**

```c
// From aioc.h
#define AIOC_IRQ_PRIO_AUDIO      2   // Must never be delayed
#define AIOC_IRQ_PRIO_SERIAL     3
#define AIOC_IRQ_PRIO_USB        4
#define AIOC_IRQ_PRIO_IO         5
#define AIOC_IRQ_PRIO_SYSTICK    6
#define AIOC_IRQ_PRIO_LED        7   // Can wait
```

### Interrupt Service Routine (ISR) Best Practices

1. **Keep ISRs short** - Do minimal work, set flags for main loop
2. **No blocking calls** - Never wait or delay in an ISR
3. **Volatile variables** - Variables shared with main code must be `volatile`
4. **Atomic access** - Use `__disable_irq()` / `__enable_irq()` for multi-word operations

```c
// Example: ISR sets flag, main loop processes
volatile bool dataReady = false;

void TIM6_DAC_IRQHandler(void) {
    // Quick: read sample and set flag
    audioSample = ADC1->DR;
    dataReady = true;

    // Clear interrupt flag
    TIM6->SR &= ~TIM_SR_UIF;
}

int main(void) {
    while (1) {
        if (dataReady) {
            dataReady = false;
            Process_Audio_Sample(audioSample);
        }
    }
}
```

### Interrupt Vector Table

The vector table maps interrupt numbers to handler functions. It's placed at address 0x08000000:

```c
// From startup_stm32f302xc.s (simplified)
.section .isr_vector
    .word _estack           // Initial stack pointer
    .word Reset_Handler     // Reset
    .word NMI_Handler       // Non-maskable interrupt
    .word HardFault_Handler // Hard fault
    // ... more handlers ...
    .word TIM6_DAC_IRQHandler  // TIM6 and DAC
    .word USB_LP_IRQHandler    // USB low priority
```

---

## Memory Regions

### FLASH Memory

FLASH stores your code permanently (survives power off):

```
┌────────────────────────────────────────────┐
│              FLASH (128 KB)                │
│              0x0800_0000                   │
├────────────────────────────────────────────┤
│  Vector Table (first 0x188 bytes)          │
│  - Stack pointer, Reset_Handler, etc.      │
├────────────────────────────────────────────┤
│  .text section                             │
│  - All compiled code                       │
│  - Functions, switch tables                │
├────────────────────────────────────────────┤
│  .rodata section                           │
│  - const variables                         │
│  - String literals                         │
├────────────────────────────────────────────┤
│  .eeprom section @ 0x0801_F000             │
│  - Settings storage (4 KB)                 │
│  - Placed at top of FLASH by linker        │
└────────────────────────────────────────────┘
```

**FLASH characteristics:**
- Read: Fast, any byte
- Write: Slow, must erase first (page = 2KB)
- Endurance: ~10,000 write cycles per page

### SRAM Memory

SRAM holds runtime data (lost on power off):

```
┌────────────────────────────────────────────┐
│               SRAM (16 KB)                 │
│              0x2000_0000                   │
├────────────────────────────────────────────┤
│  .data section                             │
│  - Initialized global/static variables    │
│  - Copied from FLASH at startup            │
├────────────────────────────────────────────┤
│  .bss section                              │
│  - Zero-initialized global/static vars     │
│  - Cleared to zero at startup              │
├────────────────────────────────────────────┤
│                                            │
│            (unused space)                  │
│                                            │
├────────────────────────────────────────────┤
│  Stack (grows downward) ↓                  │
│  - Local variables                         │
│  - Function call frames                    │
│  - Interrupt context saves                 │
└────────────────────────────────────────────┘
               0x2000_A000 (_estack)
```

### The Stack

The stack grows downward and holds:
- Local variables
- Function return addresses
- Saved registers during interrupts

```c
void foo(void) {
    int local = 5;      // On stack
    int array[10];      // On stack (40 bytes)
    bar();              // Return address pushed to stack
}
```

**Stack overflow** is a common embedded bug - if you use too much stack space, you corrupt other memory. AIOC reserves 256 bytes minimum.

### Linker Script

The linker script (`stm32f30_flash.ld`) tells the compiler where to place each section:

```ld
MEMORY {
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 128K
    RAM (xrw)   : ORIGIN = 0x20000000, LENGTH = 16K
}

SECTIONS {
    .isr_vector : { *(.isr_vector) } > FLASH
    .text       : { *(.text*) } > FLASH
    .rodata     : { *(.rodata*) } > FLASH
    .data       : { *(.data*) } > RAM AT > FLASH
    .bss        : { *(.bss*) } > RAM
}
```

---

## Clock Configuration

### Clock Tree

The STM32F302 has a complex clock system:

```
                    ┌─────────┐
  8 MHz Crystal ───►│   HSE   │───┐
                    └─────────┘   │
                                  │    ┌─────────┐
                                  ├───►│   PLL   │──► 72 MHz SYSCLK
                    ┌─────────┐   │    │   x9    │
  Internal 8 MHz ──►│   HSI   │───┘    └─────────┘
                    └─────────┘              │
                                             ▼
                              ┌──────────────┴──────────────┐
                              │                             │
                         ┌────▼────┐                   ┌────▼────┐
                         │  AHB    │                   │   USB   │
                         │ 72 MHz  │                   │ 48 MHz  │
                         └────┬────┘                   └─────────┘
                              │                        (÷1.5)
                    ┌─────────┴─────────┐
               ┌────▼────┐         ┌────▼────┐
               │  APB1   │         │  APB2   │
               │ 36 MHz  │         │ 72 MHz  │
               │ (÷2)    │         │ (÷1)    │
               └─────────┘         └─────────┘
```

### AIOC Clock Configuration

From `main.c` `SystemClock_Config()`:

```c
// 1. Enable HSE (external 8 MHz crystal)
RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
RCC_OscInitStruct.HSEState = RCC_HSE_ON;

// 2. Configure PLL: 8 MHz × 9 = 72 MHz
RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;

// 3. Set bus prescalers
RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;    // 72 MHz
RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;     // 36 MHz
RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;     // 72 MHz
```

### Why Clocks Matter

Different peripherals need different clock speeds:
- **USB**: Requires exactly 48 MHz (72 ÷ 1.5 = 48)
- **Timers**: Use APB clock for counting
- **Audio sample rates**: Derived from timer clocks

---

## HAL vs Direct Register Access

### Hardware Abstraction Layer (HAL)

ST provides a HAL library that wraps register access:

```c
// HAL style - portable, verbose
GPIO_InitTypeDef GPIO_InitStruct = {0};
GPIO_InitStruct.Pin = GPIO_PIN_1;
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
GPIO_InitStruct.Pull = GPIO_NOPULL;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET);
```

**Pros**: Portable, well-documented, handles edge cases
**Cons**: Overhead, larger code size, can hide what's happening

### Direct Register Access

You can also write directly to hardware registers:

```c
// Direct register style - fast, compact
RCC->AHBENR |= RCC_AHBENR_GPIOAEN;  // Enable GPIOA clock
GPIOA->MODER |= (1 << 2);            // PA1 as output
GPIOA->BSRR = GPIO_PIN_1;            // Set PA1 high
```

**Pros**: Faster, smaller code, full control
**Cons**: Less portable, requires reading datasheet

### AIOC's Approach

AIOC uses a mix:
- **HAL for complex peripherals** (FLASH, some clocks)
- **Direct register access for performance-critical code** (audio IRQs, timers)

Example from `usb_audio.c`:

```c
// Direct access for speed in audio ISR
void TIM6_DAC_IRQHandler(void) {
    TIM6->SR = 0;  // Clear interrupt flag (direct)
    uint16_t sample = USB_AudioReadSample();
    DAC1->DHR12L1 = sample;  // Write to DAC (direct)
}
```

### Reading the Reference Manual

To understand registers, consult the **Reference Manual** (RM0365):

1. Find the peripheral chapter (e.g., "General-purpose timers")
2. Look at the register map
3. Read each bit field description

Example: TIM6 registers
```
TIM6_CR1  @ offset 0x00  - Control register 1
  Bit 0: CEN (Counter Enable) - 1 = start counting
TIM6_SR   @ offset 0x10  - Status register
  Bit 0: UIF (Update Interrupt Flag) - 1 = overflow occurred
TIM6_ARR  @ offset 0x2C  - Auto-reload register
  Bits 15:0 - Counter reloads to 0 when it reaches this value
```

---

*[Back to main guide](../new_dev_intro.md)*
