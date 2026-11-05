# Swale

> Turn any moment of a video into a live wallpaper, or a crisp 4K still, on
> macOS and Windows.

<p align="center">
  <strong>Status:</strong> 🚧 Design phase. There is no code yet; see the
  <a href="docs/DESIGN.md">design document</a>.
</p>

---

## What it will do

- **Clip to wallpaper.** Paste a YouTube link, pick a range (say `1:10`–`1:20`)
  and a quality from 1080p to 4K. Swale downloads only that range, fits it to
  your screen and loops it on your desktop.
- **Stills too.** Use your own images or grab a single frame from a video.
  Static wallpapers are set through the OS, so nothing keeps running in the
  background.
- **Library.** Everything you make lands in one place. Previews play on hover;
  click to apply. Each display can have its own wallpaper.
- **Best-moment finder.** Ask Swale for the most wallpaper-worthy shots in a
  video and get 1, 2, 4 or 16 options: frames for stills, loopable segments for
  video. Steer it with text such as "sunsets" or "night city".
- **Recipes.** Share how you made a wallpaper (source link, time range, edit
  settings) instead of the video file. Others open the recipe in Swale and it
  renders on their machine.
- **AI Edit** *(later)*. Edits with transitions, speed ramps and color
  grading. Describe what you want or let the AI direct. It runs on local
  models, and you can bring your own API key if you want more.
- **Free.** Open source, no subscription. The heavy work runs on your own
  machine.

## How it works

```
YouTube URL + range ─▶ yt-dlp (that range only) ─▶ ffmpeg (crop · scale · loop) ─▶ library ─▶ desktop
```

On macOS a small native helper plays the loop in a desktop-level window with
AVFoundation. On Windows the player sits behind the desktop icons and uses
libmpv. Both decode on the GPU, so a 4K loop costs very little.

## Planned stack

| Layer | Choice |
|---|---|
| App shell | Tauri 2 (Rust) |
| UI | React + TypeScript |
| Local database | SQLite |
| macOS player | Swift helper (`AVPlayerLooper`) |
| Windows player | Rust + libmpv |
| Media | yt-dlp, ffmpeg |
| ML | ONNX Runtime (CLIP and aesthetic scoring), Ollama for local VLMs |
| Website & community | Next.js on Vercel, MongoDB Atlas, Backblaze B2 |

Details and trade-offs are in [docs/DESIGN.md](docs/DESIGN.md).

## Roadmap

0. Spike: macOS player and the download / convert pipeline
1. macOS MVP: app, library, video and still wallpapers, per-display
2. Windows support
3. Best-moment finder
4. Accounts, website, community recipes
5. AI Edit

## Contributing

The project is in the design phase, so feedback on the design is the most
useful contribution right now. See [CONTRIBUTING.md](CONTRIBUTING.md).

## A note on content

Swale downloads video on your own machine for personal use. Respect the terms
of the platforms you use and the rights of the people who made the videos. The
community shares recipes, never video files.

## License

[MIT](LICENSE) © 2026 Yigit Erdogan
