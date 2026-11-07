# ConvertX — Product Requirements & Technical Build Specification (Revised)

**Project type:** Windows desktop application
**Working name:** ConvertX
**Primary language:** Python
**Primary UI framework:** PySide6
**Target platform:** Windows 11 (initially)
**Development approach:** AI-assisted / agentic development
**Budget:** Zero-cost development using free and open-source dependencies
**Processing model:** Local-first, offline-capable
**Document status:** Revision 2. Supersedes the original brief. See Appendix A for what changed and why.

---

# PART 0 — READ THIS FIRST: HONEST CONSTRAINTS

These constraints shape every decision below. They were underplayed in the original brief.

1. **Windows 11 hides classic context-menu entries.** A registry-based shell verb appears only under "Show more options" (or Shift+F10), not in the top-level Windows 11 menu. Getting into the top-level menu requires an `IExplorerCommand` COM handler with a package identity (sparse or MSIX package). That typically means code signing, which is not free, plus native COM work that is awkward in Python.
   - **Decision for MVP:** accept the classic verb under "Show more options". The "two clicks" goal is **dropped** for Windows 11 MVP. It is a future goal conditional on the modern-menu work (Part 4.3).
2. **The radial menu is a differentiator, not a speed feature.** Conversion is an occasional task, so users have no muscle memory to exploit. A radial menu adds process launch, Qt initialisation and an animated window between the click and the choice. It must earn its place by being fast and polished, and must never be the only way to use the product.
3. **Python + PySide6 + PyInstaller start slowly.** Onefile builds unpack to a temp folder on every launch. A context-menu tool that lags before drawing feels broken. Startup time is measured in the Phase 0 spike, not later.
4. **External tools are heavy and sometimes unreliable.** LibreOffice is a large install with slow cold starts. Pandoc cannot produce PDFs on its own. The PDF toolchain is not "solved" by naming tools.
5. **Zero-cost distribution has a price.** An unsigned `.exe` triggers SmartScreen warnings and occasional antivirus false positives. This is documented and accepted for the MVP.
6. **"Universal converter" is an overclaim.** The MVP supports a deliberately small, reliable set of format pairs. Marketing copy must say so.

---

# PART 1 — PRODUCT VISION

## 1.1 What are we building?

ConvertX is a privacy-first, offline file conversion utility integrated into Windows File Explorer.

The user selects a file, invokes ConvertX from the context menu, and chooses an output format. ConvertX offers only the conversions that are genuinely supported and currently possible on that machine for that file.

Two presentation modes share one core:

- **Radial mode (signature UI):** an interactive segmented pie menu near the cursor.
- **List mode (default fallback and accessibility mode):** a plain, fast, keyboard-friendly popup list.

The user never has to upload files to a website or server.

## 1.2 Single source of truth for formats

The tables in Part 5 are the **only** authoritative list of supported conversions. Any example elsewhere in this document is illustrative. Formats mentioned in early drafts (RTF, EPUB, ICO, BMP/TIFF as outputs) are **not in the MVP** and appear only in the "Candidate later" column until validated.

## 1.3 Product principles

1. **Local-first:** files never leave the user's machine.
2. **Privacy:** no telemetry, analytics or cloud uploads. No network access during conversion, including remote resources referenced by input files.
3. **Context-aware:** only show conversions that are valid for the file **and** possible with the tools currently installed.
4. **Minimal interaction:** as few steps as the Windows version allows. On Windows 11 MVP this is right-click → Show more options → ConvertX → choose format.
5. **Reliability:** never silently corrupt, overwrite or delete an original file.
6. **Extensibility:** new converters can be added without rewriting the core.
7. **Performance:** measured, budgeted startup time (Part 8).
8. **Accessibility:** keyboard operation and screen-reader-friendly list mode are MVP requirements, not future work.
9. **Honesty:** never advertise fidelity or capabilities that are not tested.

## 1.4 Competitive context (must be reviewed before building)

Existing tools already add "Convert to…" context-menu entries on Windows (for example, the open-source **File Converter**). Before Phase 1, the agent must review at least two such tools and record in `TECHNICAL_DECISIONS.md`:

- What they already do well.
- Where ConvertX is meaningfully different.

If the only difference is the radial UI, that is acceptable, but it must be a conscious decision. The reliable conversion pipeline, not the radial menu, is the product's baseline value.

---

# PART 2 — USER EXPERIENCE

## 2.1 Primary workflow (Windows 11 MVP)

