# Firmware

Application firmware for the bidirectional buck-boost converter, targeting the **STM32H755ZI-Q** Nucleo-144.

The H755 is dual-core. Real-time work — PWM generation, the PI control loops, and SPI/DMA traffic to the external ADC — belongs on the **Cortex-M7**; networking and telemetry were intended for the **Cortex-M4**, keeping Ethernet stack activity off the control-loop timing. The code here is the M7 application.

## What's included

`Core/Src/main.c` is the application layer, extracted from the working build:

- TIM1 CH2/CH2N complementary PWM with hardware dead-time insertion
- AD7328 external ADC driver over SPI1
- UART telemetry on USART3
- Clock, GPIO and peripheral initialisation

It is **not** a complete buildable project. STM32CubeMX-generated board support — `main.h`, `stm32h7xx_hal_msp.c`, `stm32h7xx_it.c`, `HAL_TIM_MspPostInit`, startup assembly and linker scripts — is not committed. To build it, generate a CubeMX project for the STM32H755ZI-Q with the peripheral configuration below and drop this `main.c` in.

## Peripheral configuration

### Clock

HSI → PLL1 (M=4, N=60, P=2), SYSCLK = 480 MHz, VOS0, flash latency 4.
AHB /2, APB1–4 /2.

### TIM1 — gate drive

| Setting | Value | Why |
|---|---|---|
| Counter mode | Centre-aligned 1 | Symmetric edges, lower harmonic content than edge-aligned |
| Prescaler | 0 | Full timer clock |
| Period (ARR) | 2999 | 40 kHz |
| Channel | CH2 + CH2N | Complementary pair into the IR2101 |
| Dead time | 100 (register value) | See [`../docs/results.md`](../docs/results.md) |
| CCR2 at start-up | 1500 | 50 % duty |

Dead time is inserted by the timer's break/dead-time unit. The non-overlap guarantee is therefore a hardware property and does not depend on software timing or interrupt latency.

> The committed dead-time value is the one that was in the build. It is **not** a recommended value — both dead-time settings tried on this board ended in a failed low-side MOSFET, for opposite reasons. Read the failure analysis before powering a half-bridge with it.

### SPI1 — AD7328 link

| Setting | Value |
|---|---|
| Mode | Master, 2-line, 16-bit |
| CPOL / CPHA | 1 / 0 |
| Baud prescaler | /128 (~750 kHz) |
| Bit order | MSB first |
| NSS | Software, PA4 |

**Initialisation order matters.** The control register must be written *before* the two range registers. Written the other way round, the AD7328 silently keeps its default ±10 V bipolar span and every channel reads near mid-scale — a failure that looks like a wiring fault but isn't.

Both range registers are set to `0xFE0`, placing all eight channels on the **0–10 V unipolar** span. The sensor front end is scaled to that range so all channels share one conversion constant:

```c
volts = (10.0f / 4096.0f) * code;   /* 2.44 mV per LSB */
```

A `span_lost()` check on channel 0 detects the part reverting to its default range after a brown-out and triggers re-initialisation automatically, rather than streaming plausible-looking garbage.

### USART3 — telemetry

115200-8-N-1, routed through the Nucleo's on-board ST-LINK virtual COM port. No external USB-serial adapter required — the host sees a standard serial device.

Output format, one block per sweep:

```
--- Readings ---
CH0: code=   0  V= 0.000
CH1: code= 512  V= 1.250
...
----------------
```

### GPIO

| Pin | Function |
|---|---|
| PA4 | ADC chip select (idle high) |
| PB14 | Heartbeat LED, toggled every 10 sweeps |
| PE1 | Spare output |

## Known limitations

- **The PI control loops are not in this source.** Loop structure and tuning were validated in Simulink; the hardware firmware got as far as open-loop PWM plus telemetry before the power stage failed.
- **SPI reads are blocking.** `HAL_SPI_TransmitReceive` with `HAL_MAX_DELAY` and busy-wait inter-frame delays are fine for a 2 Hz telemetry sweep but unusable inside a 40 kHz control loop. DMA-driven acquisition is required before closing the loop.
- **No framing or CRC on the UART link**, so a corrupted frame is indistinguishable from a valid one downstream.
- **No protection logic** — no cycle-by-cycle current limit, soft-start, or over-temperature derating.

See [`../docs/results.md`](../docs/results.md) for the full analysis.
