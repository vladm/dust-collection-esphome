# Dust Collection Automation (ESPHome)

Automatic blast-gate and dust-collector control for a 4-machine workshop, built
entirely on ESPHome. A **hub** watches the current draw of each machine's supply
line, tells the matching **blast gate** to open over ESP-NOW, and starts the dust
collector on a relay.

All control logic runs on the ESP32s themselves — **Home Assistant is optional**.
HA (or the ESPHome dashboard) is used only for monitoring, calibration and remote
override; if the network or HA is down, the system still senses, gates and
collects. Validated against **ESPHome 2026.7.4**.

## What it does

- Detects which machine is running from AC current (ACS712 hall sensors, RMS with
  hysteresis) — no wiring changes to the machines beyond clamping the supply line.
- Opens only the blast gate for that machine; closes it when the machine stops.
- Sequences the collector: gate opens first (3 s), collector runs on for 10 s after
  the last machine stops, and a 30 s minimum-off lockout protects the motor.
- Manual paths: a latching hardware switch at the collector, a second latching
  "external" switch that also opens one configured gate (`external_gate_num`,
  currently gate 4), plus a "Remote Collection" switch for HA/voice.
- Fails safe: gates close if the hub goes quiet for 15 s, and every node closes its
  valve and starts the relay OFF on boot.

## Architecture

```
                     ESP-NOW broadcast (XXTEA-encrypted, rolling code)
  esp32-dust-hub  ──────────────────────────────────────────────►  esp32-dust-gate1..4
  (ESP32-C6 devkit)                                                (gate 2: ESP32-C6 devkit,
   • 4x ACS712-30A current sensors (ADC GPIO0..3)                   gates 1/3/4: XIAO ESP32-C5)
   • dust collector relay (GPIO10)                                  • MG996R servo ball valve
   • latching external switch (GPIO11): collector + one gate        • cover entity (damper)
   • collector-on LED (GPIO18)                                      • open/closed LEDs
   • latching manual switch (GPIO19)
```

- **Transport** — native ESPHome `espnow` + `packet_transport` in broadcast mode.
  The hub is the provider; each gate registers the hub's MAC as a peer and consumes
  only its own line channel (`gateN` ↔ `lineN_current`). Chosen over UDP/TCP so the
  link does not depend on Wi-Fi infrastructure, and for lowest latency.
- **Security** — XXTEA encryption with a shared PSK plus `rolling_code_enable` for
  replay protection.
- **Traffic** — state changes transmit immediately; a full-state reconciliation
  broadcast goes out every 5 s; the heartbeat (uptime sensor) every 2 s.

### Hub control logic

Per line: `adc` (8-sample average) → `ct_clamp` (AC RMS, cancels the ACS712 DC
offset) → `calibrate_linear` (volts → amps) → `analog_threshold` (`lineN_detect`:
1.5 A on / 0.8 A off, 3 s `delayed_off`). What the gates receive is a template
wrapper `lineN_current` = detection OR the external switch on its configured gate,
so the external switch opens its gate through the normal ESP-NOW path.

Collector demand = **any machine running** OR **latching manual switch** OR
**latching external switch** OR **"Remote Collection"**. Every path except Remote
Collection gives the gate a 3 s travel head start before spin-up: the machine path
via `delayed_on` on `any_machine_on`, the manual switch via `delayed_on` on its GPIO
sensor, and the external switch via the internal `external_demand` template — its
raw state opens the gate immediately, while all demand checks read the delayed view,
so the 5 s reconciliation cannot bypass the head start.

The relay itself is `internal:` — every path goes through
the `collector_start_req` / `collector_stop` scripts, so the 30 s lockout can never
be bypassed. A start requested during the lockout is *queued*: it waits out the
remainder and then re-checks that demand still exists. Relay and LED use
`restore_mode: ALWAYS_OFF`, and a fresh boot implies a full 30 s lockout. A 5 s
reconciliation interval makes the relay level-driven, so it converges on demand even
if a boot-time edge was missed. Actual relay state is published as
"Dust Collector Running".

### Gate control logic

