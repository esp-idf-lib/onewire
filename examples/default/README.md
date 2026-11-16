# 1-Wire bus example

## What it does

The example puts one DS18B20 on a 1-Wire bus, reads its temperature once a
second and prints it. The bus is bit-banged by the `onewire` component, the
sensor is driven by the `ds18x20` component, so the example doubles as a
demonstration of how a device driver uses the bus.

The bus pin and the ROM address of the sensor are taken from `menuconfig`; the
address is the part that has to be changed, because it is unique to every
sensor.

## Requirements

* An ESP32 series board and a DS18B20 (or a DS18S20, a DS1822, a DS1825 or a
  MAX31850 — anything `ds18x20` understands).
* A pull-up resistor from DQ to 3.3 V. The driver enables the internal pull-up of
  the pin, which is enough for one sensor on short leads, but a 4.7 kΩ external
  resistor is what the datasheet asks for and what makes it reliable.

## Wiring

| Signal | Default pin | Where it comes from |
| --- | --- | --- |
| DQ | 17 | `CONFIG_EXAMPLE_ONEWIRE_GPIO` |
| VDD | 3.3 V | — |
| GND | GND | — |

The default depends on the target: 17 on ESP32, ESP32-S2 and ESP32-S3, 4 on
ESP32-C2, C3, C5, C6, C61, H2 and P4.

A DS18B20 in the 3-pin package is powered from VDD and GND. The 2-pin version
is powered from the bus itself (parasitic power) and needs a strong pull-up
during the conversion; the `ds18x20` driver handles that when it is asked to
wait for the conversion, but the external transistor of the datasheet is still
the reliable way to do it.

## Configure, build, flash

```sh
idf.py set-target esp32
idf.py menuconfig
```

Under `Example configuration`:

* `GPIO Number of 1-Wire bus, or DQ` — the pin the sensor hangs on
* `Address of the sensor` — the 64-bit ROM code of your sensor

The ROM code printed by the scanner in the `ds18x20` component's `default`
example is exactly the value to paste here. A wrong address means no answer,
not a wrong temperature.

Then:

```sh
idf.py build flash monitor
```

## Example output

```
I (1234) ds18x20_test: Sensor 27041685c771ff28: 22.50°C
I (2234) ds18x20_test: Sensor 27041685c771ff28: 22.44°C
```

If the sensor does not answer, the error is printed with its name, for example
`ESP_ERR_INVALID_CRC` (the ROM code or the wiring is wrong) or
`ESP_ERR_INVALID_RESPONSE` (the sensor did not convert, which happens to a
parasitically powered one without a strong pull-up).

## Notes

* The address is printed as eight bytes in the order the bus sends them, so the
  first two characters are the family code (`28` for a DS18B20).
* Only one sensor is read here to keep the example short. A bus with several
  sensors is the `multi` example of the `ds18x20` component.
* The internal pull-up is weak (~45 kΩ). With longer wires, several sensors or
  parasitic power, "random" CRC errors are almost always the pull-up and the
  wiring, not the driver.
