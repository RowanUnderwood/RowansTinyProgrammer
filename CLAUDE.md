# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

TinyProgrammer is an autonomous Raspberry Pi device that writes, runs, and watches Python programs in an infinite loop. It renders output to an SPI TFT display (480×320) using a retro Mac OS IDE aesthetic, and has a Flask web dashboard for monitoring and configuration.

This is a fork of [cuneytozseker/TinyProgrammer](https://github.com/cuneytozseker/TinyProgrammer) with these additions: pygame 32-bit SPI display fix, `/api/shutdown` route, LM Studio provider support, and a provider/model split settings UI.

---

## Deployment Workflow

There is no local run environment — the code runs on a Raspberry Pi. The workflow is:

1. **Edit** files here in `RowansTinyProgrammer/`
2. **Deploy** changed files to the Pi via `pscp.exe` (in the parent `tinyprogramer/` directory)
3. **Restart** the service and verify on device

```bash
# Deploy a changed file (run from parent tinyprogramer/ directory)
pscp.exe -pw raspberry -batch RowansTinyProgrammer/<path/to/file> pi@192.168.2.180:/home/pi/TinyProgrammer/<path/to/file>

# Restart service
plink.exe -ssh pi@192.168.2.180 -pw raspberry -batch "echo raspberry | sudo -S systemctl restart tinyprogrammer"

# Check logs
plink.exe -ssh pi@192.168.2.180 -pw raspberry -batch "echo raspberry | sudo -S tail -n 50 /var/log/tinyprogrammer.log"

# Check service status
plink.exe -ssh pi@192.168.2.180 -pw raspberry -batch "echo raspberry | sudo -S systemctl status tinyprogrammer --no-pager"
```

Pi details: IP `192.168.2.180` (wireless), user `pi`, password `raspberry`, project path `/home/pi/TinyProgrammer`.

---

## Architecture

### The Main Loop

`main.py` boots all subsystems then hands control to `Brain.run()`. The brain cycles through states indefinitely:

```
THINK → WRITE → REVIEW → RUN → WATCH → ARCHIVE → REFLECT
                                  ↓ (30% chance)
                              BBS_BREAK → back to THINK
```

- **THINK**: Picks mood-influenced creative dimensions, selects program type (variation/core/creative split), chooses LLM model, retrieves past lessons
- **WRITE**: Streams code from LLM token-by-token, types it to the terminal with human-like delays
- **REVIEW**: Checks for banned imports, validates syntax via `compile()`; sends to FIX on error (max 2 attempts)
- **RUN**: Writes code to `programs/temp_execution.py`, runs it in a subprocess with restricted env
- **WATCH**: Reads subprocess stdout for `CMD:` draw commands and renders them to the canvas popup; monitors for 20–120s
- **ARCHIVE**: Saves code + metadata, updates mood based on success/failure streak
- **REFLECT**: Streams an LLM lesson about what went wrong/right, stores it for future THINK cycles
- **BBS_BREAK**: Visits the TinyBBS bulletin board — reads boards, optionally posts based on mood

### Three-Way Program Selection (THINK state)

Every cycle, `brain._do_think()` picks a generation mode with this priority:

1. **Variation** (15% if liked programs exist) — remixes a user-liked program
2. **Core** (50%) — picks from `config.CORE_PROGRAMS`, uses simpler prompt
3. **Creative** (remaining ~35%) — full creative dimensions (style, palette, inspiration seed)

### Mood System

`Personality` tracks a `Mood` enum (hopeful/focused/curious/proud/frustrated/tired/playful/determined) updated after each ARCHIVE based on consecutive success/failure streaks and time of day. Mood affects:
- Which program categories are preferred (`creativity.MOOD_CREATIVITY`)
- Creative style and palette choices
- BBS break probability (+20% if tired/playful, −15% if focused/determined)
- THINKING_COMMENTS shown on display

### Creative Dimensions (`programmer/creativity.py`)

`pick_creative_dimensions(mood)` returns a dict with `style`, `palette`, `inspiration_seed` (60% chance), and `directive`. Program types are grouped into `CATEGORIES` (motion/grid/generative/natural/abstract/math); mood biases selection toward preferred categories with 4× weight.

---

## Key Subsystems

### LLM Integration (`llm/generator.py`)

Three providers, routed by model ID prefix:

| Prefix | Provider | Backend |
|--------|----------|---------|
| `ollama/` | Ollama | `POST {OLLAMA_ENDPOINT}/api/generate` |
| `lmstudio/` | LM Studio | `POST {LMSTUDIO_ENDPOINT}/v1/chat/completions` |
| anything else | OpenRouter | `POST https://openrouter.ai/api/v1/chat/completions` |

Endpoints are read from `config` at call time (not import time) via `_get_ollama_endpoint()` and `_get_lmstudio_endpoint()` — this means web UI endpoint changes take effect immediately without a restart.

Local models (`ollama/`, `lmstudio/`) receive a simplified prompt (`_build_simple_prompt`) due to weaker instruction-following. Cloud models get the full creative prompt.

`SURPRISE_ME = "surprise_me"` picks a random cloud model each program cycle. `SURPRISE_ME_LOCAL` picks a random Ollama model from the live server.

### Display Pipeline (`display/`)

`Terminal` renders an in-memory pygame surface, then `FramebufferWriter` converts it to RGB565 and writes directly to `/dev/fb2` (bypassing SDL's broken fbcon driver). The pygame 32-bit fix in `_init_display()` is critical — without `pygame.display.set_mode(..., 0, 32)` before creating any `pygame.Surface`, pygame 1.9.x defaults to 8-bit palette mode and all drawing produces black pixels.

`frame_stream.py` maintains a rate-limited JPEG buffer (10fps cap) for the Flask `/stream` MJPEG endpoint, independent of the display framerate.

### Configuration System (`web/config_manager.py`)

Two layers of config:
1. **`config.py`** — base defaults, checked into the repo
2. **`config_overrides.json`** — persisted dashboard settings, in project root

`ConfigManager.save_overrides(updates)` writes to JSON and immediately applies via `setattr(config, key, value)`, so dashboard changes are live without a restart. `config_mgr.get_all()` returns the merged view.

Settings that apply immediately: LLM model switch, color scheme, LLM endpoint.  
Settings that apply on next cycle: timing, personality values.

### Web UI (`web/app.py`, `web/templates/`)

Flask app started in a daemon thread by `main.py`. Key routes:

| Route | Purpose |
|-------|---------|
| `GET /` | Dashboard with status, session history |
| `GET/POST /settings` | All config overrides |
| `GET/POST /prompt` | Program type weights, descriptions, custom types |
| `POST /api/restart` | Skip to next program cycle |
| `POST /api/shutdown` | `sudo shutdown -h now` |
| `POST /api/like` | Add current program to liked store |
| `GET /api/local-models?provider=&endpoint=` | Detect live models from Ollama or LM Studio |
| `GET /api/screenshot` | Current display surface as PNG |
| `GET /stream` | MJPEG live stream (requires `WEB_STREAM_ENABLED=true`) |

### Canvas Drawing (generated programs)

Generated programs use `tiny_canvas.py` (`Canvas` class). Draw commands are printed to stdout as `CMD:COMMAND,args...` (e.g., `CMD:FILLRECT,10,10,100,100,255,0,0`). The brain's WATCH state reads these and calls `terminal.process_draw_command()` to render them onto the canvas popup surface.

### BBS Integration (`bbs/client.py`)

Connects to TinyBBS via Supabase REST + Edge Functions. Device identity is derived from the Pi's serial number and stored in `~/.tinyprogrammer/bbs_token`. Boards: `code_share` (threaded), `chat`, `news`, `science_tech`, `jokes`, `lurk_report` (all flat feeds).

### Liked Programs (`programmer/liked_store.py`)

`liked_programs.json` stores up to 20 user-liked programs. Weighting favours less-remixed programs (`1 / (times_remixed + 1)`). Users like programs via the dashboard heart button. The THINK state has a 15% chance to enter variation mode if the store is non-empty.

---

## Important Config Variables

| Variable | Where set | Purpose |
|----------|-----------|---------|
| `DISPLAY_PROFILE` | `.env` | `pizero-spi` (480×320) or `pi4-hdmi` (800×480) |
| `LLM_MODEL` | `.env` / dashboard | Full model ID e.g. `ollama/qwen3:8b` |
| `OLLAMA_ENDPOINT` | `.env` / dashboard | Base URL, no path e.g. `http://192.168.2.83:11434` |
| `LMSTUDIO_ENDPOINT` | `.env` / dashboard | Base URL e.g. `http://localhost:1234` |
| `FB_DEVICE` | `.env` | Framebuffer device, e.g. `/dev/fb2` |
| `FB_ROTATION` | `.env` | `0`/`1`/`2`/`3` (90° increments) |
| `CORE_PROGRAMS` | `config.py` | Curated list for core-mode rotation |
| `WEB_STREAM_ENABLED` | env var | Enable MJPEG stream (CPU-heavy, off by default) |

Dashboard-saved overrides live in `config_overrides.json` at the project root. The `.env` file at `/home/pi/TinyProgrammer/.env` is not checked in — it holds API keys and device-specific settings.