`on_press` / `on_release` of the received line state run the calibrated servo sweep
scripts. Every 5 s the gate reconciles against the hub's state, which is
authoritative. If no hub packet arrives for **15 s**, the gate closes (one-shot,
re-arms when the link returns) — and since the relay lives on the hub, a dead hub
also means the collector is off.

Boot drives the valve straight to closed; `open_valve` / `close_valve` therefore
`script.stop: boot_close` first, otherwise `boot_close`'s 2 s delay could resume
*after* an early ESP-NOW open and clobber the position. The servo detaches at the
closed seat (the seat holds it) but **stays attached at open** — detaching there
caused a few-degree fall-back.

## Repository layout

| Path | Role |
|---|---|
| `esphome/esp32-dust-hub.yaml` | Complete hub config (standalone) |
| `esphome/dust-gate-common.yaml` | Shared gate package: all gate logic, C5 board/pin defaults |
| `esphome/esp32-dust-gate1.yaml` | Gate 1 (XIAO C5): servo trim + `wifi: band_mode: 2.4GHZ` |
| `esphome/esp32-dust-gate2.yaml` | Gate 2 (C6 devkit): board + pin + servo-trim overrides |
| `esphome/esp32-dust-gate3.yaml` | Gate 3 (XIAO C5): servo trim + `wifi: band_mode: 2.4GHZ` |
| `esphome/esp32-dust-gate4.yaml` | Gate 4 (XIAO C5): servo trim + `wifi: band_mode: 2.4GHZ` |
| `esphome/secrets.yaml-sample.txt` | Template for `esphome/secrets.yaml` (git-ignored) |
| `.github/workflows/validate.yml` | CI: `esphome config` over all device configs (Docker) |
| `CLAUDE.md` | Working notes / design rationale |

Per-device gate files contain only substitutions (device name, line channel, board,
pins, servo trim) plus `packages: !include dust-gate-common.yaml`. Main-file
substitutions override package defaults, and local config merges over package config
— that is how the C5 gates (1, 3, 4) add `band_mode` without touching the shared
file.

## Hardware

### Bill of materials

- 5× ESP32 dev board — **any ESP32 variant works** (see board note below).
  Reference build: 2× ESP32-C6 devkit (hub, gate 2), 3× Seeed XIAO ESP32-C5
  (gates 1, 3, 4)
- 4× ACS712 current sensor (**prefer the 20 A part** — see note below) + 8× divider
  resistors (10 kΩ or 5 kΩ — see wiring below) + 4× 100 nF capacitors
- 4× MG996R servo driving a ball-valve blast gate
- 1× relay module rated for the collector motor, 2× latching switches (manual +
  external), LEDs + resistors
- 5 V supply for the sensor rail and servos (**common ground with the ESP32s**)

### Board choice — C5, C6, or any other ESP32

The boards above are simply what this build uses: **C5 and C6 are interchangeable
on both the hub and the gates**, and any other ESP32 variant works as long as it
supports ESP-NOW (they all do) and has the required number and type of GPIOs:

- **Hub** — 4× ADC-capable inputs, 2× digital inputs (with internal pull-ups),
  2× digital outputs.
- **Gate** — 1× PWM/LEDC-capable output (servo), 2× digital outputs (LEDs).

The provided YAML configs are **suggestions**: set `board:` and the pin
substitutions to match your particular board, and avoid its strapping pins. Two
board-specific caveats: dual-band variants (the C5) must be hard-restricted to
2.4 GHz with `band_mode: 2.4GHZ` or ESP-NOW can go deaf (see the Wi-Fi section),
and on the classic ESP32/S-series pick **ADC1** pins for the current sensors —
ADC2 conflicts with Wi-Fi.

### Hub pinout (reference build: ESP32-C6, ADC1 = GPIO0..GPIO6)

| Pin | Function |
|---|---|
| GPIO0..GPIO3 | ACS712 line 1..4, via 10k/10k dividers |
| GPIO10 | Dust collector relay |
| GPIO11 | Latching external switch (to GND, `INPUT_PULLUP`, inverted): collector + gate `external_gate_num` |
| GPIO18 | "Collector on" indicator LED |
| GPIO19 | Latching manual switch (to GND, `INPUT_PULLUP`, inverted) |

