# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Bidirectional, audio-first project transfer between Logic Pro (`.logicx`) and Ableton Live (`.als`). Ships as one Python package (CLI) plus one Electron desktop app that wraps it, versioned and released together.

Two directions, one CLI entry point:
- `logic2ableton` — Logic `.logicx` → Ableton `.als` + copied media + conversion report
- `ableton2logic` — Ableton `.als` → a *Logic transfer package* (stems, timeline MIDI, per-track MIDI files, audio clips, manifests, import guide). It does **not** synthesize a native `.logicx`.

## Commands

```bash
# Python tests (no third-party deps required)
python -m pytest tests -q
python -m pytest tests/test_logic_parser.py -q          # single file
python -m pytest tests/test_logic_parser.py::test_name   # single test

# Run the CLI from source
python -m logic2ableton.cli "MySong.logicx" --output ./out
python -m logic2ableton.cli "MySet.als" --output ./out   # mode auto-detected from .als

# Build Python package / standalone binary
python -m build
pyinstaller logic2ableton.spec                            # -> dist/logic2ableton[.exe]

# Desktop app (from app/)
cd app && npm ci && npm run dev      # electron-vite dev
npm run build                         # electron-vite build
npm run dist:mac / npm run dist:win   # packaged installers
```

Tests marked `needs_test_project` (a real `.logicx`) and `needs_vst3` (local VST3 plugins) auto-skip when those local fixtures are absent — see `tests/conftest.py`. CI runs the full suite on Windows + macOS across Python 3.11–3.13.

## Architecture

**The Python package has zero third-party runtime dependencies** — only the stdlib (`plistlib`, `struct`, `gzip`, `wave`, `xml.etree`). Keep it that way; it's a hard constraint for the PyInstaller bundle and PyPI install. The Ableton `.als` template is bundled at `logic2ableton/data/DefaultLiveSet.als`, so no DAW install is needed to generate output.

Pipeline shape (mirrored in `cli.py` for both directions): **parse → match/report → generate → write report**. Reports are first-class artifacts emitted on *both* success and failure paths — they are the primary support/debugging artifact.

Module map:
- `cli.py` — single `main()` for both commands. Mode resolution order: `argv[0]` stem (`logic2ableton`/`ableton2logic`), then `--mode`, then `.als`-extension detection. `--json-progress` emits one JSON line per stage (consumed by the Electron app).
- `models.py` — all dataclasses (`LogicProject`, `AbletonProject`, note/clip/track types) plus shared helpers (`parse_audio_filename`, `samples_to_beats`). Logic audio filenames encode take/comp/bip structure that `parse_audio_filename` decodes.
- `logic_parser.py` — reads the `.logicx` bundle: plists (`ProjectInformation.plist`, `Alternatives/{NNN}/MetaData.plist`) **and the reverse-engineered binary `ProjectData`** for plugins and MIDI notes. Logic numbers alternative folders arbitrarily (the only one may be `004`, not `000`), so `discover_alternatives`/`resolve_alternative` find the active one rather than assuming an index.
- `ableton_parser.py` — reads the gzipped-XML `.als` into an `AbletonProject`.
- `ableton_generator.py` — generates `.als` by **cloning the bundled template, stripping its tracks, and injecting ours**. Every XML `Id` must be globally unique — reassign via a global counter when cloning. Starting from a known-valid file is deliberate (guarantees structural correctness).
- `logic_transfer.py` — builds the reverse-direction transfer package (stems, `Logic Timeline.mid`, per-track MIDI, timestamped clip WAVs, manifests, `IMPORT_TO_LOGIC.md`).
- `smf.py` — shared Standard MIDI File writer. `build_midi_note_file` is duck-typed over any track with `.notes` items exposing `pitch`/`velocity`/`start_beats`/`duration_beats` (works for both Logic and Ableton tracks). `MIDI_TICKS_PER_QUARTER = 960`.
- `plugin_matcher.py` / `plugin_database.py` / `vst3_scanner.py` — identify Logic AU plugins (4CC codes) and suggest VST3 equivalents in the report.

**Desktop app** (`app/`, Electron + React 19 + TypeScript + Tailwind 4 via electron-vite): the UI does not reimplement conversion. `app/src/main/converter.ts` **spawns the Python CLI as a subprocess** and parses its `--json-progress` JSONL stdout. In dev it runs `python3 -m logic2ableton.cli` from the repo root; when packaged it runs the bundled PyInstaller binary from `process.resourcesPath`. The `ProgressEvent` interface there must stay in sync with the JSON payloads emitted by `_emit(...)` in `cli.py`.

## Reverse-engineered binary format

Logic stores arrangement MIDI in an undocumented binary blob (`ProjectData`), inside `EvSq` (`"qSvE"`) chunks. The decode in `logic_parser.py` was reverse-engineered and verified against Apple's shipping demo songs. See `docs/reverse-engineering-logic-pro-midi.md` before changing MIDI extraction. Notes are decoded at Logic's 960 PPQ and emitted relative to the project's earliest note.

## Version & release

The version lives in **two** places that must be bumped together: `logic2ableton/__init__.py` (`__version__`) and `app/package.json` (`version`). `pyproject.toml` reads its version dynamically from `__init__.py`. Releases are cut by pushing a `v*` git tag — GitHub Actions (`.github/workflows/release.yml`) runs tests, builds the PyInstaller binary, and packages/uploads the Windows + macOS desktop installers.

@.claude/dean-guidelines.md
