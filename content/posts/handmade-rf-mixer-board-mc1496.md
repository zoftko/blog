---
title: "Handmade RF Mixer Board with a MC1496"
date: 2023-04-17T12:38:33-06:00
tags: ["Electronics", "RF"]
---
In an RF context, a mixer is a device which "mixes" two frequencies, producing the sum
and difference of said frequencies at its output. In the time domain it works as a multiplier,
multiplying both input signals.

There are several circuits capable of behaving as mixers, this article will cover the design
and behavior of a mixer using a Gilbert Cell encapsulated in a MC1496 IC. This post assumes
previous knowledge of the MC1496 and its applications. To familiarize yourself with the chip you can
read this [post about it]({{< ref "/posts/simple-am-transmitter-mc1496.md" >}} "Simple AM Transmitter with a MC1496").

This handmade board is the first step to creating a proper evaluation board for this chip. I don't trust myself so
ordering a batch of PC boards with no prototype seemed dangerous. It is also a great excuse to play around
with SMD components (this was my first time soldering them).

# Building the board
Board layout is very important when designing high speed or high frequency circuits. Some basic
guidelines are:

* Traces need to be kept as short as possible to minimize the effects associated with transmission lines.
* Component leads should be kept short, or else they will act as inductors. Their parasitic
  properties need to be taken into account above certain speeds/frequencies. Tolerances also play
  an important role.
* Signals need a plane to couple to for effective transmission and good signal integrity, this usually
  involves a ground plane below the signal traces.

Therefore, I decided to use a one-sided copper board and copper tape. The copper in the board
will act as the ground plane, and the copper tape on top will ideally behave as a microstrip. The
board has a 1.6 mm thickness, which means a 3.2 mm wide strip of copper tape should behave as a 50 ohm
transmission line (approx.).

The KiCad design files this board is based on can be found [on this repository](https://github.com/crazybolillo/pcrap/tree/main/ev1496).
Basically most if not of it consists on the DC biasing network for the Gilbert Cell. The target
voltages are:

| Pin (s) | Function | Voltage (V) |
|:-------:|:--------:|:-----------:|
|    5    |   Bias   |    1.25     |
|  1, 4   |  Signal  |      5      |
|  8, 10  | Carrier  |     8.2     |
|  6, 12  |  Output  |    11.1     |

Said voltages are obtained with voltage divider networks, any valid resistor combination will do, however
the data sheet recommends the networks to sink 1mA.

I used 0603 SMD components, a DC barrel jack, and a potentiometer to make the DC bias of one
of the signal pins variables. For the few ground connections needed, I made poor man vias by
drilling a hole and sticking leads cut from THT components.

## Images
Here are some images which document the build process, a schematic and components values can be visually obtained
from these pictures. All capacitors were 10nF.


{{< figure src="/img/handmade-rf-mixer-board-mc1496/build-1.jpg" title="Voltage bus and future pads. The bus needs to go below the IC so no traces collide. They all must fit in the top layer" align="center" >}}


{{< figure src="/img/handmade-rf-mixer-board-mc1496/build-3.jpg" title="Soldered socket for the IC" align="center" >}}


{{< figure src="/img/handmade-rf-mixer-board-mc1496/build-8.jpg" title="Left side of the circuit completed" align="center" >}}

{{< figure src="/img/handmade-rf-mixer-board-mc1496/build-9.jpg" title="Finished board, just missing a matching impedance network" align="center" >}}

# Results
The test signals were generated with a GW Instek AFG-2012 function generator and the oscilloscope
used for the measurements was a Tektronix TBS 1072B. The function generator had a 50 Ohm output impedance and the
output was measured at a 50 Ohm SMA dummy load. No impedance matching was made, so results should be horrendous.

## DC Bias Point

| Pin (s) | Function | Voltage (V) |
|:-------:|:--------:|:-----------:|
|   NA    |   Vin    |    15.13    |
|    1    |  Signal  |    5.08     |
|  2, 3   |   Gain   |    4.37     |
|    4    |  Signal  |    5.08     |
|    5    |   Bias   |    1.194    |
|    6    |  Output  |    11.5     |
|    8    | Carrier  |    8.15     |
|   10    | Carrier  |    8.15     |
|   12    |  Output  |    11.44    |

## Mixing two signals

{{< figure src="/img/handmade-rf-mixer-board-mc1496/lo.jpg" title="Local Oscillator at 7MHz" align="center" >}}

{{< figure src="/img/handmade-rf-mixer-board-mc1496/rf.jpg" title="RF Input at 9MHz" align="center" >}}

{{< figure src="/img/handmade-rf-mixer-board-mc1496/mixer-out.jpg" title="Mixer output" align="center" >}}

# Conclusions
The circuit worked at some degree, which is already good. The DC biasing network worked almost perfectly, with the
biggest discrepancies being found at the output. Both outputs were about 400mV higher than expected.

This means that the biasing current was not 1mA. According to the calculations it was 930.76uA, an error of around
6.923%. Considering that no biasing rules were broken, and that part of it depends on the temperature and
characteristics of the internal diode, it seems good enough.

The output however is definitely horrendous, as expected. The mixer circuit presented in the MC1496's application
note is stated to have a 13dB conversion gain, in this case the mixer had a negative conversion gain, since the
output was smaller than the RF input.

This is probably caused by the lack of an impedance matching network. Ideally this mixer
will be used for IF down conversion, bringing a 153 MHz signal down to 21.4 or 48 MHz (to be determined), that'll
be the real test for this circuit, and a matching network will definitely be used there. In this case it did not make
much sense to make a matching network for a quick test.
