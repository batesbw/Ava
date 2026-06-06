# Ava — Development Setup

## Project

Ava is an Android voice assistant app that turns Android devices into Home Assistant smart home panels. It implements the ESPHome Native API protocol (Protobuf over TCP port 6053), so Home Assistant discovers it natively — no MQTT or HACS required.

- **Language**: Kotlin + C++ (JNI for audio frontend)
- **UI**: Jetpack Compose
- **Min SDK**: 24 (Android 7.0)
- **Target SDK**: 36 (Android 16)
- **ABI**: arm64-v8a, armeabi-v7a
- **Current version**: 0.4.7 (versionCode 47)

## Module Layout

```
app/            Main application (Kotlin)
esphomeproto/   ESPHome Protobuf definitions
microfeatures/  C++ audio frontend (MicroFrontend, KissFFT, TFLite)
```

## Test Device — Pixel 6

| Property | Value |
|---|---|
| Model | Pixel 6 |
| Codename | oriole |
| Serial | 21171FDF6001YA |
| Android | 16 (API 36) |
| Build | CP1A.260405.003.A1 |
| ABI | arm64-v8a |
| Screen | 1080×2400 @ 420 dpi |
| Connection | USB (USB debugging enabled) |

The device runs the production-signed release build. It was first installed 2026-06-02.

### ADB Quick Reference

```bash
# Connect (restart server if device not listed)
adb kill-server && adb start-server
adb devices -l

# Target device explicitly
adb -s 21171FDF6001YA shell <cmd>

# Install release APK
adb -s 21171FDF6001YA install -r app/build/outputs/apk/release/Ava-0.4.7-release.apk

# External control broadcasts
adb shell am broadcast -a com.example.ava.ACTION_WAKE
adb shell am broadcast -a com.example.ava.ACTION_TOGGLE_MIC
adb shell am broadcast -a com.example.ava.ACTION_START_SERVICE
adb shell am broadcast -a com.example.ava.ACTION_STOP_SERVICE

# View logs
adb logcat -s Ava
```

## Home Assistant Integration

Ava connects to Home Assistant via the ESPHome integration (TCP port 6053). No extra add-ons needed — HA discovers the device automatically the same way it discovers ESPHome nodes.

**Voice pipeline**: Microphone → 16 kHz PCM → on-device wake word detection (microWakeWord / TFLite) → audio stream to HA Assist pipeline → STT → intent → TTS response back to device.

**Default wake word**: "Hey Jarvis"

## Build & Sign

Release keystore lives at `~/ava-key.jks` (alias: `ava`). Build a release APK:

```bash
./gradlew assembleRelease
```

Output: `app/build/outputs/apk/release/Ava-<versionName>-release.apk`

Version is driven by `version.json` at repo root (`versionCode` + `versionName`).

## Key Features

- **Voice satellite**: ESPHome protocol, works with HA Assist pipelines
- **Bluetooth proxy**: BLE gateway to extend HA Bluetooth coverage (closed-source, release-only)
- **Floating windows**: Clock, vinyl cover, subtitles, notification scenes overlaid on any app
- **Screensaver / kiosk**: Designed for 24/7 operation with battery optimization exemption
- **Presence detection**: RSSI-based home/away triggers via Bluetooth scanning
- **Camera**: Photo and video streaming to HA
- **External control**: Broadcast intents for Tasker/automation integration
