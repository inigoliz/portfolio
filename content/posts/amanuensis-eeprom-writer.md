---
date: '2024-12-10T14:27:41+01:00'
draft: false
title: 'Minimalistic EEPROM programmer'
slug: amanuensis-eeprom-interface
cover:
  image: "/images/amanuensis-eeprom-interface/shield_full.jpeg"
  alt: "Cover image"
  relative: false
---
![(Amanuensis Header Art)](/images/amanuensis-eeprom-interface/amanuensis_header.png#center "700px")

---
## What is it?

A command-line tool to interact with a 28c256 256KB EEPROM, which is the one used in 
[Ben Eater's 6502 pc](https://www.youtube.com/watch?v=LnzuMJLZRdU&list=PLowKtXNTBypFbtuVMUVXNR0z1mu7dp7eH&index=1).
Allows reading contents from the EEPROM and writing data to it.

![(Reading EEPROM preview)](/images/amanuensis-eeprom-interface/read_range_presentation.png#center "700px")

> **Note:** The command used to drive the EEPROM is`nuensis`. It is named after the Amanuensis monks, which provided services as scribes.

Check the project on [Github](https://github.com/ignigoliz/amanuensis) to make your own!

---
## Motivation

Some years ago I went down the rabbit hole of building an 8-bit pc after getting inspired by Ben Eater's Youtube videos.
The goal was to learn computer microarchitecture and to demistify the transition from logic gates to an *actual device* that runs computations.
It was very fun, and I plan to keep on playing with it in the near future.

However, I found the interaction with the EEPROM (the memory of the 8-bit pc) quite rough.
Ben Eater himself has a [video](https://www.youtube.com/watch?v=K88pgWhEb1M) explaining how
to use an Arduino to write data to the EEPROM. Nonetheless, the Arduino code has to be changed
and re-uploaded with every new content one wants to write to the EEPROM.
I thought a Terminal-based interaction would make the experience so *muuuuuch* better.
So, **I decided to build it**.

> **Note:** The core of the 8-bit pc is the 6502 microprocessor, a mythical chip that powered devices such as Apple II, NES, Commodore 64, and many others.

![(Ben's EEPROM programmer)](/images/amanuensis-eeprom-interface/eater_setup.png#center "750px")

In particular, the reasons that make interacting with the EEPROM a painful process are the following:

1. With every new EEPROM data, the Arduino script has to be updated and re-uploaded.
2. Likewise, when reading the EEPROM, selecting different addresses requires updating and re-uploading the Arduino code.
3. The Arduino pins must be connected to the EEPROM's pins thorugh 28 wires that have to be patiently and carefully placed. Also,
repeatedly plugging in and out the EEPROM for writing can end up damaging the EEPROM's pins.

All this is annoying, specially considering that reading or writing the EEPROM is something that has to be done very frequently.


My solution:

1. A CLI Tool to control the EEPROM using Terminal commands. One of the commands, `nuensis write --file program.bin`, writes a hexdump to the EEPROM.

![(Binary program contents)](/images/amanuensis-eeprom-interface/program.png#center "700px")

2. Other commands allow reading the EEPROM. To read a range of addresses, `nuensis read --range 0000 003f`, which reads addresses `0x000` to `0x003f`. To read a single address, `nuensis read --address 002f`. 

![(Gif showing EEPROM reading)](/images/amanuensis-eeprom-interface/nuensis_read_alone.gif#center "700px")

3. The protoboard and the cables are replaced by a custom shield that is plugged directly into the Arduino Mega header.
Rather than plugging the chip and progressively damaging the pins, a ZIF (Zero Insertion Force) connector grips the EEPROM with delicacy.
![(Closeup of the mounted shield)](/images/amanuensis-eeprom-interface/shield_full.jpeg#center "500px") 

## How it Works

The command parser for the Terminal runs in Python. Python communicates over the *Serial Port* with Arduino and sends what operation to perform.
Arduino executes the low level electric pulses and register conigurations that eventually read/write the EEPROM.

Each time a new command is submitted on the Terminal, a communication schema like the following takes place:

![(Closeup of the mounted shield)](/images/amanuensis-eeprom-interface/schema_2.png#center "650px") 

I decided to implement an acknowledgment-based communication structure to make sure that data was sent and received correctly.
After data is sent, the sender enters a halting state waiting for an acknowledgment (`ACK`) response. This applies to any sent data, except the ACK itself.

Besides this basic structure, the code has several optimizations. For example, when reading data, Arduino does not transfer it in single values but in batches of 16 bytes.

## Serial Communication Python - Arduino

I use the Python `pyserial` library, and the built-in Arduino `Serial` library. Using serial communication in Python is not difficult but it's a bit slippery so I want to document the best practices I found.

When dealing with the EEPROM, everything is best thought in terms of *bytes*. Memory addesses are 2 bytes long and
each memory address stores data 1 byte long. Therefore, I need a way to *send byte-level information between Python and Arduino*.

> **Note:** 1 byte can store numbers in the range 0-255. Using hex encoding, a byte is represented using 2 hex numbers.
`255 = 0xFF; 0 = 0x00; 58=0x3A`. Likewise, 4 hex numbers represent 2 bytes of information.

```python
# PSEUDOCODE: The kind of communication desired:

# Goal: Write data 0x00 to address 0x0407
package = [0x00, 0x04, 0x07]

# Send data from Python to Arduino
serial.send(package)

# Arduino receives data and writes the EEPROM
package =  serial.read()
data = package[0]
address = combine(package[1], package[2])
eeprom.write(data, address)
```

> **Note:** I use *Big Endian* encoding: `0x0407 == [0x04, 0x07]`.

### Hex Arithmetics in Arduino
The Arduino programming language, which is based in C++, makes it easy to define byte-sized information:

```cpp
unsigned int address = 0x000f;
byte data = 0xea;
```

> **Note:** `byte` is exclusively an Arduino datatype; it's equivalent to `char` in C++,
or `std::byte` in C++17.

Also, hex arithmetics behave intuitively in Arduino:
```cpp
unsigned int x = 0xabcd; // 0xABCD = 43981

x == 0xabcd; // True
x == 43981; // True
```

### Hex Arithmetics in Python

Things get more challenging in Python. Python being by default a non-typed language makes it confusing to work with byte-level data directly in hex.

Python provides two classes to handle **lists** of byte data: `bytes()`, immutable; and `bytearray()`, mutable.

First, some features about the `bytes()` class. An instance of such an object can be 

```python
a = b'\xab'  # 0xAB = 171
list(a)  # prints [171]

b = b'\xab\xcd'
list(b)  # prints [171, 205]
```

So far so good. Feeling lucky. What will happen with the following...?

```python
c = list(b'\xabcd')
list(c)
```

Let's see...
![(Weird behaviour of bytes())](/images/amanuensis-eeprom-interface/bytes_weird.png#center "500px")

Ehm... OK. That made me wink. Why 3 elements?

Well, it seems that Python, being *string-o-centric*, interprests the previous statement as "Create a list of byte information with the contents '0xab', 'c', and 'd'". Then, it uses ASCII encoding for 'c' (`c = ASCII 99`) and d (`d = ASCII 100`).

Why this behaviour? Well, it is because the `bytes()` object is not meant to behave nicely with HEX numbers. It is meant to store *byte strings*.

![(Byte string vs normal string)](/images/amanuensis-eeprom-interface/bytes_strings.png#center "500px")

I'd like to work with `bytes()` using a constructor like `some_magic_constructor('abcd')`. That way I could treat separately the bytes that compose an address, like `some_magic_constructor('ab' + 'cd')`.

Fortunately, there is `bytes.fromhex()` that does just that. And `.hex()` prints a its content as a hex string:

```python
a = bytes.fromhex('ab' + 'cd')

a.hex()  # prints 'abcd'
```

The good thing about `bytes()` is that it allows list access to its elements. The bad thing is that it does not allow item assignment:

```python
a = b'\xab\xcd'

a[0]  # prints 171 (0xab = 171)
a[0] = b'\xef'  # TypeError: 'bytes' object does not support item assignment
```

Fortunately, the class `bytearray()` behaves pretty much like `bytes()` but allows item assignment. It is instantiated by wrapping a `bytes()` object.

```python
a = byetearray(b'\xab\xcd')
b = bytearray.fromhex('ab' + 'cd')

b[0] = bytes.fromhex('ef')  # TypeError: 'bytes' object cannot be interpreted as an integer
b[0] = 239  # 0xEF = 239
```

The only problem is that its elements cannot be assigned directly to `bytes()`- Weird!. But that's a shortcoming I can live with.

### Sending information over Serial Port

The Python `pyserial` library makes it straigthforward sending byte information over the Serial Port to Arduino:

```python
dev = '/dev/' + os.popen("ls /dev/ | grep cu.usbmodem").read().rstrip()
port = serial.Serial(port=dev, baudrate=9600, timeout=0.1)

packet = bytearray()
packet += bytearray.fromhex('ab')
packet += bytearray.fromhex('cd')

port.write(packet)
```

The Arduino is configured to just send back the data it receives:

```cpp
...
void loop() {
  while (Serial.available() > 0)
  {
    unsigned char incomingByte[1];
    Serial.readBytes(incomingByte, 1);
    Serial.write(incomingByte, 1);
  }
}
```

The data that Arduino sends back gets stored in the serial buffer of the pc. Upon executing `port.read()`, the values are fetched one by one until the buffer is exhausted.

![(Reading serial data)](/images/amanuensis-eeprom-interface/write_data.gif#center "500px")


## The Shield

The shield simply connects certain Arduino Mega pins with certain EEPROM pins. The EEPROM is delicately held during programming using a ZIF connector (Zero Insertion Force). It also has a couple of LEDs for visual feedback.

![(Closeup of the shield)](/images/amanuensis-eeprom-interface/shield.png#center)

![(Closeup of the shield)](/images/amanuensis-eeprom-interface/shield_mounted.jpeg#center "400px")

I ordered a custom PCB for my shield, but you can also make your own, as I did for the first prototype:

![(Closeup of the first shield)](/images/amanuensis-eeprom-interface/shield_prev.png#center "900px")

![(Visitor Count)](https://komarev.com/ghpvc/?username=amanuensis-hugo&style=pixel&label=VISITOR+COUNT)