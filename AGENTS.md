# AGENTS.md

ESPHome firmware for the RemoteQTH WiFi DIN module. See the README for the
hardware and the control API. This file covers what isn't obvious from the
code.

## The vendor config is reused, not copied

`common/base.yaml` pulls the vendor's config
(`ok1hra/eth-din-dev-kit`, `esphome-solarbeam-version-linear-regulation.yaml`)
as a remote package pinned to a commit, and patches it with `!remove` /
`!extend`. Keep it that way:

- **Reuse as much of the vendor config as possible.** Patch it rather than
  rewriting parts of it. This is a deliberate project choice.
- **Never copy vendor YAML into this repo.** Upstream has no licence, so we
  can reference it but not redistribute it. For the same reason, don't publish
  prebuilt binaries until upstream adds a licence.

What the patches do, and why:

- **`interval` and `climate` are removed.** Upstream's SOC loop (every 2 s)
  and PID thermostat both republish POWER OUT, so a remotely set value is
  lost within seconds. The ESPHome PID writes 0 on every sensor reading even
  in OFF mode, so turning it off is not enough. The entities that only fed
  those (the SOC numbers and switches, `pid_output_temp`, `pid_temp_enable`)
  are removed with them.
- **The `temp` sensor is replaced.** Upstream hard-codes its author's DS18B20
  address, so any other sensor never reports.
- **Web UI firmware upload is added** (`ota: platform: web_server`). The
  vendor documents it as the update path, but ESPHome no longer enables it by
  default.
- **Security is deliberately left as upstream:** no authentication and no
  encryption. Don't add it back without asking.

## Gotchas

- **Removals must go in the file that includes the package.** A `!remove` in
  one package does not remove something another package brought in; it
  silently does nothing. Check the merged result, not just "Configuration is
  valid".
- **Moving the pinned commit:** check that every id patched in `base.yaml`
  still exists upstream. `!extend` and `!remove` on a missing id fail at
  validation, but a new upstream loop that writes POWER OUT would not. Read
  the upstream diff.
- **One network interface per build** (Wi-Fi or Ethernet), hence the two
  top-level files. The Ethernet variant removes `wifi` and `captive_portal`.
- **Expected build warnings:** GPIO12 and GPIO15 are strapping pins (the PCB
  fixes them). This is known and accepted.

## Contract with the controller

The Infigy core (`~/Projects/app`) depends on this. Don't change it without
changing core too:

- **Control is a single `POWER OUT` number (0–1).** The upstream cascade
  splits it across the three phases: the SSR modulates phase 1, and
  Stykač 1 / Stykač 2 switch in at ⅓ and ⅔.
- **Failsafe:** the `failsafe_timer` script restarts on every POWER OUT
  write. After `failsafe_timeout` (30 s) without a write, it sets POWER OUT
  to 0. The controller must re-send at least every 15 s.
- **REST addresses entities by name** (ESPHome ≥ 2026.8), for example
  `/number/POWER%20OUT`. Renaming an entity breaks the controller.

## Build and check

ESPHome is pinned in `requirements.txt`. Use the local venv, not a global
`esphome`:

```sh
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
.venv/bin/esphome config solarbeam-din-module-wifi.yaml    # prints the merged config
.venv/bin/esphome compile solarbeam-din-module-wifi.yaml   # and solarbeam-din-module-eth.yaml
```

After changing `base.yaml`, grep the output of `esphome config` to confirm
that no upstream `interval:`, `platform: pid` or hard-coded
`address: 0x5a…` survived. Then compile both variants.

## Comments

- Comment only *why*: non-obvious decisions, workarounds, upstream quirks.
  Never restate the YAML.
- Never write comments about the change itself ("updated", "fixed"). That
  goes in the commit message.

## Open items

- Ask upstream (OK1HRA) to add a licence. Possibly also contribute a
  "remote control" mode and the sensor fix, so fewer patches are needed.
- No GitHub remote yet.
- Core integration: add a new `HeatWaterDevice` in `core/src/shared/enums.ts`,
  with temperature and output branches in
  `core/src/modules/heat_water/HW_control.ts` next to AzRouter. It sends
  `sum(HW_EK_POWER_PERC) / 300`.
