# PCE_Controller_Info

Information about controller signalling on the PC Engine

## Electrical

The PC Engine is based on CMOS 5V logic, so the voltage supply is 5V and the logic levels are set as such.

There are 6 active lines (besides power and ground) on the connector, with two being driven by
the console (SEL and CLR), and 4 being return signals from the joypad/peripheral.

Electrically, the signals of SEL and CLR are generally pulled up by a 47K resistor on the peripheral
to prevent issues on the internals of the peripherals if power is supplied but CLR/SEL arr floating;
this is not a likely case, but this is a protective measure.

Similarly, the return signals from the peripheral are gnereally sent back through current-limiting
300-ohm resistors in order to limit the amount of current driven by the internal chips, in the event
of an unexpected short circuit on the console side of the controller.

### 2-button Controller Protocol

### Multitap Protocol

### 6-button Controller Protocol

## Special Controller peripherals

### Mouse Protocol Protocol

### Pachinko Controller Protocol

### Memory Base 128 Protocol

### Develo Box Protocol

