---
title: "STM32 Bare Metal USB Implementation"
date: 2023-07-20T23:35:19-06:00
tags: ["Electronics", "ARM", "STM32", "USB"]
---
Implementing your own bare metal USB stack may provide certain benefits like little to no 
dependencies, smaller code size, better understanding of internals and suffering. 

This article documents the process I went through to manually setup and get the USB
core of an STM32F401CCU6 to do what I wanted. My goal was to successfully return a device descriptor.
Once I got that working I had a greater appreciation for TinyUSB and went with it, however this was
still a ~~painful~~ great learning experience.

It is expected for the reader to already be familiar with USB as a protocol,
to get up to speed there are great resources like 
[USB Made Simple](https://www.usbmadesimple.co.uk/).

# Implementation
Remember this is specific to a STM32F401CCU6 MCU, however the process is likely to be similar
for most other chips.

## Clock setup
First step is to produce a 48 MHz clock. The board I had uses a 25 MHz crystal which will be used
as a source. The system clock will also be derived from the PLL and set to 72 MHz. This last step
is optional. Be warned that **setting the system clock too low will prevent the USB core from correct 
functioning**. The exact frequency can be found on your MCU manual, in this case the AHB frequency
should be higher than 14.2 MHz. Here is the code used for setup:

```c
// Update Flash latency as suggested for a 72 MHz system clock.
FLASH->ACR |= FLASH_ACR_LATENCY_2WS;
while ((FLASH->ACR & FLASH_ACR_LATENCY) != FLASH_ACR_LATENCY_2WS) continue;

// Output the clock to MCO1 for verification
// Set prescaler for APB1 as to not exceed the max frequency recommended (42 MHZ)
RCC->CFGR |= (0x06 << RCC_CFGR_MCO1PRE_Pos) | (0x03 << RCC_CFGR_MCO1_Pos) | 
    (0x04 << RCC_CFGR_PPRE1_Pos);

// Turn on HSE (crystal oscillator) which will be used as the PLL source
RCC->CR |= RCC_CR_HSEON;
while ((RCC->CR & RCC_CR_HSERDY) == 0) continue;

// SYSCLK = (25 / 25) * 288 / 4
// USBCLK = (25 / 25) * 288 / 6
RCC->PLLCFGR = (6 << RCC_PLLCFGR_PLLQ_Pos) | (288 << RCC_PLLCFGR_PLLN_Pos) |
    (25 << RCC_PLLCFGR_PLLM_Pos) | (0x01 << RCC_PLLCFGR_PLLP_Pos) |
    RCC_PLLCFGR_PLLSRC_HSE;

RCC->CR |= RCC_CR_PLLON;
while ((RCC->CR & RCC_CR_PLLRDY) == 0) continue;

// Wait for clock switch to PLL
RCC->CFGR |= RCC_CFGR_SW_PLL;
while ((RCC->CFGR & RCC_CFGR_SWS) == 0) continue;
```

You must ensure your clocks are correctly setup. Failing to do so with result in a bunch of wasted
hours finding the source of errors which may simply caused by a wrong clock setup.

## Peripheral setup
The next step is to setup the GPIOs that will function as D+ and D-. They must be set
to their alternate function for USB. VBUS sensing was also disabled and USB clock needs to be
enabled as well.

```c
RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
GPIOA->MODER |= (0x02 << GPIO_MODER_MODER11_Pos) |
    (0x02 << GPIO_MODER_MODER12_Pos);
GPIOA->OSPEEDR |= (0x02 << GPIO_OSPEEDR_OSPEED11_Pos) |
    (0x02 << GPIO_OSPEEDR_OSPEED12_Pos);
GPIOA->AFR[1] |= (0x0A << GPIO_AFRH_AFSEL11_Pos) | (0x0A << GPIO_AFRH_AFSEL12_Pos);

USB_OTG_FS->GCCFG |= USB_OTG_GCCFG_NOVBUSSENS;
USB_OTG_FS->GCCFG &= ~USB_OTG_GCCFG_VBUSBSEN;
USB_OTG_FS->GCCFG &= ~USB_OTG_GCCFG_VBUSASEN;

RCC->AHB2ENR |= RCC_AHB2ENR_OTGFSEN;
while ((OTG->GRSTCTL & USB_OTG_GRSTCTL_AHBIDL) == 0) continue;
```

## USB Core
This is really where the pain starts. The USB core may at times be finicky so that even the
slightest misconfiguration results in nothing working at all. The recommended setup process is
documented in the MCU's manual. In my case it consists of:

1. Disabling all USB OTG stuff like Session Request Protocol.
2. Force device mode
3. Setup device speed to FS (not that there is any other option), set device address to 0.
4. Enable clock (yes, again, but this time somewhere else)
5. Set USB on soft disconnect until everything is ready
6. Enabling the following USB interrupts:
    * USBRST: On USB reset
    * ENUMDNEM: Once reset is over
    * RXFLVLM: When the RX FIFO has data
    * OEPINT: Certain events for OUT endpoints
    * IEPINT: Certain events for IN endpoints

```c
static volatile uint32_t* const OTG_CLK = (void*)(USB_OTG_FS_PERIPH_BASE + USB_OTG_PCGCCTL_BASE);
static void enable_otg_clk() { *OTG_CLK = 0; }
static void enable_soft_disconnect() { OTGD->DCTL |= USB_OTG_DCTL_SDIS; }

OTG->GUSBCFG &= ~(USB_OTG_GUSBCFG_SRPCAP | USB_OTG_GUSBCFG_TRDT);
OTG->GUSBCFG |= (USB_OTG_GUSBCFG_FDMOD | (0x06 << USB_OTG_GUSBCFG_TRDT_Pos));

enable_otg_clk();
enable_soft_disconnect();

OTGD->DCFG &= ~USB_OTG_DCFG_DAD;
OTGD->DCFG |= (0x03 << USB_OTG_DCFG_DSPD_Pos) | USB_OTG_DCFG_NZLSOHSK;

OTG->GINTMSK = USB_INTERRUPTS;
OTG->GINTSTS = 0xFFFFFFFF;
OTG->GAHBCFG |= USB_OTG_GAHBCFG_GINT;
```

## FIFOs
Read over and over the FIFO map available on your MCU's manual. In the case of the STM32F401CCU6
and probably most devices, a single RX FIFO is shared by all endpoints. Its size depends on
the max packet size chosen (64 for this article). Here is how FIFOs for EP0 were initialized:

```c
// Sizes are in 4 byte words, so the RX FIFO is of 384 bytes.
OTG->GRXFSIZ = 0x30;
// Start after the RX FIFO and have max packet size (16 * 4 = 64) as its size.
OTG->DIEPTXF0_HNPTXFSIZ = (16 << USB_OTG_TX0FD_Pos) | 0x30;
```

To obtain the memory address for the FIFOs used in this example, the following function
was used. You should really read your FIFO map as this may not apply for other MCUs.

```c
uint32_t* usbEpFifo(uint8_t ep) {
    return (uint32_t*)(USB_OTG_FS_PERIPH_BASE + USB_OTG_FIFO_BASE + (ep << 12));
}
```

When dealing with TX FIFO for EP0 you can add the offset specified above or not. Apparently
the USB core can read minds because it seems to make no difference, as of this day I don't know why.

**Remember FIFOs are always handled in 4 byte words.**

### Reading
To read you need to pop the RX FIFO status register and proceed to read word by word from the FIFO.
This was the sample Implementation used:

```c
struct UsbControlRequest {
    uint8_t bmRequestType;
    uint8_t bRequest;
    uint8_t wValueL;
    uint8_t wValueH;
    uint16_t wIndex;
    uint16_t wLength;
};

void usbReadSetup(const volatile uint32_t* fifo, struct UsbControlRequest* request) {
    uint32_t* buffer = (uint32_t*)request;

    for (uint16_t idx = 0; idx < 2; idx++, buffer++) {
        *buffer = *fifo;
#ifdef BUILD_MACHINE
        fifo++;
#endif
    }
}
```

**You don't need to increment your pointer**. The hardware FIFO does it. The macro is used to enable
testing on host machines where there is no hardware FIFO and the address does need to be incremented
on every read.

### Writing
Just as reading, the same tips apply to writing to a FIFO. It is done in 4 byte words and no address
needs to be incremented, the following implementation may be improved but servers as a start:

```c
void usbRawWrite(volatile uint32_t* fifo, void* data, uint8_t len) {
    uint32_t fifoWord;
    uint32_t* buffer = (uint32_t*)data;
    uint8_t remains = len;
    for (uint8_t idx = 0; idx < len; idx += 4, remains -= 4, buffer++) {
        switch (remains) {
            case 0:
                break;
            case 1:
                fifoWord = *buffer & 0xFF;
                *fifo = fifoWord;
                break;
            case 2:
                fifoWord = *buffer & 0xFFFF;
                *fifo = fifoWord;
                break;
            case 3:
                fifoWord = *buffer & 0xFFFFFF;
                *fifo = fifoWord;
                break;
            default:
                *fifo = *buffer;
                break;
        }
#ifdef BUILD_MACHINE
        fifo++;
#endif
    }
}
```

## Endpoints
The setup for EP0 endpoints mostly consists of setting some parameters like their sizes and NAK
acknowledge status, any interrupts wished to be handled must also be set.

```c
USB_OTG_INEndpointTypeDef* usbEpin(uint8_t ep) {
    return (void*)(USB_OTG_FS_PERIPH_BASE + USB_OTG_IN_ENDPOINT_BASE + (ep << 5));
}

USB_OTG_OUTEndpointTypeDef* usbEpout(uint8_t ep) {
    return (void*)(USB_OTG_FS_PERIPH_BASE + USB_OTG_OUT_ENDPOINT_BASE + (ep << 5));
}

OTGD->DAINTMSK = ((1 << 16) | 1);
OTGD->DOEPMSK = USB_OTG_DOEPMSK_STUPM | USB_OTG_DOEPMSK_XFRCM | USB_OTG_DOEPMSK_OTEPDM |
                USB_OTG_DOEPMSK_OTEPSPRM;
OTGD->DIEPMSK = USB_OTG_DIEPMSK_XFRCM;

USB_OTG_INEndpointTypeDef* in0 = usbEpin(0);
in0->DIEPCTL &= (0x03 << USB_OTG_DIEPCTL_MPSIZ_Pos);

USB_OTG_OUTEndpointTypeDef* out0 = usbEpout(0);
out0->DOEPTSIZ = (1 << USB_OTG_DOEPTSIZ_STUPCNT_Pos) | (1 << USB_OTG_DOEPTSIZ_PKTCNT_Pos);
out0->DOEPCTL |= USB_OTG_DOEPCTL_SNAK;
```

## Returning a descriptor
This is where the fun starts. The complete flow looks something like this:

1. Device is plugged in. `USBRST` is received since the host always resets new devices.
   Use this interrupt to reset your stack to a known initial state.
2. Reset finished. `ENUMDNEM` received. Setup your endpoints.
3. A SETUP request is sent from the host (OUT). `RXFLVLM` signals this. Data is read from the RX
   FIFO.
4. `RXFLVLM` is triggered again, this time signaling the SETUP status is complete. Pop the RX FIFO
   status register.
5. This triggers `STUPCNT` on OEP0. You can now process the request and write to the IEP0 FIFO to
   start an IN transfer.
6. Transfer is complete and triggers `XFRC` on IEP0INT to signal this. Program OEP0 to receive an
   empty OUT packet (status phase of the control transfer).

A complete code example integrated with FreeRTOS that illustrates this can be found below.

# Further Reading
This is not an exhaustive article and some specifics may be lacking. While this article may
certainly be of help, more information will be needed for hand writing a full USB stack. Here are a
few items which may prove useful.

## References
* I based a lot of my work on [libusb_stm32](https://github.com/dmitrystu/libusb_stm32). This is
  basically a bare metal USB implementation. In fact if you have read the code you will notice I
  basically copied the methods used to get EP FIFOs or configuration registers. This was of immense
  help to get an idea of how to implement it myself. Code may be confusing at times, but it is still
  worth a read.

* The [DWC2 driver from TinyUSB](https://github.com/hathach/tinyusb/blob/8fa0b74d8016554d23f84981f78c656f215d76e8/src/portable/synopsys/dwc2/dcd_dwc2.c) was also a valuable resource. Code is better documented but also more complex.
It was also of great help, chances are your device is supported by this library and you can read
the source code for the driver.

* During implementation I had a horrible bug which took me a week to solve. The device was successfully
writing the descriptor, but was not ACK the Status OUT packet which caused transactions to timeout. You
can read the whole adventure on the [ST community forums](https://community.st.com/t5/stm32-mcu-products/ep0-in-transfer-times-out/td-p/573140). 
Hint: I failed to follow the datasheet and was using an RXFIFO which was not big enough.

## Full code
The full code [is available on GitHub](https://github.com/crazybolillo/stell/tree/custom-usb). Take
with an extra grain of salt and don't blindly copy it, it probably is not that good. There may be
some extra code which is not really needed but was a result of the paranoia induced by the trauma of
dealing with the USB core.

# Debugging
I got around by using dmesg and Wireshark with the kernel module usbmon on Linux. I had to debug so often
I created a little script to start them automatically:

```bash
#!/usr/bin/env bash

sudo modprobe usbmon
sudo wireshark &
sudo dmesg -w
```

Try plugging working devices to see how they behave and respond to control transfers. Then connect
your device and compare their behavior. The tail mentioned above which can be read on the ST forums
shows how I debugged my way out of the situation.

dmesg on the other hand will probably state one too many times that it failed to read the device
descriptor. It is useful to know how the kernel is reacting to the device and what address it was
assigned (remember at the start it will have 0 as its address).

# Conclusion
Implementing a bare metal USB stack is a painful task and as long as there are good quality USB
libraries around you will probably just reinvent the wheel with no discernible advantages. It is a
great learning experience so it may be worth trying it just for the fun(?).

If this is your first time doing it be ready to spend a bit of time getting familiar with the core
and USB protocol itself. At the time it will seem that there is no information available and you
will read over and over any resources you can find on implementing USB manually. Over time it gets
easier and once you are on the other side it does seem obvious. Writing this post now makes it seem
like it wasn't that bad after all. Hindsight is always 20/20.

