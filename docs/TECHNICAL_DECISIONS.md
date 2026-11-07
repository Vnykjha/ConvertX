# ConvertX — Technical Decisions

**Status:** Proposed, awaiting owner approval. No application code has been written.
**Baseline spec:** `Project_Descritpion.md` (Revision 2). It supersedes the original `PROJECT_BRIEF.md`.
**Research date:** 7 November 2026.
**Companion docs:** `ARCHITECTURE.md`, `IMPLEMENTATION_PLAN.md`, `RISK_REGISTER.md`, `DEPENDENCIES.md`.

## How to read this document

Every decision lists: **Recommendation, Why, Alternatives considered, Trade-offs, Complexity, Evidence.**

Evidence tags:

- **[V]** verified against current sources during this research session (listed in Appendix A).
- **[K]** from background knowledge and **not** re-verified. These must be confirmed in the Phase 0 spike before anyone builds on them.
- **[S]** can only be settled by measurement or experiment (the spike).

Complexity scale: **S** (hours), **M** (1–3 days), **L** (about a week), **XL** (multiple weeks or a new skill set).

## Decision index

| ID | Decision | Status |
|---|---|---|
| TD-01 | Keep Python + PySide6, but build a stable CLI seam | Recommended |
| TD-02 | MVP Explorer integration = per-user classic registry verb | Recommended |
| TD-03 | Passing file paths and the command-line contract | Recommended |
| TD-04 | Windows 11 top-level menu = gated post-MVP track | Recommended |
| TD-05 | Process model and single-instance mechanism | Recommended |
| TD-06 | Menu UI built with QWidget + QPainter | Recommended |
| TD-07 | Radial geometry, positioning, focus, DPI | Recommended |
| TD-08 | Fallback entry points: list popup, cascading submenu, drop window | Recommended |
| TD-09 | Converter plugin architecture | Recommended |
| TD-10 | Execution safety: subprocesses, Job Objects, timeouts | Recommended |
| TD-11 | Document engine: LibreOffice; Pandoc optional | Recommended |
| TD-12 | Image engine: Pillow (plus image-to-PDF) | Recommended |
| TD-13 | PDF engine: pypdf now, pypdfium2 as pre-approved swap | Recommended |
| TD-14 | Audio/video: FFmpeg, detect-only, post-MVP | Recommended |
| TD-15 | File detection: extension plus signature | Recommended |
| TD-16 | Output naming and atomic, never-overwrite writes | Recommended |
| TD-17 | External tools are detected, not bundled (MVP) | Recommended |
| TD-18 | Packaging: PyInstaller onedir + Inno Setup | Recommended |
| TD-19 | Code signing and distribution reality | Needs owner decision |
| TD-20 | License of ConvertX itself | Needs owner decision |
| TD-21 | Config, logging and privacy | Recommended |
| TD-22 | Testing strategy and tooling | Recommended |
| TD-23 | AI-assisted development workflow | Recommended |

---

## TD-01 — Language and UI stack

**Recommendation.** Stay with Python 3.12 and PySide6. Define a **stable command-line contract** (TD-03) so that the shell integration, any future native extension, and any future UI rewrite all talk to the same `ConvertX.exe` interface.

**Why.**
- You chose it, it is free, and agentic tools are strongest in Python.
- PySide6 is LGPL (or GPL/commercial) and current releases support Python 3.10 to 3.14. [V]
- Everything that carries the product's value (detection, registry, converters, safety) is language-agnostic logic.

**Alternatives considered.**
- *C# / .NET with WPF or WinUI 3.* Faster cold start, native `IExplorerCommand`, first-class MSIX tooling in Visual Studio, which you already have. Cost: new language for you, and the AI-assisted workflow is less proven for WinUI packaging.
- *Rust or C++ for the shell piece only.* Needed only for the modern menu (TD-04).
- *Electron/Tauri.* Heavier or unnecessary, and a poor fit for a transparent pie overlay launched from the shell.

**Trade-offs.** Python start-up is the weak spot (TD-05). Python also cannot implement an in-process `IExplorerCommand` handler comfortably, so a modern-menu track means a second language.

**Complexity.** S.

**Honest note.** If the Phase 0 spike shows start-up well over budget and the cascading submenu (TD-08) is not acceptable, the right move is a native launcher or a C# rewrite of the thin front end, not heroic Python tuning.

**Evidence.** PySide6 version and license: [V]. Python 3.12 as the safest pin: [K].

