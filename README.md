# PCE_Controller_Info

Information about controller signalling on the PC Engine

## Electrical

The PC Engine is based on CMOS 5V logic, so the voltage supply is 5V and the logic levels are set as such.

There are 6 active lines (besides power and ground) on the connector, with two being driven by
the console (SEL and CLR), and 4 being return signals from the joypad/peripheral.

Electrically, the signals of SEL and CLR are generally pulled up by a 47K resistor on the peripheral
to prevent issues on the internals of the peripherals if power is supplied but CLR/SEL arr floating;
this is not a likely case, but this is a protective measure.

Similarly, the controller inputs also use 47K resistors as pullups, and return signals from the
peripheral are gnereally sent back through current-limiting 300-ohm resistors in order to limit
the amount of current driven by the internal chips, in the event of an unexpected short circuit
on the console side of the controller.

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


## Signalling Protocols - Overview

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
| LOW | bit 3 (ie $8) | 'run' |
| LOW | bit 2 (ie $4) | 'select' |
| LOW | bit 1 (ie $2) | II |
| LOW | bit 0 (ie $1) | I |


### Multitap Protocol

### 6-button Controller Protocol

## Special Controller peripherals

### Mouse Protocol

### Pachinko Controller Protocol

### Memory Base 128 Protocol

### Develo Box Protocol

