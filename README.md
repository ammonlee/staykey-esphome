# ESPHome Wiegand Keypad Lock

ESPHome configurations for a Wiegand keypad + RFID reader that controls a door lock through Home Assistant. Supports **ESP32-C3 Super Mini** and **Xiao ESP32-C6** boards.

![Wiring Diagram](docs/wiring-diagram.svg)

## Features

- **30 code slots** with NVS persistence (codes survive reboots)
- **30 RFID tag slots** with NVS persistence and Wiegand 26-bit card reads
- **Tag learn mode** -- flip a switch in HA, scan a tag, and it auto-registers in the next empty slot
- **5-attempt lockout** with 60-second cooldown (shared across codes and tags)
- **Configurable relay pulse** duration (1-30s, default 5s)
- **Auto-relock timer** (0-120s, 0 to disable)
- **Home Assistant API actions** for codes (`set_code`, `clear_code`, `get_codes`), tags (`set_tag`, `clear_tag`, `get_tags`), and settings (`set_relay_pulse_time`, `set_relock_time`)
- **Home Assistant events**: `esphome.keypad_code_entered` with `source` field (`code` or `tag`) and event types `code_valid`, `code_invalid`, `code_lockout`, `code_timeout`, `tag_valid`, `tag_invalid`, `tag_lockout`
- **Lock entity** in HA with full lock/unlock control
- **Diagnostic sensors**: WiFi signal, uptime, CPU temperature, IP/MAC address
- **Last Scanned Tag** text sensor -- always shows the most recently scanned tag ID

## Supported Hardware

| Component | Tested Models |
|---|---|
| **ESP32 board** | ESP32-C3 Super Mini, Seeed Xiao ESP32-C6 |
| **Keypad** | Wiegand WG26/34 keypads with RFID (125kHz EM4100) |
| **Relay** | Any 3.3V-trigger relay module |
| **Lock** | Electric door strike, magnetic lock, or garage door opener |

## Wiring

Both boards use the same pin assignments:

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

## Project Structure

```
esp32c3/
  keypad-lock.yaml     # Production lock config for ESP32-C3 Super Mini
  keypad-test.yaml     # Hardware test config for ESP32-C3 Super Mini
esp32c6/
  keypad-lock.yaml     # Production lock config for Xiao ESP32-C6
  keypad-test.yaml     # Hardware test config for Xiao ESP32-C6
blueprints/
  keypad_notifications.yaml    # HA blueprint: notify on code entry events
  keypad_garage_toggle.yaml    # HA blueprint: toggle garage door on unlock
docs/
  wiring-diagram.svg   # Wiring diagram
secrets.yaml.example   # Template for secrets.yaml
```

- **`keypad-lock.yaml`** -- The full lock configuration with code slots, lockout, relay control, relock timer, and HA integration.
- **`keypad-test.yaml`** -- A minimal test configuration for validating hardware wiring. Tests GPIO inputs/outputs, Wiegand reads, and key entry before deploying the full lock config.

## Installation

1. **Copy files** -- Copy the appropriate board directory (`esp32c3/` or `esp32c6/`) to your ESPHome config directory.

2. **Create secrets** -- Copy the secrets template and fill in your values:
   ```bash
   cp secrets.yaml.example secrets.yaml
   ```
   Edit `secrets.yaml` with your WiFi credentials and generate an API key:
   ```bash
   openssl rand -base64 32
   ```

3. **Flash the test config first** -- Upload `keypad-test.yaml` to verify wiring:
   ```bash
   esphome run keypad-test.yaml
   ```
   Press keys on the keypad and check the log for `KEY PRESS` events. Scan a card and look for `TAG` events.

4. **Switch to production** -- Once wiring is confirmed, flash `keypad-lock.yaml`:
   ```bash
   esphome run keypad-lock.yaml
   ```

5. **Add to Home Assistant** -- The device will appear in HA under **Settings > Devices & Services > ESPHome**. You'll see a Lock entity, diagnostic sensors, and configurable numbers for relay pulse and relock time.

6. **Set codes** via the HA Developer Tools > Services:
   ```yaml
   action: esphome.keypad_lock_set_code
   data:
     slot: 1
     code: "1234"
   ```

7. **Register RFID tags** -- Two options:

   **Option A: Learn mode (easiest)**
   - In HA, toggle the **Tag Learn Mode** switch ON.
   - Scan the tag on the keypad within 30 seconds.
   - The tag is saved in the next available slot. The switch turns itself OFF.

   **Option B: API action**
   ```yaml
   action: esphome.keypad_lock_set_tag
   data:
     slot: 1
     tag: "0123456789"
   ```
   Use the "Last Scanned Tag" sensor value to get the tag ID if you don't know it.

## Home Assistant Blueprints

### Keypad Notifications

Get notified when codes are entered, invalid attempts occur, or the keypad locks out.

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/ammonlee/esphome-keypad-lock/main/blueprints/keypad_notifications.yaml)

### Garage Door Toggle

Toggle a garage door open/closed when a valid code is entered on the keypad.

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/ammonlee/esphome-keypad-lock/main/blueprints/keypad_garage_toggle.yaml)

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

The lock config exposes these actions through the ESPHome API:

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

## License

[MIT](LICENSE)
