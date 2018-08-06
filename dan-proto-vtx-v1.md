# VTX protocol for Drone Area Network Protocol, version 1

This document describes the commands implemented by a DAN compatible VTX.
It supports VTX with support for any arbitrary sets of bands/channels
and frequencies. VTX with means to change the current settings externally
(e.g. button/NFC/bluetooth) are also supported.

# DAN Messages


## VTX_IDENTIFY: 0x0020

- Input: None
- Output: *struct* DAN-PERIPHERAL-IDENTITY-PAYLOAD-V1

## VTX_CONFIGURE: 0x0021

- Input: *struct* DAN-PERIPHERAL-CONFIGURE-V1
- Output: *DAN-PERIPHERAL-ACK* ACK

## VTX_CAPABILITIES: 0x0022

Returns a list of the supported frequencies as closed ranges (i.e. a range
specifed as [F1, F2] includes support for F1 and F2). VTX should use this not
only to specify their minimum and maximum supported frequencies, but also to
exclude frequencies not supported due to other factors (e.g. regulatory
compliance). For example, a VTX supporting the 5600Mhz to 5900Mhz range and
excluding 5645Mhz should use these two ranges: [5600, 5644], [5646, 5900].

- Input: None    
- Output: *struct* DAN-VTX-CAPABILITIES-PAYLOAD-V2
    - *uint16_t* Capabilities: Bitmask. See DAN-VTX-CAPABILITIES-MASK.
    - *array[struct]* Bands
        - *string* Name: Long band name (e.g. "BOSCAM A")
        - *string* Short Name: Short name (usually 1 character long) for
        showing the band in small indicators or next to a channel number
        (e.g. "A").
    - *array[struct]* Power levels
        - *uint8_t* Type: DAN-VTX-POWER-LEVEL-TYPE-ENUM
        - *string* Name: Name, should be meaningful for the user (e.g. 25mw)
    supported power levels by the VTX. See additional considerations.
    - *array[struct]* Frequencies
        - *uint16_t* Minimum: Lowest frequency supported in this range (MHz).
        - *uint16_t* Maximum: Highest frequency supported in this range (MHz).


### DAN-VTX-CAPABILITIES-MASK

This 16-bit bitmask describes additional capabilities that a VTX might support.
Bits not defined in this specification MUST be zero.

- *CAP_POWERON = bit0*
  
    The VTX supports POWER-ON settings and MUST implement
    VTX_GET_POWERON_SETTINGS and VTX_SET_POWERON_SETTINGS.

- *CAP_RGB_LED = bit1*

    The VTX contains a controllable RGB LED. TODO: Implement commands for it.

### DAN-VTX-POWER-LEVEL-TYPE-ENUM

This enum defines how a power level should be handled by the DAN master.

- DAN-VTX-POWER-LEVEL-OFF = 0: No RF output. VTX MUST NOT include more than one
power level of this type.
- DAN-VTX-POWER-LEVEL-PIT = 1: Extremely low power level, used for producing
some RF output suitable for very close reception while minimizing the
possibility of interference with other aircrafts that might be flying.
- DAN-VTX-POWER-LEVEL-NORMAL = 2: Power level intended to be used while the
aircraft is flying.

### Additional considerations

Bands and channel names are intended to be displayed directly to the user.

All "Power level" should be sorted in increase order. This lets the
DAN master to set the VTX to its lowest power when the aircraft is unarmed.

## VTX_GET_BAND_CHANNELS: 0x0023

Returns the frequencies for the channels in the given band.

Channels MUST be returned in the same order that they should be presented to
the user. A band with a missing channel can be represented by setting its
frequency to zero. For example, if channels E4, E7 and E8 are not supported, the
channels array for the E band could be represented by [5705, 5685, 5665, 0,
5885, 5905, 0, 0]. A DAN master SHOULD read all channels, but it might decide to
read up to a maximum number due to memory constrains. 


