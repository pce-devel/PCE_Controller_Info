# PCE_Controller_Info

Information about controller signalling on the PC Engine

## Electrical Characteristics

The PC Engine is based on CMOS 5V logic, so the voltage supply is 5V and the logic levels are set as such.

There are 6 active lines (besides power and ground) on the connector, with two being driven by
the console (SEL and CLR), and 4 being return signals from the joypad/peripheral.

The original purpose of SEL was as a multiplex selector for joypad inputs (which 4 inputs out of 8 to
read), and the purpose of CLR was to reset a port counter for the multitap... but over time, these
signal have been used for more complicated purposes on sophisticated peripherals (more on this in the
"Signalling Protocols" section below).

Electrically, the signals of SEL and CLR are generally pulled up by a 47K resistor on the peripheral
to prevent issues on the internals of the peripherals if power is supplied but CLR/SEL are floating;
this is not a likely case, but this is a protective measure.

Similarly, the controller inputs also use 47K resistors as pullups, and return signals from the
peripheral are generally sent back through current-limiting 300-ohm resistors in order to limit
the amount of current driven by the internal chips, in the event of an unexpected short circuit
on the console side of the controller.

Oscilloscope traces of the CLR and SEL signals into a multitap device as a load indicate that rise
time for these signals is on the order of 10ns, and fall time on the order of 20ns.


### Controller Schematics and Connector Pinout

This schematic shows the wiring diagram for a basic PC Engine controller. Schematics of additional
configurations can be found here:
https://console5.com/wiki/NEC_PC_Engine#Controller_Schematics

(Many thanks to console5.com for sourcing, preserving and improving console information like this.)

As can be seen in the schematic:
 1) All inputs default to high, unless they are specifically driven low
 2) The 74HC163 creates a divide-by-2/4/8 counter based on the CLR signal, which is then used by the
'turbo' switches to modulate auto-repeat keying.

![https://console5.com/wiki/File:PC-Engine---TG16-Controller-Schematic.png](https://console5.com/techwiki/images/9/9f/PC-Engine---TG16-Controller-Schematic.png)


### Power Consumption of Joypad Peripherals

Power consumption numbers are currently being confirmed, but are almost certainly lower than one might expect.

#### "Simple" power consumption:

This test was conducted by connecting a current-limiting test power supply to the devices, with the CLR and SEL
lines tied to ground (to prevent rapid state changes), and observing the digital readout on the power supply.
This is not a comprehensive test - rather, it represents the lower limit of current/power usage:

| Device | Observed Usage |
|:------:|:--------------:|
| 2-button joypad (NEC) | < 1 mA |
| 6-button joypad (Hori) | < 1 mA |
| 5-port Multitap (NEC) | < 1 mA |
| 5-port Multitap (NEC) | < 1 mA |
| PC Engine Mouse | 6 mA |
| XHE-3/XE-1AP | 33 mA |
| Memory Base 128 (NEC), fresh batteries | 0 mA |
| Memory Base 128 (NEC), no batteries | 30 mA |
| Save-kun (Koei), fresh batteries | 0 mA |
| Save-kun (Koei), no batteries | 10 mA |
| Replacement MB128 (my RP2040 design) | 25 mA |
| USB Mouse (my RP2040 design), no USB device | 30 mA |
| USB Mouse (my RP2040 design), Logitech Universal Receiver | 53 mA |


## Signalling Protocols - Overview

The program will scan the joypad controls with code similar to the below.

Notice that normally:
1) The CLR bit is set only briefly, once per scan across all joypads
2) SEL is kept HIGH for the initial read of joypad values, and toggled LOW to read the second group
3) There is normally a brief delay - although the size of this can vary - between changing the value
of the SEL line, and reading back return values from the joypad. This is above and beyond the ~680ns
delay (5 cycles) which would be implicit betwen the write cycle of a 'STA $1000' opcode and the read
cycle of an immediately-following 'LDA $1000' opcode, for a total of roughly 1.93 microseconds.

One further point which is not highlighted in the code below is that subsequent joypads are read
simply by toggling SEL high again to advance to the next joypad in the multitap. Toggling CLR resets
the sequence to the first port all over again.

Normal reading of joypads in this way is generally carried out as part of the VSYNC interrupt, or
as a direct result of its completion, so that values are always available, always 'fresh', and always
based on a regular timebase.

