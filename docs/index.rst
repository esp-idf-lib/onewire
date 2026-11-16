onewire: Bit-banging 1-Wire driver
==================================

The Dallas 1-Wire bus needs one GPIO and one pull-up resistor, and no
peripheral at all: every pulse on the line is produced by the CPU. This
component is such a bit-banging master — it resets the bus, writes and reads
bytes, selects devices by their ROM address, enumerates the bus and computes
the CRCs the protocol uses. Devices such as the DS18B20 are driven on top of it
(see `ds18x20`), and it runs on every ESP32 series chip and on the ESP8266.

Features
--------

* Reset and presence detection, byte and bit transfers, ROM select and skip ROM.
* Bus enumeration with the standard search algorithm, with or without a family
  code filter, so a bus may hold devices of several kinds.
* Strong pull-up for parasitically powered devices: ``onewire_power()`` drives
  the line high with the pin itself, ``onewire_depower()`` gives it back to the
  pull-up resistor, and a reset does the same automatically.
* Dallas CRC8 for ROM codes and scratchpads, and CRC16 with the inverted form
  that devices send.
* No hardware peripheral, no interrupt handlers, no task: everything runs in the
  caller's context.

Wiring
------

.. list-table::
   :header-rows: 1

   * - Signal
     - Connection
   * - DQ
     - Any GPIO, one pin per bus
   * - VDD, GND
     - Power and ground of the devices
   * - Pull-up
     - 4.7 kΩ from DQ to 3.3 V

The driver configures the pin as an open-drain output with the internal pull-up
enabled, so on a breadboard with a short bus nothing else is needed: the
internal pull-up of an ESP32, roughly 45 kΩ, is enough for a couple of devices
and a few centimetres of wire. That is not the 4.7 kΩ the datasheets ask for,
though, and it becomes unreliable with longer wires, more devices or parasitic
power. For anything beyond a quick test, add the external resistor.

A bus is not a tree: the pull-up sits at one end, the devices hang off the wire,
and long stubs make the edges ring. A star with four sensors on one pin is how
"random" CRC errors are usually made.

Timing
------

The protocol is timing-critical, and the timings are busy-waits: a bit takes
about 70 µs, a byte about 560 µs, a reset cycle about 1 ms. Enumeration costs
roughly 1 ms per device and ROM bit, so a bus with several sensors takes a few
milliseconds to scan.

Each bit is transferred inside a critical section, so interrupts are disabled
for up to about 80 µs, not for a whole byte: other drivers keep working, but
their handlers should stay short. The functions must not be called from an
interrupt handler themselves.

The component is not thread-safe. Call it from one task, or take your own mutex
around every exchange with the bus: the library has no lock of its own, because
a lock cannot protect the critical sections that the protocol needs.

ROM address
-----------

Every device has a 64-bit ROM code, which the driver keeps in a
``onewire_addr_t``. The least significant byte is the family code: ``0x28`` for
the DS18B20, ``0x10`` for the DS18S20, ``0x3B`` for the MAX31850, and so on. Two
devices of the same kind can share a bus, so the ROM code, not the family,
identifies an address.

``ONEWIRE_NONE`` is a value that cannot be a valid ROM code (its CRC does not
match) and is returned by ``onewire_search_next()`` when there is nothing more
to find.

CRC8 and espressif/onewire_bus
------------------------------

``espressif/onewire_bus`` exports a function called ``onewire_crc8()`` as well,
with a different signature, so the two components cannot both link their symbol
into one binary. The implementation here is therefore exported as
``onewire_crc8_dallas()``, and ``onewire_crc8()`` is an inline wrapper that keeps
the old name working for existing code.

Code that calls the Espressif function on purpose has to include
``onewire_crc.h`` and call ``onewire_crc8(0, data, len)`` with its three
arguments; the wrapper of this component takes two.

Configuration
-------------

.. list-table::
   :header-rows: 1

   * - Option
     - Default
     - Meaning
   * - ``CONFIG_ONEWIRE_CRC8_TABLE``
     - ``y``
     - Compute CRC8 from a table in flash instead of bit by bit

Example
-------

``examples/default`` is a complete 1-Wire application: it reads a DS18B20 through
the ``ds18x20`` driver, with the bus pin and the sensor address taken from
``menuconfig``. It doubles as a demonstration of how this component is used from
a device driver.

API reference
-------------

 .. doxygenfile:: onewire.h

