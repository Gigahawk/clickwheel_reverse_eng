# iPod Clickwheel Protocol

## Physical layer
- Clickwheel acts as SPI master, interfacing device should act as SPI slave.
- Connection setup (for AVR):
    - Both clock and data lines should be pulled high to 3.3V
    - `CPOL` = 1 (clock is idle HIGH, falling edge leads)
    - `DORD` = 1 (LSB first)

## First Byte
First byte is a packet header, should always be `0x35`

## Second Byte
Second byte is button statuses (1 for pressed)

| MSB |   |      |            |      |      |        | LSB |
|-----|---|------|------------|------|------|--------|-----|
| ?   | ? | Menu | Play/Pause | Prev | Next | Center | ?   |

## Third Byte
Third byte is clickwheel touch location, it starts from `0x00` at the top (Menu button), increases in increments of 2 going clockwise, stopping at `0xBE`, for a total of 96 unique positions.
> Note: pressing buttons may cause the clickwheel to report random touch locations, these should be ignored (see below)

## Fourth Byte
The fourth byte is clickwheel touch enabled.
It is `0x00` when there is no finger on the clickwheel (any changes to clickwheel touch location should be ignored).
It is `0x80` when there is a finger on the clickwheel (clickwheel is reporting correct clickwheel touch location).
