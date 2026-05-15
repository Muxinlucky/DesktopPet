# SmartTodo Assistant

[简体中文](README.md)

> A desktop intelligent assistant powered by LLM intent parsing and system audio awareness -- Tauri v2 + Rust kernel, neon floating UI, zero-framework pure DOM rendering.

## Core Features

### Smart Intent Todo

Connects to DeepSeek or other LLM APIs. Natural language input is automatically classified as either an alarm reminder or a permanent memo.

```
"remind me to drink water in 30 min"  -->  countdown timer, looping alarm + visual pulse
"learn deep learning"                  -->  permanent memo, persisted to localStorage
"call me for a meeting in 1 hour"     -->  countdown timer
```

Ambiguous time expressions fall back to LLM semantic parsing -- no strict format required.

### AI Persona Chat

Connects to any OpenAI-compatible API with six built-in personality presets and a custom persona panel:

| Preset | Style |
|--------|-------|
| Tsundere Cat | Aloof exterior, secretly caring |
| Genki Girl | Bubbly energy, exclamation mark overflow |
| Toxic Friend | Sarcastic roasts, zero malice |
| Gentle Senior | Warm, patient, soft-spoken |
| Chuunibyou | Dramatic monologues, delusional flair |
| Zen Shiba | Stoic philosophy, minimal words |

First click triggers a full LLM response, then falls back to lightweight quotes. 10-minute cooldown before the next LLM trigger -- balancing experience and cost.

### System Audio Awareness

Real-time peak detection via Windows Core Audio API at the system output level. When music is playing, particles float to the beat. When silent, the pet stays quiet.

### Geek Aesthetic UI

Dark neon floating panels -- frosted glass backgrounds, rounded glowing borders, custom neon scrollbars. All panels are overlay layers, never intruding on the desktop layout.

### Deep System Integration

- Auto-start on boot (Tauri autostart plugin)
- Always-on-top (preference persisted across sessions)
- File drop to recycle bin (drag files onto the pet to delete)
- GitHub commit monitoring (pet cheers when new commits are detected)

### Focus Mode

5 / 15 / 30 / 45 / 60 minute Pomodoro timer. During focus, the pet stays quiet and displays a countdown. A bell rings when time is up.

## Quick Start

**Install**

Download the latest installer (`.msi` or `setup.exe`) from the [Releases](../../releases) page. Launch after installation.

**Configure API**

Right-click the pet > Chat Mode > Connect API, and fill in:

| Field | Description |
|-------|-------------|
| Endpoint | API service URL (any OpenAI-compatible endpoint, `/v1` auto-appended) |
| Key | Your API key |
| Model | Model name (defaults to `gpt-3.5-turbo`) |

Click "Save" to test connectivity. Green means success.

## Basic Controls

| Action | Effect |
|--------|--------|
| Left click | Pet greets you, spawns particle effects |
| Click and drag | Pet follows cursor, bounces on release |
| Right click | Opens context menu |
| Drop files on pet | Files sent to recycle bin |
| Long idle | Pet zones out, thinks, then falls asleep |
| Move mouse to wake | Pet jumps up in surprise |

## Development

```bash
npm install            # Install dependencies
npm run tauri dev      # Development mode (hot reload)
npm run tauri build    # Production build (generates installer)
```

Build output is located at `src-tauri/target/release/bundle/nsis/`.

## Technical Specs

- **Runtime**: Tauri v2 (Rust backend + WebView2 frontend)
- **Frontend**: Vite + TypeScript, zero-framework pure DOM rendering
- **Animation**: CSS `steps()` sprite engine, no Canvas/WebGL
- **Installer**: NSIS, customizable install directory

See [ARCHITECTURE.md](ARCHITECTURE.md) for detailed architecture documentation.

## Pet Package Format

Supports importing custom pets (`.zip` files):

```
pet.json              -- Pet manifest (id, displayName, description, spritesheetPath, version)
spritesheet.webp      -- Sprite sheet (8 columns x N rows, 192x208 per cell)
```

Right-click > Settings > Import Pet (.zip) to swap.

## License

MIT
