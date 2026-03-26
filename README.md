# StayKey ESPHome

Modular ESPHome configurations for StayKey smart-home devices, managed through Home Assistant. Currently supports a **Wiegand keypad + RFID lock controller** and a **Balboa spa controller**.

![Wiring Diagram](docs/wiring-diagram.svg)

## Project Structure

```
staykey-keypad-lock-c3.yaml         Keypad lock -- ESP32-C3 Super Mini
staykey-keypad-lock-c6.yaml         Keypad lock -- Xiao ESP32-C6
staykey-spa-controller.yaml         Balboa spa controller -- ESP32-S3
components/                         Local ESPHome external components
  balboa_spa/                         Balboa spa integration (vendored, BSD-2-Clause)
blueprints/                         Home Assistant automation blueprints
  keypad_notifications.yaml           Notify on code entry events
  keypad_garage_toggle.yaml           Toggle garage door on unlock
docs/
  wiring-diagram.svg
secrets.yaml.example                Template for secrets.yaml
```

Each device file is a complete, self-contained ESPHome configuration. The flat layout matches the ESPHome Dashboard's expected structure -- copy the YAML files, `components/`, and `secrets.yaml` into `/config/esphome/` and they work as-is.

## Devices

### Keypad Lock

A Wiegand keypad + RFID reader that controls a door lock through Home Assistant. Supports **ESP32-C3 Super Mini** and **Xiao ESP32-C6** boards.

**Features:**

- **30 code slots** with NVS persistence (codes survive reboots)
- **30 RFID tag slots** with NVS persistence and Wiegand 26-bit card reads
- **Code learn mode** -- flip a switch in HA, type a code on the keypad, and it auto-registers in the next empty slot
- **Tag learn mode** -- flip a switch in HA, scan a tag, and it auto-registers in the next empty slot
- **5-attempt lockout** with 60-second cooldown (shared across codes and tags)
- **Configurable relay pulse** duration (1-30s, default 5s)
- **Auto-relock timer** (0-120s, 0 to disable)
- **Home Assistant API actions** for codes (`set_code`, `clear_code`, `get_codes`), tags (`set_tag`, `clear_tag`, `get_tags`), and settings (`set_relay_pulse_time`, `set_relock_time`)
- **Home Assistant events**: `esphome.keypad_code_entered` with `source` field (`code` or `tag`) and event types `code_valid`, `code_invalid`, `code_lockout`, `code_timeout`, `tag_valid`, `tag_invalid`, `tag_lockout`
- **Lock entity** in HA with full lock/unlock control
- **Last Scanned Tag** text sensor -- always shows the most recently scanned tag ID

### Spa Controller

A Balboa spa controller using the `esphome-balboa-spa` external component on an ESP32-S3. Provides climate control, jet control, and light control through Home Assistant.

## Supported Hardware

| Component | Tested Models |
|---|---|
| **ESP32 board** | ESP32-C3 Super Mini, Seeed Xiao ESP32-C6, ESP32-S3-DevKitC-1 |
| **Keypad** | Wiegand WG26/34 keypads with RFID (125kHz EM4100) |
| **Relay** | Any 3.3V-trigger relay module |
| **Lock** | Electric door strike, magnetic lock, or garage door opener |
| **Spa** | Balboa-based hot tubs (via UART) |

## Keypad Wiring

Both C3 and C6 boards use the same pin assignments:

| Function | GPIO | Board Pin |
|---|---|---|
| Wiegand D0 | GPIO0 | Pin labeled `0` |
| Wiegand D1 | GPIO1 | Pin labeled `1` |
| Relay output | GPIO2 | Pin labeled `2` |

**Keypad wiring:**

| Keypad Terminal | Connect To |
|---|---|
| 12V | 12V power supply (+) |
| GND | Power supply (-) AND ESP32 GND |
| D0 | ESP32 GPIO0 |
| D1 | ESP32 GPIO1 |

The keypad **must** be powered at 12V (per its spec).

> **No level shifter needed!** The Wiegand D0/D1 lines are open-collector outputs -- they only pull LOW to signal, and float when idle. The ESP32's internal pullups pull D0/D1 to 3.3V. The data lines never carry 12V, so you can connect D0 and D1 directly to the ESP32 GPIO pins without any voltage divider or level shifter.

