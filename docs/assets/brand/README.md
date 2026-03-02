# Brand assets

Put repository-level, documentation/marketing assets here so they are easy to link from `README.md` and other docs without shipping them with app binaries.

Suggested files
- app-icon.png — 1024×1024 PNG, transparent background, used in README and social images
- logo-light.svg — logo for light backgrounds
- logo-dark.svg — logo for dark backgrounds
- social-card.png — 1200×630 Open Graph preview for repo/app links

Usage in README
- Simple image (single icon):
  ![Community Library app icon](docs/assets/brand/app-icon.png)

- Light/dark-aware logo (GitHub supports <picture>):
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="docs/assets/brand/logo-dark.svg">
    <source media="(prefers-color-scheme: light)" srcset="docs/assets/brand/logo-light.svg">
    <img alt="Community Library" src="docs/assets/brand/logo-light.svg" width="200">
  </picture>

Platform-specific locations (do not mix with docs assets)
- Web (favicons/PWA icons): `web/src/assets/icons/` (e.g., favicon.svg, favicon.ico, manifest icons)
- iOS app icons: `ios/YourApp/Assets.xcassets/AppIcon.appiconset/`
- Android (deferred): `android/app/src/main/res/mipmap-*` (when added)

Notes
- Prefer SVG for logos (crisp at any size). Use PNG for the app icon.
- Keep docs assets small (≤300 KB) and use Git LFS only if you must store large binaries.
- When you update visuals, keep filenames stable so README links don’t break.
