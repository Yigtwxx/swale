# Swale — Design

**Status:** design phase, no code yet · **Last updated:** 2026-11-05

This document records the decisions made so far and the reasoning behind them.
Open questions are listed in [section 13](#13-open-questions). When a decision
changes, update the relevant section and the [decision log](#14-decision-log).

---

## 1. Goal

A free, open-source desktop app for **macOS and Windows** that turns a moment of
a video into a looping live wallpaper, or a frame / image into a 2K–4K static
wallpaper, and keeps everything in a library.

Principles:

- **Free for users.** Heavy work (download, render, ML) runs on the user's
  machine. No mandatory paid API.
- **Local-first.** After the first sign-in, local features keep working
  without network access (cached session). Only downloads and community
  features need the network.
- **Light at rest.** A running wallpaper should cost close to nothing:
  hardware decoding, paused when it is hidden.

## 2. Core flow (v1)

1. The user enters a YouTube URL, a start and end time, and a quality
   (1080p / 1440p / 2160p / best available).
2. `yt-dlp` downloads only that range (`--download-sections "*1:10-1:20"`).
3. `ffmpeg` crops and scales it to the target display, makes the loop seamless
   (crossfade or ping-pong) and encodes an MP4.
4. The item lands in the library. One click applies it.

Formats:

- Wallpapers are stored as **MP4**. H.264 by default (plays everywhere,
  including Windows machines without the HEVC extension); HEVC is optional on
  macOS.
- **GIF is not used for wallpapers.** It is limited to 256 colors, very large
  at 4K, and decoded on the CPU. GIF exists only as a low-resolution export for
  sharing.
- 4K is real 4K only when the source is 4K. Upscaling (Real-ESRGAN) is an
  explicit option, never a silent default.

## 3. Features

### 3.1 Library

- Grid of items with filters (All / Animated / Still) and sorting.
- Animated previews are low-resolution proxies that **play on hover only**.
- Click applies the item. Right click: edit, delete, export (MP4 / GIF / PNG),
  show in folder.
- **Per-display wallpapers from v1.**
- Every item keeps its recipe (section 5), so it can be re-rendered or edited.

### 3.2 Static wallpapers

- Sources: the user's own files, "grab this frame" from a video, optional
  upscaling to 4K.
- Applied through the OS wallpaper API. The video player shuts down, so a
  static wallpaper uses no CPU or GPU.

### 3.3 Best-moment finder

The user picks a mode (**Still** or **Video**), a count (1, 2, 4, 8 or 16) and
optionally a text prompt ("sunsets", "night city").

1. Sample 1–2 frames per second and detect scene cuts.
2. Reject bad candidates: blurry (Laplacian variance), too dark or bright,
   subtitles or text, frames in the middle of a transition.
3. Score with an aesthetic predictor on CLIP / SigLIP embeddings. A text prompt
   adds CLIP text–image similarity.
4. Video mode scores 5–10 s segments instead of frames and adds loopability
   (first / last frame similarity), camera stability, and "no cut inside the
   segment".
5. Diversity: cluster the embeddings and take the best item per cluster, so 16
   results are 16 different shots.
6. Optional: rerank the top candidates with a local VLM or a BYOK model.

If the video does not contain enough distinct candidates, say so ("found 9, not
16") instead of padding with near-duplicates.

### 3.4 AI Edit (later, not in v1)

A real edit with transitions, speed ramps, punch-in zooms, shake / flash,
color grading and beat-aware timing. Not just a cut-down highlight reel.

- **Two modes:** the user describes or picks the edit, or leaves it to the AI.
  Both fill the same edit plan (JSON) and use the same renderer.
- Quality comes from **hand-made presets** (e.g. velocity, cinematic, smooth).
  The AI picks a preset and fills in clips, cut points and transitions. It does
  not invent effects.
- Renderer: `ffmpeg` + GL Transitions (MIT). Remotion was considered and
  rejected because of its licensing.
- Building blocks: `xfade` and GL transitions, `setpts` speed ramps with RIFE
  frame interpolation and motion blur, keyframed zoom / shake / flash, LUT
  grading, `librosa` beat detection, SAM 2 masks for subject-masked transitions
  (advanced).
- Models: local by default (Ollama, e.g. Qwen3-VL or Gemma 3). BYOK Claude /
  Gemini is optional. Keys live in the OS keychain and are never bundled.
- Expectation: consistent "good template" quality, not a professional editor
  every time. Wallpaper presets need calm, loop-friendly variants; ten
  transitions in a ten-second background is tiring to look at.

### 3.5 Discover (later)

In-app browsing of free wallpaper sources such as Wallhaven.

## 4. Desktop architecture

| Layer | macOS | Windows | Language |
|---|---|---|---|
| UI | shared | shared | TypeScript (React) |
| Core: library, jobs, sidecars, AI calls | shared | shared | Rust (Tauri 2) |
| Local database | SQLite | SQLite | Rust (`rusqlite` / `sqlx`) |
| Video wallpaper player | Swift helper, `AVPlayerLooper` | WorkerW + libmpv | Swift / Rust |
| Static wallpaper | `NSWorkspace.setDesktopImageURL` | `IDesktopWallpaper` | Swift / Rust |
| ML scoring (CLIP, aesthetic) | ONNX Runtime → CoreML | ONNX Runtime → CUDA / DirectML | Rust (`ort`) |
| Upscale / interpolation | `realesrgan-ncnn-vulkan`, `rife-ncnn-vulkan` | same | prebuilt sidecars |
| Download / encode | `yt-dlp`, `ffmpeg` | same | prebuilt sidecars |
| Local VLM | Ollama | Ollama | HTTP |

Why Tauri: the app runs all day, so Electron's memory cost (~200 MB+) is too
high. PySide6 packages poorly. Two fully native apps would mean writing
everything twice. With Tauri about 90% of the code is shared.

Platform notes:

- **macOS** has no public API for video wallpapers. The Swift helper opens one
  borderless `NSWindow` per `NSScreen` at desktop window level, ignores mouse
  events, and loops the file with `AVPlayerLooper`. It builds with `swiftc`, so
  Xcode is not required.
- **Windows:** the player window is parented to the `WorkerW` window behind the
  desktop icons and plays through libmpv (LGPL build) with hardware decoding.
  `WorkerW` is undocumented and has broken with Windows updates (24H2), so this
  code stays isolated and gets its own tests.
- Pause playback when a fullscreen app covers the display, and optionally on
  battery.
- ML execution-provider order: **CUDA → DirectML → CoreML → CPU** (the ONNX
  equivalent of the CUDA → MPS → CPU rule).
- Python is used only during development (exporting models to ONNX, scoring
  experiments). It never ships.

### 4.1 Local data

SQLite is embedded in the app binary, so there is nothing extra to install.
The database file is created on first launch. It holds metadata only. Media
files stay on disk and are never stored as BLOBs.

- macOS: `~/Library/Application Support/com.yigtwxx.swale/`
- Windows: `%APPDATA%\com.yigtwxx.swale\`

```
swale.db            library metadata (SQLite)
media/              rendered wallpapers (MP4 / PNG / JPEG)
previews/           low-res hover proxies and thumbnails
cache/              downloads in progress, ML scratch
models/             ONNX models (downloaded on first use)
```

Draft tables:

```sql
CREATE TABLE items (
  id              TEXT PRIMARY KEY,          -- UUID v7
  kind            TEXT NOT NULL CHECK (kind IN ('video', 'image')),
  title           TEXT,
  width           INTEGER NOT NULL,
  height          INTEGER NOT NULL,
  fps             REAL,                      -- NULL for images
  file_path       TEXT NOT NULL,
  preview_path    TEXT,
  thumbnail_path  TEXT,
  recipe_json     TEXT NOT NULL,             -- see section 5
  created_at      TEXT NOT NULL,
  last_applied_at TEXT
);

CREATE TABLE display_assignments (
  display_id  TEXT PRIMARY KEY,              -- stable per-OS display identifier
  item_id     TEXT NOT NULL REFERENCES items(id) ON DELETE CASCADE
);
```

## 5. Recipes

A recipe is a small JSON document that describes how a wallpaper was made. It
is stored with every library item and is the **only** thing the community
shares. Receiving a recipe, Swale downloads the source and renders the
wallpaper on the user's own machine.

Draft (`swale.recipe/v1`):

```json
{
  "schema": "swale.recipe/v1",
  "kind": "video",
  "source": { "type": "youtube", "videoId": "dQw4w9WgXcQ" },
  "range": { "start": 70.0, "end": 80.0 },
  "output": { "maxHeight": 2160, "fps": 30, "loop": "crossfade" },
  "edit": null,
  "meta": { "title": "Neon rain", "tags": ["city", "night"] }
}
```

- Image recipes use `"kind": "image"` with `"at": 72.4` in place of `range`.
- `edit` is `null` until AI Edit exists. Then it holds the edit plan: preset,
  clips, transitions, grade and effects with timings.
- The `schema` field is versioned from day one.

## 6. Accounts

- **Required** on both the website and the desktop app. The desktop app asks
  for sign-in on first launch. The session is cached in the OS keychain, so the
  app keeps working offline and refreshes the session once it is back online.
- Sign-in methods: **email + password**, **GitHub**, **Google**.
- Passwords are hashed with Argon2id (or scrypt) and are never stored or logged
  in plain text. Email verification, password reset, and rate-limited login
  are required.
- Desktop sign-in uses the system browser with OAuth 2.0 + PKCE (RFC 8252). The
  callback returns through the `swale://auth/callback` deep link, and tokens
  are kept in the OS keychain.
- Auth library candidates: Better Auth or Auth.js. Both are TypeScript and both
  support MongoDB.

## 7. Website & community

**Hosting:** deployed to **Vercel** first. When the maintainer's personal
website exists, Swale moves under it at `<personal-domain>/projects/swale`, so
every project shares one domain. Next.js `basePath: "/projects/swale"`, with the
personal site rewriting that path to the Swale deployment.

Pages:

- **Landing page** (scrolling): hero with a live wallpaper demo, how it works in
  three steps, features, OS-aware download button, FAQ.
- **Community:** recipe gallery with filters (animated / still, resolution,
  color, tags), likes, collections, weekly picks.
- **"Open in Swale":** a `swale://recipe/<id>` link opens the app and adds the
  recipe to the library.
- Profiles, preset sharing, changelog (from GitHub Releases), help,
  report / DMCA, privacy, terms. Languages: Turkish and English.

**Recipes only, no video files.** The server stores the recipe JSON and a
small thumbnail (WebP, about 50 KB). Not hosting clips removes most of the
copyright (DMCA) exposure and almost all of the storage cost.

Backend:

- **Database:** MongoDB Atlas (free M0 tier, 512 MB). Recipes are documents,
  so the document model fits.
- **Object storage (thumbnails only):** **Backblaze B2**, chosen because
  Cloudflare R2's free tier is already used by another project. B2 is
  S3-compatible and comes from a separate company (not part of Cloudflare). The first 10 GB are
  always free, egress is free up to 3× the stored data each month, and
  downloads served through Cloudflare's CDN are free (Bandwidth Alliance). At
  about 50 KB per thumbnail, 10 GB is roughly 200,000 recipes.
- Move to paid tiers when traffic needs it. S3 compatibility keeps that
  migration simple.
- **Moderation:** report button, NSFW classifier on uploaded thumbnails, rate
  limits.
- **Donations:** [GitHub Sponsors](https://github.com/sponsors/Yigtwxx). The
  repository's Sponsor button comes from `.github/FUNDING.yml`.

## 8. Distribution

- GitHub Actions (`tauri-action`) builds `.dmg` (macOS arm64 / x64) and
  `.msi` / `.exe` (Windows) on every version tag and attaches them to GitHub
  Releases. This is free for public repositories.
- The website's download button detects the OS and links to the latest release.
- Auto-update: the Tauri updater plugin, pointed at GitHub Releases.
- Code signing: without an Apple Developer account ($99/year) macOS shows an
  "unidentified developer" warning, and unsigned Windows builds trigger
  SmartScreen. This is acceptable for early open-source releases.

## 9. Naming

- Product and display name: **Swale**. Technical name: **swale** (repository,
  packages, folders, CLI). Bundle identifier: `com.yigtwxx.swale`. In Tauri:
  `productName: "Swale"`, `identifier: "com.yigtwxx.swale"`.
- Checked 2026-11-05: no software product with this name was found. "Swale" is
  also a UK borough and the name of a wallpaper pattern at Happywall, both
  unrelated. This was not a formal trademark search.

## 10. Legal & licensing

- Swale itself: **MIT**.
- `yt-dlp`: Unlicense. `ffmpeg`: ship an LGPL build as a separate sidecar.
  libmpv: LGPL build (`-Dgpl=false`). GL Transitions: MIT. Model weights each
  carry their own license; check before bundling or auto-downloading.
- YouTube's terms restrict downloading. Swale downloads on the user's machine
  for personal use, and the project never hosts clips.

## 11. Risks

- `WorkerW` behavior changing with Windows updates.
- Local AI needs real hardware: about 16 GB RAM on a Mac, or a GPU with 6–8 GB
  VRAM. Manual mode must work on every machine. The first AI use downloads
  3–6 GB of models.
- `yt-dlp` breaks when YouTube changes. The sidecar needs a self-update path.
- Free-tier limits on the web backend.

## 12. Roadmap

0. **Spike:** macOS player helper plus the `yt-dlp` → `ffmpeg` pipeline as a
   CLI. Goal: URL + range → looping desktop wallpaper.
1. **macOS MVP:** Tauri app, SQLite library, video and still wallpapers, hover
   previews, per-display assignment, GIF export.
2. **Windows:** `WorkerW` + libmpv player, `IDesktopWallpaper`, `.msi`, CI for
   both platforms.
3. **Best-moment finder.**
4. **Accounts, website, community recipes,** desktop sign-in, `swale://` links.
5. **AI Edit.**

Later: Discover.

## 13. Open questions

- Auth library: Better Auth or Auth.js?
- UI components: shadcn/ui proposed.

## 14. Decision log

All entries dated 2026-11-05.

- v1 ships without AI. AI Edit comes later.
- Platforms: macOS and Windows. Shell: Tauri. UI: React. macOS player: Swift
  helper.
- Wallpapers are stored as MP4. GIF is export-only.
- AI runs on local models, with optional BYOK. Free for users.
- Open source under MIT.
- Desktop app, not a web app. A browser cannot set the desktop wallpaper.
- Library: hover previews, per-display wallpapers from v1, GIF export.
  Discover comes later.
- Static 2K / 4K wallpapers are supported, not only video.
- Best-moment finder searches by mode: frames for stills, loopable segments for
  video.
- Name: Swale.
- Community shares recipes only.
- Accounts: email + password, GitHub, Google, on both web and desktop.
  Sign-in is required in the desktop app too, with a cached session for
  offline use.
- Web database: MongoDB Atlas. Object storage: Backblaze B2, thumbnails only
  (Cloudflare R2's free tier is already in use).
- Desktop database: SQLite, embedded in the app.
- Website: Vercel first, later `<personal-domain>/projects/swale`.
- Donations: GitHub Sponsors.