```
LDA #1     ; SEL High (Least-significant bit), CLR low (next-least significant bit)
STA $1000  ; output to joypad port
LDA #3     ; both SEL and CLR high
STA $1000
LDA #1     ; Keep SEL high; drive CLR low again
STA $1000
           ; the following instructions are a delay to allow transition of the joypad device
PHA        ; 3 cycles
PLA        ; 4 cycles
NOP        ; 2 cycles  total = 9 cycles, roughly 1.25 microseconds (CPU clock is normally 7.16 MHz)

LDA $1000  ; read from Joypad port
AND #$0F   ; strip the unnecessary bits

LDA #0     ; Keep SEL high; drive CLR low again
STA $1000
           ; the following instructions are a delay to allow transition of the joypad device
PHA        ; 3 cycles
PLA        ; 4 cycles
NOP        ; 2 cycles  total = 9 cycles, roughly 1.25 microseconds (CPU clock is normally 7.16 MHz)

LDA $1000  ; read from Joypad port
AND #$0F   ; strip the unnecessary bits
```

### 2-button Controller Protocol

A 2-button controller is keyed primarily on the SEL signal, and sends 4 key values when SEL is high,
and another 4 key values when SEL is low.

A 2-button controller doesn't care about the CLR line, but due to the way it is connected to the 74HC157,
all 4 outputs will be LOW when CLR is HIGH.

| SEL value | Bit Number | Key |
|:------:|:--------------:|:--------:|
| HIGH | bit 3 (ie $8) | LEFT |
| HIGH | bit 2 (ie $4) | DOWN |
| HIGH | bit 1 (ie $2) | RIGHT |
| HIGH | bit 0 (ie $1) | UP |
| LOW | bit 3 (ie $8) | RUN |
| LOW | bit 2 (ie $4) | SELECT |
| LOW | bit 1 (ie $2) | II |
| LOW | bit 0 (ie $1) | I |


### Multitap Protocol

Initially, multitap devices were built only for 2-button joypads, so subsequent devices which
were designed to be used with them needed to be built to accept the multitap design.

The multitap is essentially a multiplexer which takes CLR and SEL values from the console, and
creates CLR and SEL values on each of the ports; likewise, it takes the output values from the
active port and routes those back to the console.

One interesting point is that while the console only generates a short CLR pulse at thebeginning of a
joypad scan, the multitap presents it as a sustained high signal across all inputs except the active port.

| CLR | SEL | Active Port | Port 1 CLR | Port 1 SEL | Port 2 CLR | Port 2 SEL | Port 3 CLR | Port 3 SEL | Port 4 CLR | Port 4 SEL | Port 5 CLR | Port 5 SEL |
|-----|-----|-------------|------------|------------|------------|------------|------------|------------|------------|------------|------------|------------|
| L | H | None | H | H | H | H | H | H | H | H | H | H |
| H | H | None | H | H | H | H | H | H | H | H | H | H |
| L | H | 1 | L | H | H | H | H | H | H | H | H | H |
| L | L | 1 | L | L | H | H | H | H | H | H | H | H |
| L | H | 2 | H | H | L | H | H | H | H | H | H | H |
| L | L | 2 | H | H | L | L | H | H | H | H | H | H |
| L | H | 3 | H | H | H | H | L | H | H | H | H | H |
| L | L | 3 | H | H | H | H | L | L | H | H | H | H |
| L | H | 4 | H | H | H | H | H | H | L | H | H | H |
| L | L | 4 | H | H | H | H | H | H | L | L | H | H |
| L | H | 5 | H | H | H | H | H | H | H | H | L | H |
| L | L | 5 | H | H | H | H | H | H | H | H | L | L |

**Compatibilty:** All games are expected to be compatible with the Multitap.<br>
However, one game - Youkai Douchuuki - is known to be incompatible (it shows a black screen with no movement when multitap
is plugged in). There may be a 'fix' version of this HuCard available, as there is a version of the ROM which functions properly.


### 6-button Controller Protocol

When fighting games like Street Fighter II became popular at the arcades, a 6-button joypad was required
to support games like it on the PC Engine.

The signalling format is the same as the 2-button joypad, except that on alternating scans (inferred by
counting 'CLR' pulses), the direction buttons scan (SEL = HIGH) will show either the actual direction buttons,
or a value of '0000', which is effectively impossible (as it would imply that opposing directions are
simultaneously pressed, which is not possible on a standard controller). The additional buttons are sent
in the corresponding (SEL = LOW) scan in place of the original RUN/SELECT/II/I buttons.

Any game checking for the existence of a 6-button pad should scan twice, and not make any assumptions about
which scan number would yield the '0000' number.

