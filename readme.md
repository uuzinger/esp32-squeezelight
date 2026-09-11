# ESP32 Spotify + Music Assistant Audio Player — Build Guide

A DIY network audio player built on **squeezelite-esp32**, supporting native Spotify Connect
playback and Music Assistant (via Slimproto/Squeezebox), with clean line-level output to a
stereo and an OLED status display.

## Why squeezelite-esp32

Music Assistant drives players over several protocols, but the two most ESP32-friendly ones are
**Slimproto** (the Squeezebox/LMS protocol) and **Spotify Connect**. Rather than building custom
firmware around separate libraries (cspot for Spotify Connect, a Slimproto client), squeezelite-esp32
already bundles both into one actively maintained project — plus optional AirPlay 1 support.

Protocols intentionally *not* targeted: DLNA/UPnP (heavier, clunkier) and Google Cast (too much
TLS/protobuf overhead for an ESP32).

## Hardware

| Component | Part | Price | Notes |
|---|---|---|---|
| MCU board | [Olimex ESP32-DevKit-Lipo (ESP32-WROVER-E)](https://www.digikey.com/en/products/detail/olimex-ltd/ESP32-WROVER-DEVKIT-LIPO/19204279) | ~$9 | **Must have 4MB PSRAM** — squeezelite-esp32's hard minimum. A plain ESP32-WROOM board (no PSRAM) will not work, even if the listing looks nearly identical. |
| DAC | [Adafruit PCM5102 I2S DAC (line-level)](https://www.digikey.com/en/products/detail/adafruit-industries-llc/6250/26249971) | ~$5 | Chosen over a class-D amp module (e.g. MAX98357A) because the target is a stereo's line-in, not a speaker — the PCM5102 has no amplification stage, giving a cleaner, lower-power signal matched to line-in impedance. |
| Display | [Generic 0.96" SSD1306 OLED, 128×64, I2C, 4-pin](https://www.amazon.com/dp/B0D2RMQQHR) | ~$3.33 (3-pack ~$9.99) | Confirm I2C address on the module (0x3C is standard; some ship 0x3D). |

### Wiring

| Signal | ESP32 GPIO | DAC / Display pin |
|---|---|---|
| I2S Bit Clock (BCK) | GPIO33 | DAC `BCK` |
| I2S Word Select (WS/LRCK) | GPIO25 | DAC `WSEL` |
| I2S Data (DIN) | GPIO32 | DAC `DIN` |
| DAC Power | 3.3V | DAC `VIN` (**not** the DAC's `3V` pad — that's a regulated *output*, not an input) |
| I2C Clock | GPIO22 | OLED `SCL` |
| I2C Data | GPIO23 | OLED `SDA` |
| OLED Power | 3.3V | OLED `VCC` |
| Ground | GND | Common ground across ESP32, DAC, and OLED |

**Gotchas we hit:**
- **WSEL and DIN are easy to cross** — they're adjacent and both carry digital signal, but a swap
  produces *silence*, not noise or distortion, since the DAC can't lock onto a valid frame. If audio
  is completely silent but everything else (display, network, DAC power) checks out, verify WSEL and
  DIN against your `dac_config` before anything else.
- The DAC's top-row control pins (`DE`, `FIL`, `MCK`, `MU`, `FM`) can be left unconnected on the
  Adafruit PCM5102 board — its onboard resistor network already biases them to sensible defaults
  (I2S format, internal PLL clock, unmuted, normal filter, de-emphasis off). No jumper needed.
- Double-check the OLED module's actual pin order printed on its silkscreen before wiring — cheap
  4-pin I2C breakouts aren't consistent about pin order (some are GND-VCC-SCL-SDA, others differ).

## Flashing

1. Go to the [squeezelite-esp32 web installer](https://sle118.github.io/squeezelite-esp32-installer/)
   in **Chrome or Edge** (needs WebSerial — Safari/Firefox won't work).
2. Leave **Preset Options / Known Board Name** as `--` unless your board is one of the specific
   named presets (SqueezeAmp, ESP32-A1S variants, etc.) — picking one overwrites your custom GPIO
   assignments.
3. Flash the **generic I2S** build. This first flash only installs a minimal recovery/bootstrap
   app — the page title will show `[recovery]`.
4. The device broadcasts a temporary setup AP named `squeezelite-<id>`.
   Default password: **`squeezelite`**.
5. Connect to that AP; a captive portal should prompt for your home WiFi credentials (or browse to
   `192.168.4.1` if it doesn't appear automatically).
6. Once joined to your network, go to the device's **Updates** tab and flash the actual application:
   platform **I2S-4MFlash**, and use the **16-bit** build (not 32-bit) — the maintainer designed
   16-bit as the default to preserve CPU/memory headroom, since the chip is already stretched
   running WiFi + Spotify Connect + Slimproto together. 32-bit only matters if you need heavy
   in-device DSP (EQ, high-quality resampling).
7. After this second flash and reboot, the `[recovery]` tag should disappear — that's the real
   application running.

## NVS Configuration

Reachable via the device's web UI → **NVS Editor**.

**DAC Options:**

| Field | Value |
|---|---|
| DAC Model Name | `I2S` |
| Clock GPIO | `33` |
| Word Select GPIO | `25` |
| Data GPIO | `32` |
| Mute GPIO / SDA / SCL / I2C address | leave blank — not applicable to the PCM5102 (no I2C control) |

**I2C Bus Parameters** (for the display):

| Field | Value |
|---|---|
| Port | `1` |
| Frequency | `400000` |
| SDA GPIO | `23` |
| SCL GPIO | `22` |

**Display:**

| Field | Value |
|---|---|
| Interface | `I2C` |
| Driver | `SSD1306` |
| I2C address | `60` (0x3C) |
| Width | `128` |
| Height | `64` |

Equivalent raw config strings, for reference:
```
dac_config: model=I2S,bck=33,ws=25,do=32
display_config: I2C,driver=SSD1306,width=128,height=64,sda=23,scl=22,address=60,port=1
```

## Home Assistant Integration

No ESPHome needed. Once the device is added to Music Assistant as a Squeezebox/LMS player
(auto-discovered via Slimproto), it automatically shows up as a Home Assistant `media_player`
entity through the Music Assistant integration — with full playback control, state, and progress.
ESPHome's media_player component has no Spotify Connect or Slimproto support, so switching to it
would mean losing native Spotify Connect for no HA-integration benefit, since MA already bridges
squeezelite-esp32 into HA.

## Known Issue

The OLED's now-playing display doesn't use the full 128×64 canvas when showing **Spotify Connect**
metadata — text is confined to a smaller region even with width/height correctly configured. This
appears to be a limitation of cspot's display integration rather than a wiring or config problem;
the maintainer describes Spotify/AirPlay/Bluetooth support as "add-ons stitched to" the core
LMS/Squeezebox design, with some rough edges. Worth checking whether Music Assistant/Slimproto
playback renders correctly at full resolution as a way to isolate whether this is specific to the
Spotify Connect path.
