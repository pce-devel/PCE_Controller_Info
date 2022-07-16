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

This schematic (courtesy of console5.com) shows the wiring diagram for a basic PC Engine controller.
Schematics of additional configurations can be found here:
https://console5.com/wiki/NEC_PC_Engine#Controller_Schematics

As can be seen in the schematic:
 1) All inputs default to high, unless they are specifically driven low
 2) The 74HC163 creates a divide-by-n counter based on the CLR signal, which is then used by the
'turbo' switches to modulate auto-repeat keying.

![https://console5.com/wiki/File:PC-Engine---TG16-Controller-Schematic.png](https://console5.com/wiki/File:PC-Engine---TG16-Controller-Schematic.png)


## Signalling Protocols - Overview

```
LDA #1     ; SEL High (Least-significant bit), CLR low (next-least significant bit)
STA $1000  ; output to joypad port
LDA #3     ; both SEL and CLR high
STA $1000
LDA #1     ; Keep SEL high; drive CLR low again
STA $1000
           ; the following instructions are a delay to allow transition of the joypad device
PSHA       ; ? cycles
PULA       ; ? cycles
NOP        ; ? cycles  total = ? cycles, roughly ? microseconds
```

### 2-button Controller Protocol

A 2-button controller is keyed primarily on the SEL signal.

### Multitap Protocol

### 6-button Controller Protocol

## Special Controller peripherals

### Mouse Protocol Protocol

### Pachinko Controller Protocol

### Memory Base 128 Protocol

### Develo Box Protocol

