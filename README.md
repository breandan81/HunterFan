# HunterFan

Arduino library for controlling Hunter ceiling fans (and similar 433 MHz OOK
remotes) from ESP8266, ESP32, or AVR boards. Supports both transmit and
receive — you can capture a packet from the OEM remote and replay it
byte-for-byte.

## Hardware

You need a **433 MHz OOK transmitter** (e.g. SYN115, FS1000A). If you
also want to capture your OEM remote's codes, add a matching
**433 MHz OOK receiver** (e.g. SYN480R). Both are cheap on Amazon /
AliExpress.

Wiring is the same on any board — only the pin names change:

| Signal              | ESP8266 NodeMCU | Arduino Uno | Notes                          |
|---------------------|-----------------|-------------|--------------------------------|
| TX module **DATA**  | D1 (GPIO5)      | Pin 10      | any GPIO                       |
| RX module **DATA**  | D2 (GPIO4)      | Pin 2       | **must be interrupt-capable**  |
| TX/RX **VCC**       | VU (5V) or 3V3  | 5V          | TX range improves at 5V        |
| TX/RX **GND**       | GND             | GND         |                                |
| TX module antenna   | —               | —           | **solder a 17.3 cm wire**      |

A few gotchas the modules don't tell you:

- **The TX module is useless without an antenna.** Solder a 17.3 cm
  (¼-wavelength of 433 MHz) wire to the ANT pad. Without it you'll
  get maybe 10 cm of range and waste hours blaming software.
- **The RX module wants 5V** for best sensitivity. The data pin is
  ~5V logic but the SYN480R will work into a 3.3V MCU input on a
  short bus. If you see garbage, use a divider or a 3.3V receiver.
- **Interrupt-capable RX pin.** On AVR Uno that's pin 2 or 3
  (`INT0`/`INT1`). On ESP8266/ESP32 every GPIO except the strapping
  pins is fine.

On AVR, the only relevant in-band ISR (Timer0, ~1 µs) is well inside
the library's ±45 µs tolerance, so you can leave `millis()`
interrupts enabled. On ESP8266/ESP32, see `disableInterruptsDuringTx`
in `HunterFan.h` if you need to suppress WiFi jitter during transmits.

## Install

**Arduino IDE.** Clone this repo (or download the ZIP) into your
sketchbook's `libraries/` folder:

- Linux: `~/Arduino/libraries/HunterFan`
- macOS: `~/Documents/Arduino/libraries/HunterFan`
- Windows: `Documents\Arduino\libraries\HunterFan`

Restart the IDE; the examples will appear under
**File → Examples → HunterFan**.

**PlatformIO.** Add to `platformio.ini`:

```ini
lib_deps =
    https://github.com/breandan/HunterFan.git
```

## Capture your remote's codes

The codes in the examples are paired to *my* fan — they won't do
anything to yours. The first thing you'll want to do is capture your
own remote's buttons:

1. Wire up a 433 MHz receiver as in the table above.
2. Flash `examples/ESP8266_RX` (or `Uno_RX`).
3. Open the serial monitor at 115200 baud.
4. Point the OEM remote at the receiver and press a button.
5. The sketch prints something like:
   ```
   #1  RX 66b  A6FF346CBB18067F80
   ```
6. Note the bit count (often 65 or 66 — varies by remote model) and
   the hex string. That's the complete packet for that button. Repeat
   for every button you want to control.

To replay, paste the hex into a TX sketch and call:

```cpp
fan.sendHex("A6FF346CBB18067F80", 66);
```

That's the whole workflow — the library doesn't need to know what
each button means, it just transmits exactly what it received.

## Quick start — transmit

```cpp
#include <HunterFan.h>

HunterFan fan(/* txPin */ 12);   // GPIO12 = D6 on NodeMCU

void setup() {
    fan.begin();
    // 66-bit "light toggle" command captured from a Hunter remote
    fan.sendHex("A6FF346CBB18067F80", 66);
}

void loop() {}
```

## Quick start — receive (learn a button)

```cpp
#include <HunterFan.h>

HunterFan fan(/* txPin */ 12, /* rxPin */ 13);

void setup() {
    Serial.begin(115200);
    fan.begin();
}

void loop() {
    uint8_t buf[16], bits;
    if (fan.receive(buf, sizeof(buf), bits, /* timeoutMs */ 5000)) {
        Serial.printf("Got %u bits: %s\n", bits, HunterFan::toHex(buf, (bits + 7) / 8));
    }
}
```

The `toHex` output and `sendHex` input use the same format, so a capture
can be replayed verbatim.

## Protocol

OOK at 433 MHz, T = 400 µs nominal bit clock.

- **Preamble:** ~60 alternating HIGH/LOW pulses (1T each) to settle the
  receiver's AGC.
- **Anchor gap:** ~13T of LOW that marks the boundary between preamble
  and data.
- **Data:** 65–66 bits, each a PWM symbol over 3T:
  - `1` → 2T HIGH, 1T LOW
  - `0` → 1T HIGH, 2T LOW
- **Repeat:** the whole frame is sent 3× with a 26 ms gap.

All timing constants (`clkUs`, `preamblePairs`, `anchorClocks`, `repeats`,
…) are public fields and can be overridden before `begin()` to match
similar OOK remotes. See `HunterFan.h`.

## Examples

- `examples/HunterFanBasic` — combined TX+RX example, builds on ESP8266 or Uno.
- `examples/ESP8266_TX`, `examples/ESP8266_RX` — ESP8266 sender/receiver.
- `examples/Uno_TX`, `examples/Uno_RX` — same on an Arduino Uno.

## License

MIT — see `LICENSE`.
