# onju-voice-ha-satellite

ESPHome configuration for the [Onju Voice](https://github.com/justLV/onju-voice) board (ESP32-S3 replacement PCB
for the Google Nest Mini) as a Home Assistant voice satellite, with on-device wake word (microWakeWord
"Okay Nabu") and acoustic echo cancellation.

Based on the [Onju Voice ESPHome gist](https://gist.github.com/rmeissn/a6bc1c91f65a47cb5e37d6e2fcfa8849) by
Roy Meissner (rmeissn). See [Credits](#credits).

## What it uses

- [n-IA-hane/esphome-audio-stack](https://github.com/n-IA-hane/esphome-audio-stack): `esp_audio_stack` (full-duplex
  I2S shared by the SPH0645 mics and the DAC) and `esp_aec` (software echo cancellation, 16 kHz).
- `micro_wake_word` with `okay_nabu` (20241226.3), `voice_assistant`, `media_player: speaker` with a mixer
  (announcement + media + intercom) and per-stream resamplers.
- Touch buttons (VOL-, VOL+, center), LED ring states, mute switch, timers, alarm siren.

## Setup

1. Add these keys to your ESPHome `secrets.yaml`: `ha_url` (e.g. `http://homeassistant.local:8123`), `wifi_ssid`,
   `wifi_password`, `onju_api_key`.
2. Copy the files in [`sounds/`](sounds/) to Home Assistant `www/resource/` (served at `/local/resource/`). They must be
   **FLAC, mono, 48 kHz, 16-bit**: the speaker media player rejects 24-bit files ("Incompatible bits per sample"),
   and then the wake word action waits forever for the beep.
3. Set `name` in `substitutions` to your device name, then compile and flash.

The sounds are fetched over plain HTTP from `ha_url`. This is fine on a trusted LAN; use an `https://` URL otherwise.
`esp_audio_stack` is pinned to a commit: update the hash on purpose, after reading the upstream changes.

## Behavior

- Center button, single click: stop the timer / announcement / assistant; if music is playing or paused, pause or
  resume it; when idle, start the assistant.
- Center button, double click: start the assistant, also while music plays (`on_start` ducks the music by 30 dB).
- Double / triple / long press: exposed as an `event` entity for automations. Long press also resets the audio
  components and restarts the wake word.
- The wake word stays on while music plays (the AEC removes the echo). A 5 s watchdog restarts it if nothing else
  did, e.g. after an OTA flash.
- `Echo Cancellation` switch: turns the AEC off at runtime, to compare.
- `Wake word sensitivity` select: per-model probability cutoffs (0.85 / 0.75 / 0.69 / 0.56).

## Current tuning

| Setting | Value | Why |
|---|---|---|
| `voice_assistant` `gain_factor` | 1 | higher values clipped the STT audio at 0 dBFS |
| `voice_assistant` `auto_gain` / `noise_suppression_level` | 0 dBFS / 0 | stacked with HA's own processing; cut soft consonants |
| `micro_wake_word` `gain_factor` | 8 | 32 saturated the mic with music playing |
| `esp_aec` `mode` / `filter_length` | `sr_low_cost` / 8 | `sr_*` is recommended with a wake word; 8 = 128 ms tail |
| `mic_channel` | right | one mic; stereo capture lowered the SNR |

## Known limits

- **Wake word while music plays** works, but less reliably than in silence: the Onju has no hardware loopback, so
  the AEC uses a software reference of the speaker output. Use the double click when it does not fire.
- Free internal heap is low (~30-45 KB with music and the assistant running), so `esp_afe` with both mics may not fit.

## Credits

- Roy Meissner ([@rmeissn](https://github.com/rmeissn)): the base configuration,
  [gist](https://gist.github.com/rmeissn/a6bc1c91f65a47cb5e37d6e2fcfa8849), also published as
  [rmeissn/onju-voice-esphome](https://github.com/rmeissn/onju-voice-esphome) (MIT).
- [n-IA-hane/esphome-audio-stack](https://github.com/n-IA-hane/esphome-audio-stack) (MIT): `esp_audio_stack`, `esp_aec`.
- [kahrendt/microWakeWord](https://github.com/kahrendt/microWakeWord): the `okay_nabu` model.
- [justLV/onju-voice](https://github.com/justLV/onju-voice): the hardware.
- Sounds: authors and licenses (CC BY 4.0, CC0) in [`sounds/CREDITS.md`](sounds/CREDITS.md).

## License

MIT, see [LICENSE](LICENSE).