The relay module connects to GPIO2 (signal), 3.3V (VCC), and GND. The relay's NO (normally open) and COM terminals switch 12V to the electric lock/strike.

### Powering everything from one 12V supply

You can use a small DC-DC buck converter (e.g. MP1584EN mini module) to step down 12V to 5V and power the ESP32 through its 5V pin. This lets you run the entire setup -- keypad, ESP32, relay, and lock -- from a single 12V power supply.

```
12V PSU ──┬── Keypad 12V
          ├── Relay COM (for lock)
          └── Buck converter IN
                  │
                  └── 5V OUT ── ESP32 5V pin
```

## Keypad Setup (Important!)

Most Wiegand keypads ship in **standalone access-control mode**, where they buffer PIN entry internally and send the full code as a single Wiegand "card" transmission. This does **not** work with ESPHome's Wiegand key-by-key input.

You must switch the keypad to **head reading mode** so it sends each key press individually over Wiegand:

```
Press: * 123456 # 0 31 #
```

Where `123456` is the factory default programming password. After this, each key press sends a 4-bit Wiegand code immediately, and card scans send standard 26-bit data.

> **Note:** If the keypad is factory-reset, you will need to run this sequence again. The default door-opening password is `7890` and the programming password is `123456`.

## Installation

### Using the ESPHome Dashboard (Home Assistant)

1. **Initial flash** -- Connect the ESP32 via USB to a machine with ESPHome CLI installed, then flash the config:
   ```bash
   pip install esphome
   esphome run staykey-keypad-lock-c3.yaml
   ```

2. **Adopt in Home Assistant** -- Once flashed, the device connects to your WiFi and appears in the ESPHome Dashboard. Click **Adopt** and the Dashboard will import the full configuration from GitHub automatically.

3. **Set your secrets** -- After adopting, edit the device in the Dashboard and fill in your WiFi credentials and API key in the secrets section.

### Using the ESPHome CLI

1. **Create secrets** -- Copy the secrets template and fill in your values:
   ```bash
   cp secrets.yaml.example secrets.yaml
   ```
   Edit `secrets.yaml` with your WiFi credentials and generate an API key:
   ```bash
   openssl rand -base64 32
   ```

2. **Flash the device** -- Upload the config for your board:
   ```bash
   # Keypad lock (pick your board):
   esphome run staykey-keypad-lock-c3.yaml
   esphome run staykey-keypad-lock-c6.yaml

   # Spa controller:
   esphome run staykey-spa-controller.yaml
   ```

3. **Add to Home Assistant** -- The device will appear in HA under **Settings > Devices & Services > ESPHome**.

### Keypad Setup

4. **Set codes** -- Two options:

   **Option A: Learn mode (easiest)**
   - In HA, toggle the **Code Learn Mode** switch ON.
   - Type a code on the keypad (min 4 digits) and press `#` within 30 seconds.
   - The code is saved in the next available slot. The switch turns itself OFF.

   **Option B: API action**
   ```yaml
   action: esphome.staykey_keypad_lock_c3_set_code
   data:
     slot: 1
     code: "1234"
   ```

5. **Register RFID tags** -- Two options:

   **Option A: Learn mode (easiest)**
   - In HA, toggle the **Tag Learn Mode** switch ON.
   - Scan the tag on the keypad within 30 seconds.
   - The tag is saved in the next available slot. The switch turns itself OFF.

   **Option B: API action**
   ```yaml
   action: esphome.staykey_keypad_lock_c3_set_tag
   data:
     slot: 1
     tag: "0123456789"
   ```
   Use the "Last Scanned Tag" sensor value to get the tag ID if you don't know it.

### Spa Controller

Connect the ESP32-S3 to your Balboa spa's UART interface (TX on GPIO19, RX on GPIO22). The device will appear in HA with climate, fan (jets), and light entities.

## Home Assistant Blueprints

### Keypad Notifications

Get notified when codes are entered, invalid attempts occur, or the keypad locks out.

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/ammonlee/staykey-esphome/main/blueprints/keypad_notifications.yaml)