GPIO4/GPIO5 are ADC-capable but double as strapping/JTAG pins; GPIO8/GPIO9 are boot
strapping pins — all left free.

### Gate pinout

| Pin | XIAO C5 (package default) | C6 (gate 2 override) |
|---|---|---|
| Servo PWM (LEDC, 50 Hz) | GPIO10 | GPIO3 |
| "Opened" LED | GPIO9 | GPIO10 |
| "Closed" LED | GPIO8 | GPIO1 |

### ACS712 input chain (per line)

```
   5 V ────────── VCC ┌────────────┐
                      │   ACS712   │   machine supply wire runs
                      │            │   through IP+ / IP-
   GND ──┬─────── GND └─────┬──────┘
         │                 OUT
         │                  │
         │               [ R1 ]  10 kΩ
         │                  │
         │        node A    ●──────────────── ESP32 ADC pin
         │                  │
         │            ┌─────┴──────┐
         │         [ R2 ]         ─┴─  C1
         │         10 kΩ          ─┬─  100 nF
         │            │            │
         └────────────┴────────────┴───────── GND (common with the ESP32!)
```

R1 = R2 halves the sensor output — the divider **ratio** is what matters, not the
absolute value, so **5 kΩ pairs work just as well as 10 kΩ**. Just use the same
nominal for both resistors of a given sensor. Keep C1 (node A → GND) close to the
ESP32 pin; it quiets ADC sampling noise.

The ACS712-30A puts out 66 mV/A around a 2.5 V idle midpoint at 5 V. The divider
halves that to **33 mV/A with a ~1.25 V midpoint**, which keeps the signal inside the
ESP32 ADC's linear range. `calibrate_linear: 0.033 -> 1.0` assumes the divider is
present — without it, use `0.066 -> 1.0`.

> **Prefer the ACS712-20A in most cases** — reserve the 30 A part for lines feeding
> really high-powered machines. The whole family has roughly the same absolute noise
> at the output, so sensitivity is what sets the noise floor *in amps*: the 20 A part's
> 100 mV/A (50 mV/A after the divider) is ~1.5× the 30 A part's, which lowers the idle
> noise floor accordingly and makes the gap between "shop idle" and the smallest
> machine much easier to threshold. With a 20 A sensor, change `calibrate_linear` to
> `0.050 -> 1.0` (divider) or `0.100 -> 1.0` (no divider). ±20 A of headroom covers
> most single-phase workshop machines.

> **Floating ADC pins read 1–2 phantom amps.** That is noise RMS and is expected on
> the bench with no sensors attached; it disappears once the low-impedance ACS712
> outputs are connected. Do not wire the real collector to the relay until the
> sensors are in — phantom demand will run it.

## Setup

### 1. Secrets

Copy the sample and fill it in. ESPHome looks for `secrets.yaml` next to the config
files, so it belongs in `esphome/` (already git-ignored):

```bash
cp esphome/secrets.yaml-sample.txt esphome/secrets.yaml
```

| Key | Purpose |
|---|---|
| `wifi_ssid`, `wifi_password`, `wifi_domain` | Wi-Fi (monitoring/OTA only) |
| `wifi_fallback_password` | Fallback AP password |
| `api_key` | Shared HA API key across the fleet |
| `ota_password` | OTA password |
| `espnow_psk` | ESP-NOW XXTEA pre-shared key (same on all 5 nodes) |
| `dust_hub_mac` | Hub MAC, registered as an ESP-NOW peer by the gates — **uppercase hex** |
| `wifi_bssid` | Optional — pin gates to one AP's 2.4 GHz radio |

The sample's dummy values are deliberately **format-valid** (password lengths,
32-byte base64 `api_key`, uppercase MACs) because CI validates against them —
keep the formats intact if you edit the sample.

### 2. Flash the hub first

```bash
esphome run esphome/esp32-dust-hub.yaml
```

Read the hub's MAC address out of its boot log and put it in `dust_hub_mac` before
flashing any gate — a gate with the wrong peer MAC will hear nothing.

