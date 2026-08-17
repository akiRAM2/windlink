# WindLink

**English** | [日本語](README.ja.md)

WindLink is an experimental project that turns a VRChat avatar's movement speed into real-time airflow. It receives VRChat OSC data on a Windows PC, maps movement speed to a fan target, and controls a 12 V four-wire PWM PC fan through a USB-connected ESP32.

![WindLink prototype with a 120 mm PWM fan, ESP32 breadboard circuit, and 12 V power supply](assets/windlink-prototype.jpg)

*The tested WindLink prototype: 120 mm PWM fan, ESP32 breadboard controller, and 12 V power supply.*

> [!NOTE]
> The source code, web UI, and documentation in this project were created and organized with assistance from generative AI. A human tested the wiring and core behavior on physical hardware. Review the implementation and verify all electrical details for your own components before use. See [AI-assisted development](#ai-assisted-development).

```text
VRChat
  │ OSC / UDP 127.0.0.1:9001
  ▼
Python OSC monitor and HTTP server
  │ JSON / HTTP 127.0.0.1:8766
  ▼
Browser UI
  │ Web Serial / USB at 115200 baud
  ▼
ESP32
  │ GPIO4 / 25 kHz PWM / open collector
  ▼
12 V four-wire PWM fan
```

## Bill of materials

### Required hardware

| Qty. | Item | Requirement / purpose |
| ---: | --- | --- |
| 1 | ESP32 development board | Tested with Freenove ESP32 WROOM-32E. Must support USB serial |
| 1 | 12 V four-wire PWM PC fan | Tested with a 120 mm fan; pin 4 must be the PWM control input |
| 1 | 12 V DC power supply | Must cover the fan's rated maximum current, preferably with 20–30% headroom |
| 1 | NPN transistor | Tested with 2SC1815GR; used only to pull the PWM signal low |
| 1 | 1 kΩ resistor | 1/4 W or greater; connects ESP32 GPIO to the transistor base |
| 1 | Breadboard | For prototype signal wiring |
| As needed | Jumper wires and connectors | For GPIO, common ground, and fan PWM connections |
| 1 | USB data cable | Must carry data; charge-only cables will not work |
| 1 | Windows PC | Runs VRChat, Python, a Web Serial browser, and Arduino IDE |

Choose the 12 V supply from the maximum current printed on the fan label or specified in its datasheet. Add the maximum currents when using multiple fans, and use appropriately rated power wiring and connectors.

### Required software

| Software | Purpose |
| --- | --- |
| VRChat | Sends avatar movement parameters over OSC |
| Python 3 | Runs the OSC bridge and local HTTP server; no third-party packages required |
| Chrome or Edge | Provides Web Serial access to the ESP32 |
| Arduino IDE 2.x | Builds and uploads ESP32 firmware |
| Arduino ESP32 Core 3.x | ESP32 platform; tested with version 3.3.11 |

## Wiring

The fan must be powered directly from the 12 V supply. Do not route its power current through the ESP32 or signal transistor.

```text
ESP32 GPIO4
    │
   1 kΩ
    │
2SC1815 Base

2SC1815 Collector ─── Fan PWM (pin 4)
2SC1815 Emitter   ─── Common GND

12 V supply + ─────── Fan +12 V (pin 2)
12 V supply - ──┬──── Fan GND (pin 1)
                 └──── ESP32 GND

Fan TACH (pin 3) ─── Not connected
```

Confirm the emitter, collector, and base pinout from the datasheet for the exact transistor in your hand. Package pinouts can vary.

The NPN stage inverts the PWM logic:

```text
requestedDuty = fanPercent × 255 / 100
gpioDuty      = 255 - requestedDuty
```

| Fan command | ESP32 GPIO duty |
| ---: | ---: |
| 0% | 255 |
| 50% | approximately 127 |
| 100% | 0 |

## Published web view

Open [`index.html`](index.html) to inspect the final interface. This public repository is intentionally limited to the self-contained web view and documentation. The original OSC bridge and ESP32 firmware are described below as part of the tested architecture but are not distributed here.

The page expects the original local API at `/api/state` for live OSC data. Without that bridge it remains a static interface preview; Web Serial controls also require a compatible browser and an ESP32 using the documented serial protocol.

## Technology stack

### Browser interface

- One self-contained `index.html`
- HTML5, responsive CSS, and Vanilla JavaScript
- Fetch API for 100 ms state polling
- Web Serial API for direct ESP32 communication
- `TextEncoder` / `TextDecoder` for serial messages
- `requestAnimationFrame` for display smoothing
- `localStorage` for speed thresholds

No external JavaScript or CSS packages, CDNs, images, web-font files, or other third-party assets are embedded in the published page.

### OSC / HTTP bridge architecture

The tested prototype used Python 3 standard-library modules only: `socket`, `struct`, `threading`, `http.server`, `pathlib`, and `collections.deque`. It listened for OSC on UDP `127.0.0.1:9001` and served state JSON on HTTP `127.0.0.1:8766`.

### ESP32 firmware architecture

- C++ / Arduino framework
- Arduino ESP32 Core 3.3.11
- ESP32 LEDC API: `ledcAttach(GPIO4, 25000, 8)` and `ledcWrite(...)`
- 115200-baud newline-delimited serial protocol

## Speed-to-airflow mapping

Default mapping:

| `VelocityMagnitude` | Fan target |
| ---: | ---: |
| Below 0.2 m/s | 0% |
| 3.150 m/s | 50% |
| 5.4 m/s or above | 100% |
| `InStation = true` | 0% |

The two ranges are linearly interpolated. The browser UI lets you change `deadZone`, `midSpeed`, and `maxSpeed`, subject to `0 <= deadZone < midSpeed < maxSpeed`.

Output is smoothed with a 150 ms rise time and 400 ms fall time, then sent to the ESP32 every 250 ms. If OSC packets stop for five seconds, the automatic target returns to 0%. If valid serial commands stop for four seconds, the ESP32 also returns to a 0% command.

## Serial protocol

Connection: 115200 baud, 8 data bits, no parity, one stop bit, newline-delimited ASCII/UTF-8 commands.

PC to ESP32:

```text
SET <fanPercent> <sequence>
STATUS
PING
```

ESP32 response example:

```json
{"fan":50,"gpio":127,"frequency":25000,"seq":123}
```

## Repository layout

```text
windlink/
├─ assets/
│  └─ windlink-prototype.jpg        Prototype hardware photo
├─ index.html                       Final self-contained web view
├─ README.md                        English project and technical guide
├─ README.ja.md                     Japanese project and technical guide
└─ LICENSE                          The Unlicense
```

## AI-assisted development

Generative AI assisted with implementation proposals, refactoring, HTML/CSS/JavaScript, documentation, and preparation of the public repository layout.

- A human tested the wiring, PWM control, OSC reception, and Web Serial integration on physical hardware.
- AI output was not treated as a substitute for electrical safety review.
- Errors and hardware-specific differences may remain. Verify component pinouts, current ratings, and power requirements against manufacturer datasheets.
- Contributors who use AI are encouraged to disclose its scope in their pull requests.

## Third-party technologies and licenses

The published `index.html` contains no bundled third-party source code or assets. The following tools and platforms were used by the original prototype but are not redistributed in this repository; their own terms apply when installed or used independently.

| Technology | Upstream license / terms | Included here? |
| --- | --- | --- |
| Arduino IDE 2.x | [GNU AGPL-3.0](https://github.com/arduino/arduino-ide) | No |
| Arduino CLI | [GNU GPL-3.0](https://github.com/arduino/arduino-cli) | No |
| Arduino ESP32 Core 3.3.11 | [LGPL-2.1-or-later](https://github.com/espressif/arduino-esp32/blob/master/package.json) | No |
| Python 3 and its standard library | [Python Software Foundation License Version 2](https://docs.python.org/3/license.html) | No |

HTML, CSS, JavaScript, Fetch, Web Serial, and related browser APIs are web technologies used through the user's browser; no browser implementation is copied into this repository. Product names and trademarks belong to their respective owners.

## License

WindLink-owned content in this repository is released under [The Unlicense](LICENSE), to the extent the contributors have the right to do so. This does not relicense third-party products, services, trademarks, or separately obtained software listed above.