---

## TD-02 — MVP Explorer integration

**Recommendation.** Register a **per-user classic shell verb** under `HKCU\Software\Classes\SystemFileAssociations\.<ext>\shell\ConvertX`, one key per supported input extension. It runs `ConvertX.exe --pick "<path>"`. Expect it to appear under **Show more options** on Windows 11. Accept this for the MVP and drop the "two clicks" claim.

**Why.**
- Microsoft's current documentation says the Windows 11 top-level menu lists only commands that implement `IExplorerCommand` and are registered through a package with identity. Classic registry verbs remain in the legacy menu. [V]
- Fresh reports from late October 2026 confirm that even a Windows 11 build with current updates still places classic entries under "Show more options". [V]
- Popular open-source converters such as File Converter have the same limitation on Windows 11. [V]
- Per-user `HKCU` needs no administrator rights and is trivially reversible.
- Writing under `SystemFileAssociations` does not touch ProgIDs or default-app choices.

**Alternatives considered.**

| Approach | Verdict |
|---|---|
| Classic registry verb (`shell\verb\command`) | **Chosen for MVP** |
| `IExplorerCommand` + sparse/MSIX package | Real fix for the top-level menu; deferred (TD-04) |
| Legacy `IContextMenu` COM shell extension | Obsolete direction; DLL loads into Explorer, which is a stability risk. Rejected |
| `SendTo` folder shortcut | Lives in the legacy menu too, so no gain. Rejected |
| `DelegateExecute` / `IExecuteCommand` handlers | COM complexity without a menu-placement benefit. Rejected |
| Global hotkey that reads Explorer's selection | Needs shell automation or UI Automation; fragile across Explorer changes. Rejected |
| Process injection or hooking Explorer | Prohibited by project rules. Rejected |
| Drag-and-drop window / desktop shortcut | Not a replacement, but a good supplementary entry point (TD-08) |

**Trade-offs.** One extra click on Windows 11. Windows 10 users get the entry directly in the main menu, so supporting Windows 10 is nearly free and gives those users the better experience.

**Registry rules.**
- Every key and value is written from a **manifest** (JSON) that the installer also stores in `%LOCALAPPDATA%\ConvertX\install-manifest.json`.
- Uninstall removes only what the manifest lists, and only if the values still point at ConvertX.
- Installer runs a **dry run first** and prints the exact keys it will write. The agent must get your approval before it runs any real registry write.
- Set `MultiSelectModel` to `Single` so a multi-selection does not spawn N windows. [K] The exact behaviour on Windows 11 is a spike item.

**Complexity.** S–M.

**Evidence.** Menu behaviour: [V]. `MultiSelectModel` semantics: [K]/[S].

---

## TD-03 — Passing file paths and the command-line contract

**Recommendation.** The shell verb's command is:

```
"<install dir>\ConvertX.exe" --pick "%1"
```

The executable exposes a small, versioned CLI. This is the **seam** that every other integration uses.

| Command | Purpose |
|---|---|
| `ConvertX.exe --pick <path>` | Show the format menu (radial or list) for one file |
| `ConvertX.exe --convert <path> --to <fmt>` | Direct conversion, no menu (used by the cascading submenu and scripts) |
| `ConvertX.exe --probe` | Print tool availability as JSON |
| `ConvertX.exe --install` / `--uninstall` | Register or remove shell entries, with `--dry-run` |
| `ConvertX.exe --version` | Version and build info |

Rules:
- Always quote `"%1"`. Windows delivers the command line as UTF-16, so Unicode names are safe in Python's `sys.argv`.
- Normalise the received path: absolute path, long-path form via `GetLongPathNameW` (the shell can hand over short 8.3 names in some cases [K]), reject directories and device paths.
- Use a **windowed** PyInstaller build (no console) so no console window flashes when launched from Explorer. Ship a separate console build only for debugging.
- External tools are started with the `CREATE_NO_WINDOW` flag for the same reason.
- Never build a shell string. Always pass argument lists.

**Why.** A versioned CLI makes the registry layer dumb and safe, lets the cascading submenu mode and a future native shell extension reuse the same entry point, and makes testing possible without a GUI.

**Alternatives considered.** Passing a JSON blob (unnecessary), named pipes from the shell (not available to a plain verb), `%L` instead of `%1` (equivalent for our purpose; we normalise anyway).

**Complexity.** S.

**Evidence.** [K]/[S]. Verify `%1` behaviour with Unicode, spaces and long paths in the spike.

