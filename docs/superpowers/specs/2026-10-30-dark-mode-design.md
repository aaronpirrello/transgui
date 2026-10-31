# Dark Mode for Windows — Design

Date: 2026-10-30
Branch: `dark-mode` (fork: `aaronpirrello/transgui`)

## Goal

Give Transmission Remote GUI a dark mode on Windows, selectable as
System / Light / Dark, defaulting to System (follow the Windows app theme).

## Scope

- In scope: Windows (win32 widgetset), Windows 10 1809 and later.
- Out of scope: Linux and macOS. They already follow the system theme via
  GTK/Qt/Cocoa; their builds must keep compiling but get no new behavior.
  No live theme switching: changing the theme requires a restart.

## Approach

Use the third-party Lazarus package **MetaDarkStyle**
(https://github.com/zamtmn/metadarkstyle, LGPL 2.1+ with linking exception).
It patches the win32 widgetset so native controls, menus, scrollbars, list
headers, status bar and title bars render dark, and it replaces the system
color table. The app's own custom drawing already uses system colors
(`clWindow`, `clHighlight`, ...) and theme-adaptive math (`GetLikeColor`),
so most of it follows automatically.

Rejected: hand-rolled uxtheme calls (only partly dark — menus, buttons, tabs
stay light) and switching Windows to the Qt widgetset (adds Qt DLLs and
reworks packaging).

## Components

### 1. Vendored dependency

- Copy the runtime package `metadarkstyle.lpk` and `src/` from upstream
  commit `e38933a` into `metadarkstyle/`. Exclude design-time packages
  (`metadarkstyledsgn*`), `examples/`, and `docs/`.
- Add `metadarkstyle/README.md` stating upstream URL, pinned commit and
  license. Treat the directory as third-party code like `synapse/`.
- Add `metadarkstyle` to `transgui.lpi` RequiredPackages with a default
  filename pointing at the vendored `.lpk`. On non-Windows the package
  compiles to stubs.
- Update Windows build scripts (`setup/win/make_setup.bat`,
  `setup/win/make_zipdist.bat`, `.github/scripts/build-windows.ps1`) only if
  `lazbuild` cannot resolve the package from the `.lpi` alone; if needed,
  build it the same way `trcomp.lpk` is built.

### 2. Startup

In `CheckAppParams` (`main.pas`), right after `Ini` is created and before
`Application.Initialize`, under `{$ifdef windows}`:

- Read `[Interface] Theme` as an integer: 0 = System (default), 1 = Light,
  2 = Dark. Out-of-range values are treated as 0.
- Map to `PreferredAppMode`: 0 → `pamAllowDark`, 1 → `pamForceLight`,
  2 → `pamForceDark`.
- Call `ApplyMetaDarkStyle(DefaultDark)`.

On Windows older than 10 1809 the library detects missing APIs and leaves the
app light; no extra handling.

### 3. Options UI

- Advanced tab of the Options dialog: add label "Theme:" and combo box with
  items System / Light / Dark, placed with the Language controls. Visible on
  Windows only (hidden via `{$ifdef}` elsewhere).
- Load from `[Interface] Theme`; on save, write it, and if it changed show
  the existing "You need to restart the application to apply changes"
  message.
- Keep `options.pas` and `options.lfm` in sync; `.lfm` edits stay minimal and
  Lazarus-compatible.

### 4. Localization

Add "Theme:", "System", "Light", "Dark" to `lang/transgui.template`. Other
`lang/transgui.*` files fall back to English for missing keys.

### 5. Custom-drawing audit

Run in Dark mode and inspect, fixing only what is visibly wrong:

- Torrent grid progress cells (`main.pas` progress cell drawing) and the
  pieces/progress bar on the General tab.
- Alternating row color (`FAlterColor`) and Transfer/Torrent header shading.
- Disconnected/disabled state (`clBtnFace` on grids and edits in `main.pas`
  and `EnableControls` in `utils.pas`).
- `vargrid.pas`: check-box painting, sort arrow, selection text workaround.
- Status bar, toolbar, splitters, filter list separators, icons.

Fixes must use system colors or theme-adaptive math, never hard-coded dark
values, so Light mode is unchanged. Icons unreadable on dark are reported,
not redrawn.

## Verification

No automated test suite exists. Verification is:

1. `lazbuild -B transgui.lpi` and `make` succeed on Windows.
2. Launch with Theme = System, Light, Dark; flip the Windows app theme to
   confirm System follows it. Screenshot main window, Options, Torrent
   properties, Add torrent, menus.
3. Light mode matches an upstream build visually.
4. Linux build still compiles (CI on the fork, or note if skipped).

## Commits

Focused commits on `dark-mode`: vendor package; startup + setting; drawing
fixes; localization. Imperative capitalized subjects per `AGENTS.md`.
