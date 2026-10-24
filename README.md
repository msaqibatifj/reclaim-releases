# Reclaim

**Find out what is eating your Windows drive and safely get the space back.**

Reclaim is a free, open-source desktop app for Windows that maps your disk, cleans the caches that quietly grow to tens of gigabytes, moves heavy folders to another drive without breaking apps, and finds duplicate files. It is built around one rule: **a cleaner that breaks your PC is worse than a full disk.**

![Clean up](docs/screenshots/overview-light.png)

## Features

| | |
|---|---|
| **Clean up** | Free space on your system drive, what you've selected to free, and what you'll have afterwards, with the cleanup catalog under it: temp and Windows Update leftovers, Chrome/Edge/Brave caches (logins untouched), npm/pip/Yarn/NuGet caches, shader and game caches, crash dumps, Recycle Bin, hibernation file and DISM component-store cleanup. Every item is tagged Safe, Re-downloads or Check first, and "Details" explains what it is, whether it comes back, what happens if removed and how to undo. |
| **Review, then confirm** | Nothing is deleted, moved or uninstalled in one click. The review step lists every path that will change, where it goes and how to undo it, with a total that matches the drive bar exactly. |
| **Disk usage** | Every folder sized, largest first, filled in as the scan finishes each folder. Click into folders, see the largest files and a breakdown by type. Hard links are counted once, links are never followed. |
| **Move folders** | Moves a folder (game data, package caches) to another drive and leaves a junction, so apps keep the same path. Copy with progress, verify, rename, link, automatic rollback, and **Move back**. |
| **Duplicates** | Size → partial hash → full SHA-256, so matches are exact. You pick which copy to keep in each set (one is always kept). Removed copies go to the Recycle Bin, and **Undo** puts them back. |
| **Programs** | Installed software by size. Uninstall runs the program's own uninstaller after a review. |
| **History** | Every action, newest first, with its undo. Totals at the top. Export as text or CSV. |

<p>
  <img src="docs/screenshots/preview.png" width="49%" alt="Review step" />
  <img src="docs/screenshots/space.png" width="49%" alt="Disk usage" />
  <img src="docs/screenshots/duplicates.png" width="49%" alt="Duplicates" />
  <img src="docs/screenshots/ledger.png" width="49%" alt="History" />
</p>

## Design

The interface follows [DESIGN.md](DESIGN.md) ("Plain"): plain words, familiar Windows controls, one column, and a review step before anything changes. It uses the Windows system font and follows Windows light/dark mode (or pick one with the theme button). Motion is small and turns off with Windows "Animation effects".

Keyboard: everything is reachable with `Tab`, `Space` and `Enter`. `Esc` leaves a review, and goes up one folder in Disk usage.

## How it stays safe

These rules are enforced in code ([`electron/lib/safety.js`](electron/lib/safety.js)) and covered by tests:

- **Your files are off-limits to cleanups.** Documents, Desktop, Downloads, Pictures, Videos, Music, OneDrive, Windows, Program Files, ProgramData, drive roots and your profile folder are on a hard deny list. A cleanup root that *is* one of these, *contains* one, or (for personal folders) sits *inside* one is rejected.
- **Every delete is fenced.** Each cleanup declares its folders up front, and every file is checked to be inside one of them before removal.
- **Links are never followed.** Symlinks and junctions are unlinked, never traversed.
- **In-use files are skipped, not forced.** Temp cleanup keeps files from the last 2 days by default.
- **Moves keep the original** as `<name>.reclaim-old` until you confirm the app works and choose to delete it. The old copy is only deleted after Reclaim confirms the junction still points at the new location and that drive is connected. Undo checks free space first and copies the live data back, so changes made after the move are not lost.
- **Moves refuse what they can't copy completely:** folders containing links/junctions, unreadable subfolders, OneDrive folders, system and Program Files trees.
- **Duplicate removal is enforced in the main process,** not just the UI: only files from the last search can be recycled, and never the last remaining copy.
- **No shell strings.** `powercfg`, `DISM`, `robocopy` and `net` are started with fixed argument arrays. Uninstallers are looked up from the registry by the main process, never accepted from the UI.
- **One job at a time.** A second cleanup or move is rejected while one is running, and only one copy of Reclaim can run.
- **Admin only when needed.** Reclaim runs as a normal user; Windows-level cleanups are skipped until you restart it as administrator.
- **Offline.** No telemetry, no network calls. The renderer runs sandboxed with context isolation and a strict CSP, and can only reach a whitelisted set of IPC channels.

## Download