| Scan number | SEL value | Bit Number | Key |
|:-----------:|:------:|:--------------:|:--------:|
| 1 | HIGH | bit 3 (ie $8) | LEFT |
| 1 | HIGH | bit 2 (ie $4) | DOWN |
| 1 | HIGH | bit 1 (ie $2) | RIGHT |
| 1 | HIGH | bit 0 (ie $1) | UP |
| 1 | LOW | bit 3 (ie $8) | RUN |
| 1 | LOW | bit 2 (ie $4) | SELECT |
| 1 | LOW | bit 1 (ie $2) | II |
| 1 | LOW | bit 0 (ie $1) | I |
| 2 | HIGH | bits 3-0  | value 0000 |
| 2 | LOW | bit 3 (ie $8) | VI |
| 2 | LOW | bit 2 (ie $4) | V |
| 2 | LOW | bit 1 (ie $2) | IV |
| 2 | LOW | bit 0 (ie $1) | III |

**Compatibility:** Games which are not specifically written to support 6-button controllers will not work properly.<br>
Compatible games include:
 - HuCard
   - Street Fighter II
 - CDROM
   - Advanced Variable Geo
   - Battlefield '94 in Tokyo Dome
   - Emerald Dragon
   - Fire Pro Jyoshi - Dome Choujyo Taisen
   - Flash Hiders
   - Garou Densetsu II - Aratanaru Tatakai
   - Garou Densetsu Special
   - Kakutou Haou Densetsu Algunos
   - Linda Cube
   - Mahjong Sword Princess Quest Gaiden
   - Martial Champions
   - Princess Maker 2
   - Ryuuko no Ken
   - Sotsugyou II - Neo Generation
   - Super Real Mahjong P II - P III Custom
   - Super Real Mahjong P V Custom
   - Tengai Makyo - Kabuki Itouryodan
   - World Heroes 2
   - Ys IV


## Special Controller Peripherals

The devices identified up to this point are easily implemented using simple discreet logic ICs with limited
statefulness.  The next group of devices rely on statefulness and have more complexity to their protocols.

### Mouse Protocol

Similar to the 6-button joypad, the mouse outputs different pieces of data across a series of multiple scans.

Timing is important on this protocol, as it must automatically reset so that the 'Delta X' scan is the first
value returned in a scan sequence; as a result, there are two timeouts:
1) If on any scan, the SEL line does not transition from HIGH to LOW within ~550 microseconds, the mouse will
time out, consider the scan completed, and reset the delta values until the next scan.  It can be assumed that
this is also true of LOW-to-HIGH transitions.
2) If the CLR line is not retriggered within a similar timeframe (for example, 600 microseconds), the mouse will
also consider the scan complete.

In-between scans, any X/Y movement is accumulated and presented as a net 'Delta X/Y' value at scan time.
There does not seem to be any specific requirements for scanning frequencies, other than the fact that a
subsequent scan should not appear within the timeout period for a preceding scan.

In order to detect a mouse, a game will generally scan a number of times, and verify that a '0000' value returns
in place of the UP/DOWN/LEFT/RIGHT keys; however, this has a danger of identifying a 6-button pad as a mouse.
However, within a large number of scans in a single frame, two sequential '0000' values sould be able to identify
a mouse.

| Scan number | SEL value | Data Returned |
|:-----------:|:------:|:--------:|
| 1 | HIGH | bits 7-4 of 'Delta X' |
| 1 | LOW  | buttons (RUN,SELECT,II,I) |
| 2 | HIGH | bits 3-0 of 'Delta X' |
| 2 | LOW  | buttons (RUN,SELECT,II,I) |
| 3 | HIGH | bits 7-4 of 'Delta Y' |
| 3 | LOW  | buttons (RUN,SELECT,II,I) |
| 4 | HIGH | bits 3-0 of 'Delta Y' |
| 4 | LOW  | buttons (RUN,SELECT,II,I) |

**Compatibility:** Games which are not specifically written to support the PC Engine Mouse will not work properly.<br>
Compatible games include:
 - HuCard : none
 - CDROM
   - 1552 Tenka Tairan
   - A. III - Takin' the A Train
   - Atlas Renaissance Voyage
   - Brandish
   - Dennou Tenshi Digital Angel
   - Doukyuusei
   - Eikan ha Kimini - Koukou Yakyuu Zenkoku Taikai
   - Hatsukoi Monogatari
   - Jantei Monogatari III - Saver Angels
   - Lemmings
   - Metal Angel
   - Nemurenumori no Chiisana Ohanashi
   - Power Golf 2 Golfer
   - Princess Maker 2
   - Tokimeki Memorial
   - Vasteel 2


### Pachinko Controller Protocol

