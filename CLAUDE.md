# CLAUDE.md — Swale

Project-level instructions for Claude Code. These complement the user's global
`~/.claude/CLAUDE.md`; anything here is specific to Swale.

## What this project is

Swale is a free, open-source **desktop wallpaper app for macOS and Windows**. A
user pastes a YouTube link and a time range; Swale downloads only that range,
fits it to the screen and loops it on the desktop. It also handles 2K / 4K
static wallpapers, keeps everything in a library, finds the most
wallpaper-worthy moments in a video, and (later) produces AI-directed edits.
The website hosts a landing page and a community that shares **recipes**
(source + range + edit plan JSON), never video files.

> Status: design phase, no code. The source of truth is
> [`docs/DESIGN.md`](docs/DESIGN.md). Read it before starting work, and update
> its decision log whenever a decision changes.

## Stack (planned)

- **Desktop shell:** Tauri 2. Core in Rust, UI in React + TypeScript.
  Targets macOS 14+ (Apple Silicon only) and Windows 11.
- **Local DB:** SQLite (embedded), metadata only; media files on disk.
- **macOS player:** Swift helper (`AVPlayerLooper`, desktop-level `NSWindow`),
  built with `swiftc` (Xcode is not installed on the maintainer's Mac).
- **Windows player:** Rust (`windows` crate) + libmpv (LGPL build), WorkerW.
- **Media sidecars:** `yt-dlp`, `ffmpeg` (LGPL build), `realesrgan-ncnn-vulkan`,
  `rife-ncnn-vulkan`.
- **ML:** ONNX Runtime via `ort`. Execution-provider order: CUDA → DirectML →
  CoreML → CPU. Local VLM through Ollama; optional BYOK Claude / Gemini.
- **Web:** Next.js on Vercel (later under `<personal-domain>/projects/swale`),
  MongoDB Atlas, Backblaze B2 (S3-compatible) for thumbnails and avatars only.
- **Python:** development only (ONNX export, experiments). Never shipped.

## Planned layout

```
apps/desktop/            Tauri app (React UI)
apps/desktop/src-tauri/  Rust core
native/macos-player/     Swift wallpaper helper
apps/web/                Next.js website + community
packages/recipe-schema/  shared recipe JSON schema
docs/                    design and decisions
```

## Conventions

- Code, identifiers and comments in **English**.
- Naming: **Swale** in UI and docs; **swale** for technical names; bundle id
  `com.yigtwxx.swale`.
- Rust: `cargo fmt` + `cargo clippy -- -D warnings`.
- TypeScript: `strict: true`, `prettier`, `tsc --noEmit` clean.
- Swift: `swift-format`.
- Python (dev scripts): type hints, `ruff check` + `ruff format --check`.
- Wallpapers are MP4 (H.264 by default). GIF is export-only.
- Commits follow Conventional Commits. Do not commit or push unless asked.

## Security

- No secrets in code or commits; `.env` files stay local.
- BYOK API keys and auth tokens live in the OS keychain.
- Passwords are hashed with Argon2id (or scrypt), never logged.
- Never pass user input to a shell. Sidecars get argument arrays, not strings.
