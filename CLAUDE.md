# Dust Collection Automation — Project Summary

ESPHome-based, Home-Assistant-independent automation for a 4-machine workshop
dust collection system. A hub senses which machine is drawing current, opens the
matching blast gate over ESP-NOW, and starts the dust collector. All control
logic runs on the ESP32s themselves; HA is used only for monitoring and optional
remote control. Validated against **ESPHome 2026.7.4**.

## Architecture

```
                      ESP-NOW broadcast (encrypted, rolling code)
  esp32-dust-hub  ────────────────────────────────────────────────►  esp32-dust-gate1..4
  (ESP32-C6 devkit)                                                   (gate 2: ESP32-C6 devkit,
   • 4x ACS712-30A current sensors (ADC GPIO0..3)                      gates 1/3/4: XIAO ESP32-C5)
   • dust collector relay (GPIO10)                                    • MG996R servo ball valve
   • latching external switch (GPIO11): collector + one gate          • cover entity (damper)
   • collector-on LED (GPIO18)                                        • open/closed LEDs
   • latching manual switch (GPIO19)
```

- **Transport**: native ESPHome `espnow` + `packet_transport` (broadcast mode).
  Hub is the provider; gates register the hub MAC as a peer. Chosen over
  UDP/TCP for independence from Wi-Fi infrastructure and lowest latency.
- **Security**: XXTEA encryption (shared PSK) + `rolling_code_enable` for
  replay protection. On a **consumer**, the key goes under
  `providers: - name: ... encryption:` — top-level `encryption:` is for
  outgoing broadcasts and fails on a receive-only node
  ("No sensors or binary sensors to encrypt").
- **Traffic**: immediate transmission on any state change; full-state
  reconciliation broadcast every 5 s; heartbeat (uptime sensor) every 2 s.

## Control logic