1. User right-clicks a file in File Explorer.
2. User chooses **Show more options** (or presses Shift+F10).
3. User chooses **ConvertX**.
4. ConvertX receives the selected file path.
5. ConvertX identifies the file type and probes which tools are available.
6. ConvertX displays the menu (radial or list mode) with supported formats.
7. User selects an output format (mouse, or keyboard: arrows / number keys / Enter).
8. Conversion starts.
9. A small progress/result window owned by ConvertX shows status (see 2.4).
10. The converted file is saved alongside the original.
11. The user sees a clear success or failure state with an "Open folder" action.

## 2.2 Radial menu design

Design requirements:

- Circular segmented pie layout (not buttons arranged in a circle).
- Transparent window background, dark modern aesthetic.
- Smooth open/close animation, kept short so it never delays the user.
- Individually interactive segments with hover highlighting.
- Format icons and short labels.
- Centre area shows the original filename and extension.
- Clear indication of the highlighted segment.
- Closes on Escape or click outside.
- Positioned near the invocation cursor, clamped to the monitor's work area.
- Correct on multi-monitor setups and mixed DPI scaling.
- Geometry code is independent of conversion logic.

### Interaction requirements (MVP)

- Mouse: hover highlights, click selects and triggers conversion.
- **Keyboard (MVP):** arrow keys move between segments, number keys select directly, Enter confirms, Escape closes.
- **List mode** is available at all times (setting plus automatic fallback) and is the accessible path. It uses standard Qt widgets so screen readers can read it. The custom-painted radial widget should expose accessible names/roles where Qt allows, but list mode is the accessibility guarantee.
- The menu must not block the desktop unnecessarily.

### Focus and dismissal

A frameless, transparent window launched from Explorer may not receive foreground focus reliably, which breaks Escape and click-outside-to-close. The Phase 0 spike must verify and solve this (for example with `activateWindow()`, `raise_()`, and if necessary the documented foreground-lock workarounds). If focus cannot be made reliable, the menu must fall back to a mode that does not depend on it.

### Dynamic generation

The menu is generated from the format registry for the specific file. Never hardcode a universal menu. Unsupported conversions never appear. Conversions that are supported in principle but whose tool is missing are handled per Part 5.8.

## 2.3 Multi-selection and single instance