> **Write the MAC in uppercase hex** — `AA:BB:CC:11:22:33`, not `aa:bb:cc:11:22:33`.
> Lowercase letters are not accepted for the ESP-NOW peer address, and the failure is
> silent at runtime: the gate boots and joins Wi-Fi normally but never receives a
> packet ("Hub Link OK" off, "Machine Running (remote)" unknown), which looks exactly
> like the 5 GHz band problem below. Digits and the `:` separators are unaffected.

### 3. Flash each gate

```bash
esphome run esphome/esp32-dust-gate1.yaml
```

### 4. Calibrate the servo endpoints (per rig)

Endpoints are mechanical and differ per valve. Uncomment the **Test Min Endpoint** /
**Test Max Endpoint** buttons in `dust-gate-common.yaml` (they write the raw
endpoints, ignoring `servo_direction`), then:

1. Raise `level_min` until the valve just seats closed without the servo buzzing.
2. Lower `level_max` until Test Max lands exactly on seated-open.
3. Set `servo_direction` to `-1.0` if the valve travels the wrong way.
4. Re-comment the buttons.

At 50 Hz: 5 % = 1000 µs, 7.5 % = 1500 µs, 10 % = 2000 µs. Current per-rig trims:
gate 1 `5%` / `10.6%` (step `0.047`), gate 2 `5.4%` / `10.3%`, gate 3 `5.1%` /
`10.0%`. Gate 4 ships gate 1's values and **still needs calibration on its own rig**.

### 5. Calibrate the current thresholds

Watch the throttled "Line N Current" sensors with the shop idle, then with the
smallest machine running, and set `on_threshold` / `off_threshold` between the two.
The 30 A parts have the highest noise floor in the family, so the idle reading will
not be zero.

## CI validation

Every push or PR touching `esphome/**` runs `esphome config` on all five device
configs (`.github/workflows/validate.yml`) inside the ESPHome Docker image pinned to
the release the fleet runs — full schema validation without the slow ESP-IDF
compile. `dust-gate-common.yaml` is covered through each gate that includes it, and
the git-ignored `secrets.yaml` is stood in for by the sample file. When upgrading
ESPHome, bump the image tag in the workflow together with the fleet.

## Tuning

Hub timings, in `esp32-dust-hub.yaml` `substitutions:`

| Substitution | Default | Meaning |
|---|---|---|
| `sample_duration` | `200ms` | RMS window (several AC cycles) |
| `sense_interval` | `1s` | How often each line is measured |
| `on_threshold` / `off_threshold` | `1.5` / `0.8` A | Machine ON/OFF hysteresis |
| `current_off_delay` | `3s` | Rides through brief current dips |
| `collector_start_delay` | `3s` | Gate head start before spin-up |
| `collector_runon` | `10s` | Run-on after the last machine stops |
| `collector_min_off_ms` | `30000` | Minimum-off lockout |
| `reconcile_interval` | `5s` | Full-state broadcast |
| `heartbeat_interval` | `2s` | Heartbeat publish |

Gate settings, in `dust-gate-common.yaml` (override per device):

| Substitution | Default | Meaning |
|---|---|---|
| `line_channel` | — | Which hub line this gate follows |
| `failsafe_timeout_ms` | `15000` | Silence before the fail-safe close |
| `sweep_step` | `0.045` | Position change per 20 ms tick (≈0.8 s sweep) |
| `servo_direction` | `1.0` | `-1.0` reverses travel |
| `level_min` / `level_max` | `4.1%` / `9.15%` | Closed / open endpoints |

Confirm that `collector_start_delay` is at least the real gate travel time.

## Home Assistant entities

**Hub** — Machine 1..4 Running · Manual Collection · External Collection ·
Dust Collector Running ·
Remote Collection (switch) · Line 1..4 Current (30 s averages) · WiFi Signal ·
Device Info · Reset Reason · Restart

**Each gate** — Ball Valve (cover, `damper`) · Machine Running (remote) ·
Hub Link OK (diagnostic) · Uptime · WiFi Signal · Device Info · Reset Reason ·
Restart device

"Remote Collection" is a *request*, not the relay: switching it on during the lockout
queues the start and the switch shows ON immediately, while "Dust Collector Running"
reflects the actual relay.

## Wi-Fi, mesh and dual-band — the main operational hazard

