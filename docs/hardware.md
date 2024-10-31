# LED Blinker: Hardware Requirements

Users will need two hardware elements to follow the LED Blinker tutorial: an embedded ARM Linux computer, and an LED capable of withstanding the operating voltage of the computer.

## Embedded Computer Requirements

The embedded computer must be an ARM Linux machine with little-endian architecture and available GPIO pins. Some platforms that should work include: the RaspberryPI, Odroids, the BeagleBone, and similar platforms. The user is expected to set up Linux running Kernel 4.8+ or newer and enable the GPIO pins in the platform configuration.

> Write down the ARM architecture bus-width: 32-bit, or 64-bit.

## LED Requirements

The user may use any LED that can withstand the GPIO voltage of the chosen platform (typically 3.3 or 5 volts). This usually means choosing an LED with sufficient diode drop or inlining a resistor.

## Wiring Diagram

For this tutorial, GPIO pin 13 will be used. For platforms that do not have GPIO pin 13 readily available another pin should be chosen, noted, and used in-place of GPIO 13.

```
GPIO 13 ----> LED + (cathode) 
GND     <---- LED - (anode)     
```