The pachinko controller has two sections to it - all the standard buttons of a 2-button controller, plus a
spring-loaded dial control which acts like a paddle with approximately 90 degreess of rotational travel.  The
spring returns it to its counterclockwise limit, and the player is intended to rotate it clockwise to the
appropriate amount.

To the console, the pachinko controller appears like a multitap with a regular 2-button pad in port 1 (echoed
on ports 3 and 5); in port 2 (and echoed on port 4), the data should be interpreted in the following manner:
- Scan 'A' of the pad with SEL=HIGH will show with bits 3 and 1 as LOW (i.e. either $5, or $0 - but possibly $1 or $4).
- The rest of the data on scan 'A' is not meaningful; it may show as zeroes, or it may have the same values as scan 'B'
(except for those two bits).
- Scan 'B' of the pad with SEL=HIGH will definitely return bit 3 as HIGH (i.e. $80 to $FF).
- When taken as a whole, scan 'B' will show a digital value of the analog pachinko dial as follows:
  - SEL = HIGH retrieves the most-significant 4 bits, and SEL = LOW retrieves the least signifiacnt 4 bits
  - When taken as a byte, the value ranges between $FC (spring-returned counterclockwise limit) to $83 (at
full-range of clockwise motion).

**Compatibility:** Games which are not specifically written to support pachinko controllers may or may not work properly.<br>
Compatible games include:
 - HuCard
   - Pachiokun Juuban Shoubu
 - CDROM
   - Pachiokun Maboroshi no Densetsu
   - Pachiokun Warau Uchuu
   - Pachiokun III - Pachisuro & Pachinko


### Memory Base 128 Protocol

Memory Base 128 is a console (host)-driven protocol which resembles SPI to a certain degree, and covers both
NEC-made "Memory Base 128" units as well as KOEI-made "Save-Kun" units.

For Memory Base 128 protocol, the 'CLR' line is similar to the 'CLK' line of SPI, and when CLR transitions
from LOW to HIGH, the value of SEL is taken as a single bit of data. Under normal conditions, the joypad read
routines always set SEL to HIGH when CLR is set HIGH, so the data stream would normally be a series of '1' bits.

In order to transition from 'joypad' mode to 'MB128' mode, the Memory Base device monitors the joypad scanning bitstream
for a particular sequence of bits, and transitions to MB128 mode when it sees them.

Memory Base 128 Mode is initiated when it sees the sequence '00010101' arrive.  This corresponds to a value of $A8,
since the bitstream is least-significant-bit first (unlike SPI implementations elsewhere).

After the $A8 value is triggered, the Memory Base 128 will ignore the inputs of the daisy-chained device, and instead
communicate back to the console with:
- Bit 3 = No meaning. Zero on Save-Kun units.
- Bit 2 = Identification.  Transfers data back to the console only at predefined moments in the protocol; otherwise 0.
- Bit 1 = No meaning. Zero on Save-Kun units.
- Bit 0 = Data to be returned to the console

Note that all data is sent least-significant bit first (in both directions):

| Sequence | Number of Bits | Meaning |
|----------|----------------|---------|
| 1 | 2 | Send identification bit - echo the incoming bit back on bit 2 |
| 2 | 1 | Request type (0 = Write; 1 = Read) |
| 3 | 10 | Address (minimum granularity 128-byte offset within memory) |
| 4 | 20 | Transfer length in bits (can address 128KB of memory down to the exact number of bits) |
| 5 | (xfer len) | Data (to or from the console) |
| 6 | 0 or 2 | Trailing empty bits (**Only if a write command**). Data out is set to zero after the first bit is received  |
| 7 | 3 | Trailing empty bits. Data out is set to zero after the first bit is received  |

At the conclusion of a transfer, the Memory base 128 disengages and allows the daisy-chained controller outputs
to be returned to the console until the next $A8 sequence is received from the console.

**Compatibility:** Games which are not specifically written to support the Memory Base 128 should work properly.<br>
Compatible games include:
 - HuCard : None
 - CDROM
   - A. III - Takin' the A Train
   - Atlas Renaissance Voyage
   - Bishoujo Senshi Sailor Moon Collection
   - Brandish
   - Eikan ha Kimini - Koukou Yakyuu Zenkoku Taikai
   - Emerald Dragon
   - Fire Pro Jyoshi - Dome Choujyo Taisen
   - Ganchouhishi - Aoki Ookami to Shiroki Mejika
   - Linda Cube
   - Magicoal
   - Mahojng Sword Princess Quest Gaiden
   - Nobunaga no Yabou - Bushou Fuuunroku
   - Nobunaga no Yabou Zenkokuban
   - Popful Mail
   - Princess Maker 2
   - Private Eye Dol
   - Sankokushi III
   - Shin Megami Tensei
   - Super Mahjong Taikai
   - Super Real Mahjong P II - P III Custom
   - Super Real Mahjong P V Custom
   - Tadaima Yuusha Boshuuchuu
   - Vasteel 2


