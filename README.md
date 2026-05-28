# HunterFan

Arduino library for controlling Hunter ceiling fans (and similar 433 MHz OOK
remotes) from ESP8266, ESP32, or AVR boards. Supports both transmit and
receive — you can capture a packet from the OEM remote and replay it
byte-for-byte.

## Hardware

- A 433 MHz OOK transmitter (e.g. SYN115, FS1000A) wired to any GPIO.
- *(Optional)* A 433 MHz OOK receiver (e.g. SYN480R) wired to an
  interrupt-capable GPIO. Skip this if you only want to transmit.

```
MCU GPIO ──► TX module DATA          (transmit)
MCU GPIO ◄── RX module DATA          (receive)
```

## Install

Drop this directory into your Arduino `libraries/` folder, or symlink it.
PlatformIO users can add it to `lib_deps` as a Git URL.

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

- `examples/HunterFanBasic` — minimal TX example.
- `examples/ESP8266_TX`, `examples/ESP8266_RX` — ESP8266 sender/receiver.
- `examples/Uno_TX`, `examples/Uno_RX` — same on an Arduino Uno.

## License

MIT — see `LICENSE`.
