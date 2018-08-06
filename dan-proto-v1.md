# The Drone Area Network Protocol, version 1

## Authors

- Alberto Garcia Hierro (@fiam)
- Andrey Mironov (@diehertz)
- Michael Keller (@mikeller)

Drone Area Network (DAN) is a joking moniker we came up with to describe what
we're trying to achieve. It's purpose is to allow unified connection of
peripherals to the Flight Controller (FC) by the means of a unified protocol,
with possible support for other peripherals in the future, e.g. smart Raven
receivers.

The rationale for such a technology is to have an open, standardised and free
for everyone peripheral control protocol controlled by the community, not
separate vendors.

DAN will define a list of hardware and software requirements for the devices
implementing it to play along nicely. DAN's goal is to allow multiple
peripherals of different types to share a single half-duplex uart, used as a
bus. Multiple devices of the same type are supported by using multiple uarts.

## Hardware requirements

- DAN peripherals MUST initialy use a baudrate of 9600, 1 stop bit and no
parity bits.
- Half-duplex.
- No pulling of the line when inactive.

## Software requirements

- Peripherals MUST ignore messages they do not support.
- Predefined message IDs and payload formats.
- Listen Before Talk.

## Common data type definitions

- uint8_t: Unsigned byte.
- uint16_t: Unsigned 16-bit little endian.
- uvarint_t: Variable encoded unsigned integer. Allows [0, 127] using a single
byte.
- array[T]: uvarint_t length - [T... elements]
- string: [array(uint8_t)]
- struct: A simple concatenation of fields with no padding between them.
- union: A list of fields where only of them can be used at any given time,
  usually preceded by an enum to determine the field used.

## General message type

A DAN message is described by the following struct. All the fields are
mandatory:

- *struct* DAN-MESSAGE-V1
    - *uvarint_t* Sync byte = 'D'.
    - *uvarint_t* Message code: Request/response code. Must be > 0, except for
    errors.
    - *array[uint8_t]* Payload: Message payload. Array length MUST be zero for
    messages with no payload.
    - *uint8_t* CRC: Checksum covering all bytes after the sync byte,
    calculated using the crc8_dvb_s2 algorithm.

A DAN message MUST be answered with a response using the same message code. If
the peripheral doesn't understand the request message code, it should ignore
it. On the other hand, if the message code is a known one but the request is not
well formed (e.g. payload doesn't adhere to the expected format), the peripheral
MUST reply with a message where *Message code* is zero and *Message payload* is
an *struct DAN-ERROR-MESSAGE-V1*.

- *struct* DAN-ERROR-MESSAGE-V1
    - *uvarint_t* Message code: The request code in the incoming message that
    produced this error.
    - *uvarint_t* Error code: Reserved, must be zero.
    - *string* Error description: Textual description of the error. Might be
    empty. 

## Common payloads

- *struct* DAN-PERIPHERAL-IDENTITY-PAYLOAD-V1
    - *uint8_t* DAN version: DAN version supported by this peripheral.
    Must be 1 for version 1.
    - *string* Manufacturer name
    - *string* Model name
    - *string* HW version/revision
    - *string* Software version
    - *uint16_t* DAN-PERIPHERAL-SUPPORTED-BAUDRATES-MASK-V1
    - *uint16_t* DAN-PERIPHERAL-CAPABILITIES-MASK-V1

- *struct* DAN-PERIPHERAL-CONFIGURE-PAYLOAD-V1
    - *uint8_t* DAN-PERIPHERAL-CONFIGURATION-REQUEST-TYPE-ENUM-V1
    - *union* DAN-PERIPHERAL-CONFIGURATION-REQUEST-FIELDS-V1
        - *uvarint_t* Requested baud rate: Baud rate the master wants to use to
        communicate with this device, in bps.

- *enum* DAN-PERIPHERAL-ACK
    - DAN-PERIPHERAL-ACK-OK = 0: Indicates an operation was successfully
    performed.
    - DAN-PERIPHERAL-ACK-FAIL = 1: Indicates an operation failed. When needed,
    responses will also define some means to indicate the failure reason.

### DAN-PERIPHERAL-SUPPORTED-BAUDRATES-MASK-V1

This 16-bit bitmask describes higher baud rates supported by the peripheral.
Bits not defined in this spec MUST be zero.

A DAN master MIGHT use this information to communicate with this device at
higher speeds, but devices should not assume the DAN master will use any
specific BPS. The following bits are defined:

- *BPS_19200 = bit0*

    The device supports a baud rate of 19200bps.

- *BPS_38400 = bit1*

    The device supports a baud rate of 38400bps.

- *BPS_57600 = bit2*

    The device supports a baud rate of 57600bps.

- *BPS_115200 = bit3*

    The device supports a baud rate of 115200bps.

### DAN-PERIPHERAL-CAPABILITIES-MASK-V1

This 16-bit bitmask describes additional DAN capabilities that a peripheral
might support. Currently, all bits are reserved and this field MUST be zero.

### DAN-PERIPHERAL-CONFIGURATION-REQUEST-TYPE-ENUM-V1

This 8-bit enum tells a peripheral the type of configuration request sent by
the DAN master, allowing the peripheral to read the appropriate field in
DAN-PERIPHERAL-CONFIGURATION-REQUEST-FIELDS-V1. The following values are
defined.

- DAN-PERIPHERAL-CONFIGURATION-REQUEST-TYPE-BPS = 0: Configure the bps used
for all future communication. Value is in the "Requested baud rate" field.


## Well known DAN command codes

In order to allow using the serial port as a bus, each device type uses a
different command code space. See each peripheral type's document for the
message definitions. 

See other documents for peripheral specific features.

## Principle of operation

All DAN peripherals should be connected in a star configuration to the DAN
master, via a single wire (i.e. all peripherals should be connected to a single
pin in the master).

During startup, the master will perform the discovery phase by probing for each
device type defined in the DAN spec on each DAN-enabled UART at 9600bps. After
the discovery phase is done, no new peripherals will be detected (i.e. hot
plugging is not supported).

Once all the devices present in the bus are discovered, the master will send a
CONFIGURE request as appropriate. Devices that are requested to run at a given
baud rate, should continue to do so until they're power cycled.