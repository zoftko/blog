---
title: "Simple AM Transmitter with a MC1496 IC"
date: 2022-08-04T12:31:54-06:00
tags: ["Electronics", "RF"]
---
The MC1496 is a Balanced Modulator/Demodulator, capable of producing amplitude modulated signals.
Its internal structure resembles a Gilbert Cell biased by two current sources (one for each leg of the circuit), 
both of them being biased by a single diode with a 500 Ohm resistor.

{{< figure src="/img/simple-am-transmitter-mc1496/internals.png" title="MC1496 - Internal Schematic" align="center" >}}

The internal schematic was gathered from
[Application AN531 from onsemi](https://www.onsemi.com/pub/Collateral/AN531-D.PDF) which is a great resource
for in-depth explanations on proper usage of the IC.
This post is based on it and will detail the design and construction procedures used to create a simple
AM transmitter whose output can be heard through a commercial AM Radio.

# Design
At least three components are required in order to generate an AM signal:

* Carrier Wave
* Message Wave
* Mixer (modulator)

In order for the transmitter to be heard by a commercial receiver, it needs to transmit in the AM commercial band
(illegal without a license), this band normally goes from 
[530 to 1700 MHz](https://en.wikipedia.org/wiki/Medium_wave#Spectrum_and_channel_allocation).
I have no regards for the law, and this design does not have enough power nor an antenna good enough (would need to
be at least 75 m long) to interfere with commercial broadcasts, so a 1 MHz crystal is used for generating the carrier
wave.

## Local Oscillator
The Local Oscillator is in charge of producing the carrier wave.
A Pierce topology (common emitter) was used since it is easy to design and creates a decently
clean and strong signal.

[From Foundations of Oscillator Design](https://www.amazon.com/Foundations-Oscillator-Circuit-Microwave-Hardcover/dp/1596931620)
(talking about Clap, Colpitts and Pierce topologies):
> The circuit biasing resistors and stray capacitance appear in shunt with
different elements in each of the three configurations. This makes the performance
of the three configurations different. The Pierce configuration is usually the most
desirable.

And from [Crystal Oscillator Circuits](https://www.amazon.com/gp/product/0471874019):
    > It has very good short-term stability because the crystal’s source and load impedance are mostly capacitive rather
than resistive, which give it a high in-circuit Q. 
The circuit provides a large output signal and simultaneously drives the crystal at a low power level.

The output is taken from the base of the transistor, where there exists a (relatively) big capacitance, so any
extra stray capacitance introduced by the next stage won't affect the oscillator too much. 

The next stage consists of a common collector buffer, being biased by a constant current source in an attempt to
provide a high input impedance, trying to isolate the oscillator as much as possible from the next stages.
Small signal diodes are usually a better choice for this task, but this was all I had on hand at the time.

{{< figure src="/img/simple-am-transmitter-mc1496/oscillator.png" title="1 MHz Crystal Pierce Oscillator" align="center" >}}

## Message Wave
The message wave contains the information to be superimposed on the carrier wave. In this case it will simply
be the AUX output from a phone. I have a handy Bluetooth to AUX converter which ensures I burn the converter
and not my phone in case I do something horribly wrong.

The amplitude of the message wave has a direct impact on the modulation index and higher frequency components
need to be filtered out, but since this is a simple test those two points will be conveniently ignored.

## Modulator
Following the application note mentioned at the beginning, I arrived at the schematic shown below. Containing at
least 8 transistors, DC biasing is a crucial step for the proper functioning of the MC1496 chip. A brief
overview of the guidelines and the design steps to follow them were:

1. Output pins (6, 12) must be at a higher DC potential than all other pins.
2. Carrier pins (8, 10) must be 2V below the output pins.
3. Modulating pins(1, 4) must be 2.7V below the carrier pins.
4. The bias pin(5) must be 2.7V below the modulating pins.

This leaves very little headroom, and it's why inductors were used at the output stage. 
They act as resistances at the operating frequency, providing the gain necessary, but act as a short circuit when
it comes to DC biasing.

Bias current (I5) was chosen to be 1mA, considering the diode forward voltage to be 0.75 Vdc at ambient
temperature, the voltage at the bias pin turns out to be 1.25 V.

Knowing the max and min voltage limits, a resistor network was added to provide bias to the two missing stages
of the circuit. 

Audio is inserted into pin 4, where a variable resistor controls the modulation index.
[This video](https://www.youtube.com/watch?v=38OQub2Vi2Q) explains how varying the bias voltage affects the
modulation index.

{{< figure src="/img/simple-am-transmitter-mc1496/schematic.png" title="MC1496 Amplitude Modulator" align="center" >}}

# Results
{{< figure src="/img/simple-am-transmitter-mc1496/board.jpg" title="Constructed board (crystal goes into the empty sockets)" align="center" >}}
{{< figure src="/img/simple-am-transmitter-mc1496/wave.png" title="Message and resulting AM signal" align="center" >}}
{{< figure src="/img/simple-am-transmitter-mc1496/fft.png" title="FFT of the AM signal" align="center" >}}
{{< youtube id="MBk8WKM4mkQ" >}}
