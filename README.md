# ECE 528/L – Homework 3

## Section I: Review Questions

### 1. eUSCI_A registers

- **UCAxBRW (Baud Rate Control Word):** 16-bit clock prescaler that divides the input clock (BRCLK) to set the baud rate.
- **UCAxMCTLW (Modulation Control Word):** Fine-tunes the baud rate. It holds UCOS16 (bit 0, oversampling enable), UCBRF (bits 7-4, first modulation stage) and UCBRS (bits 15-8, second modulation stage).
- **UCAxCTLW0 (Control Word 0):** Main configuration register. It sets reset (UCSWRST), clock source (UCSSEL), UART/SPI mode (UCSYNC, UCMODE), stop bits (UCSPB), data length (UC7BIT), bit order (UCMSB), and parity (UCPAR, UCPEN).
- **UCAxRXBUF (Receive Buffer):** Holds the last received byte. Reading it clears the receive flag.
- **UCAxTXBUF (Transmit Buffer):** Write a byte here to transmit it. Writing it clears the transmit flag.

### 2. UCAxSTATW

UCAxSTATW is the status register. It reports what the UART is doing and whether a receive error happened.

- Transmitting: **UCBUSY**, bit 0 (`0x01`)
- Errors: **UCFE** bit 6 (`0x40`) framing, **UCPE** bit 4 (`0x10`) parity, **UCOE** bit 5 (`0x20`) overrun. **UCRXERR** bit 2 (`0x04`) is set on any receive error.
- Unread data in RX buffer: this is the **UCRXIFG** flag, bit 0 of UCAxIFG (`0x01`), not UCAxSTATW.

### 3. Baud rate for eUSCI_A3

N = 12,000,000 / 115,200 = 104.1667

Since N > 16, use oversampling (UCOS16 = 1):

- UCAxBRW = INT(N/16) = INT(6.5104) = 6
- UCBRF = INT(0.5104 x 16) = 8
- UCBRS = 0x20 (from the modulation table)

**UCA3BRW = 6**

### 4. UART2

#### (a)

```c
#include "msp.h"

void UART2_Init(void)
{
    // hold the module in reset
    EUSCI_A2->CTLW0 |= 0x0001;

    // MSB first, SMCLK, reset on
    EUSCI_A2->CTLW0 = 0x2000 | 0x0080 | 0x0001;

    // 12 MHz / 115200 with oversampling
    EUSCI_A2->BRW = 6;
    EUSCI_A2->MCTLW = (0x20 << 8) | (8 << 4) | 0x01;

    // P3.2 and P3.3 primary function
    P3->SEL0 |= 0x0C;
    P3->SEL1 &= ~0x0C;

    // release reset
    EUSCI_A2->CTLW0 &= ~0x0001;
}
```

#### (b)


### 5. eUSCI_B and SPI vs. UART

The eUSCI_B module provides SPI and I2C serial communication.

- **Synchronization:** UART is asynchronous. SPI is synchronous.
- **Clocking:** UART has no shared clock, so both sides use the same baud rate. In SPI the controller generates SCLK.
- **Signaling:** UART uses TX and RX. SPI uses SCLK, MOSI, MISO and CS.
- **Data framing:** UART uses start, stop and optional parity bits. SPI has no start/stop bits and CS marks the start and end of a transfer.

### 6. SPI signals

#### (a)

- **SCLK:** Clock from the controller that times each bit.
- **CS:** Active-low line from the controller that selects a peripheral.
- **MISO:** Data line from the peripheral to the controller.
- **MOSI:** Data line from the controller to the peripheral.

#### (b)

Bus contention. Multiple peripherals drive MISO at the same time, so the data read by the controller is corrupted, and all of them respond to the same MOSI data.

### 7. UCBxCTLW0

#### (a)

UCBxCTLW0 is the main control register of eUSCI_B. It sets the mode (SPI or I2C), reset, clock source, controller/peripheral, clock phase and polarity, bit order and data length.

#### (b)

- **UCSYNC** (bit 8): set to 1 for synchronous mode (SPI).
- **UCMODEx** (bits 10-9): selects the SPI format. 00 = 3-pin SPI, 01 = 4-pin SPI (STE active high), 10 = 4-pin SPI (STE active low), 11 = I2C.
- **UCCKPH** (bit 15): clock phase. 0 = data changed on first edge and captured on the next. 1 = data captured on first edge and changed on the next.
- **UCCKPL** (bit 14): clock polarity. 0 = clock idles low. 1 = clock idles high.

#### (c)

The **UCSSELx** field (bits 7-6):

- 00b: reserved
- 01b: ACLK
- 10b: SMCLK
- 11b: SMCLK

### 8. eUSCI_B0 SPI

#### (a)

```c
#include "msp.h"

void EUSCI_B0_SPI_Init(void)
{
    // hold the module in reset
    EUSCI_B0->CTLW0 |= 0x0001;

    // mode 0 (UCCKPH = 1), MSB first, controller, sync, SMCLK, reset on
    EUSCI_B0->CTLW0 = 0x8000 | 0x2000 | 0x0800 | 0x0100 | 0x0080 | 0x0001;

    // 12 MHz / 12 = 1 MHz
    EUSCI_B0->BRW = 12;

    // P1.5, P1.6, P1.7 primary function
    P1->SEL0 |= 0xE0;
    P1->SEL1 &= ~0xE0;

    // P9.0 chip select, GPIO output, idle high
    P9->SEL0 &= ~0x01;
    P9->SEL1 &= ~0x01;
    P9->DIR |= 0x01;
    P9->OUT |= 0x01;

    // release reset
    EUSCI_B0->CTLW0 &= ~0x0001;
}
```

#### (b)


