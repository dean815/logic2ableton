# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Audio-first project transfer between **three DAWs** — Logic Pro (`.logicx`), Ableton Live (`.als`), and Pro Tools (`.ptx`/`.pts`/`.ptf`). Ships as one Python package (CLI) plus one Electron desktop app that wraps it, versioned and released together.

Six transfer lanes, all sharing a single `cli:main`:

| Lane | Input | Output |
| --- | --- | --- |
| `logic2ableton` | Logic `.logicx` | native Ableton `.als` + copied media + report |
| `protools2ableton` | Pro Tools session | native Ableton `.als` + copied media + report |
| `ableton2logic` | Ableton `.als` | Logic *transfer package* |
| `protools2logic` | Pro Tools session | Logic *transfer package* |
| `ableton2protools` | Ableton `.als` | Pro Tools *transfer package* |
| `logic2protools` | Logic `.logicx` | Pro Tools *transfer package* |

Only the two `*2ableton` lanes synthesize a native project file. Every other lane emits a **transfer package** — stems, timeline MIDI, per-track MIDI, clip audio, manifests, and an import guide — because neither `.logicx` nor `.ptx` can be safely synthesized from scratch. Do not "improve" a package lane into a native-file lane without reading the format docs first.

## Commands

```bash
# Python tests (no third-party deps required)
python -m pytest tests -q
python -m pytest tests/test_protools_parser.py -q         # single file
python -m pytest tests/test_logic_parser.py::test_name    # single test

# Run the CLI from source (mode is auto-detected for the *2ableton/*2logic lanes)
python -m logic2ableton.cli "MySong.logicx" --output ./out
python -m logic2ableton.cli "MySet.als" --output ./out
python -m logic2ableton.cli "MySession.ptx" --output ./out
python -m logic2ableton.cli --mode logic2protools "MySong.logicx" -o ./out

# Build Python package / standalone binary
python -m build
pyinstaller logic2ableton.spec                            # -> dist/logic2ableton[.exe]

# Desktop app (from app/)
cd app && npm ci && npm run dev      # electron-vite dev
npm run build
npm run dist:mac / npm run dist:win   # packaged installers
```

Tests marked `needs_test_project` (a real `.logicx`) and `needs_vst3` (local VST3 plugins) auto-skip when those fixtures are absent — see `tests/conftest.py`. CI runs the full suite on Windows + macOS across Python 3.11–3.13.

## Architecture

**The Python package has zero third-party runtime dependencies** — only the stdlib (`plistlib`, `struct`, `gzip`, `wave`, `csv`, `xml.etree`). Keep it that way; it's a hard constraint for the PyInstaller bundle and the PyPI install. The Ableton template is bundled at `logic2ableton/data/DefaultLiveSet.als`, so no DAW install is needed to generate output.

Pipeline shape, mirrored in `cli.py` for every lane: **parse → match/report → generate → write report**. Reports are first-class artifacts emitted on *both* success and failure paths — they are the primary support and debugging artifact.

Module map:

- `cli.py` — one `main()` for all six lanes; also owns failure-report construction and `--json-progress` emission (one JSON line per stage, consumed by the Electron app).
- `models.py` — all dataclasses (`LogicProject`, `AbletonProject`, note/clip/track types) plus shared helpers `parse_audio_filename` and `samples_to_beats`. Logic audio filenames encode take/comp/bip structure that `parse_audio_filename` decodes.
- `logic_parser.py` — reads the `.logicx` bundle: plists (`ProjectInformation.plist`, `Alternatives/{NNN}/MetaData.plist`) **and the reverse-engineered binary `ProjectData`** for plugins and MIDI. Logic numbers alternative folders arbitrarily (the only one may be `004`, not `000`), so `discover_alternatives`/`resolve_alternative` find the active one rather than assuming an index.
- `ableton_parser.py` — reads the gzipped-XML `.als` into an `AbletonProject`.
- `protools_parser.py` — parses `.ptx`/`.pts`/`.ptf` into a session model. Reverse-engineered; see the format doc before touching it.
- `protools_import.py` — maps a parsed Pro Tools session onto the transfer models the Ableton and Logic lanes already consume. This adapter is why Pro Tools input reuses the existing generators instead of adding parallel ones.
- `ableton_generator.py` — generates `.als` by **cloning the bundled template, stripping its tracks, and injecting ours**. Every XML `Id` must be globally unique — reassign via a global counter when cloning. Starting from a known-valid file is deliberate: it guarantees structural correctness.
- `logic_transfer.py` / `protools_transfer.py` — build the transfer packages for their respective destinations.
- `smf.py` — shared Standard MIDI File writer. `build_midi_note_file` is duck-typed over any track whose `.notes` expose `pitch`/`velocity`/`start_beats`/`duration_beats`, so it serves all three DAWs. `MIDI_TICKS_PER_QUARTER = 960`.
- `report.py` — `generate_report` is **Logic-specific** (it takes a `LogicProject`). Other lanes build their reports inside `cli.py`. Don't mistake it for a shared reporting layer.
- `plugin_matcher.py` / `plugin_database.py` / `vst3_scanner.py` — identify Logic AU plugins (4CC codes) and suggest VST3 equivalents in the report.

### Mode resolution

`_resolve_mode` tries, in order: a lane name as the **first positional arg** → the `--mode` flag → `_detect_mode`.

`_detect_mode` checks `argv[0]`'s stem (each lane is its own console script in `pyproject.toml`, so the installed `logic2protools` binary self-identifies), then falls back to the first non-flag token's extension: `.als` → `ableton2logic`, a Pro Tools suffix → `protools2ableton`, anything else → `logic2ableton`.

**Extension detection can only ever reach `logic2ableton`, `ableton2logic`, and `protools2ableton`.** The three package-producing lanes to Logic and Pro Tools are unreachable by inference — they need an explicit lane name or `--mode`. That is why `app/src/main/converter.ts` always passes `--mode` rather than relying on detection.

### Desktop app

`app/` is Electron + React 19 + TypeScript + Tailwind 4 via electron-vite. The UI does not reimplement conversion: `app/src/main/converter.ts` **spawns the Python CLI as a subprocess** and parses its `--json-progress` JSONL stdout. In dev it runs `python3 -m logic2ableton.cli` from the repo root; when packaged it runs the bundled PyInstaller binary from `process.resourcesPath`. The `ProgressEvent` interface there must stay in sync with the payloads emitted by `_emit(...)` in `cli.py`.

## Reverse-engineered formats

Two of the three DAWs store their data in undocumented binary formats, and both decoders were reverse-engineered against real projects. Read the relevant doc before changing either parser:

- Logic arrangement MIDI lives in `ProjectData` inside `EvSq` (`"qSvE"`) chunks, decoded at Logic's 960 PPQ and emitted relative to the project's earliest note — `docs/reverse-engineering-logic-pro-midi.md`.
- Pro Tools sessions — `docs/reverse-engineering-pro-tools-sessions.md`.

## Version & release

The version lives in **two** places that must be bumped together: `logic2ableton/__init__.py` (`__version__`) and `app/package.json` (`version`). `pyproject.toml` reads its version dynamically from `__init__.py`. Releases are cut by pushing a `v*` git tag — GitHub Actions (`.github/workflows/release.yml`) runs tests, builds the PyInstaller binary, and packages/uploads the Windows + macOS desktop installers.

@.claude/dean-guidelines.md