- **MVP:** register verbs so they apply to a **single selected file only** (use the shell verb's `MultiSelectModel` set to `Single`; verify actual behaviour on Windows 11 in the spike). The user must not get N simultaneous menus.
- **Single-instance mechanism from day one:** if ConvertX is already running, a new launch forwards its file path to the existing instance (for example via `QLocalServer`/`QLocalSocket` or a named mutex) and exits. This is required even in single-file MVP, because rapid repeated invocations must not spawn competing windows.
- **Future:** true multi-file support uses the same single-instance channel to queue paths into one batch.

## 2.4 Process lifetime and post-menu UX

The original brief did not define this. Defined now:

- The ConvertX process stays alive until all its conversions finish, then exits. No resident background process in MVP.
- After the user picks a format, the menu closes and a small **ConvertX-owned status window** (compact, bottom-right of the work area, always-on-top while active) shows progress, then success or error.
- Success state: output filename, **Open file** and **Show in folder** actions, auto-dismiss after a few seconds unless hovered.
- Failure state: plain-language reason plus a "Copy details" action. Details go to the log file.
- Do **not** rely on `QSystemTrayIcon` toasts or OS notifications as the primary feedback. They may be used as optional extras only.

---

# PART 3 — TECHNICAL ARCHITECTURE

## 3.1 Stack (revised)

| Layer | Technology | Notes |
|---|---|---|
| Language | Python 3.11+ (pin exact version) | |
| Desktop UI | PySide6 | LGPL; see license table |
| UI drawing | Qt Widgets / QPainter | |
| Windows integration | stdlib `winreg` | **`pywin32` removed**; only add if a concrete need is demonstrated and documented |
| Document conversion | LibreOffice (headless) | External install; probed at runtime |
| Structured text conversion | Pandoc | External; **not** used for PDF output unless a PDF engine is confirmed |
| Image conversion | Pillow | |
| PDF text extraction | `pypdf` | **PyMuPDF excluded by default (AGPL)** |
| Audio/video | FFmpeg | Phase 2+ |
| Configuration | JSON | |
| Logging | Python `logging` (rotating file) | |
| Testing | pytest | |
| Packaging | PyInstaller **onedir** | Not onefile (startup cost) |
| Installer | Inno Setup (free) | |
| Version control | Git and GitHub | |

Every dependency must be justified in `TECHNICAL_DECISIONS.md` before it is added.

## 3.2 License review (to be verified, not assumed)

The agent must confirm each license from the project's official source before release and record the result. Starting expectations, **to be verified**:

| Component | Expected license | Concern |
|---|---|---|
| PySide6 | LGPL v3 (and commercial) | Keep dynamically linked (onedir helps); include license notices |
| Pillow | HPND (permissive) | Low |
| pypdf | BSD-style | Low |
| LibreOffice | MPL 2.0 | Invoked as an external program |
| Pandoc | GPL | Invoked as a separate process; if bundled, redistribute with license and source offer |
| FFmpeg | LGPL or GPL depending on build | Choice of build matters if bundling |
| PyMuPDF | AGPL (or commercial) | **Avoid** unless the project is knowingly released under AGPL |
| `pdf2docx` | Depends on PyMuPDF | Inherits the AGPL concern; PDF→DOCX is deferred |

## 3.3 Architecture overview

Responsibilities are strictly separated.

**A. Launcher (fast path)**
- Receives file path from Windows, validates arguments, handles single-instance forwarding, then starts the UI.
- Keep imports minimal. Do not import PySide6, Pillow or converter modules until needed.
- Entry points: `main.py` is the only executable entry point. `launcher.py` is removed; launcher logic lives in `src/convertx/launcher/` as a module imported by `main.py`.

**B. File detection engine**
- Extension-based detection plus signature (magic-byte) verification.
- Existence, readability, size checks.
- Handles OneDrive online-only placeholders: detect the offline/recall attributes and **do not** trigger a silent download. Report "file is not available offline" to the user (see Part 6).
- Handles long paths (>260 characters) using proper extended-path handling.

**C. Format registry**
- Single source of truth for input formats, output formats, handlers, display names and icons.
- Returns valid conversion options for a file, filtered by tool availability (see D2).

**D. Radial/list menu UI**
- Draws the menu, handles input, animations and file info display.
- Emits a "format selected" signal only. Contains no conversion logic.

**D2. Tool probe service (new)**
- At launch, locate LibreOffice, Pandoc and FFmpeg: PATH, then known install directories, then a bundled location, then user-configured paths.
- Cache results briefly; keep the probe fast (target: well under the startup budget).
- Provides capability status per converter: `available`, `missing`, `broken`.

**E. Conversion manager**
- Receives input and output formats, selects the handler, runs conversions asynchronously (QThread/QThreadPool or subprocess workers), reports progress and errors, returns the output path.

**F. Conversion modules**
- Documents, images, PDF, audio, video. Archives are future.

**G. Windows integration**
- Registers/unregisters the context-menu entry. Installs and uninstalls safely.

**H. Status window (new)**
- Progress, success and failure UI owned by ConvertX (Part 2.4).

## 3.4 Project structure

```text
ConvertX/
├── main.py                      # single executable entry point
├── requirements.txt
├── pyproject.toml
├── README.md
├── PROJECT_DESCRIPTION.md
├── ARCHITECTURE.md
├── TECHNICAL_DECISIONS.md
├── IMPLEMENTATION_PLAN.md
├── RISK_REGISTER.md
├── .gitignore
│
├── src/convertx/
│   ├── __init__.py
│   ├── launcher/        # arg parsing, single-instance forwarding
│   ├── core/
│   │   ├── file_detector.py
│   │   ├── format_registry.py
│   │   ├── tool_probe.py
│   │   ├── conversion_manager.py
│   │   ├── conversion_result.py
│   │   └── job_manager.py
│   ├── ui/
│   │   ├── list_menu.py          # plain popup (built first)
│   │   ├── radial_menu.py
│   │   ├── radial_painter.py
│   │   ├── radial_segment.py
│   │   ├── status_window.py
│   │   ├── animations.py
│   │   └── theme.py
│   ├── converters/
│   │   ├── base.py
│   │   ├── images.py
│   │   ├── documents.py
│   │   ├── pdf.py
│   │   ├── audio.py
│   │   └── video.py
│   ├── windows/
│   │   ├── registry.py
│   │   ├── context_menu.py
│   │   └── integration_manager.py
│   ├── services/
│   │   ├── file_service.py
│   │   ├── process_service.py
│   │   └── settings_service.py
│   └── resources/icons/
│
├── tests/
│   ├── fixtures/                 # sample files (Part 10)
│   └── ...
├── scripts/
│   ├── install_context_menu.ps1
│   └── uninstall_context_menu.ps1
└── build/                        # git-ignored
```

This is a proposal. The agent may improve it after architectural analysis, but must document any change.

---

# PART 4 — WINDOWS FILE EXPLORER INTEGRATION

## 4.1 MVP implementation

Use a conventional per-user registry shell verb. Expect it under "Show more options" on Windows 11.

Example invocation (conceptual):

```powershell
ConvertX.exe "C:\Users\User\Documents\Assignment.docx"
```

Paths must be properly quoted. Registry command values use `"%1"` with the executable path quoted.

## 4.2 Registry integration rules

- Register under **`HKCU\Software\Classes\SystemFileAssociations\.<ext>\shell\ConvertX`**. Do **not** modify ProgIDs or default-app associations, so users' default programs are never disturbed.
- No administrator rights. Per-user only.
- Register only for extensions that appear as inputs in the format registry.
- Set `MultiSelectModel` to `Single` for the MVP (verify behaviour).
- Use `winreg` from the standard library.
- Every key and value written is recorded in `docs/REGISTRY_CHANGES.md` and in an install manifest file.
- The uninstaller removes **only** keys listed in the manifest. It never deletes unrelated entries.
- Installation and uninstallation are idempotent and reversible.
- **Ask for explicit user approval before the agent first writes to the registry** (agent rule 15).

## 4.3 Two integration modes

**Mode 1: Launch verb (MVP).** Context menu entry "ConvertX" launches the app, which shows the radial or list menu.

**Mode 2: Cascading submenu fallback (build alongside or immediately after MVP).** Register a native cascading submenu per extension (`SubCommands` / `ExtendedSubCommandsKey`) listing output formats directly: `ConvertX ▸ PDF / TXT / HTML …`. Each submenu item runs ConvertX in direct-convert mode with arguments naming the target format. Benefits: no Python process needed at menu-render time, faster feel, works for users who dislike the radial menu. The radial menu then becomes the "rich mode".

Note: a static submenu cannot reflect tool availability at menu time. Direct-convert mode must therefore re-probe tools and report a clear error if a tool has gone missing.

## 4.4 Modern Windows 11 menu (future, gated)

Investigate only after the MVP is stable:

- `IExplorerCommand` implementation with sparse or MSIX package identity.
- Signing options and their real cost.
- Native component (C++/C#/Rust) rather than Python.
- Multi-file selection and flyouts.

The agent must produce a go/no-go note with cost and effort estimates before any work begins. Do not inject DLLs into Explorer or use process hooking.

---

# PART 5 — CONVERSION ENGINE

## 5.1 Philosophy

Plugin-like converters. Each converter declares:

- Supported input formats and output formats.
- Required external tools/dependencies.
- Lossless vs lossy.
- Offline capability (must be true for everything shipped).
- Known limitations (shown to the user where relevant).

The conversion manager selects the handler automatically. Never assume every format converts to every other format.

## 5.2 MVP scope (deliberately small)

The MVP ships a short list that is tested end to end. Anything else is Post-MVP.

### MVP tier 1 — Images (Pillow, no external tools)

| Input | Output |
|---|---|
| PNG | JPG, WEBP |
| JPG | PNG, WEBP |
| WEBP | PNG, JPG |

### MVP tier 2 — Documents (LibreOffice)

| Input | Output |
|---|---|
| DOCX | PDF, TXT |

### MVP tier 2 — PDF (pypdf)

| Input | Output |
|---|---|
| PDF | TXT (text-based PDFs only; see limitations) |

### Post-MVP (validate each before promising)

| Input | Output | Notes |
|---|---|---|
| BMP, TIFF | PNG | Pillow |
| GIF | PNG (first frame, clearly labelled) | Pillow |
| DOCX | HTML, Markdown | Pandoc and/or LibreOffice; Word-specific features will not all survive |
| Markdown | HTML | Pandoc or a Python Markdown library |
| Markdown / HTML / TXT | PDF | Toolchain **must be chosen and tested first** (see 5.3) |
| TXT / Markdown | DOCX | Pandoc |
| PDF | DOCX | Best-effort only; layout, tables and columns degrade; scanned PDFs need OCR |

### Candidate later (not committed)

RTF, EPUB, ICO, other image formats, archives.

## 5.3 PDF toolchain (must be defined before promising any PDF output)

Pandoc alone cannot create PDFs. It needs a PDF engine (typically LaTeX, or an HTML-to-PDF engine). Requiring users to install LaTeX violates the lightweight goal. Options to evaluate in the spike and record in `TECHNICAL_DECISIONS.md`:

| Route | Pros | Cons |
|---|---|---|
| LibreOffice headless for DOCX/TXT/HTML → PDF | One tool already required for DOCX | Heavy; HTML layout varies |
| Markdown → HTML → PDF via Qt (`QTextDocument` / `QPrinter`) | No new dependency | Limited CSS; basic layout |
| Pandoc + external PDF engine | Good typography | Large extra dependency |

Until a route is chosen and tested, **no row with PDF as an output (other than DOCX→PDF) may appear in the UI.**

## 5.4 Image rules

- Preserve transparency where the target supports it.
- JPEG output: configurable background colour for transparent inputs (default white).
- Preserve EXIF orientation (apply it) and keep metadata where the format supports it.
- Animated inputs: first frame only, and the UI must say so.
- Set Pillow's `Image.MAX_IMAGE_PIXELS` to a defined limit and handle `DecompressionBombError` with a clear message.
- Stream or process safely; do not load huge files without limits.

## 5.5 Document rules

- Run LibreOffice with a **dedicated profile directory** (`-env:UserInstallation=file:///…`) so conversion does not collide with, hang on, or modify the user's own LibreOffice instance.
- Disable or set the strictest macro security; never execute macros.
- Enforce a timeout; kill the process tree on timeout or cancellation.
- Serialize LibreOffice runs or use isolated profiles per job. Do not run parallel conversions against one profile.
- Detect password-protected inputs and report clearly instead of hanging.
- HTML inputs: block all network access and remote resource loading. A conversion must never contact the internet.
- Never promise perfect fidelity. Surface a limitation note where relevant.

## 5.6 PDF rules

- `pypdf` text extraction quality varies. If extraction yields no text, treat the file as likely scanned and tell the user OCR is not available yet.
- Password-protected PDFs: report clearly. Do not attempt to bypass encryption.
- PDF→DOCX is deferred; see 5.2.

## 5.7 Audio and video (Phase 2/3, post-MVP)

FFmpeg-based. Requirements unchanged from the original brief: quality presets, bitrate selection, metadata preservation, progress reporting, async execution, clear errors. Candidate formats: MP3, WAV, FLAC, AAC, OGG, M4A; MP4, MKV, MOV, AVI, WEBM. FFmpeg build licensing (LGPL vs GPL) must be decided before bundling.

## 5.8 Behaviour when a tool is missing

Decision (replaces the ambiguity in the original brief):

- If a conversion's required tool is **missing**, the output format is **shown disabled** (greyed) with a one-line reason, e.g. "Requires LibreOffice — not found. Install or set path in Settings." It is never silently hidden, because silent disappearance looks like a bug.
- If the tool is `broken` (found but failing a version check), show the same with a "tool not working" reason.
- Conversions that are not supported at all never appear.
- The status shown reflects the probe at launch time.

## 5.9 Archives (future)

Distinguish extraction, repackaging and true conversion. Do not label unsupported operations as conversion.

---

# PART 6 — FILE SAFETY AND RELIABILITY

Critical requirement. The application handles arbitrary user files.

1. Never modify the original file.
2. **Never overwrite an existing output file. Never prompt about it.** On a name collision, automatically generate a unique name (`report.pdf` → `report (1).pdf` → `report (2).pdf`). This is the single naming policy and replaces the contradictory rules in the original brief.
3. Use temporary directories for intermediates; write the final output atomically (write to temp, then rename into place) so a failed run never leaves a corrupt output.
4. Clean up temporary files after success **and** after failure or cancellation.
5. Handle locked or inaccessible files with a clear message.
6. Handle filenames with spaces, Unicode and special characters.
7. Handle paths longer than 260 characters.
8. Handle very large files without loading everything into memory; set sensible size and pixel limits.
9. Use subprocess **argument lists**, never shell string concatenation. Never use `shell=True` with user-controlled input.
10. Validate input and output paths. Output folder must be writable; if it is read-only or on an unavailable network location, fail clearly (and optionally offer a fallback folder in a later release).
11. Handle missing external executables (Part 5.8).
12. Never execute embedded macros or scripts from input documents.
13. Block network access during conversion, including HTML remote resources and document external links.
14. **OneDrive / cloud placeholders:** if the file is online-only, do not read it in a way that triggers a hidden download. Tell the user it must be made available offline first.
15. Guard against decompression bombs and pathological inputs (pixel limits, timeouts, output size sanity checks).
16. Handle password-protected files by reporting, not bypassing.
17. Do not silently process unsupported file types.
18. Keep error messages understandable to normal users. Technical detail goes to the log.
19. All conversions are local. No cloud API or LLM in the conversion process.

---

# PART 7 — USER INTERFACE AND DESIGN

## 7.1 Visual direction

Premium, modern, minimal dark interface; Windows 11 inspired; subtle glass-like effects; smooth but fast animations; clean typography. Avoid excessive gradients, too many colours, clutter, oversized labels and generic dashboard aesthetics.

## 7.2 Radial geometry

- QPainter-based; evaluate alternatives in `TECHNICAL_DECISIONS.md`.
- Divide the circle into N equal sectors for any N the registry returns (including N = 1 and large N with a sensible overflow strategy, e.g. a "More…" segment or switch to list mode above a threshold).
- Compute segment angles, hit-testing, hover state and label placement independent of conversion logic.
- Handle DPI scaling and per-monitor DPI correctly; clamp to the monitor's available geometry.
- Disabled (tool-missing) segments are visibly distinct and not clickable but expose their reason on hover/focus.

## 7.3 Deferred interface elements

Settings window, supported-format explorer, conversion history, theme selection, default output directory. Do not build a large settings dashboard in the MVP. A minimal settings file (JSON) plus a few essentials (menu mode: radial/list, tool paths) is enough.

---

# PART 8 — PERFORMANCE AND APPLICATION LIFECYCLE

## 8.1 Performance budget (targets to validate in Phase 0)

These are initial targets, not guarantees. Phase 0 measures actuals and the plan is adjusted.

| Measure | Target |
|---|---|
| Click on context-menu entry → menu visible (warm, SSD) | ≤ ~500 ms |
| Same, cold start (first launch after boot) | ≤ ~1.5 s |
| Tool probe time | ≤ ~100 ms (cached) |
| UI thread blocking during conversion | None |

If the targets cannot be met with PySide6 + PyInstaller onedir, evaluate: lazy imports, a thinner launcher that defers Qt, trimming the Qt module set, or the cascading submenu mode (4.3) as the primary path.

## 8.2 Lifecycle rules

- Long-running conversions never block the UI thread (QThread/QThreadPool or subprocess workers).
- Support cancellation where the underlying tool allows; kill process trees cleanly.
- Release resources after each conversion.
- No persistent background process in the MVP.
- Measure first, optimize second, but **measure in week one**.

---

# PART 9 — DEVELOPMENT PHASES (RE-ORDERED BY RISK)

Principle: prove the riskiest assumptions first. Do not polish UI before the pipeline works.

## Phase 0 — Feasibility spike (1–2 days, throwaway code allowed)

Goal: confirm or kill key assumptions before committing to the architecture.

Tasks:
1. With user approval, register a per-user shell verb for `.png` under `SystemFileAssociations`, and confirm where it appears on Windows 11.
2. Launch a minimal PySide6 window at the cursor from that verb. Measure cold and warm start times (source run **and** a PyInstaller onedir build).
3. Verify foreground focus, Escape and click-outside dismissal for a frameless transparent window launched from Explorer.
4. Run one real PNG→JPG conversion with Pillow and verify the output opens.
5. Probe for LibreOffice; run one headless DOCX→PDF with a dedicated profile; record cold-start time.
6. Verify `MultiSelectModel=Single` behaviour.
7. Record findings.

Go/no-go gates:
- If startup is far over budget, adopt mitigation options in 8.1 or promote cascading submenu mode to primary.
- If focus handling is unreliable, list mode becomes the default and radial becomes optional.

Deliverable: `SPIKE_REPORT.md` with measurements and decisions.

## Phase 1 — Research and architecture documents

Informed by the spike. Produce:
- `ARCHITECTURE.md`, `TECHNICAL_DECISIONS.md` (including the competitor review and PDF toolchain decision), `IMPLEMENTATION_PLAN.md`, `RISK_REGISTER.md`.

No application code in this phase except what the spike already produced.

## Phase 2 — Core foundation

- Project structure, `pyproject.toml`, pinned dependencies, logging.
- Launcher with argument validation and **single-instance forwarding**.
- File detection (extension + signature, placeholders, long paths).
- Format registry (single source of truth).
- Tool probe service.

Acceptance: accepts a path, categorizes the file, returns correct options (including disabled-with-reason states), handles invalid paths gracefully, and a second launch forwards to the first instance.

## Phase 3 — Vertical slice (ugly but real)

End to end with a **plain list popup**, no radial yet:

Right-click `.png` → ConvertX → list popup → PNG→JPG → output beside original → status window with success/failure.

Acceptance: works from Explorer with the registry verb, original untouched, collision handled with auto-suffix, UI responsive, uninstall script removes only ConvertX keys.

This phase proves the product. Everything after it is expansion.

## Phase 4 — Conversion engine expansion

- Converter interface, conversion manager, job manager, worker threads.
- All MVP tier 1 image conversions.
- DOCX→PDF and DOCX→TXT (LibreOffice, dedicated profile, timeout, macro-safe).
- PDF→TXT (pypdf, scanned/encrypted detection).
- Error handling and tool-missing behaviour.

Acceptance: each converter works independently, outputs open correctly, originals untouched, failures reported clearly, UI never blocks.

## Phase 5 — Radial menu

Build the radial UI on top of the working pipeline.

- Dynamic segments, hover and click, animation, Escape and click-outside, placement near cursor, DPI and multi-monitor support.
- **Keyboard navigation (arrows, number keys, Enter) is required here, not deferred.**
- Disabled-segment rendering.
- Menu mode setting (radial / list) with automatic fallback.

Acceptance: identical results to the list popup, no regression in latency beyond budget, accessible via list mode.

## Phase 6 — Windows integration hardening and cascading mode

- Final install/uninstall scripts with manifest-based removal.
- Cascading submenu mode (4.3).
- Registry documentation completed.
- Clean install/uninstall verification.

Acceptance: right-clicking a supported file shows ConvertX (under Show more options on Win11); correct file received; correct formats shown; uninstall restores original state.

## Phase 7 — Complete MVP verification

Test through File Explorer: DOCX→PDF, DOCX→TXT, PNG→JPG, PNG→WEBP, JPG→PNG, PDF→TXT. No internet required. No original file damaged. Errors clear.

## Phase 8 — Post-MVP expansion

Only after MVP is stable: Pandoc-based document conversions, additional image formats, chosen PDF-output route, audio, video, batch/multi-file, history, settings, advanced radial interactions, PDF→DOCX (with the AGPL decision made), OCR, modern-menu investigation (4.4).

## Phase 9 — Packaging and distribution

- PyInstaller **onedir** build.
- Application icon.
- **Inno Setup** installer that performs registration and offers uninstall.
- Decide external tool strategy: detect-only (preferred for MVP) versus bundle (requires license compliance and size increase). Do not claim portability until this is resolved.
- Document SmartScreen/antivirus behaviour for unsigned builds; investigate free signing options only if they genuinely exist and are acceptable.
- Clean install/removal testing on a fresh machine or VM.

---

# PART 10 — TESTING STRATEGY

Create `tests/fixtures/` with sample files. Test:

- Small and large documents; empty files; corrupted documents.
- Password-protected PDFs and Office files.
- Scanned (image-only) PDFs.
- Transparent PNGs; animated GIFs; EXIF-rotated JPEGs.
- Decompression-bomb image (confirm it is rejected safely).
- HTML referencing remote resources (confirm no network access).
- Unicode filenames, filenames with spaces, very long paths.
- Read-only inputs; read-only or unavailable output folders.
- Existing output files (auto-suffix works, nothing overwritten).
- OneDrive online-only placeholder behaviour.
- Missing LibreOffice, Pandoc, FFmpeg (disabled-with-reason UI).
- Broken tool installs.
- Rapid repeated launches (single-instance).
- Unsupported extensions.
- DPI scaling at 100%, 150%, 200% and mixed multi-monitor.
- Keyboard-only operation and list-mode accessibility.
- Startup-time regression check against the budget.

Rules:
- Every converter has unit tests; every workflow has an integration test.
- A feature is not complete because code runs. The resulting file must be opened and verified.
- Registry tests must be isolated or reversible and require user approval before running on a real registry.

---

# PART 11 — GIT AND DEVELOPMENT MANAGEMENT

- Use GitHub from the start.
- Solo project: `main` (stable) plus short-lived feature branches. A permanent `develop` branch is optional and not required.
- Commit after every working milestone.
- Never commit virtual environments or build outputs.
- Record dependency versions (pinned).
- Document breaking changes.
- Tag stable releases.
- Maintain the README.
- Use an appropriate `.gitignore`.

---

# PART 12 — DEFINITION OF DONE (MVP)

The MVP is complete only when all of the following are true:

1. ConvertX can be launched from File Explorer (via "Show more options" on Windows 11).
2. It receives the correct selected file.
3. It detects the input format (extension plus signature).
4. It generates the menu dynamically.
5. It shows only supported conversions, with unavailable-tool cases shown disabled with a reason.
6. The user can select an output format by mouse **and** keyboard, in radial or list mode.
7. The conversion executes successfully.
8. The output appears beside the original with a collision-safe name.
9. The original file is unchanged.
10. Errors are displayed clearly in the status window.
11. The application stays responsive and meets the measured performance budget (or the deviation is documented).
12. No network access occurs during conversion.
13. The integration uninstalls cleanly using the manifest, touching only ConvertX keys.

A beautiful radial menu without a working, tested conversion pipeline is not an MVP.

---

# PART 13 — AGENT DEVELOPMENT RULES

1. Never implement the application in one massive operation.
2. Never rewrite working modules without a clear reason.
3. Never install unnecessary dependencies; justify each one.
4. Never silently change the approved architecture.
5. Never invent conversion capabilities.
6. Never hardcode conversion options into the UI.
7. Never mix UI drawing code with conversion logic.
8. Never block the UI thread with long-running operations.
9. Never overwrite user files; use the auto-suffix policy.
10. Never introduce paid APIs or services.
11. Never introduce mandatory cloud processing.
12. Never claim a feature works without testing it.
13. Maintain documentation as the architecture evolves.
14. Report incomplete functionality honestly.
15. Ask for approval before administrator-level changes, registry modifications or destructive operations.
16. Prefer small, testable changes.
17. Run relevant tests after every meaningful implementation.
18. Preserve the ability to roll back to a working version.
19. **Prove risky assumptions with a spike before building on them.**
20. **Measure startup time against the budget at every phase, not just at the end.**
21. **Verify licenses from official sources before adding or bundling any dependency.**
22. **Do not build radial polish before the vertical slice works.**
23. When uncertain about a Windows API or library, consult official documentation instead of guessing.

---

# PART 14 — FUTURE PRODUCT POSSIBILITIES

Ideas, not MVP requirements: customizable radial menu, user-defined format groups, drag-and-drop, batch conversion, presets, image resizing/compression, PDF merge/split, audio extraction from video, metadata inspection, OCR, plugin SDK for third-party converters, portable mode, better multi-monitor behaviour, modern Windows 11 top-level menu integration.

Keep the architecture extensible enough to support these.

---

# PART 15 — OPEN DECISIONS (resolve in Phase 0/1)

| # | Decision | Where resolved |
|---|---|---|
| 1 | Is startup within budget with PySide6 + PyInstaller onedir? | Phase 0 |
| 2 | Is focus/dismissal reliable for a transparent frameless window from Explorer? | Phase 0 |
| 3 | Which route produces PDFs from TXT/Markdown/HTML? | Phase 0–1 |
| 4 | Is the radial menu the default, or list mode / cascading submenu? | After Phase 0 data |
| 5 | Detect external tools only, or bundle them? | Phase 9 planning |
| 6 | Accept AGPL (PyMuPDF/pdf2docx) or stay with pypdf only? | Before PDF→DOCX work |
| 7 | Pursue modern Windows 11 menu (signing, native component)? | Post-MVP go/no-go |
| 8 | What differentiates ConvertX from existing context-menu converters? | Phase 1 |

---

# FINAL PRODUCT GOAL

Build a genuinely useful, privacy-first Windows utility that makes file conversion fast and trustworthy. The radial interface is the signature feature, but product quality comes from the conversion engine, safe file handling and dependable Windows integration.

**Prioritize a working, tested product over an impressive prototype.**

---

# APPENDIX A — CHANGES FROM THE ORIGINAL BRIEF

| # | Change | Reason |
|---|---|---|
| 1 | Added Part 0 stating real constraints | Original underplayed Windows 11, startup and dependency risks |
| 2 | Windows 11 behaviour decided, "two clicks" dropped | Classic verbs live under "Show more options"; modern menu needs signing and native COM |
| 3 | Phases re-ordered: spike → vertical slice → radial | Riskiest assumptions were scheduled last |
| 4 | Added plain list popup mode and made it the accessibility path | Radial menu is slower, less accessible, unproven |
| 5 | Added cascading submenu fallback | Faster, no Python at menu time |
| 6 | Registry location fixed to `SystemFileAssociations`, manifest-based uninstall | Avoid disturbing default apps; safe removal |
| 7 | Removed `pywin32`; use `winreg` | Unnecessary dependency |
| 8 | Removed PyMuPDF by default | AGPL licensing risk; `pdf2docx` inherits it |
| 9 | Defined PDF toolchain requirement | Pandoc cannot make PDFs alone |
| 10 | Added LibreOffice dedicated profile, timeout, macro and password handling | Known hang and collision issues |
| 11 | Added tool probe and disabled-with-reason UI | Original left missing-tool behaviour undefined |
| 12 | Added single-instance handling and `MultiSelectModel=Single` | Prevent multiple simultaneous menus |
| 13 | Defined process lifetime and ConvertX-owned status window | Original had no post-menu UX |
| 14 | Keyboard navigation moved into MVP | Accessibility was listed as a principle but deferred |
| 15 | One naming policy: auto-suffix, never prompt, never overwrite | Original rules 2 and 3 contradicted each other |
| 16 | Added decompression-bomb, network-blocking, OneDrive placeholder and long-path rules | Security and real-world gaps |
| 17 | MVP format matrix shrunk; single source of truth established | Reliability over breadth; vision examples contradicted priority tables |
| 18 | Removed ambiguous `launcher.py` vs `main.py` | Single entry point |
| 19 | Added performance budget with measurable targets | "Measure first" needed concrete numbers and an earlier schedule |
| 20 | Added competitor review requirement | Differentiation was undefined |
| 21 | Packaging: onedir plus Inno Setup; unsigned-build caveat documented | Onefile startup cost; distribution realism |
| 22 | Added license review table (to be verified) | Original said "review licenses" without a method |
| 23 | Simplified Git branching for solo work | `develop` branch adds overhead without benefit |
| 24 | Added open-decisions register (Part 15) | Make unresolved choices explicit and tracked |
