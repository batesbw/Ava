# Pixel 6 Dev Setup & Troubleshooting Guide

This document covers the concrete hardware setup for the Pixel 6 running Ava, the on-device llama.cpp instance, the voice pipeline integration with Home Assistant, and how to connect remotely for troubleshooting.

---

## Hardware Overview

| Property | Value |
|---|---|
| Device | Pixel 6 (oriole) |
| Serial | 21171FDF6001YA |
| Android | 16 (API 36) |
| IP address | 192.168.1.127 |
| ADB port | 5555 (wireless) |

---

## Wireless ADB

ADB over WiFi is enabled on port 5555. To connect from any machine on the same network:

```bash
adb connect 192.168.1.127:5555
adb -s 192.168.1.127:5555 shell echo ok
```

If the connection drops (e.g. after a reboot or ADB daemon restart), plug in via USB and re-enable TCP mode:

```bash
adb kill-server && adb start-server
adb -s 21171FDF6001YA tcpip 5555
adb connect 192.168.1.127:5555
```

Verify both transports are active:

```bash
adb devices -l
# Should show both:
#   21171FDF6001YA   device usb:...
#   192.168.1.127:5555  device ...
```

---

## On-Device llama.cpp (Termux)

The Pixel 6 runs a `llama-server` instance inside Termux, serving as the local LLM backend for Home Assistant's conversation agent.

### Process

```
llama-server
  --model /data/data/com.termux/files/home/models/gemma-2-2b-it-Q8_0.gguf
  --alias gemma-2-2b
  --host 0.0.0.0
  --port 8080
  --ctx-size 4096
  --n-predict 512
  --n-gpu-layers 99
  --threads 4
  --log-disable
```

### Check status

```bash
# From dev machine (after port forwarding)
adb -s 192.168.1.127:5555 forward tcp:8080 tcp:8080
curl http://localhost:8080/health
# {"status":"ok"}

curl http://localhost:8080/props | python3 -m json.tool
```

### Model: Gemma 2 2B Instruct (Q8_0)

The Gemma 2 chat template enforces two hard constraints that are the root cause of the voice pipeline error:

```jinja
{# 1. System role is not supported — throws immediately #}
{% if messages[0]['role'] == 'system' %}
  {{ raise_exception('System role not supported') }}
{% endif %}

{# 2. Roles must strictly alternate user/assistant/user/assistant #}
{% if (message['role'] == 'user') != (loop.index0 % 2 == 0) %}
  {{ raise_exception('Conversation roles must alternate user/assistant/user/assistant/...') }}
{% endif %}
```

**`--n-gpu-layers 99` note:** The Pixel 6 GPU (Mali-G78) does not support Vulkan compute for GGML. All layers fall back to CPU. This flag has no effect on this device and can be removed or set to `0`.

---

## Voice Pipeline Architecture

```
Pixel 6
  └── Ava app (VoiceSatelliteService)
        │  ESPHome Native API (TCP :6053)
        ▼
  Home Assistant
        └── Assist Pipeline
              ├── STT (Whisper or similar)
              ├── Conversation Agent ──► llama-server on Pixel 6 (:8080)
              └── TTS (EdgeTTS or similar)
                    │  TTS audio URL
                    ▼
              Ava app plays response
```

HA calls the on-device llama-server via the OpenAI-compatible `/v1/chat/completions` endpoint at `http://192.168.1.127:8080`.

---

## Known Issue: Voice Pipeline Speaks Raw Error Text

### Symptom

The voice assistant speaks the entire error message aloud, e.g.:

> "Sorry, there was a problem talking to the backend: HomeAssistantError Failed to communicate with the API Error code 400 Unable to generate parser for this template Conversation roles must alternate user assistant..."

### Root Cause

HA's conversation agent sends a message list to llama-server that violates Gemma 2's alternating-role constraint. The llama-server returns a 400 error. HA formats that error as a string and passes it to TTS. TTS generates audio for the full error string. Ava plays it — correctly, from its perspective.

This is **not an Ava bug**. Ava faithfully plays whatever TTS URL Home Assistant sends.

### Log Evidence

From `adb logcat --pid=$(adb shell pidof com.example.ava)`:

