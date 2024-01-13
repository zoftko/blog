---
title: "How to use an ST LED1642GW"
date: 2024-01-13T11:40:18-06:00
tags: ["Electronics", "STM32"]
---
We have all probably turned on an LED with the classical 5V + 330 (or so) Ohm resistor combination.
Failing to limit the current would result in a burnt LED. Besides resistors, one may also use
LED drivers to control their brightness and limit the current going through them.

The [LED1642GW](https://www.st.com/en/power-management/led1642gw.html) by ST is one of such LED
drivers. It sinks current, has 16 channels and can be controlled by an external MCU through
a serial link. This article documents how to achieve such control with the help of an STM32F401CCU6
microcontroller.

The end result is the following, a constant fade in/out of LEDs achieved by varying the brightness
to which the LED1642GW channels are set. 

*Note that the blinking can't be perceived by the human eye but gets picked up by the camera.*

{{< youtube id="GjykLighhdw" >}}

# Implementation
As can be seen in the video, 5V are used to power the LED1642GW IC and LEDs. Capacitors were placed
on the LED rail as well as on the LED1642GW supply terminals, this is recommended by the datasheet.

To properly function, the LED driver requires a PWCLK signal which the internal counter uses.
The required frequency depends on whether the internal counter is using 12 or 16 bits. In this case
the counter was configured to use 12 bits, so a 500 KHz frequency is used for PWCLK.

Finally an external resistor (R-EXT) is needed to set the reference current.
A value of 12k Ohms was used.

Since this chip is only available in SMD packages, a breakout board was used, this makes it possible
to place it on a protoboard.

{{<figure src="/img/how-to-use-led1642gw/led1642gw.jpg" title="LED1642GW used for the article" align="center">}}

## Serial Communication
The protocol used is custom so I was not able to integrate it with any existing hardware module
on the MCU. Three pins are used:

* Clock
* Data
* Latch

A transmission block consists of 16 data bits together with N amount of clock cycles with latch HIGH,
the amount of cycles determines the register where data will be written (command executed).

A bit bang approach was used, it is based on a timer which works as a reference by generating
interrupts at 100kHz (same frequency used by I2C in standard mode). A register keeps track
of whether the clock should rise/fall and sends the data and latch bits before turning the clock
pin HIGH.

**This last point is important. During testing it was found that communications would fail if the
clock goes HIGH before the data line.**

All of this is performed in the timer ISR. A code excerpt can be seen below. It uses FreeRTOS API
calls to notify the main task that transmission has ended. The link to the full code
can be found on the "Tips" section.

```c
// Datastore to keep track of the transmission state
struct Message {
    uint32_t data;
    int size;
    int latch;
};

// State machine, called by the ISR
void led1642_transmit(void) {
    if ((message.data >> message.size) & 0x01) { sdo_on(); }
    if (message.latch > message.size) { le_on(); }
    message.size--;
}

void led1642_set_brightness(const uint32_t brightness) {
    message.data = brightness;
    for (int idx = 0; idx < BRIGHTNESS_REG_COUNT - 1; idx++) {
        message.size = TWO_BYTES;
        message.latch = BRIGHTNESS_LATCH;
        // Turn on timer and transmit 16 bit block
        TIM2->CR1 |= TIM_CR1_CEN;
        // Wait for transmission to end
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
    }
    message.size = TWO_BYTES;
    // Last brightness value must be sent with a different latch, see datasheet
    message.latch = BRIGHTNESS_GLOBAL_LATCH;
    TIM2->CR1 |= TIM_CR1_CEN;
    ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
}

void TIM2_IRQHandler(void) {
    if (TIM2->SR & TIM_DIER_UIE) {
        if (message.size >= 0) {
            if (rising) {
                led1642_transmit();
                clock_on();
            } else {
                sdo_off();
                clock_off();
            }
            rising = !rising;
        } else {
            sdo_off();
            le_off();
            clock_off();
            // Turn off timer and notify message has been sent
            TIM2->CR1 &= ~(TIM_CR1_CEN); 
            vTaskNotifyGiveFromISR(led1642TaskHandle, NULL);
        }
    }
    TIM2->SR = 0;
}
```

## Transmission examples
To better illustrate the protocol, two transmission blocks recorded with a logic analyzer will be
shown. The upper signal corresponds to the latch, the middle one to the clock, and the bottom one
to data.

The command below places latch HIGH for the last 7 clock cycles, according to the datasheet
this corresponds to the "Write Configuration Register" command. All but the 15th bit are set to
their default value of 0. This last bit sets the counter to its 12 bit mode.

{{<figure src="/img/how-to-use-led1642gw/write-config.png" title="Setting the counter to 12 bits" align="center">}}

This other command turns ON all 16 channels. It can be seen that the latch now corresponds to the
"Write switch" command.
{{<figure src="/img/how-to-use-led1642gw/write-switch.png" title="Turning ON all channels" align="center">}}

# Tips
During the realization of this project, a few issues were found, here are some tips that may help
other souls.

* Setting up timers on the STM32 platform may be confusing at first. Check your clock tree to see
  which frequency is fed to your timers, and remember **that APB1 frequency is multiplied by two (2x)
  if the prescaler is not equal to 1.**
* If your LEDs blink or turn on/off randomly, try increasing PWCLK or using the counter in its 12
  bit mode.
* A Logic Analyzer will help you keep your sanity, for such low frequencies a cheap one will be more
  than enough.

Finally, the complete [code implementation can be consulted here](https://github.com/crazybolillo/stm32f401xc/blob/main/src/led1642.c).