---

## TD-04 — The Windows 11 top-level menu (gated, post-MVP)

**Recommendation.** Treat the top-level menu as a **separate track with a go/no-go gate after the MVP**. Do not start it until the conversion pipeline is proven. When started, build:

1. A tiny **native** `IExplorerCommand` DLL (Rust with the `windows` crate, or C++/WinRT, or C#) that does nothing except launch `ConvertX.exe --pick <path>`. It must do no heavy work, because it runs inside Explorer.
2. A **sparse MSIX package** (package with external location) declaring the `windows.fileExplorerContextMenus` extension, so the unpackaged ConvertX gets package identity.

**Why.**
- Microsoft documents exactly this route for unpackaged desktop apps. [V]
- Multiple current open-source projects (7-Zip shell extension, QuickLook, and others) use a native DLL plus a sparse package. [V]
- A thin DLL that only launches your exe keeps all logic in Python and honours "no DLLs in Explorer that do real work".

**The identity and signing problem (the real cost).**

| Route | Cost | Notes |
|---|---|---|
| Unsigned sparse package registered with `Add-AppxPackage -AllowUnsigned -ExternalLocation` | Free | Reports are inconsistent: one project needs Developer Mode, another registered it from an elevated installer without Developer Mode. Needs a spike on your target machines. [V]/[S] |
| Sparse package signed with a self-made certificate trusted on the user's machine | Free | Requires installing a trusted root. Poor security practice for distribution. Fine for your own machine. [V] |
| **Microsoft Store MSIX** | **Free** | Individual developer registration is now free worldwide, and Store-hosted MSIX packages are signed by Microsoft with no SmartScreen warning. Adds Store certification review and requires full MSIX packaging. [V] |
| SignPath Foundation | Free | Only for qualifying open-source projects: OSI-approved license, public repo, free downloads. Publisher shown is the foundation, not you. Approval is not guaranteed. [V] |
| Azure Artifact Signing | About US$10/month | **Not available to you**: individual developers are limited to the USA and Canada, and a paid Azure subscription is required (free or sponsored subscriptions are not accepted). [V] |
| Commercial OV certificate | Paid (open-source tiers start around €49) | Out of budget. [V] |

**Recommended order of attack when the gate opens.** (1) Prove the native shim and sparse package on your own machine with a self-signed cert. (2) Try the unsigned `-AllowUnsigned` route and document its limits. (3) Decide between Microsoft Store distribution and SignPath Foundation, which both depend on your TD-20 licensing choice.

**Alternatives considered.** Registry hacks that restore the classic menu globally (changes the user's whole system; rejected), shipping a full MSIX of the Python app as the *only* distribution (possible via Store, but a large packaging project).

**Trade-offs.** Two languages and a packaging toolchain. High reward for polish, high cost for the MVP.

**Complexity.** XL (for a Python-only developer).

**Evidence.** Mechanism: [V]. Cost and eligibility: [V]. Unsigned registration behaviour: [S].

---

## TD-05 — Process model and single instance

**Recommendation.** Start a **fresh short-lived process per invocation** (no resident background process in MVP). Add a **single-instance guard** using a named mutex plus a named pipe from the standard library (`ctypes` for the mutex and `multiprocessing.connection` with `AF_PIPE` for forwarding). A second launch sends its path to the running instance and exits.

**Why.**
- The spec forbids unnecessary background processes. A resident process also makes uninstall and updates harder.
- Using the standard library keeps the "forward and exit" path from importing Qt at all, so double-launches cost almost nothing.
- Single instance is needed even without multi-select, because users double-click and repeat invocations.

**Alternatives considered.**
- *`QLocalServer`/`QLocalSocket`.* Idiomatic Qt, but requires importing Qt in the second process before it can forward.
- *Resident tray process with a tiny native launcher.* Best latency, but breaks the no-background principle. It remains the **escape hatch** if the spike shows the budget cannot be met.
- *Socket on localhost.* Opens a network port. Rejected.

**Start-up measures to apply** (measure each, keep only what helps):
- Import only what each path needs. `QtWidgets` and Pillow load lazily.
- Do not use QtQuick or WebEngine.
- Trim the Qt modules in the PyInstaller spec.
- Compare PyInstaller against Nuitka in the spike (Nuitka is free and sometimes starts faster and triggers fewer antivirus false positives). [K]/[S]

**Complexity.** M.

---

## TD-06 — Menu UI technology

**Recommendation.** One frameless, translucent **QWidget** sized to the menu's bounding square (not a full-screen overlay), custom-painted with **QPainter**, animated with `QVariantAnimation`/`QPropertyAnimation`.

Window flags: `Qt.FramelessWindowHint | Qt.WindowStaysOnTopHint | Qt.Tool`, with `WA_TranslucentBackground`.

**Why.**
- Smallest dependency surface (QtWidgets only), fastest start-up, and a QPainter pie gives pixel-exact control of segments.
- A bounding-square window avoids a full-screen overlay that would block the desktop (a spec requirement).
- Custom painting keeps geometry testable (pure functions).

**Alternatives considered.**

| Option | Verdict |
|---|---|
| QML / QtQuick | Better animation tooling, but pulls in a larger runtime and slower start. Rejected for MVP |
| `QGraphicsView` + path items | Hit-testing for free, but heavier than needed |
| Native layered window with Win32 | Maximum control, no benefit over Qt here |
| Acrylic/blur-behind effect | System backdrop effects apply to rectangular top-level windows, so a circle would look wrong. Use faked glass (alpha gradients) instead. Skip for MVP |

**Trade-offs.** Custom painting means custom accessibility (TD-08 list mode covers it).

**Complexity.** M–L.

---

## TD-07 — Radial geometry, positioning, focus and DPI

**Recommendation (geometry).** Put all geometry in one pure module, `radial_geometry.py`, with no Qt imports. See `ARCHITECTURE.md` §6 for formulas. Key points:
- Segment *i* of *N* spans `360/N` degrees, with segment 0 centred at 12 o'clock and ordering clockwise.
- Hit-testing: compute `dx, dy` from the centre, radius `r = hypot(dx, dy)`, angle `θ = atan2(dy, dx)` in screen coordinates (y down, so angles are clockwise-positive), then normalise to the segment index. Ignore the centre dead zone and anything past the outer radius plus a small tolerance.
- QPainter's `arcTo` uses counter-clockwise degrees from 3 o'clock, so convert: `qt_angle = -screen_angle`.
- Cap at **8 radial segments**. More than that switches to list mode or adds a "More…" segment. Large N makes the labels unreadable and targets small.

**Recommendation (positioning).** Centre the menu on `QCursor.pos()` at launch, find the monitor with `QGuiApplication.screenAt(pos)`, clamp the centre so the entire circle fits inside `screen.availableGeometry()` (move the centre, never the geometry off-screen).

**Limits that cannot be solved.**
- A classic verb receives **no coordinates** from Explorer. The cursor position at launch is the only signal. It is a good approximation after a mouse click, but when the menu is opened with the keyboard (Shift+F10 or the Menu key) the cursor may be nowhere near the selected file. Fallback: centre the menu on the screen containing the cursor.
- Positioning relative to the Explorer context menu itself is not possible without hooking Explorer, which is prohibited.

**Recommendation (focus and dismissal).** A frameless, translucent window launched from another process may not get foreground focus, which breaks Escape and click-outside. Plan:
1. Call `show()`, `raise_()`, `activateWindow()` after positioning.
2. Dismiss on `QEvent.WindowDeactivate` (click-outside) because the window is not full-screen and cannot receive outside clicks.
3. If activation fails, fall back to the Win32 foreground helpers via `ctypes` (`AllowSetForegroundWindow`, `SetForegroundWindow`).
4. If still unreliable, default to list mode (TD-08). **[S]**: the spike decides.

**Recommendation (DPI and multi-monitor).** Qt 6 is per-monitor DPI aware by default. Rules: set the window's screen before showing it, keep all geometry in logical (device-independent) units, render icons as SVG, test 100%, 125%, 150%, 200% and mixed-DPI dual monitors, and test a primary/secondary monitor with different scale factors. Known risk: moving a window between monitors with different scales has produced size glitches in Qt before. [K]

**Recommendation (animation).** 120–160 ms ease-out, scale from about 0.9 and opacity from 0. Animate one property, repaint only on hover change, never animate window geometry. The menu must be interactive from the first frame.

**Complexity.** M–L. **Evidence.** [K]/[S].

---

## TD-08 — Fallback entry points

**Recommendation.** Build three non-radial paths. They are the real safety net.

1. **List popup** (standard Qt widgets). First UI built (vertical slice). Screen-reader friendly and the automatic fallback if focus or rendering fails.
2. **Cascading submenu mode.** Register native submenus per extension (`ExtendedSubCommandsKey` pointing to a key under `HKCU\Software\Classes\...`) so right-click → ConvertX → PDF runs `ConvertX.exe --convert <file> --to pdf` with no menu process at all. Instant to render. [K]: HKCU support for command stores must be verified in the spike; prefer `ExtendedSubCommandsKey` over a CommandStore entry.
3. **Drop window.** Launching `ConvertX.exe` with no arguments shows a small window that accepts dragged files. Useful for testing and as an entry point that does not depend on any shell integration.

**Why.** If the radial menu turns out slower or flakier than hoped, you still ship a good product. The radial menu becomes a "rich mode", chosen in settings.

**Trade-offs.** A static submenu cannot reflect tool availability at render time (it was registered earlier), so direct-convert mode must re-probe and show a clear error.

**Complexity.** List: S. Submenu: M. Drop window: S.

---

## TD-09 — Converter plugin architecture

**Recommendation.** An in-process, explicitly-registered **converter protocol**. Converters are plain classes that describe themselves; the registry is built from them.

Each converter provides:
- `specs()` → list of `ConversionSpec(input_format, output_format, lossy: bool, requires: list[ToolRequirement], notes: str, options_schema)`.
- `convert(request, ctx) -> ConversionResult`, where `ctx` carries a progress callback, a cancel token, a temp directory and the tool locations.

Rules:
- Converters never import Qt or the UI.
- The registry is the single source of truth. The UI and the registry-install scripts both read it, so the Explorer entries and the menu can never drift apart.
- Third-party plugins (Python entry points) are a **later** feature; design for it, do not build it.

**Alternatives considered.** Dynamic plugin discovery now (premature), one giant switch statement (violates extensibility), subprocess-per-plugin (heavy).

**Complexity.** M.

---

## TD-10 — Execution safety

**Recommendation.**
- External tools run as **subprocesses with argument lists**, never `shell=True`, started with `CREATE_NO_WINDOW`.
- Every subprocess is assigned to a **Windows Job Object** with `KILL_ON_JOB_CLOSE`. This is how you reliably kill LibreOffice's child `soffice.bin` on timeout, cancel or crash. Without it, orphaned background processes are the most common real-world failure. Use `ctypes` (no `pywin32`). [K]
- Mandatory per-job **timeouts** with sensible defaults (for example 120 s documents) and a cancel button.
- Concurrency: one LibreOffice job at a time per profile; use a ConvertX-owned profile directory (TD-11).
- Pillow runs in-process in a worker thread with hard pixel and size limits (TD-12). Malformed-file crashes in native decoders are a known risk class. For MVP the process is short-lived, so a crash is contained; a worker-process isolation mode is a flagged enhancement.
- Pass **copies in a temp directory** to tools when the source path is very long (over 260 characters) or on a network location, so external tools do not choke.

**Complexity.** M.

---

## TD-11 — Document engine

**Recommendation.** **LibreOffice headless** for DOCX → PDF and DOCX → TXT in the MVP. Detect it, never bundle it (TD-17).

Operational rules:
- Locate `soffice.exe` via registry keys and standard install paths, then PATH, then a user-set path. Read the version from the install's version file rather than launching the app. [K]
- Always use a dedicated `-env:UserInstallation=file:///…/ConvertX/lo-profile` so conversions never collide with, hang on, or modify the user's own LibreOffice session. [K]
- Pre-seed that profile with the strictest macro-security level; never enable macros. [K]
- Convert in a temp directory, then move the result to its final name (TD-16).
- Timeout, Job Object, cold-start measurement. Cold start is the dominant cost of every document conversion.

**New finding that changes the plan.** LibreOffice 26.2 added native **Markdown import and export**. [V] If its Markdown quality is acceptable, LibreOffice alone may cover DOCX ↔ Markdown and Markdown → PDF/DOCX, and **Pandoc could be dropped from the dependency set entirely**. This must be validated with real documents in the spike before the dependency list is final.

**Release channel note.** The 26.2 series ends its life on 30 November 2026, and 26.8.x is the current branch. [V] Test against both and state a supported-version range.

**Alternatives considered.**

| Option | Verdict |
|---|---|
| Pandoc | Good structured-text converter, but cannot create PDFs on its own (needs a PDF engine such as LaTeX or an HTML renderer). Optional, post-MVP. [K] |
| Microsoft Word automation via COM | Best DOCX fidelity, but requires licensed Word, runs macros unless carefully controlled, and is slow and fragile. Possible optional engine later |
| Python libraries (`python-docx`, `docx2pdf`, etc.) | `python-docx` only reads/writes the file structure and cannot render to PDF; `docx2pdf` just drives Word. Rejected |
| Qt `QTextDocument` printing | Cannot faithfully load DOCX. Possible for plain-text/HTML-to-PDF only (TD-13) |

**Residual risk.** LibreOffice may fetch linked external resources while loading a document, and **network access cannot be fully blocked without a firewall rule or sandbox**. Mitigation: pre-scan archives for external relationships, disable link updates in the profile, and document this as a known residual risk (R-09).

**Complexity.** M–L.

---

## TD-12 — Image engine

**Recommendation.** **Pillow** in-process, with these rules:
- Set `Image.MAX_IMAGE_PIXELS` to an explicit cap and handle `DecompressionBombError` with a clear user message.
- Apply EXIF orientation, preserve metadata where the target format allows it, preserve transparency, flatten onto a configurable background for JPEG.
- Animated inputs: first frame only, and the status window says so.
- **Add image → PDF as an MVP-plus conversion.** Pillow can write PDF directly with no extra dependency. It is high-value and nearly free.
- **HEIC/HEIF → JPG** is the most-wanted phone-photo conversion; plan `pillow-heif` as a post-MVP add-on (verify its license and wheel size).

**Alternatives considered.** ImageMagick (large, external, many historical security issues; File Converter uses it [V]); Wand (needs ImageMagick); `img2pdf` (loss-less JPEG-to-PDF; optional later).

**Complexity.** S–M.

---

## TD-13 — PDF engine

**Recommendation.**
- **PDF → TXT:** `pypdf` for the MVP (pure Python, BSD-3). Detect "no extractable text" and tell the user it is probably a scanned PDF (OCR is future work). Detect encryption and report it; never attempt to bypass it.
- **Pre-approved swap:** `pypdfium2` (Apache-2.0 or BSD-3, wraps Google's PDFium) if pypdf's text quality is judged poor. It extracts text faster and also renders pages to images, which an OCR feature would need. [V]
- **Excluded:** `PyMuPDF` is AGPL-3.0, so shipping it would require ConvertX itself to be AGPL or the purchase of a commercial licence from Artifex. [V] `pdf2docx` builds on it. [K]: verify before any PDF → DOCX work.

**PDF output from TXT/Markdown/HTML.** Not committed for the MVP. Candidate routes: LibreOffice (once its Markdown import is validated), Qt printing for simple text, or Pandoc plus an external engine. Until a route is tested, no such output may appear in the UI.

**PDF → DOCX.** Best-effort only; layout, tables and columns degrade, and scans need OCR. Post-MVP, behind an explicit "best effort" label.

**Complexity.** S (extraction). M (any PDF output).

---

## TD-14 — Audio and video

**Recommendation.** FFmpeg, **detect-only**, post-MVP. Use `-progress pipe:1` for progress, a Job Object for cancellation, and quality presets rather than raw flags. Decide the build licence (LGPL vs GPL) only if bundling is ever considered. [K]

**Alternatives considered.** `ffmpeg-python` (thin wrapper; direct `subprocess` is simpler and safer), PyAV (links FFmpeg libs into the process; heavier).

**Complexity.** M for audio, L for video.

---

## TD-15 — File detection

**Recommendation.** Extension is the **hint**, content signature is the **check**. Use a small built-in table of magic bytes for the supported formats; use ZIP inspection to tell DOCX from other Office files (look for the main document part); use Pillow's own verification for images. No `python-magic`/libmagic dependency (it needs a native library on Windows).

Also in this module:
- Reject directories, devices, zero-byte files and unreadable files with specific errors.
- Detect **cloud placeholders**: check `st_file_attributes` for the recall-on-open and recall-on-data-access bits. If set, do **not** read the file; tell the user it must be made available offline first. [K]
- Detect and report the 260-character limit and long-path forms.
- Cheap **zip-bomb guard** for Office files: sum declared uncompressed sizes before handing to LibreOffice.

**Mismatch policy.** If the extension and signature disagree (a `.png` that is really a JPEG), trust the content, tell the user, and offer the conversions for the real type.

**Complexity.** M.

---

## TD-16 — Output naming and writes

**Recommendation.** One policy, no prompts:
1. The output is `<stem>.<newext>` beside the original.
2. If that name exists, use `<stem> (1).<newext>`, then `(2)`, and so on.
3. **Never overwrite.** To make this race-free, write the result to a temp file *in the output directory*, then publish it with `os.rename` in a retry loop. On Windows, `os.rename` **fails if the destination exists** (unlike `os.replace`), which gives an atomic "create only if absent" primitive. [K]
4. A failed or cancelled job leaves no partial output.
5. If the output folder is read-only or unavailable, fail clearly. A fallback folder (for example Documents\ConvertX) is a later setting.

**Alternatives considered.** Prompting on collision (interrupts flow, contradicts "never prompt"), overwriting with a backup (risky), timestamps in names (noisy).

**Complexity.** S.

---

## TD-17 — External tool strategy

**Recommendation.** **Detect, do not bundle**, for the MVP. A `ToolProbe` service finds LibreOffice (and later Pandoc, FFmpeg), reports `available / missing / broken`, and the menu shows unavailable conversions **disabled with a one-line reason and a "how to fix" link**. An optional user setting points to a custom path.

**Why.** Bundling LibreOffice (hundreds of MB) is not realistic for a free project, and GPL builds of other tools add licence obligations. Images and PDF→TXT work with **zero** external installs, so the product is useful immediately.

**Trade-offs.** The installer cannot promise "everything works". The README and first-run message must state that DOCX conversion needs LibreOffice. A later winget-based "install helper" is a possible convenience, not an MVP item. [K]

**Complexity.** M.

---

## TD-18 — Packaging

**Recommendation.** **PyInstaller in onedir mode**, then **Inno Setup** (free) to produce a **per-user** installer (no administrator rights, `HKCU` registration, install under `%LOCALAPPDATA%\Programs`). The installer calls `ConvertX.exe --install` so registry logic exists in exactly one place, and the uninstaller calls `--uninstall`.

**Why.** Onefile unpacks to a temp folder on every launch and is the single worst choice for a context-menu tool. Per-user install matches the per-user registry approach and avoids UAC.

**Alternatives considered.** Nuitka (evaluate in the spike), MSIX (required later for the modern menu or Store, TD-04), cx_Freeze/py2exe (smaller communities, no advantage), a plain zip plus scripts (fine for early testers).

**Trade-offs.** Antivirus false positives on PyInstaller bootloaders are common. Mitigations: build only in public CI, report false positives to Microsoft, keep versions reproducible, and prefer signing (TD-19).

**Complexity.** M.

---

## TD-19 — Code signing and distribution reality

**This needs your decision, because the free options depend on it.**

What is true today for an individual developer in India. [V]
- **Azure Artifact Signing is not available to you** (individuals must be in the USA or Canada) and it requires a paid Azure subscription, so the Student Pack's Azure credit does not help.
- **Microsoft Store registration is free for individuals worldwide**, and Store-hosted MSIX packages are signed by Microsoft and install without SmartScreen warnings.
- **SignPath Foundation** signs qualifying open-source projects for free, but needs an OSI licence, a public repository, free downloads, and an existing maintained project. The publisher name shown is the foundation's.
- No signature means SmartScreen warnings, and some organisations block unsigned installers outright.

**Recommendation.**
1. **MVP and early testers:** unsigned, distributed through GitHub Releases, with honest README instructions about SmartScreen. Build in public CI so the binary is reproducible.
2. **Public release:** pursue **SignPath Foundation** if you choose an OSI licence (TD-20), or the **Microsoft Store** if you take the MSIX path (TD-04). Decide after the MVP.

**Complexity.** M (SignPath application) to L (Store plus MSIX).

---

## TD-20 — Licence for ConvertX itself

**Recommendation.** Choose an OSI-approved licence and publish the source. **MIT or Apache-2.0** keeps options open.

**Why.**
- It unlocks the free SignPath Foundation route (TD-19).
- It is compatible with the dependency set when external tools are detect-only and PySide6 stays dynamically linked (LGPL).
- It avoids having to relicense later.

**If you want a closed-source or commercial product,** the picture changes: avoid PyMuPDF, check PySide6's LGPL obligations, and forget SignPath Foundation. This is the one decision that is awkward to reverse.

**Evidence.** [V] for SignPath eligibility and PySide6 licensing; confirm the licence text of each dependency before release (see `DEPENDENCIES.md`).

---

## TD-21 — Config, logging and privacy

**Recommendation.**
- Settings: JSON at `%APPDATA%\ConvertX\settings.json`, schema-validated, with safe defaults and a "reset" command.
- Logs: rotating file at `%LOCALAPPDATA%\ConvertX\logs\`, **never** includes file contents, and file paths are logged only at debug level.
- **No telemetry, no analytics, no update pings, no crash upload.** Crash details stay local, with a "Copy details" button for bug reports.
- No network code in the application at all; add an automated test that fails if a network module is imported by the core.

**Complexity.** S.

---

## TD-22 — Testing

**Recommendation.**
- `pytest`, plus `pytest-qt` for widget tests.
- Fixtures are **generated by code** (small PNGs, JPEGs, a transparent PNG, an animated GIF, a decompression-bomb PNG, a tiny DOCX, a text PDF, an encrypted PDF, a corrupted file) so the repo stays small and legal.
- Geometry tests are pure and exhaustive (N = 1…8, boundary angles, centre dead zone).
- Registry tests use a **throwaway subtree** (for example `HKCU\Software\ConvertX-Test`) via an injectable root, never the real file associations.
- Clean-machine tests (install, run, uninstall, verify registry is restored) run in **Windows Sandbox** or a free Windows 11 VM.
- `import-linter` (free) enforces the layer boundaries from `ARCHITECTURE.md`; `ruff` for lint and format.
- A start-up benchmark script runs in CI-adjacent manual checks and its results are stored in the repo.

**Complexity.** M.

---

## TD-23 — AI-assisted workflow

**Recommendation.** Use Google Antigravity (agentic IDE), VS Code, or Visual Studio as the editor/agent host. The tool itself is interchangeable; the discipline is not.
- Put the agent rules (spec Part 13) in whichever project-rules file your agent tool reads (check its documentation for the filename; this was not verified).
- One milestone per agent task. Each task starts with "write a plan, wait for approval", then small commits, then tests.
- **The agent cannot see File Explorer.** All Explorer, DPI and focus checks are manual, on real Windows 11, and recorded in the milestone's checklist.
- Never let the agent edit the registry directly. It generates scripts and dry-run output; you review and run them.
- GitHub Student Pack: GitHub Pro (private repos, Actions minutes), and JetBrains tools are useful. Copilot Student sign-ups were reported as paused in 2026, so do not plan around it. [V]
- Visual Studio Community is the right tool if you reach the native shell shim (TD-04), because it includes the C++/C# toolchains and MSIX packaging support. [K]

**Complexity.** S.

---

## Appendix A — Sources consulted (7 November 2026)

Windows 11 menu and packaging:
- Microsoft Learn, "Add a File Explorer context menu command to a packaged desktop app" (IExplorerCommand, sparse packages).
- GitHub issue #150, open-file-lock-handle (30 October 2026): identity-package requirement, unsigned registration result, classic entries still under "Show more options".
- GitHub: QuickLook PR #2048, 7-Zip Explorer Extension, Arca release notes, ZipNest PR #1, OSS Share: real-world sparse-package implementations and their signing/Developer Mode caveats.
- Microsoft Learn, `Add-AppxPackage` reference (`-AllowUnsigned`, `-ExternalLocation`).

Signing and distribution:
- Microsoft Learn, "Code signing options for Windows app developers".
- Microsoft Q&A, "Artifact Signing" (eligibility; free/sponsored subscriptions not supported).
- Windows Developer Blog (10 September 2025; 7 May 2026): free Store registration for individuals and companies.
- SignPath Foundation eligibility summaries (Zenn article; GordianXI issue #279, checked 5 October 2026).

Libraries and licences:
- PyPI/CERN mirror: PySide6 6.10.0 and 6.11.0 pages.
- LibreOffice download page and 2026 release coverage (26.2 Markdown import/export; 26.8 branch; 26.2 end of life 30 November 2026).
- PDF library comparisons (PyMuPDF AGPL-3.0; pypdf BSD-3; pypdfium2 Apache-2.0/BSD-3).
- GitHub: Tichau/FileConverter (GPL-3.0; ImageMagick, FFmpeg, Ghostscript back ends).
- Student pack coverage (Copilot sign-ups paused; Azure credit; JetBrains).

**Not verified this session:** Antigravity features and rules-file conventions, Pandoc PDF-engine details, FFmpeg build licences, `pdf2docx` licence, exact `MultiSelectModel` and `%1` behaviour, and every start-up timing. They are tagged [K] or [S] above.
