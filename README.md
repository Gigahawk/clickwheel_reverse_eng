# iPod Clickwheel Protocol

## Pulseview/Sigrok files

All signals were captured on an iPod Photo (color 4th gen) via [clickwheel_breakout_4th_gen](https://github.com/Gigahawk/clickwheel_breakout_4th_gen), meaning signals were measured at the flat flex cable between the iPod mainboard and the headphone jack board.

- `clickwheel_pulseview`: Pulseview capture of signals coming from clickwheel
- `clicker_pulseview`: Pulseview capture of signal going to the clicker on the headphone board.
    - The clicker appears to be AC coupled by a capacitor on the headphone board.

## Clicker Sound

The waveform was reproduced with GPIO from an [Adafruit STM32F405 Feather Express](https://learn.adafruit.com/adafruit-stm32f405-feather-express) runnining MicroPython to get finer control of timing.

When driving the clicker directly from GPIO, the sound is quieter than the iPod, the Feather does not appear to have enough drive strength.
When driving the speaker through a low Rds(On) MOSFET switching the 3.3V output on the Feather, the sound is nearly identical to the original iPod.
> Note: test was done with an N-channel MOSFET (IRLB8748PbF), but for implementing in a design a P-channel MOSFET is probably recommended since the clicker is directly connected to the digital GND of the iPod

## Clickwheel Communication Protocol
### Physical layer
- Clickwheel acts as SPI master, interfacing device should act as SPI slave.
- Connection setup (for AVR):
    - Both clock and data lines should be pulled high to 3.3V
    - `CPOL` = 1 (clock is idle HIGH, falling edge leads)
    - `DORD` = 1 (LSB first)

### First Byte
First byte is a packet header, should always be `0x35`

### Second Byte
Second byte is button statuses (1 for pressed)

| MSB |   |      |            |      |      |        | LSB |
|-----|---|------|------------|------|------|--------|-----|
| ?   | ? | Menu | Play/Pause | Prev | Next | Center | ?   |

### Third Byte
Third byte is clickwheel touch location, it starts from `0x00` at the top (Menu button), increases in increments of 2 going clockwise, stopping at `0xBE`, for a total of 96 unique positions.
> Note: pressing buttons may cause the clickwheel to report random touch locations, these should be ignored (see below)

### Fourth Byte
The fourth byte is clickwheel touch enabled.
It is `0x00` when there is no finger on the clickwheel (any changes to clickwheel touch location should be ignored).
It is `0x80` when there is a finger on the clickwheel (clickwheel is reporting correct clickwheel touch location).
