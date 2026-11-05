# Contributing to Swale

Thanks for your interest! Swale is in the **design phase**: the architecture
and features are written down, but there is no code yet. Early feedback is very
welcome.

## Ways to help right now

- Read the [design document](docs/DESIGN.md) and open an issue with feedback.
- Share experience with video wallpapers on macOS (desktop-level windows) or
  Windows (`WorkerW`), especially edge cases across OS versions and
  multi-monitor setups.
- Suggest edit presets or best-moment scoring ideas.

## Ground rules

- **Language:** all code, identifiers and comments in **English**.
- **Cross-platform:** features must work on both macOS and Windows unless they
  are explicitly platform-specific.
- **Security:** never hard-code or log secrets. API keys belong in the OS
  keychain, never in the repository.
- **Content:** do not add copyrighted media to the repository. Use your own
  footage or properly licensed samples for tests and demos.

## Commits and pull requests

- Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/)
  (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`), imperative mood,
  subject line under 72 characters.
- Keep pull requests focused, and explain *why* in the description.
- Development setup and checks will be documented here once code lands.

## Code of Conduct

This project follows the [Contributor Covenant](CODE_OF_CONDUCT.md). By
participating you agree to uphold it.