### Garage Door Toggle

Toggle a garage door open/closed when a valid code is entered on the keypad.

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/ammonlee/staykey-esphome/main/blueprints/keypad_garage_toggle.yaml)

### Example: Garage Door Toggle Automation (Manual)

If you prefer to create the automation manually instead of using the blueprint:

```yaml
alias: Keypad Garage Door Toggle
description: "Toggle garage door when keypad lock is unlocked"
triggers:
  - trigger: device
    domain: lock
    entity_id: <your_keypad_lock_entity_id>
    type: unlocked
conditions: []
actions:
  - choose:
      - conditions:
          - condition: device
            domain: cover
            entity_id: <your_garage_door_entity_id>
            type: is_open
        sequence:
          - action: cover.close_cover
            target:
              entity_id: <your_garage_door_entity_id>
      - conditions:
          - condition: device
            domain: cover
            entity_id: <your_garage_door_entity_id>
            type: is_closed
        sequence:
          - action: cover.open_cover
            target:
              entity_id: <your_garage_door_entity_id>
mode: single
```

## API Actions

The keypad lock config exposes these actions through the ESPHome API:

| Action | Parameters | Description |
|---|---|---|
| `set_code` | `slot` (1-30), `code` (string, min 4 digits) | Set a PIN code in a slot. Rejects duplicates. |
| `clear_code` | `slot` (1-30) | Clear a PIN code slot. |
| `get_codes` | _(none)_ | Returns all 30 code slots (empty string = unused). |
| `set_tag` | `slot` (1-30), `tag` (string) | Register an RFID tag in a slot. Rejects duplicates. |
| `clear_tag` | `slot` (1-30) | Clear an RFID tag slot. |
| `get_tags` | _(none)_ | Returns all 30 tag slots (empty string = unused). |
| `set_relay_pulse_time` | `seconds` (1-30) | How long the relay stays on per unlock. |
| `set_relock_time` | `seconds` (0-120) | Auto-relock delay. 0 = disabled. |

## Adding New Devices

To add a new StayKey device, create a new YAML file in the project root (e.g. `staykey-my-device.yaml`). Use one of the existing device files as a starting template and customize the `substitutions`, `esp32` board/variant, and device-specific components.

## Troubleshooting

### Keys come through as wrong numbers (inverted)

If pressing `1` shows as key `14`, `2` as `13`, etc., your **D0 and D1 wires are swapped**. In Wiegand, D0 carries 0-bits and D1 carries 1-bits. Swapping them inverts every bit in the 4-bit key code. Fix: swap the two data wires at the ESP32.

### No key presses in the log, only a "tag" read after entering full PIN

The keypad is in **standalone access-control mode** (factory default). It buffers the PIN internally and sends it as a single Wiegand card number. You need to switch to head reading mode -- see the [Keypad Setup](#keypad-setup-important) section.

### Tag reads show "invalid parity"

Usually caused by swapped D0/D1 (see above) or the keypad being in access-control mode. In head-reading mode with correct wiring, 26-bit tag reads should pass parity.

### Do I need a level shifter or voltage divider for D0/D1?

Not necessary. The Wiegand D0/D1 outputs are open-collector. They only pull the line LOW to signal a bit and float when idle. The ESP32's internal pullups hold the lines at 3.3V. Even though the keypad runs on 12V, the data lines never carry 12V -- you can wire D0 and D1 directly to the ESP32 GPIO pins. A level shifter will also work if you have one in your setup, but it is not required.

### Keypad not responding at all

- Verify the keypad is powered at **12V DC** (not 3.3V or 5V).
- Verify **GND is shared** between the keypad and the ESP32.
- Check that D0 and D1 are connected to the correct GPIO pins.

## Third-Party Components

The Balboa spa integration in `components/balboa_spa/` is vendored from [brianfeucht/esphome-balboa-spa](https://github.com/brianfeucht/esphome-balboa-spa) (commit `58a063c`). It is licensed under the BSD 2-Clause License -- see [`components/balboa_spa/LICENSE`](components/balboa_spa/LICENSE) for details. Original copyright belongs to Geoff Davis and contributors.

## License

[MIT](LICENSE)
