# Analysis of the thermostat bus of an Uponor Smatrix Base Pulse underfloor heating control system

Devices under test:
* 1x Uponor Smatrix Base Pulse Controller X-245
* 5x Uponor Smatrix Base Thermostat T-148


# Example packets

    11 0B DE 62 FF D3 3D 
    11 0B DE 62 40 02 CD 3E 00 00 3F 0C 03 42 96 30 3D 5D - Bathroom 22.0°C 48%RH
    11 0B DE 62 2D 7F FF 3D 00 00 0C 00 24 37 01 9A 38 03 B6 3B 02 85 3C 00 48 35 80 00 08 0B D5 09 E5 95 0A 00 1E 38 07 

    11 0B DD FF FF 4B AD
    11 0B DD FF 40 02 A8 3E 00 00 3F 0C 03 42 96 33 BF ED - Bedroom 20.0°C 51%RH
    11 0B DD FF 2D 7F FF 3D 00 00 0C 00 24 37 01 9A 38 03 B6 3B 02 84 3C 00 48 35 80 00 08 0B D5 09 E5 95 0A 00 1E BC B6 

    11 0B DE 72 FF DE FD 
    11 0B DE 72 40 02 BE 3E 00 00 3F 0C 03 42 96 32 70 28 - Office 21.2°C 50%RH
    11 0B DE 72 2D 7F FF 3D 00 00 0C 00 24 37 01 9A 38 03 B6 3B 02 85 3C 00 48 35 80 00 08 0B D5 09 E5 95 0A 00 1E 3F 97 

    11 0B DE 4A FF CD 3D 
    11 0B DE 4A 40 02 B6 3E 00 00 3F 0C 03 42 96 33 6E BA - Guest Bathroom 20.7°C 51%RH
    11 0B DE 4A 2D 7F FF 3D 00 00 0C 00 24 37 01 9A 38 03 B6 3B 02 85 3C 00 48 35 80 00 08 0B D5 09 E5 95 0A 00 1F F6 4F 

    11 0B DE 13 FF F7 6D 
    11 0B DE 13 40 02 B7 3E 00 00 3F 0C 03 42 96 33 08 0B D5 09 E5 95 0A 00 1F FD 31 - Living Room 20.8°C 51%RH
    11 0B DE 13 2D 7F FF 3D 00 04 0C 00 24 37 01 9A 38 03 B6 3B 02 7C 3C 00 48 35 80 00 4B 0E 


# Communication description

## Serial communication
RS485, 19200, 8N1


## Packet layout
The first four bytes contain address information.
Bytes 1 to 2 seem to contain a system ID (source address) and bytes 3 to 4 contain the thermostat ID (target address).
It is unclear if the system ID encodes information like the controller type.

This is followed by a variable number of payload bytes. The length of the payload is always divisable by 3 (except for the data request).
Each 3 byte section starts with a register number (or command byte) followed by two bytes of data value.

A modbus-like CRC16 checksum is added to the end of each packet.

All data field in the package are big-endian except for the little-endian checksum.


## Communication sequence
The controller starts a new communication cycle exactly every 10 seconds.

At the beginning of each communication cycle, a request packet (payload FF) is sent by the controller to the first known thermostat.
The thermostat responds with a set of values, after which the controller sends an update packet containing additional values the thermostat did not provide by itself (e.g. current date and time).
After these three packets (request, response, update), the controller sends a request to the next known thermostat until all known thermostats have been queried.


