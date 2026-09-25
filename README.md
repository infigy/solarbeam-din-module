# Solarbeam DIN module

Open-source [ESPHome](https://esphome.io) firmware for the
[RemoteQTH WiFi DIN module](https://remoteqth.com/w/doku.php?id=din-wifi)
(ESP32, 12–24 V DC or passive PoE), used as a remote output for a
water-heater element.

This project builds on the module vendor's own ESPHome configuration
([ok1hra/eth-din-dev-kit](https://github.com/ok1hra/eth-din-dev-kit), by
OK1HRA, with regulation by OK1BT). Their config is fetched at build time,
pinned to a commit, and patched in [`common/base.yaml`](common/base.yaml):

- **Removed:** the on-device regulation (the SOC loop and the PID
  thermostat). Both rewrite POWER OUT every few seconds, which makes remote
  control impossible. Here the controller (the Infigy app, Home Assistant,
  your own script) does the regulating.
- **Replaced:** the DS18B20 temperature sensor. Upstream hard-codes its
  author's sensor address; this build uses whichever sensor is connected.
- **Added:** a failsafe.

Everything else is theirs, unchanged: pins, the SSR slow PWM, and the
POWER OUT cascade.

Not affiliated with RemoteQTH.

## Hardware

| Output   | GPIO   | Terminal                                                        |
| -------- | ------ | --------------------------------------------------------------- |
| SSR      | GPIO12 | pin 6, open collector 500 mA, 10k pull-up to supply (+V on pin 5) |
| Stykač 1 | GPIO13 | pin 4, open collector 500 mA (+V on pin 3)                       |
| Stykač 2 | GPIO14 | pin 2, open collector 500 mA (+V on pin 1)                       |

The temperature sensor (DS18B20) connects to the JST 1-wire socket (GPIO15).

The outputs are **open-collector transistors, not relays**. They switch an SSR
input or a 12/24 V DC contactor coil between the supply voltage and the
output pin.

Only one network interface can be active per build, so there are two
variants: `solarbeam-din-module-wifi.yaml` (with a setup access point) and
`solarbeam-din-module-eth.yaml` (Ethernet with DHCP).

## Control API

The controller sets a single value, **POWER OUT** (0–1), for three-phase
heating. The upstream cascade splits it across the outputs:

| POWER OUT | SSR (phase 1)   | Stykač 1 (phase 2) | Stykač 2 (phase 3) |
| --------- | --------------- | ------------------ | ------------------ |
| 0 – ⅓     | PWM 0 → 100 %   | off                | off                |
| ⅓ – ⅔     | PWM 0 → 100 %   | on                 | off                |
| ⅔ – 1     | PWM 0 → 100 %   | on                 | on                 |

REST (ESPHome ≥ 2026.8 addresses entities by name; URL-encode the spaces).
There is no authentication, the same as the vendor config.

| Entity    | Request                                  |
| --------- | ---------------------------------------- |
| POWER OUT | `POST /number/POWER%20OUT/set?value=0.5` |
| TEMP (°C) | `GET /sensor/TEMP`                       |
| SSR Level | `GET /sensor/SSR%20Level`                |
| FAILSAFE  | `GET /binary_sensor/FAILSAFE`            |

`GET /events` streams every state change as server-sent events. The same
entities are also available over the ESPHome native API (port 6053), which
Home Assistant uses.

### Failsafe

If no POWER OUT write arrives for **30 s** (`failsafe_timeout`), POWER OUT is
set to 0 and `FAILSAFE` turns on. Repeated writes of the same value count as
commands, so a controller that holds a steady output **must re-send it at
least every 15 s**. Everything is off after every boot until the first
command arrives.

## Build and flash

```sh
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
```

The first flash goes over USB-C. The module has a CP2104 USB-serial chip with
automatic reset, so no button presses are needed. On macOS it shows up as
`/dev/cu.usbserial-*`. If it doesn't, install the Silicon Labs CP210x VCP
driver.

```sh
.venv/bin/esphome run solarbeam-din-module-wifi.yaml    # compile, flash, show logs
```

Later updates can go over the network with the same command, or through the
web UI upload. Instead of compiling, you can also flash a prebuilt
`*.factory.bin` from https://web.esphome.io in Chrome or Edge.

USB-C and RS-485 share the ESP32's UART. While USB is plugged in, RS-485 is
disconnected.

### First Wi-Fi setup

Connect to the "EMS NG" access point (password `12345678`) and open
http://192.168.4.1.

The device's address is printed in the serial log. It is also advertised over
mDNS as `solarbeam-din-module-<mac>.local`.

## Security

Like the vendor config, this build has no authentication: anyone on the
network can control the outputs, read the values, and upload firmware. Keep
the module on a trusted network. The setup access point uses the vendor's
public password.

## License

MIT. See [LICENSE](LICENSE).