ESP-NOW requires all five radios on the **same 2.4 GHz channel**. The hub and gate 2
are C6 (2.4 GHz-only), but **gates 1, 3 and 4 are dual-band XIAO C5** and a C5 node
has gone deaf by associating to 5 GHz. The symptom is total silence: "Machine Running (remote)"
unknown, "Hub Link OK" disconnected, and hub reboots do not help.

Mitigations already in the configs:

- `band_mode: 2.4GHZ` on the C5 device files — a hard radio restriction. **C5-only**;
  it breaks the C6 build, which is why it lives in the device file and not the package.
- `enable_rrm: false` / `enable_btm: false` everywhere — roaming assist invites AP and
  band hops on stationary nodes.
- `power_save_mode: none` — modem sleep drops ESP-NOW packets.
- Optional `bssid:` pin, valid **only inside a `networks:` entry**, never at the
  `wifi:` top level. Take the BSSID from the 2.4 GHz radio listing; each band has its own.

Also pin the mesh's 2.4 GHz channel if the controller allows it. Note the
`reboot_timeout: 5min` side effect: during a long AP outage, gates reboot (and
boot-close) every 5 minutes even though ESP-NOW itself would have kept working.

## Diagnostics

- **"Hub Link OK"** — ON means the heartbeat is flowing and the fail-safe is armed.
  OFF while "Machine Running (remote)" still has a state means the heartbeat is
  missing and reconciliation is disabled. Both dead means the radio path is broken:
  check band and channel first.
- **Logs tell the story.** Gates log open/close/reconcile/fail-safe decisions; the hub
  logs every collector decision, including
  `Collector start deferred: 30 s minimum-off lockout active`,
  `Starting dust collector`, and `Lockout expired but demand gone`.
  Add `logs: { packet_transport: VERBOSE }` to see every received item.
- **`[S]` lines in the log viewer are not firmware logs** — the dashboard/HA viewer
  interleaves API *state updates* with logs and no `logger:` setting removes them.
  This is why the 1 Hz `ct_clamp` sensors are `internal:` (full rate for detection)
  and HA sees `copy` sensors throttled to a 30 s average instead.
- The `debug` component plus Reset Reason / Device Info text sensors are on every
  node — watch for brownout boot loops on the bench.

## ESPHome 2026.7.4 gotchas hit while building this

1. On a **consumer** `packet_transport`, `encryption` belongs under the provider
   entry. Top-level `encryption:` is for outgoing broadcasts and fails validation on
   a receive-only node ("No sensors or binary sensors to encrypt").
2. A `packet_transport` binary sensor with `name:` requires an explicit `internal:`.
3. `analog_threshold` takes `sensor_id:`, not `sensor:`.
4. `bssid` is only valid inside a `wifi: networks:` entry.
5. `band_mode` exists only on ESP32-C5 targets.
6. `publish_initial_state` was renamed `trigger_on_initial_state` (same codegen).
7. Fallback AP SSIDs are capped at 32 characters — `${device_name} Fallback` fits.
8. GPIO binary sensors do **not** fire automations for their boot state unless
   `trigger_on_initial_state: true` — required for the latching manual switch.
9. ESP-NOW peer MACs must be **uppercase hex**; a lowercase `dust_hub_mac` gives a
   node that runs normally but never receives anything.

## Deployment checklist

- [ ] Wire the ACS712s and dividers; calibrate per-line thresholds against real draw.
- [ ] Verify the gate 2 C6 pin overrides against the actual wiring.
- [ ] Calibrate gate 4's servo trim (currently a copy of gate 1's values).
- [ ] Set `wifi_bssid` and uncomment the bssid pin if the mesh mis-channels.
- [ ] Pin the mesh 2.4 GHz channel; delete stale HA entities left by renames.
- [ ] Confirm `collector_start_delay` ≥ real gate travel time.
- [ ] Only then wire the physical collector to the relay.

## Safety

The ACS712 sensors sit on mains-voltage machine supply lines and the relay switches a
large induction motor. Do that wiring de-energized, to local code, and have it
inspected if you are not qualified. The 30 s minimum-off lockout exists to protect the
collector motor — do not shorten it without knowing what your motor tolerates.