### X-HE3 Converter for XE-1AP Analog Controller

The X-HE3 converter connects a XE-1AP controller (originally made for MSX and X68000 computers) to the PC Engine,
in both digital and analog formats. In digital format, the controller acts like a regular 2-button joypad.
In analog form, the controller has additional inputs for games which specifically support the analog pad.

For other joypad peripherals, the CLR line ls LOW under normal circumstances, and a pulse HIGH indicates either:
 a) A reset of the sequence, followed immediately by a low signal again, or
 b) Other data to be transmitted on the SEL line at the time of transition, like SPI protocol (MB128)

Normally, when CLR is HIGH, the return data is not expected to be valid; in most cases, CLR is tied to a 74HC157
on the /OE line, which will send all '0000' data back until CLR transitions low again; this is not the case for the
X-HE3/XE-1AP combination (in analog mode).

In this XE-1AP's analog case, the CLR signal is held high except for a brief pulse low, which signals the joystick
to gneerate a pulse train back to the PC Engine, which must be monitored closely for transitions (which are driven
by an internal state machine on the joypad, not the PC Engine).

During this time, SEL is held LOW as the PC Engine is expected to watch for state transitions on the buttons 'I' and 'II',
which are on D0 and D1 respectively.  D0 is LOW and D1 is HIGH during the waiting period.  After roughly 68.4 microseconds,
data is ready, which is signalled by D1 (button 'II') transitioning low.

When the console sees this, its duty is to transition SEL to HIGH, in order to read the four data bits on D0-D3
(which normally correspond to UP/DOWN/LEFT/RIGHT), which in this situation correspond to the first nybble of data
from the joystick.  Then, SEL needs to transition back to LOW in order to monitor D0 and D1 again.  The window of data
availability (when button 'II' stays LOW) is fairly short, roughly 12.1 microseconds, requiring a tight loop on the
host console.

Before the next nybble is made available, button 'II' transitions HIGH for roughly 3.8 microseconds while data lines
transition.  Button 'I' transitions HIGH, representing the second nybble - and button 'II' transitions LOW again (for
12.1 microseconds again), signalling data ready, and the console needs to fetch the nybble by toggling SEL and reading
the lines during the availability window.

When button 'II' toggles high again, this time it stays high for longer (to prepare for the next 2 nybbles), at
21.8 microseconds.  The pattern repeats, with 6 bytes (12 nybbles) in all being transferred; the entire time from the
initial pulse to the end of the data transfer is roughly 350 microseconds.

#### Data sequence output:

The sequence of data (and bit-order) is as follows (from original joystick):

All values are low when pressed, high when not pressed

 1. Buttons A, B, C, D - (Note: A is pressed if either A or A' is pressed; same with B or B')
 2. Buttons E1, E2, Start(F), Select (G)
 3. Top 4 bits of 'channel 0' (Y-axis;   limit up   = 0x00, limit down  = 0xFF)
 4. Top 4 bits of 'channel 1' (X-axis;   limit left = 0x00, limit right = 0xFF)
 5. Top 4 bits of 'channel 2' (Throttle; limit up   = 0xFF, limit down  = 0x00)
 6. 0000 (unused)
 7. Bottom 4 bits of 'channel 0' (Y-axis)
 8. Bottom 4 bits of 'channel 1' (X-axis)
 9. Bottom 4 bits of 'channel 2' (Throttle)
10. 0000 (unused)
11. Buttons A, B, A', B' (This can differentiate between the buttons, whereas scan #1 merges them)
12. 1111 (all high)


#### Logic state diagram:

![/images/XHE-3_protocol.png](/images/XHE-3_protocol.png)



**Compatibility:** Games which are not specifically written to support the X-HE3/XE-1AP in analog format will not work properly,
and should be used in digital format instead.<br>
Compatible games include:
 - AfterBurner II
 - Forgotten Worlds
 - Operation Wolf
 - Outrun
 - Thunder Blade (possibly)


## To Be Added:

### Develo Box Protocol

The Develo Box is intended to interface with a PC, so there is communications software running on the PC side
to handle the more complicated communications this involves.

This deserves its own file, which will follow in the near future.

