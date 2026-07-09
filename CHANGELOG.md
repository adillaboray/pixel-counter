# Changelog

All notable changes to the **Pixel Counter** project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.6.0] — 2026-07-09

### Added
- Idle save: count persisted to `localStorage` after 5s of inactivity
- Immediate save on tab switch, page hide, or browser close via `visibilitychange` / `pagehide` / `beforeunload`
- Resume popup on return showing last session's count with **CONTINUE** / **FRESH START** options
- Session history tracking (max 20 sessions) with timestamps, final count, and total clicks
- New `CHANGELOG.md` file to track also the documentations.

## [0.5.0] — 2026-07-06

### Changed
- Bumped service worker cache version to `0.5`
- Adjusted `.counter-container` `z-index` to `1` to ensure proper stacking context with overlay popups

## [0.4.0] — 2026-07-06

### Added
- Service worker update notification popup with 16-bit pixel aesthetic
- `SKIP_WAITING` message handler in service worker for seamless updates
- `controllerchange` listener to auto-reload page after service worker activation

### Changed
- Enhanced service worker messaging infrastructure for update workflow

## [0.3.0] — 2026-06-30

### Added
- "Press Start 2P" pixel font for retro 16-bit visual identity
- Custom font pre-cached in service worker asset list

### Changed
- Bumped service worker cache version to `0.4` to accommodate font asset

## [0.2.0] — 2026-06-30

### Added
- Cache-first with stale-while-revalidate fetch strategy in service worker
- Versioned cache management (`pixel-counter-v{N}`) with automatic old cache cleanup
- `PWA-GUIDE.md` with architecture documentation, caching strategy diagrams, and deployment workflow

### Changed
- Refined service worker install and activate lifecycle handlers
- Enhanced offline support with network-fallback error handling

## [0.1.0] — 2026-06-24

### Added
- Initial PWA project scaffold
- `index.html` — app shell with counter UI, click and keyboard (Enter) interaction, and reset button
- `sw.js` — service worker with install, activate, and basic fetch lifecycle
- `manifest.json` — PWA manifest with standalone display, theme color, and SVG icon
- `icon.svg` — SVG app icon (maskable, any size)
- `counter.html` — initial counter page (renamed to `index.html` in subsequent release)
- `PWA-GUIDE.md` — initial version with architecture overview and versioning contract

---

## Legend

| Icon | Meaning |
|------|---------|
| ✨ `Added` | New features |
| 🛠 `Changed` | Changes in existing functionality |
| 🐛 `Fixed` | Bug fixes |
| ⚠️ `Deprecated` | Soon-to-be removed features |
| 🗑 `Removed` | Removed features |
| 🔒 `Security` | Security fixes |

<!--
Version format: MAJOR.MINOR.PATCH
- MAJOR: breaking changes
- MINOR: new features (backward compatible)
- PATCH: bug fixes (backward compatible)
-->