## Registers / commands
| ID | Description                                        | Format                | Data / Example
| -- | ---------------------------------------------------| --------------------- | --------------
| FF | Data request                                       | N/A                   | Only field with no value
| 08 | Date/Time Part 1 (Year, Month, Day of Week)        |                       | See example below
| 09 | Date/Time Part 2 (Day of Month, Hour, Minute)      |                       | See example below
| 0A | Date/Time Part 3 (Seconds)                         |                       | See example below
| 0C | Unknown                                            | Bitfield?             | Observed values: 0x0342, 0x0024
| 2D | Unknown, maybe Outdoor Temperature from controller | Absolute Temperature? | 0x7FFF = No data available
| 35 | Unknown                                            | Bitfield?             | 0x8000 = Heat pump defrost off?
| 37 | Room Temperature Setpoint Minimum                  | Absolute Temperature  | 0x019A = 410 = 41°F = 5°C
| 38 | Room Temperature Setpoint Maximum                  | Absolute Temperature  | 0x03B6 = 950 = 95°F = 35°C
| 3B | Room Temperature Setpoint                          | Absolute Temperature  | 0x0284 = 644 = 64.4°F = 18°C
| 3C | ECO setback value                                  | Relative Temperature  | 0x0048 = 72 = 4°C
| 3D | Heating demand                                     | Bitfield              | 0x1000 = Cooling (0=Heating, 1=Cooling)?
|    |                                                    |                       | 0x0040 = Demanding
| 3E | Operating mode                                     |                       | 0x0002 = ? (seen when changing Control Mode to RFT)
|    |                                                    |                       | 0x0008 = ECO State (1=Active)
|    |                                                    |                       | 0x4000 = Program Schedule State (0=Prg Off, 1=Prg Selected)
| 3F | Heating/Cooling state                              | Bitfield              | 0x0100 = Control Mode Bit 1 ? (set if Control Mode is RFT or RS)
|    |                                                    |                       | 0x0200 = Control Mode Bit 2 ? (set if Control Mode is RS or R0)
|    |                                                    |                       | 0x0400 = Cooling Allowed
|    |                                                    |                       | 0x0800 = Heating Allowed
|    |                                                    |                       | 0x1000 = Cooling State (0=Heating, 1=Cooling)
|    |                                                    |                       | Control Modes ?:
|    |                                                    |                       |   0 = Room Temperature (RT)
|    |                                                    |                       |   1 = Room and Floor Temperature (RFT)
|    |                                                    |                       |   2 = Room and Outdoor Sensor (RO)
|    |                                                    |                       |   3 = Remote Sensor (RS)
| 40 | Room Temperature                                   | Absolute Temperature  | 0x02B7 = 695 = 69.5°F = 20.8°C
| 41 | External Sensor Temperature (Floor/Outdoor)        | Absolute Temperature  | 0x7FFF = No data available
| 42 | Humidity                                           | Humidity              | 0x9635 = 53%


## Formats / Value decoding

###  Absolute Temperature
    celsius = ((fahrenheit / 10) - 32) / 1.8

### Relative Temperature
    celsius = (value / 10) / 1.8

### Humidity
First byte is unknown (seen values: B300 / 96xx). Maybe status/alarm bits.
Second byte contains humidity in percent.

### Date/Time
Registers 08, 09 and 0A need to be combined for full date/time.

Example:

    Data: 08 0B D5 09 E5 EB 0A 00 3B
    0BD5 = 000010111 1010 101
                  23   10   5
                  yy   mm   w
    E5EB = 11100 10111 101011
              28    23     43
              dd    HH     MM
    003B = 00111011
                 59
                 SS
    => Saturday, 2023-10-28 23:43:59

    w = Day of Week
    0 = Monday
    1 = Tuesday
    2 = Wednesday
    3 = Thursday
    4 = Friday
    5 = Saturday
    6 = Sunday


## Possible command injections

### Set Temperature
For unknown reasons, we need to send a null setpoint first for the thermostat to react

Set temperature setpoint to 20.0°C:

    11 0B DE 72 3B 00 00 3B 02 A8 F8 28

### Set Time
The first thermostat paired to the controller will act as the time master. Time can only be manually adjusted at this first thermostat.
Changing the time can only be done by faking a packet coming from this thermostat.

    11 0B DE 13 08 0B D6 09 EB 14 0A 00 00 27 52
    11 0B DE 13 08 0B D0 09 F0 11 0A 00 00 25 9C