- Input: *uint8_t* Band Index: The index one of the bands returned
in VTX_CAPABILITIES.
- Output: *array[uint16_t]* Channels: Channel frequencies in MHz.


## VTX_GET_POWER_LEVEL: 0x0024

Returns the current power level, as well as how it was set. This allows the
DAN master to differentiate when the change was made via DAN or via an
arbitrary external method (button/NFC/etc...).

- Input: None    
- Output: *struct*
    - *uint8_t* Power level index: Index referencing one of the power
    levels advertised in VTX_CAPABILITIES.
    - *uint8_t* Set By: See DAN-VTX-SET-BY-ENUM.

### DAN-VTX-SET-BY-ENUM

This enum tracks how a parameter was set. This lets a DAN master know if the change
was performed via DAN or via an external arbitrary method. Valid values:

- DAN-VTX-SET-BY-DAN = 0: The value was set in response to a DAN request.
- DAN-VTX-SET-BY-EXTERNAL = 1: The value was set externally (button/NFC/etc..).
- DAN-VTX-SET-BY-POWERON = 2: The value was set by power-on setting.


## VTX_SET_POWER_LEVEL: 0x0025

Sets the power level to the given index. The VTX is MIGHT commit this change to
permanent memory and restore it after a power cycle. Note that setting this
value via DAN should always set the "Set By" field in the VTX_GET_POWER_LEVEL
response to DAN-VTX-SET-BY-DAN, even if it's requesting the same power level
that is already active.

If the requested power level index is greater or equal than the amount of
defined power levels, the VTX should perform no changes and indicate an error
via the output.

- Input: *uint8_t* Power level index: Index referencing one of the power
    levels advertised in VTX_CAPABILITIES.
- Output: *DAN-PERIPHERAL-ACK* ACK


## VTX_GET_FREQUENCY: 0x0026

Returns the current frequency, as well as how it was set. This allows the
DAN master to differentiate when the change was made via DAN or via an
arbitrary external method (button/NFC/etc...).

- Input: None    
- Output: *struct*
    - *uint16_t* Frequency: Frequency in MHz.
    - *uint8_t* Set By: See DAN-VTX-SET-BY-ENUM in VTX_GET_POWER_LEVEL.


## VTX_SET_FREQUENCY: 0x0027

Sets the frequency to the given value in MHz. The VTX is MIGHT commit this change to
permanent memory and restore it after a power cycle. Note that setting this
value via DAN should always set the "Set By" field in the VTX_GET_FREQUENCY
response to DAN-VTX-SET-BY-DAN, even if it's requesting the same power level
that is already active.

If the frequency is not supported, the VTX should perform no changes and indicate
an error via the output. A VTX MUST support all frequencies advertised in the ranges
returned in VTX_CAPABILITIES.

- Input: *uint16_t* Frequency: Frequency in MHz.
- Output: *DAN-PERIPHERAL-ACK* ACK

## VTX_GET_POWERON_SETTINGS: 0x0028

Returns the power-on settings. Power-on settings are intended to make the VTX
start up on a very low power and, optionally, on a given frequency. This
avoids disrupting other aircrafts that might be flying in the vicinity.

- Input: None
- Output: *struct* DAN-VTX-POWERON-SETTINGS-PAYLOAD-V1
    - *uint8_t* Power Level Index: Power index of the level to use when
    powering on. If the value is 0xFF, there's no power-on power level
    configured and the VTX will start up using the saved power level for
    normal operation.
    - *uint16_t* Frequency: Frequency to use at power-on, in MHz. If
    the value is zero, there's no power-on frequency configured and the
    VTX will start up using the frequency for normal operation.

## VTX_SET_POWERON_SETTINGS: 0x0029

- Input: *struct* DAN-VTX-POWERON-SETTINGS-PAYLOAD-V1
- Output: *DAN-PERIPHERAL-ACK* ACK

## DAN MASTER requirements for a VTX

- MUST support at least 9 bands.
- MUST support at least 8 channels per band.