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

- DAN peripherals must use a baudrate of 115200, 1 stop bit and no parity bits.
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

- *enum* DAN-PERIPHERAL-ACK
    - DAN-PERIPHERAL-ACK-OK = 0: Indicates an operation was successfully
    performed.
    - DAN-PERIPHERAL-ACK-FAIL = 1: Indicates an operation failed. When needed,
    responses will also define some means to indicate the failure reason.

## Well known DAN command codes

In order to allow using the serial port as a bus, each device type uses a different
command code space. See each peripheral type's document for the message definitions. 

See other documents for peripheral specific features