**Hub** (per line): ADC (8-sample avg) → `ct_clamp` (AC RMS, cancels ACS712 DC
offset) → `calibrate_linear` (V→A) → `analog_threshold` (`lineN_detect`, 1.5 A
on / 0.8 A off hysteresis, 3 s delayed_off). What gets broadcast is a template
wrapper `lineN_current` = `lineN_detect` OR (external switch AND N ==
`external_gate_num`) — the id the gates bind to, so the external switch opens
its gate with zero gate-side changes. The wrapper also self-latches through
the run-on: once ON it holds while the collector relay is energized and no
other demand source (other lines' detect, manual, external, remote) is active,
so the last machine's gate closes only after the relay drops — ducts purge
through the open gate, collector never dead-heads. Another active demand
releases the hold immediately (a stale open gate would bleed suction). Collector relay demand = any machine ON
(`any_machine_on` reads `lineN_detect`), OR latching manual switch, OR latching
external switch (GPIO11; manual-switch semantics + opens gate
`external_gate_num`), OR "Remote Collection" HA template switch. The 3 s
gate-travel head start applies on every path except Remote Collection: machine
via `delayed_on` on `any_machine_on`, manual via `delayed_on` on its GPIO
sensor, external via internal `external_demand` template (raw switch drives
the gate wrapper immediately; all demand checks read `external_demand`, so the
5 s reconciliation can't bypass the delay). Sequencing: 3 s start delay
(gate opens first), 15 s run-on after last machine stops, **30 s minimum-off
lockout** (starts during lockout are queued and re-check demand when it
expires; a reboot implies a fresh 30 s lockout). Relay is `internal:` — every
path goes through `collector_start_req` / `collector_stop` scripts; state is
exposed via the "Dust Collector Running" sensor. Relay + LED are
`restore_mode: ALWAYS_OFF`. A 5 s reconciliation interval makes the relay
level-driven (converges to demand even if a boot-time edge is missed).

**Gates**: `on_press`/`on_release` of the received line state drive the
calibrated servo sweep scripts. 5 s reconciliation treats hub state as
authoritative. **15 s fail-safe close** on heartbeat loss (one-shot, re-arms
on link return); since the collector relay is on the hub, a dead hub also
means the collector is off. Boot closes the valve; `open_valve`/`close_valve`
must `script.stop: boot_close` (boot race fix — boot_close's 2 s delay would
otherwise resume after an early ESP-NOW open and clobber position to closed).
Servo detaches at closed (seat holds), stays attached at open (holding torque;
detaching caused fall-back).

## Files

| File | Role |
|---|---|
| `esp32-dust-hub.yaml` | Complete hub config (standalone) |
| `dust-gate-common.yaml` | Shared gate package: all logic, C5 pin/board defaults |
| `esp32-dust-gate1.yaml` | Gate 1 (XIAO C5): trim + `wifi: band_mode: 2.4GHZ` |
| `esp32-dust-gate2.yaml` | Gate 2 (C6 devkit): overrides board + pins + trim |
| `esp32-dust-gate3.yaml` | Gate 3 (XIAO C5): trim + `wifi: band_mode: 2.4GHZ` |
| `esp32-dust-gate4.yaml` | Gate 4 (XIAO C5): trim + `wifi: band_mode: 2.4GHZ`; target of the hub's external switch |
| `.github/workflows/validate.yml` | CI: `esphome config` on all `esp32-dust-*.yaml` in the pinned Docker image (2026.7.4); secrets come from the sample file, whose dummy values must stay format-valid |

Per-device files carry only substitutions (device_name, line_channel, board,
pins, servo trim) + `packages: !include dust-gate-common.yaml`. Main-file
substitutions override package defaults; local config merges over package
config (used for `band_mode`). Gate↔line pairing: gateN ↔ lineN_current.

Secrets (`secrets.yaml`): `wifi_ssid`, `wifi_password`, `wifi_domain`,
`wifi_fallback_password`, `api_key` (shared fleet key), `ota_password`,
`espnow_psk`, `dust_hub_mac` (hub MAC for gate peer registration),
optional `wifi_bssid_2g`.

## Hardware notes

- **ACS712-30A**: 66 mV/A at 5 V supply, 2.5 V idle midpoint. Each OUT feeds a
  **10k/10k divider** (OUT → 10k → node A → 10k → GND; ADC taps node A;
  100 nF node A → GND) → 33 mV/A, ~1.25 V midpoint, inside the ESP32 ADC
  linear range. `calibrate_linear: 0.033 → 1.0` assumes the divider.
  Grounds of the 5 V sensor rail and ESP32 must be common.
- **Floating ADC pins read 1–2 phantom amps** (noise RMS) — expected on the
  bench with no sensors attached; disappears once the low-impedance ACS712
  outputs are connected. Thresholds need per-line calibration against real
  machine draw (30 A parts have the family's highest noise floor in amps).
- **Servo endpoints** are per-rig and live in each gate file: gate 1
  `5%`/`10.6%` (step `0.047`), gate 2 `5.4%`/`10.3%`, gate 3 `5.1%`/`10.0%`;
  gate 4 is a copy of gate 1's trim, not yet calibrated. Package defaults
  remain `4.1%`/`9.15%`. Use the commented Test Min/Max Endpoint buttons
  to trim.

## Wi-Fi / mesh / dual-band (the big operational hazard)

ESP-NOW requires all five radios on the **same 2.4 GHz channel**. The hub and
gate 2 are C6 (2.4 GHz-only); **gates 1, 3 and 4 are dual-band XIAO C5** and a
C5 node once went deaf by associating to 5 GHz (symptom: zero reception — remote sensor
"unknown", Hub Link OK disconnected; hub reboots don't help). Mitigations in
config: `band_mode: 2.4GHZ` on C5 nodes (hard radio restriction; **C5-only
option**, breaks the C6 build — hence it lives in the C5 device files),
`enable_rrm: false` / `enable_btm: false` everywhere (roaming assist invites
AP/band hops), optional `bssid:` pin (valid **only inside a `networks:`
entry**, never top-level). Also recommended: pin the mesh's 2.4 GHz channel.
`power_save_mode: none` matters — modem sleep drops ESP-NOW packets.
`reboot_timeout: 5min` side effect: during long AP outages gates reboot (and
boot-close) every 5 min even though ESP-NOW itself would keep working.

## Diagnostics

- **"Hub Link OK"** (gate): ON = heartbeat flowing, fail-safe armed. OFF while
  "Machine Running (remote)" has a state = heartbeat missing → reconciliation
  disabled. Both dead = radio path broken (check band/channel first).
- Log narrative: gates log open/close/reconcile/fail-safe decisions; hub logs
  every collector decision incl. "Collector start deferred: 30 s minimum-off
  lockout active" / "Starting dust collector" / "Lockout expired but demand
  gone". `logs: packet_transport: VERBOSE` shows every received item
  ("Got sensor heartbeat ...").
- **`[S]` log-viewer lines are not firmware logs** — the dashboard/HA viewer
  renders API *state updates* interleaved with logs; no `logger:` setting can
  remove them. Fixed by making the 1 Hz ct_clamp sensors `internal:` (full
  rate for detection) and exposing `copy` sensors with `throttle_average: 30s`
  for HA reporting.
- Debug component + Reset Reason / Device Info text sensors on all nodes
  (watch for brownout boot loops on the bench).

## ESPHome 2026.7.4 gotchas encountered

1. Consumer `packet_transport`: `encryption` belongs under the provider entry.
2. `packet_transport` binary sensor with `name:` requires explicit `internal:`.
3. `analog_threshold` takes `sensor_id:`, not `sensor:`.
4. `bssid` only valid inside `wifi: networks:` entries.
5. `band_mode` exists only on ESP32-C5 targets.
6. `publish_initial_state` → renamed `trigger_on_initial_state` (same codegen).
7. Fallback AP SSID limit 32 chars — `${device_name} Fallback` fits; the old
   "... Fallback Hotspot" suffix did not.
8. GPIO binary sensors do NOT trigger automations on boot state unless
   `trigger_on_initial_state: true` (needed for the latching manual switch).

## Remaining / deployment checklist

- [ ] Wire ACS712s + dividers; calibrate per-line on/off thresholds from the
      throttled amp sensors (idle floor vs smallest machine draw).
- [ ] Verify gate2 C6 pin overrides against actual wiring.
- [ ] Calibrate gate4 servo trim (currently a copy of gate1's values).
- [ ] Set `wifi_bssid` + uncomment bssid pins if the mesh mis-channels.
- [ ] Pin the mesh 2.4 GHz channel; delete stale HA entities from renames.
- [ ] Confirm 3 s `collector_start_delay` ≥ real gate travel time.
- [ ] Don't wire the physical collector to the relay until sensors are in
      (floating pins create phantom demand and will run it).
