# Changelog

All notable changes to this project are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Design document (`docs/DESIGN.md`): core flow, library, static wallpapers,
  best-moment finder, AI Edit plan, desktop architecture, recipes, accounts,
  website and community, distribution, licensing, risks, roadmap and decision
  log.
- Project scaffolding: README, MIT license, contributing guide, code of
  conduct, security policy, issue and pull request templates.
- `.github/FUNDING.yml`, which adds the GitHub Sponsors button.

### Changed
- Design: sign-in is required in the desktop app as well (cached session for
  offline use). Backblaze B2 chosen for thumbnail storage. GitHub Sponsors
  recorded.
- Design: product details settled. Covers sources (YouTube, other `yt-dlp`
  sites, local files), 30 s clips, optional audio, menu bar / tray presence,
  launch at login, battery and theme settings, macOS 14+ (Apple Silicon) and
  Windows 11, Turkish and English, community likes / follows / collections,
  public or unlisted recipes, opt-in crash reports, background updates and a
  relocatable library.
- Design: first-launch wizard, bundled sample wallpapers, username and profile
  rules.
