# Syro FMUv6X base-board bring-up tests

`i2cdetect` shows nothing on bus 1 (GPS1, POWER1) and bus 3 (base EEPROM U16, MS5611), while
buses 2 and 4 work. The firmware pin mapping was checked against the netlist and matches, so
these tests look for the fault in the hardware. Run them in order, cheapest first, and stop at
the first one that fails.

## Before you start

- Props off. Power from USB or a bench supply.
- `gpio write` keeps a pin as a GPIO until the next `reboot`. Reboot after every test.
- I2C2 and I2C4 carry live sensors (BMP581, BMM350, IMU EEPROM). Do not drive their pins.
- Meter: black probe on GND, red probe on the pin named.

| Part | Pins used below |
| --- | --- |
| U16 base EEPROM (24LC64, DFN-8) | 4 GND, 5 SDA, 6 SCL, 8 3V3 |
| J4 GPS1 (Holybro M10) | 1 5V, 4 SCL, 5 SDA, 8 3V3, 9 buzzer, 10 GND |
| J16 POWER1, J17 POWER2 | 3 SCL, 4 SDA |

## 1. Rails (multimeter)

| Measure | Expect | Reading |
| --- | --- | --- |
| U16 pin 8 to pin 4 | 3.3 V | |
| J4 pin 8 to pin 10 | 3.3 V | |
| J4 pin 1 to pin 10 | 5 V | |

Missing 3.3 V on the base would explain U16, the MS5611 and the GPS LED chip together. Fix it
before going on.

## 2. Buzzer

```
tune_control play -t 1
```

The M10's buzzer should play the startup tune if J4 pins 1, 9 and 10 are good.

## 3. I2C3 continuity (U16)

Right after a fresh reboot, drive each line from the MCU and measure at U16. Low reads ~0 V,
released reads ~3.3 V.

```
gpio write A8 0 OPENDRAIN --force   # PA8 = I2C3_SCL, U16 pin 6
gpio write A8 1 OPENDRAIN --force
gpio write H8 0 OPENDRAIN --force   # PH8 = I2C3_SDA, U16 pin 5
gpio write H8 1 OPENDRAIN --force
reboot
```

If both lines follow and U16 has 3.3 V but it still does not ACK, suspect the solder joints of
the leadless DFN-8. Reflow or replace it.

## 4. I2C1 continuity (GPS1, POWER1)

Same method. Stop the power monitor first, and `i2c_launcher stop` too if it is running.

```
ina228 stop
gpio write B8 0 OPENDRAIN --force   # PB8 = I2C1_SCL, J4 pin 4 and J16 pin 3
gpio write B8 1 OPENDRAIN --force
gpio write B9 0 OPENDRAIN --force   # PB9 = I2C1_SDA, J4 pin 5 and J16 pin 4
gpio write B9 1 OPENDRAIN --force
reboot
```

## 5. Power module swap

Plug the power module into J17 (POWER2), reboot, and run `i2cdetect -b 2`. If 0x45 shows up
next to the BMP581 at 0x46, the power module and its harness are fine and the I2C1 path is the
problem.

## Reading the results

| Result | Meaning |
| --- | --- |
| 3.3 V or 5 V missing | Supply path to the base. Fix first. |
| A line does not follow | Open or short on that net between the MCU and the measured pin. |
| Lines follow, 3.3 V present, still no ACK | Solder joints of U16 (DFN-8). |
| Module answers on J17 but not on J16 | I2C1 path: J16 and J4 wiring, PB8 / PB9. |
| Module is silent on J17 too | Power module or its harness: connector, SCL / SDA order. |

Healthy end state:

```
i2cdetect -b 1   # 0x0E (M10 compass), 0x45 (INA228), GPS LED chip
i2cdetect -b 3   # 0x51 (U16), 0x76 or 0x77 (MS5611)
```