```
VoiceSatelliteStateMachine: TTS_START received
VoiceSatellite: VoiceAssistantEventResponse: eventType=VOICE_ASSISTANT_TTS_START,
  data=[text=Sorry, there was a problem talking to the backend:
  HomeAssistantError('Failed to communicate with the API! Error code: 400 -
  {"error": {"code": 400, "message": "...Conversation roles must alternate
  user/assistant/user/assistant/...", "type": "invalid_request_error"}}')]
```

### Likely Triggers

1. **HA sends a system prompt** — Gemma 2 rejects `role=system` outright. Check whether the HA conversation agent config includes a system prompt.
2. **Back-to-back user messages** — If a second voice turn starts before the previous assistant reply is recorded in history, HA may send two consecutive `user` messages.
3. **Continue-conversation history mismatch** — Session resumes with stale history where the last role was `user`, then HA appends another `user` message.

### Fix Options

#### Option A — Remove the system prompt in HA (easiest)

In the HA conversation agent configuration for the llama-server backend, clear the system prompt field entirely. Gemma 2 does not support it.

#### Option B — Swap to a model that supports system roles

Llama 3.2 3B Instruct or Phi-3 Mini handle system prompts and do not enforce strict alternation. Drop the GGUF into `/data/data/com.termux/files/home/models/` and update the `--model` path in the Termux startup script.

#### Option C — Middleware prompt normalizer

Run a small proxy between HA and llama-server that:
- Folds any `system` message into the first `user` message as a prefix
- Merges consecutive same-role messages

#### Option D — Reset conversation history on error

Configure the HA conversation agent to clear its history when a 400 error is returned, rather than retrying with the same (broken) history.

---

## Viewing Ava Logs

Ava does not use a single `Ava` log tag. Filter by process instead:

```bash
# Wireless ADB
adb -s 192.168.1.127:5555 logcat \
  --pid=$(adb -s 192.168.1.127:5555 shell pidof com.example.ava | tr -d '\r') \
  2>&1 | grep -E "VoiceSatellite|TtsPlayer|Server|ClientConnection|MicrophoneInput"
```

Key tags:

| Tag | What it covers |
|---|---|
| `VoiceSatellite` | Main coordinator, ESPHome protocol events |
| `VoiceSatelliteStateMachine` | State transitions (Listening → Processing → Responding) |
| `VoiceSatellitePlayer` | TTS and media playback |
| `TtsPlayer` | TTS audio playback detail |
| `MicrophoneInput` | Audio capture / AGC |
| `Server` / `ClientConnection` | TCP :6053 ESPHome connection |
| `SendspinSocketServer` | Internal IPC socket (ignore `Address already in use` — benign on restart) |

---

## External Control (Broadcast Intents)

```bash
adb -s 192.168.1.127:5555 shell am broadcast -a com.example.ava.ACTION_WAKE
adb -s 192.168.1.127:5555 shell am broadcast -a com.example.ava.ACTION_TOGGLE_MIC
adb -s 192.168.1.127:5555 shell am broadcast -a com.example.ava.ACTION_START_SERVICE
adb -s 192.168.1.127:5555 shell am broadcast -a com.example.ava.ACTION_STOP_SERVICE
```

---

## Build & Install

```bash
# Build signed release APK
./gradlew assembleRelease

# Install over wireless ADB
adb -s 192.168.1.127:5555 install -r \
  app/build/outputs/apk/release/Ava-0.4.7-release.apk
```

Keystore: `~/ava-key.jks` (alias: `ava`). Version driven by `version.json` at repo root.

---

## HA-Side Troubleshooting Checklist

For a Claude Code instance with full HA access (e.g. Unraid instance):

- [ ] Check the conversation agent config — is a system prompt set? If yes, clear it.
- [ ] Inspect recent Assist pipeline runs in HA → **Settings → Voice assistants → [pipeline] → Debug**
- [ ] Confirm the conversation agent is pointed at `http://192.168.1.127:8080/v1` (OpenAI-compatible)
- [ ] Check what model/context the agent is sending: enable HA debug logging for `homeassistant.components.conversation`
- [ ] Verify the message list in the 400 error — count consecutive user/assistant roles
- [ ] If using `continue_conversation`, check whether history is cleared correctly after a failed turn
- [ ] Consider switching the pipeline to use a standard HA built-in conversation agent (no LLM) to confirm Ava itself is healthy, then re-enable llama-server
