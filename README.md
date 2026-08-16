# Rain Barrel Pump Automation

An ESPHome + Home Assistant project that automatically switches a rain
barrel pump to supply water to the garden/sprinklers. Whenever water
is drawn downstream (detected via a flow sensor), this system activates
a pump + solenoid valve to deliver water from the rain barrel — provided
the barrel is not empty. Includes built-in safety features for leak
detection, a stuck pump, and overheating.

> **Disclaimer:** these instructions were generated with the help of
> Claude (Anthropic) after the build was completed, based on the working
> setup. The author accepts no responsibility for damage, leaks,
> electrical problems or other consequences of replicating this project.
> Always work safely with electricity and water.

> **Status:** this is an extract from a private Home Assistant setup,
> intended to allow others to replicate it. See [To do](#to-do) for
> what is still missing (especially: photos of the physical setup —
> a [parts diagram](diagram.html) with full parts list is already here,
> photos to follow).

## How it works

1. A **flow sensor** in the main water line to the garden/sprinklers
   detects when water is being drawn.
2. When flow is detected (> 1 L/min) and the **float switch** in the
   rain barrel indicates water is present, the **solenoid valve** closes
   the tap and (2 seconds later) the **pump** starts — delivering rain
   water instead of tap water.
3. When flow stops (< 1 L/min), both pump and valve turn off and the
   tap supplies water again.
4. If the barrel runs dry during use (float → off), pump and valve stop
   immediately and you receive a notification. When water returns
   (float → on), another notification is sent.
5. A set of safety automations monitors the rest: leak detection (too
   high flow), a pump running too long, the solenoid valve out of sync
   with the pump, and an overheating ESP32 enclosure.

## Required hardware

### Electronics

| Component | Specifications | Source |
|---|---|---|
| ESP32 WROOM-32 | dev board with USB-C | [Amazon](https://www.amazon.nl/dp/B0D4QZ9CKD) |
| 24V 5A switching power supply | open frame, 120W | [Amazon](https://www.amazon.nl/dp/B0BX2GF5QY) |
| Buck converter 20V | adjustable DC-DC, 20A/300W | [Amazon](https://www.amazon.nl/dp/B09KC2FNLB) |
| 5V USB power supply | separate supply for ESP32, isolated from 24V circuit | Any 5V 1A+ USB charger |
| 2-channel 5V relay module | with optocoupler, active-low | [Amazon](https://www.amazon.nl/dp/B0B41KGSJ7) |
| ¾" brass flow sensor | YF-S201 variant, ~444 pulses/litre | [Amazon](https://www.amazon.nl/dp/B0DP93BWWJ) |
| Float switch | dry contact, 3 wires (COM/NO/NC) | [Amazon](https://www.amazon.nl/dp/B09XXHY7C9) |
| 24V DC NO solenoid valve | ¾" brass, normally open | Heschen 2WK200-20 |
| 30mm 5V PWM fan | for enclosure cooling | electronics supplier |
| Weatherproof enclosure | large enough for power supply + electronics | hardware store |
| Cable glands | for watertight cable entry | hardware store |
| Wago connectors | for connecting wires | hardware store |

### Pump

This project uses a **Parkside PTBP 20-Li** cordless rain barrel pump,
which normally runs on a 20V lithium battery pack. In this setup the
battery is replaced by an adjustable buck converter supplying 20V from
the 24V power supply. The pump cable is connected directly to the buck
converter output.

Note: the pump is designed for manual use with a battery. By connecting
it to a fixed power supply you lose the built-in battery protection.
Use a buck converter with sufficient current capacity (minimum 4A at
20V).

### Plumbing

| Component | Specifications | Source |
|---|---|---|
| ¾" brass Y-piece / manifold | 2 inputs, 1 output | [Bol.com](https://www.bol.com/nl/nl/p/verdeler-splitter-messing-2-aftakkingen-3-4/9300000044146766/) |
| Garden hose non-return valve | in pump outlet, prevents backflow | Gardena/hardware store |
| ¾" swivel coupling M×F | for mounting without rotation | Amazon |

## Wiring diagram

```
230V mains
    │
    ├── 24V power supply (120W)
    │       │
    │       ├── Buck converter → 20V → Relay COM1 → Pump (+)
    │       │                          Pump (−) → 24V GND
    │       │
    │       └── Relay COM2 → Solenoid valve (+)
    │                        Solenoid valve (−) → 24V GND
    │
    └── 5V USB power supply → ESP32 (USB-C)
                                  │
                                  ├── 5V pin → Wago → Relay VCC
                                  │                   Flow sensor red
                                  │                   Fan red
                                  │
                                  ├── GND → Wago → Relay GND + BGND (24V GND)
                                  │                Flow sensor black
                                  │                Float switch black
                                  │                Fan black
                                  │
                                  ├── GPIO5  → Float switch signal (blue)
                                  ├── GPIO13 → Flow sensor yellow
                                  ├── GPIO19 → Relay IN1 (solenoid valve)
                                  ├── GPIO21 → Relay IN2 (pump)
                                  └── GPIO23 → Fan blue (PWM)
```

**Important:** the relay module's BGND must be connected to 24V GND
(common ground). Use a **separate 5V USB power supply** for the ESP32
— not the 24V supply via a buck converter. A pump draws a large inrush
current on startup that temporarily overloads the 24V supply; if the
ESP32 is on the same supply it will reset.

## Plumbing diagram

```
Tap (wall tap)
    │
    └── Solenoid valve (NO, open when pump is off)
            │
            └── Y-piece / manifold ¾"  ←── Non-return valve ←── Pump (from rain barrel)
                    │
                    └── Flow sensor ¾"
                            │
                            └── Distribution manifold
                                    ├── Sprinklers
                                    └── Garden hose
```

**Flow direction:** the tap supplies water by default through the open
solenoid valve. When the pump takes over, the solenoid valve closes the
tap. The non-return valve in the pump outlet prevents tap water from
flowing back into the barrel when the pump is off.

## ESPHome entities (from `esphome/regenton.yaml`)

| Component | Role | GPIO | HA domain |
|---|---|---|---|
| ESP32 dev board (`esp32dev`) | system brain | — | — |
| Flow sensor, ~444 pulses/litre | measures flow in L/min | GPIO13 | `sensor` |
| Float switch (switches to GND) | detects whether barrel contains water | GPIO5 | `binary_sensor` |
| Relay (active-low) + NO solenoid valve | closes tap when pump is on | GPIO19 | `switch` |
| Relay (active-low) + water pump | pumps water from the barrel | GPIO21 | `switch` |
| PWM fan (via LEDC output) | cools enclosure, locally controlled above 65°C | GPIO23 | `fan` |
| ESP32 internal temperature sensor | monitors enclosure temperature | internal | `sensor` |

## Installation

1. **Flash ESPHome firmware**: copy
   [`esphome/secrets.yaml.example`](esphome/secrets.yaml.example) to
   `esphome/secrets.yaml` and fill in your WiFi credentials. Then flash
   [`esphome/regenton.yaml`](esphome/regenton.yaml) to your ESP32 via
   USB using `esphome run regenton.yaml --device COMx` (Windows) or
   via the ESPHome add-on in Home Assistant. After the first USB flash,
   further updates work via OTA (WiFi).
2. Add the device to Home Assistant via the ESPHome integration
   (auto-discovery via zeroconf, or manually with IP address).
3. Note the entity names after adding (e.g. `sensor.regenton_water_flow`,
   `binary_sensor.regenton_float_switch`, `switch.regenton_pump`,
   `switch.regenton_solenoid_valve`, `fan.regenton_fan`,
   `sensor.regenton_esp32_temperature`) — adjust the entity IDs in
   `automations.yaml` if your names differ.
4. Copy the contents of [`automations.yaml`](automations.yaml) to your
   own `automations.yaml` (or import individual automations via the UI).
5. Copy the `template:` section from
   [`template_sensors.yaml`](template_sensors.yaml) to your
   `configuration.yaml`.
6. **Replace the placeholder notification service** — search
   `automations.yaml` for `notify.YOUR_NOTIFY_TARGET` and replace it
   with your own notify service (e.g. `notify.mobile_app_phone`).
7. Reload automations and restart HA.
8. Optional: add the card from [`dashboard.yaml`](dashboard.yaml) to
   a dashboard.

## Safety automations

- **Leak emergency stop** — flow above 6 L/min indicates a leak; pump
  off immediately, valve closed, notification sent. Adjust the threshold
  (`above: 6`) to suit your normal flow rate.
- **Pump maximum 21 minutes on** — prevents a stuck pump running
  indefinitely. Adjust the duration to what is realistic for your garden.
- **Keep pump and valve in sync** — failsafe: if the pump is on, the
  valve must also be on (open).
- **Enclosure too hot** — notification if the ESP32 enclosure exceeds
  65°C.

## Important build notes

**Power supply:** use a **separate 5V USB power supply** for the ESP32,
completely isolated from the 24V supply. A motor pump draws an inrush
current several times its normal draw on startup — this temporarily
overloads the supply and resets the ESP32 if it shares the same supply.

**Relay:** use a relay module with **active-low trigger** (low level
trigger). Add `inverted: true` in ESPHome for both relay channels.

**Solenoid valve:** choose a **normally open (NO)** valve. On power
failure the valve opens and the tap supplies water — the correct
failsafe for a garden installation. A separate non-return valve on the
tap side is not needed: the solenoid valve is always closed when the
pump is running.

**Fan:** the enclosure can heat up significantly in direct sun. A 30mm
5V PWM fan with a hole at the bottom (air inlet) and side (outlet)
keeps the temperature under control. Cover holes with fly mesh to keep
insects out and fit protective caps so water cannot enter directly. The
enclosure is no longer fully weatherproof once ventilation holes are
added — condensation is the main risk. Mount the enclosure in a sheltered
location and ensure good air circulation. Fan control runs entirely
locally on the ESP32 and does not depend on Home Assistant.

**Flow sensor calibration:** the YF-S201 produces ~450 pulses per litre.
The calibration value in ESPHome (`multiply: 0.00225`) is tuned for
this, but may vary between sensors — test with a known volume of water.

## To do

- [ ] Photos of the physical setup and wiring — see [`diagram.html`](diagram.html) (parts diagram + full parts list; photos to be added to `photos/`)
- [ ] Confirm exact parts list (brand/type per component)
- [x] Choose a licence — CC BY-SA 4.0, see [`LICENSE`](LICENSE)
- [ ] Move to the final GitHub repository once created
