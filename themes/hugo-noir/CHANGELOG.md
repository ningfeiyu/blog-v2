# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [2.1.0] - 2026-09-06

### Changed
- **Minimum Hugo version raised to v0.158.0 (Extended).** The theme now uses the `hugo.Data` / `hugo.Sites` project APIs and `Language.Label`. Running on older Hugo versions will fail the version check.
- **Tailwind CSS is now compiled, no longer loaded from CDN.** The theme ships a prebuilt `static/css/tailwind.css` (includes the Typography plugin for Markdown rendering). If you override theme templates with new Tailwind classes, rebuild it: `npm install && npx tailwindcss -i assets/css/main.css -o static/css/tailwind.css --minify` (see README "CSS Build").
- **Markdown rendering:** blog posts use the Tailwind Typography plugin (`prose dark:prose-invert`), and Markdown tables are wrapped in a horizontal-scroll container via a `render-table` hook. No content changes needed.
- **Homepage blog links** (titles, "view all") now follow the heading color (dark text in light mode, white in dark mode) instead of blue, with blue on hover.
- **Blog post card** (`single.html`) now uses the same surface color as the homepage hero card in both modes.
- **`profile_image` is not rendered by the theme** (name, location, description, and social icons only). Existing `profile_image` settings are ignored, not errors.

### Removed
- **Redundant Tailwind color aliases** (they duplicated primary values): `bg-secondary-light`, `bg-tertiary-light`, `bg-tertiary-dark`, `border-secondary-light`, `border-secondary-dark`. Templates now use the `*-primary-*` / `bg-secondary-dark` equivalents.
  - *Migration:* if your own template overrides use the removed classes, replace `bg-tertiary-light` → `bg-primary-light`, `bg-tertiary-dark` → `bg-secondary-dark`, `bg-secondary-light` → `bg-primary-light`, `border-secondary-light/dark` → `border-primary-light/dark`.
- Deprecated Hugo APIs (`Site.Data`, `Site.Languages`, `Site.BaseURL` concatenation, `languageName`/`languageCode` config keys) replaced with their supported counterparts. No user action needed unless you copied these patterns into your own overrides.

## [2.0.0] - 2025-05-22

### Added
- **Light Mode:** Full support for a light color scheme, complementing the existing dark mode.
- **Multilingual Support:** Enhanced for easier setup and use with pre-configured English, Spanish, and French.
- **Region in Experience:** `country` field added to experience entries, displayed in the experience section.
- **Carousel Animations:** Implemented new criss-cross transition animations for carousels.
- **Hyperlink SVGs:** Visual indicators (SVGs) added to hyperlinks for better affordance.
- **SVG Reorganization:** Internal SVG assets have been restructured for better maintainability.
- **Enhanced Mobile Navigation:** Improved mobile menu and navigation bars for better usability on small screens.
- **`hugo.example.toml`:** Added a comprehensive example configuration file for users.
- **GitHub Release Badge:** Added to `README.md`.

### Changed
- **Theme Name:** Standardized to "Hugo Noir" across all files.
- **Documentation:** Updated `README.md` extensively to reflect new features and configurations.

### Fixed
- **SVG Optimization:** Replaced long data URI strings for SVGs with minimal, optimized versions.
- **Customization Enhancement:** Removed hardcoded titles and footers from `index.html` to allow for complete user customization via configuration files and i18n.

## [1.0.0] - YYYY-03-12
- Initial release 