**[⬇ Download Reclaim for Windows](https://github.com/msaqibatifj/reclaim-releases/releases/latest/download/Reclaim-Setup.exe)** (installer, no admin needed)

Or get the [portable version](https://github.com/msaqibatifj/reclaim-releases/releases/latest/download/Reclaim-portable.exe) (no install), or browse [all releases](https://github.com/msaqibatifj/reclaim-releases/releases). `SHA256SUMS.txt` is attached to every release.

The builds are not code-signed yet, so Windows SmartScreen shows "Windows protected your PC" on first run. Click **More info → Run anyway**.

## Getting started (from source)

Requirements: Windows 10/11, [Node.js 20.19+](https://nodejs.org) (22 LTS recommended).

```powershell
git clone https://github.com/<your-username>/reclaim.git
cd reclaim
npm install
npm run dev        # Vite + Electron with hot reload
```

Other scripts:

```powershell
npm test           # engine tests (node:test, run on any OS)
npm run dev:web    # UI only, in a browser, with demo data (works on macOS/Linux too)
npm run build      # production build of the UI into dist/
npm start          # run Electron against dist/
npm run dist       # build the installer and portable .exe into release/
```

To test Windows-level cleanups (hibernation, DISM, Windows Update cache) during development, start your terminal as Administrator before `npm run dev`.

## Project structure

```
electron/
  main.js            Electron main process: window, IPC handlers, job locking
  preload.js         whitelisted bridge exposed to the UI as window.reclaim
  lib/
    safety.js        protected-path guard used by every destructive operation
    fsx.js           bounded-concurrency fs helpers, guarded deleteContents()
    scanner.js       single-pass disk scanner (size tree, top files, categories)
    cleaners.js      the cleanup catalog (file cleaners + command cleaners)
    mover.js         move-to-drive with junctions, verify, rollback, ledger
    duplicates.js    3-pass duplicate finder
    system.js        drives, admin check, hidden system files, installed programs
    recycle.js       restore files Reclaim recycled (reads $Recycle.Bin $I index files)
    receipts.js      ledger receipts and text/CSV export
src/
  App.jsx            top bar (tabs, theme), views kept mounted
  views/             Overview (Clean up), Space (Disk usage), Duplicates, Programs, MoveDrive (Move folders), Ledger (History)
  components/        ui.jsx (page, cards, notices, buttons, drive bar, tags), Preview.jsx (review step)
  lib/motion.js      anime.js wrappers: number tweens, row fade-in; off with reduced motion
  lib/api.js         IPC wrapper; falls back to lib/mock.js outside Electron
DESIGN.md            the design system; the UI follows it
tests/engine.test.js
```

### Adding a cleanup

Cleanups live in [`electron/lib/cleaners.js`](electron/lib/cleaners.js). A file cleanup is a metadata object plus a function that returns the folders to empty. Then add its grade and margin notes to `MARGIN` in the same file (see DESIGN.md §3):

```js
fileCleaner(
  {
    id: 'gradle-cache',
    name: 'Gradle cache',
    description: 'Downloaded dependencies. Rebuilt on the next build.',
    risk: 'safe',          // 'safe' | 'low' | 'caution'
    admin: false,
    closeApps: ['Android Studio'],
    group: 'Developer',
  },
  async () => [{ root: path.join(home, '.gradle', 'caches') }],
);
```

Every root you return is validated by `assertSafeRoot()`, and every file deleted under it by `assertInside()`. Please add a test in `tests/engine.test.js` that creates a fake folder layout and checks what survives.

## Releasing

1. Bump `"version"` in `package.json` and add an entry to [CHANGELOG.md](CHANGELOG.md).
2. Commit, then tag and push: `git tag v0.1.0 && git push origin main --tags`.

The source repo is private; builds are published to the public [reclaim-releases](https://github.com/msaqibatifj/reclaim-releases) repo so anyone can download them. For a `v*` tag, [`.github/workflows/release.yml`](.github/workflows/release.yml) runs the tests on Windows and Linux, checks the tag matches `package.json`, builds the NSIS installer and the portable exe, and publishes a release there with both files (versioned and fixed-name copies, so the download links always point at the latest) and `SHA256SUMS.txt`.

It needs one secret, `RELEASES_TOKEN`: a fine-grained personal access token limited to `reclaim-releases` with **Contents: Read and write**.

The builds are not code-signed, so SmartScreen will warn until a signing certificate is added (see electron-builder's `win.signtoolOptions`).

## Known limitations

- Sizes are logical file sizes. OneDrive "online-only" files and NTFS-compressed folders show their full size, not what they take on disk.
- Windows only for cleanups and moves. The disk map and duplicate finder work elsewhere, but are untested outside Windows.
- The scanner runs in the main process; on very large drives the window can stutter briefly while results are assembled.

## Roadmap

- Run the scanner in a utility process and stream partial results to the treemap
- Code signing
- Scheduled cleanup and low-space notifications
- Per-app leftover detection after uninstall
- WSL/Docker virtual disk compaction helper
- macOS and Linux support (the engine is portable; the cleanup catalog is Windows-specific)

## License

[MIT](LICENSE)
