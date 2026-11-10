# ESP32-S3 Home Assistant Voice

ESPHome configuration package for a **bare ESP32-S3 (N16R8) voice satellite** built from
an I2S MEMS microphone and an I2S amplifier/DAC — a minimal fork of the Nabu Casa
*Home Assistant Voice PE* config, retuned for this hardware.

Use it by pointing a small device YAML at the package:

```yaml
packages:
  Nabu Casa.Home Assistant Voice PE: github://rpeng6/ESP32-S3-Home-Assistant-Voice/ESP32-S3-Voice-Assistant.yaml
```

## Files

| File | Purpose |
|---|---|
| `ESP32-S3-Voice-Assistant.yaml` | The package. Everything except per-board pin mapping. |
| `examples/device-template.yaml` | Copy this to start a new satellite. |
| `dynamic-volume.yaml` | Optional dynamic volume package. |
| `home-assistant-voice.yaml` | Alternate/upstream-referenced config. |

## Tuning applied (measured on real hardware)

Audio output chain, sized to the hardware and to the TTS source:

| Setting | Value | Why |
|---|---|---|
| speaker `sample_rate` | 24000 | matches the TTS native rate, halves decode/mix work |
| speaker `channel` | mono | single mono amplifier |
| speaker `buffer_duration` | 300ms | ~14 KB in internal RAM. At 48 kHz stereo a 1000 ms buffer needs ~192 KB and lands in PSRAM |
| mixer | mono, default source timeouts, `task_stack_in_psram` | `timeout: never` lets an idle media source stall an announcement mid-word |
| media_player pipelines | 24000 Hz, `num_channels: 1` | mono announcement + media |
| microphone | `correct_dc_offset: true`, `use_apll: true` | MEMS mics carry a DC bias; APLL gives an accurate 16 kHz clock |

Wake word input chain:

| Setting | Value | Why |
|---|---|---|
| `micro_wake_word.microphone.gain_factor` | 4 | the block is optional and defaults to **1x**, while the voice-assistant path runs 4x — without it the wake models get a quarter of the signal |
| `okay_nabu` cutoff | 0.62 | model default 0.85 |
| `hey_jarvis` cutoff | 0.85 | model default 0.97 |
| models | okay_nabu, hey_jarvis, stop | kenobi/hey_mycroft removed; only *enabled* models cost CPU |

Measured effect: one announcement went from **10.65 s** (source re-opened mid-reply,
mixer start/stop cycles, 2 slow-component warnings) to **6.04 s** continuous with zero
warnings on the same sentence.

## Starting a new satellite

1. Copy `examples/device-template.yaml` into your ESPHome config directory.
2. Fill in the substitutions: device name, OTA password, API encryption key, and your
   board's I2S pins (`i2s_lrclk`, `i2s_bclk`, `i2s_ws`, `i2s_sck`, `mic`, `spk`, plus
   `ledpin`, `vol_up`, `vol_down`).
3. First flash over USB; afterwards OTA works from the ESPHome dashboard.

## Hardware assumptions

- **Two separate I2S buses** — one for the microphone, one for the amplifier
  (`i2s_audio` blocks `i2s_input` / `i2s_output`). This matters: a board that shares one
  bus between mic and speaker cannot run full duplex in stock ESPHome, and with ESPHome
  ≤ 2026.10 it can deadlock permanently on `Parent bus is busy` while the mic holds the
  bus. Keep them separate, or run ESPHome ≥ 2026.11 which fixes the underlying race
  (esphome/esphome#21380).
- Mono speaker/amplifier. Stereo hardware works but needs `channel: stereo` and a
  matching mixer `num_channels`.
- ESPHome **2026.10.x or newer** — validated on 2026.10.2 and 2026.11.0b3.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `[W][micro_wake_word] Not enough free bytes in ring buffer` | inference task starving: reduce display/touchscreen polling rate, check heap |
| `[W][component] display took a long time` during speech | a forced full-page redraw in `on_play`/`on_announcement` is blocking the loop; remove `component.update` and let the display interval paint |
| Speaker retries `Parent bus is busy` forever | shared mic/speaker I2S bus + ESPHome ≤ 2026.10 mic-stop race — see above |
| Wake word needs shouting | set `micro_wake_word.microphone.gain_factor`, lower `probability_cutoff`, raise the ADC PGA (`audio_adc: mic_gain`) |

## Wake word sensitivity select (ported from upstream, PR #558)

The package now ships a `Wake word sensitivity` select entity with the maintainers'
measured presets. Cutoffs are quantized `uint8` values (`round(probability * 255)`):

| Preset | okay_nabu | hey_jarvis | False accepts/hour |
|---|---|---|---|
| Slightly sensitive | 0.85 (217) | 0.97 (247) | 0.00 / 0.56 |
| Moderately sensitive | 0.69 (176) | 0.92 (235) | 0.38 / 0.94 |
| Very sensitive | 0.56 (143) | 0.83 (212) | 0.75 / 1.50 |

Pick it in Home Assistant instead of reflashing. It survives reboots (`restore_value`),
and it overrides the `probability_cutoff` values written in the YAML at runtime.

## Sound files are now built into the firmware

The chimes (`wake_word_triggered`, `timer_finished`, `mute_switch_*`, button clicks,
factory-reset and error sounds) used to be fetched from `raw.githubusercontent.com` at
play time - an HTTPS round trip to GitHub in the middle of the wake path. They are now
referenced as local paths (`sounds/<name>`) and **embedded in the firmware**.

**Requirement:** the `sounds/` folder from this repo must exist inside your ESPHome
config directory (next to the device YAML):

```bash
cp -r sounds/ /config/esphome/sounds/     # add-on
# or, for a CLI/docker build: copy sounds/ into the mounted /config directory
```

A build without them fails at compile time with a missing-file error, so it cannot ship
silently broken firmware.

## Also ported from upstream

- `on_intent_progress` - starts the reply phase as soon as Home Assistant hands over a
  streaming TTS URL, and arms the stop word once (instead of double-arming it when
  `on_tts_start` fires too).
