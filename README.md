# iPod Clickwheel Protocol

## Preface

All research has been done using an iPod 4th gen (photo) clickwheel.
Some of these findings are likely to also apply to the 5th gen and newer models as well

Much of this research was inspired by Jason Garr's original post: https://jasongarr.wordpress.com/project-pages/ipod-clickwheel-hack/

## Pulseview/Sigrok files

All signals were captured on an iPod Photo (color 4th gen) via [clickwheel_breakout_4th_gen](https://github.com/Gigahawk/clickwheel_breakout_4th_gen), meaning signals were measured at the flat flex cable between the iPod mainboard and the headphone jack board.

- `clickwheel_pulseview`: Pulseview capture of signals coming from clickwheel
- `clicker_pulseview`: Pulseview capture of signal going to the clicker on the headphone board.
    - The clicker appears to be AC coupled by a capacitor on the headphone board.


## Pinout

> This is specific to the 4th gen clickwheel, notably on 5th gen and newer the click switches are situated on the mainboard, and there are extra pins on the touch flex cable so that the MCU driving the clickwheel can read them.

Pin ordering is printed on the FPC.

1. VBat (Power input)
    - Main power input, apparently accepts the full range of a Li-po, but I have only tested using a 3.3V supply
2. SCK (Logic output, tristate)
    - Serial clock, push-pull driven by the clickwheel when sending a packet, hi-Z when idle.
        - The iPod pulls this line high
3. CFG1 (Logic input)
    - See [Configuration Pins](#configuration-pins)
4. BTN1 (Logic output, push-pull)
    - Idles low, pulses high for approximately 29.83ms when the either the center or menu button is pressed, nothing happens if the button is released.
        - Pulse is sent approximately 15ms after the start of the data packet for the event
        - This pin appears to be used to cold boot the iPod (after pulling power)
5. ??? (Logic output, push-pull?)
    - Always low, even with a 10k pullup to 3.3V.
    - I can't find a condition that toggles this line
    - This line is NOT connected to GND when power is removed
6. MOSI (Logic output, open drain)
    - Serial data from clickwheel to CPU, open drain.
    - See [Event Packet](#event-packet)
    - NOTE: the clickwheel is the master, NOT the main processor
7. MISO (Logic input)
    - Serial data from CPU to clickwheel
    - See [Commands](#commands)
8. GND


## Physical layer

See [Pulseview/Sigrok files](#pulseviewsigrok-files) for more details

Without the pullup (or with a pulldown), the clock line idles low.
At the start of a packet the clickwheel clock line will:
1. pulse high for 2.9us (start pulse)
2. clock 32 times at about 55kHz
    - The last pulse lasts a little longer, about 11us instead of 9us
3. Go idle (low)

The data pin will go low while the clock is low after the start pulse.
The start data on all valid packets appears to be identical so it's not clear whether data should be read on the rising or falling edge of the clock.
This document will assume data is read on the falling edge.

If a pullup resistor is added to the clock line, the line idles high, which means the start pulse is hidden, the packet start is marked by a falling edge of the clock. Now the clock line will always just clock 32 times at ~55kHz for one packet.
In this case, we can treat the clickwheel as a SPI master with `CPOL = 1` (clock is idle high, falling edge leads).
This document will assume the data is LSB first.

## Configuration Pins

### CFG1 = 0

The clickwheel will continuously send packets at a rate of ~1.465kHz, when no touch/click events are happening the data will always be all `0xFF`s

The fastest rate the clickwheel can send touch/click events appears to be about 67Hz (when swiping fast across the touchwheel).
This means that there are always at least 21 empty packets between events.

When CFG1 = 0, the CPU can send commands back to the clickwheel via the MISO pin.
See [Commands](#commands)

### CFG1 = 1

The clickwheel will only send packets when touch events occur, up to a rate of about 67Hz

## Clicker Sound

The waveform was reproduced with GPIO from an [Adafruit STM32F405 Feather Express](https://learn.adafruit.com/adafruit-stm32f405-feather-express) runnining MicroPython to get finer control of timing.

When driving the clicker directly from GPIO, the sound is quieter than the iPod, the Feather does not appear to have enough drive strength.
When driving the speaker through a low Rds(On) MOSFET switching the 3.3V output on the Feather, the sound is nearly identical to the original iPod.
> Note: test was done with an N-channel MOSFET (IRLB8748PbF), but for implementing in a design a P-channel MOSFET is probably recommended since the clicker is directly connected to the digital GND of the iPod

## Clickwheel Communication Protocol
### Assumptions
- Pullup resistor is connected on SCK and SDA
- Clickwheel is a SPI master device (SDA is MOSI)
    - Both clock and data lines should be pulled high to 3.3V
    - `CPOL` = 1 (clock is idle HIGH, falling edge leads)
    - `CPHA` = 0
    - `DORD` = 1 (LSB first)

### Event Packet

These packets are sent from the clickwheel to the CPU when the clickwheel experiences a event (button press/release, touch down/drag/up)

#### First Byte
First byte is seems to be a packet header, always `0x35`

#### Second Byte
Second byte is button statuses (1 for pressed)

| MSB |   |      |            |      |      |        | LSB |
|-----|---|------|------------|------|------|--------|-----|
| ?   | ? | Menu | Play/Pause | Prev | Next | Center | ?   |

#### Third Byte
Third byte is clickwheel touch location, it starts from `0x00` at the top (Menu button), increases in increments of 2 going clockwise, stopping at `0xBE`, for a total of 96 unique positions.
> Note: pressing buttons may cause the clickwheel to report random touch locations, these should be ignored (see below)

#### Fourth Byte
The fourth byte is clickwheel touch enabled.
It is `0x00` when there is no finger on the clickwheel (any changes to clickwheel touch location should be ignored).
It is `0x80` when there is a finger on the clickwheel (clickwheel is reporting correct clickwheel touch location).

### Commands

Commands are sent from the CPU to the clickwheel by setting CFG1 = 0.

#### Sleep

The clickwheel can be put to sleep by sending `95 02 00 C0`.

In this state, the clickwheel will only send event packets for button events, not touch events.

As far as I can tell, the clickwheel will automatically wake when a button is pressed, however the iPod will send some commands to the clickwheel when it wakes up.
The clickwheel will start emitting touch events after a button is pressed even if MISO is severed after putting the clickwheel to sleep.

#### Init 1???

The CPU will send `1D 01 00 C0` when it wakes from sleep.
The clickwheel is expected to reply with `75 04 02 00`

#### Init 2???

The CPU will send `95 82 00 C0` after the first init command (only seems to happen if the clickwheel can send its reply)

## Low Power Modes

The iPod supports two types of low power mode:
- Power off
    - Main processor is in deep sleep (clickwheel in sleep too?), waits for interrupt from BTN1, only way to wake from this state is to press the  center or menu button
- Sleep
    - Main processor and clickwheel goes to sleep, touch events do not get emitted, only button events
- Hold switch
    - Clickwheel power is disabled (presumably there is a PMOS that cuts power to the clickwheel)
    - Upon unlocking the hold switch the clickwheel will send a bunch of `AA` to indicate it is ready
