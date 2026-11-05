# Security Policy

## Reporting a vulnerability

If you discover a security issue in Swale, please report it privately rather
than opening a public issue:

- Email: **yigiterdogan023@gmail.com**
- Or use GitHub's private **"Report a vulnerability"** feature under the
  Security tab.

Please include a description, reproduction steps, and impact. We aim to
acknowledge reports within a few days.

## How Swale handles sensitive data

- **API keys (BYOK)** are stored in the operating system's keychain (macOS
  Keychain, Windows Credential Manager). They are never written to logs,
  config files, or the library database, and never bundled with the app.
- **Passwords** for Swale accounts are hashed (Argon2id or scrypt) and are
  never stored or logged in plain text.
- **Auth tokens** in the desktop app are kept in the OS keychain.
- **External tools** (`yt-dlp`, `ffmpeg` and others) receive arguments as
  arrays. User input is never passed through a shell.
- **The community** stores recipes (JSON) and thumbnails only. No video files
  are uploaded.

## Supported versions

Swale is in the design phase. Until a `1.0.0` release, only the latest commit
on the default branch is supported.
