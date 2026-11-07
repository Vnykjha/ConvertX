# ConvertX — Dependencies

**Status:** Proposed, awaiting owner approval. Nothing has been installed.
**Research date:** 7 November 2026.
**Principle:** the smallest dependency set that delivers a working MVP. Every addition needs a written justification here before it enters `requirements.txt`.

Evidence tags: **[V]** verified from current sources this session, **[K]** background knowledge, **verify before release**.

---

## 1. MVP dependency set (the whole list)

### 1.1 Python packages shipped with the app

| Package | Purpose | Why needed | License | Evidence |
|---|---|---|---|---|
| **PySide6** (use `PySide6-Essentials` if sufficient) | Qt UI | The UI framework chosen for the project; the menu needs only QtCore, QtGui, QtWidgets | LGPL-3.0 or GPL-2.0/3.0 or commercial | [V] |
| **Pillow** | Image conversion (and image→PDF) | Covers every MVP image conversion with no external tool | Permissive (MIT-CMU/HPND style) | [V] snippet, confirm in release check |
| **pypdf** | PDF → TXT | Pure-Python, small, permissive | BSD-3-Clause | [V] |

Current PySide6 release seen: **6.11.0** (23 March 2026), requiring Python 3.10 to 3.14. A newer patch release may exist; pin whatever is current at spike time. [V]

> Note: the `PySide6` wheel is an alias for `PySide6_Essentials` plus `PySide6_Addons`. [V] Installing **Essentials only** avoids shipping unused Qt modules (WebEngine, 3D, Quick, and so on) and is the first, cheapest bundle-size win. Confirm that `QtWidgets`, `QtGui`, `QtCore` and `QtSvg` come from Essentials in the version you pin. [K]

### 1.2 Python standard library (no install, no licence concern)

`winreg` (registry), `ctypes` (Job Objects, foreground helpers, long-path API, named mutex), `multiprocessing.connection` (named pipe for single instance), `subprocess`, `zipfile`, `json`, `logging`, `pathlib`, `tempfile`, `argparse` or manual parsing, `hashlib`, `dataclasses`.

### 1.3 External tools (detected, never bundled in MVP)

| Tool | Needed for | License | Install | Evidence |
|---|---|---|---|---|
| **LibreOffice** | DOCX → PDF, DOCX → TXT (and possibly Markdown, see §4) | MPL-2.0 (some distributors list LGPL-3.0 for the suite) | User installs it | [K]; version [V] |

Supported-version guidance: current branches are **26.8.x** (latest) and **26.2.x**, whose maintenance ends **30 November 2026**. LibreOffice 26.2 and later require **Windows 10 or newer** and added **Markdown import/export**. [V] Test against both branches in the spike and state a minimum.

### 1.3 Build and development tools (not shipped)

| Tool | Purpose | License | Cost | Evidence |
|---|---|---|---|---|
| Python **3.12** | Runtime pin (3.13 acceptable) | PSF | Free | [K] |
| **PyInstaller** (onedir) | Build Windows executable | GPL-2.0 with a bootloader exception that permits bundling in any app | Free | [K] verify |
| **Inno Setup** | Per-user installer | Free licence permitting use | Free | [K] verify |
| **pytest** | Test runner | MIT | Free | [K] |
| **pytest-qt** | Widget tests | MIT | Free | [K] |
| **ruff** | Lint and format | MIT | Free | [K] |
| **import-linter** | Enforce layer rules | BSD-2-Clause | Free | [K] |
| **pip-audit** | Known-vulnerability scan of pins | Apache-2.0 | Free | [K] |
| **mypy** (optional) | Static typing on `core` | MIT | Free | [K] |
| Git + GitHub | Version control, issues, Actions CI | n/a | Free (GitHub Pro via Student Pack) | [V] |
| Windows Sandbox / Hyper-V VM | Clean install and uninstall tests | n/a | Free (feature availability depends on Windows edition) | [K] |

---

## 2. Dependencies considered and rejected (or deferred)

| Candidate | Decision | Reason |
|---|---|---|
| **pywin32** | Rejected | Everything needed is in `winreg` and `ctypes`. Adds a large binary dependency for nothing |
| **PyMuPDF** | Rejected | AGPL-3.0: distributing it requires ConvertX to be AGPL or a paid commercial licence from Artifex [V] |
| **pdf2docx** | Deferred | Believed to depend on PyMuPDF [K]. Verify before any PDF → DOCX work |
| **python-magic / libmagic** | Rejected | Needs a native library on Windows; a small magic-byte table covers our formats |
| **ImageMagick / Wand** | Rejected | Large external dependency with a long vulnerability history; Pillow suffices. (File Converter uses ImageMagick, FFmpeg and Ghostscript [V]) |
| **docx2pdf** | Rejected | Only automates Microsoft Word; needs a licensed Word install |
| **python-docx** | Not needed | Reads and writes the file structure; cannot render to PDF |
| **QtWebEngine** | Rejected | Very large, slow to start, unnecessary for MVP |
| **QML / QtQuick** | Rejected for MVP | Heavier runtime; QWidget + QPainter is enough (TD-06) |
| **ffmpeg-python** | Rejected | A thin wrapper; direct `subprocess` is simpler and safer |
| **PyAV** | Deferred | Links FFmpeg libraries into the process; heavier than needed |
| **requests / httpx / any network library** | Forbidden | Product principle: no network code. A test enforces it |
| **Telemetry / crash-reporting SDKs** | Forbidden | Product principle: privacy |
| **Ghostscript** | Rejected | AGPL; not needed for the MVP |

---

## 3. Planned post-MVP dependencies (pre-evaluated, not approved)

| Candidate | For | License | Notes |
|---|---|---|---|
| **Pandoc** (external) | Structured text conversions (Markdown, HTML, DOCX variants) | GPL-2.0-or-later [K] | May be unnecessary if LibreOffice 26.2+ Markdown support is good enough (§4). It cannot make PDFs without a separate engine [K]. Detect-only; do not bundle without a licence plan |
| **FFmpeg** (external) | Audio and video | LGPL-2.1+ or GPL depending on build [K] | Detect-only. If bundling is ever considered, choose an LGPL build and review obligations |
| **pypdfium2** | Better PDF text extraction, page rendering (OCR groundwork) | Apache-2.0 or BSD-3-Clause [V] | Pre-approved swap for `pypdf` if text quality is poor |
| **pillow-heif** | HEIC/HEIF → JPG/PNG | BSD-3 (libheif is LGPL) [K] | High user value; check wheel size and licence notices |
| **Tesseract** (external) + wrapper | OCR for scanned PDFs | Apache-2.0 [K] | Large; a feature of its own |
| **img2pdf** | Lossless JPEG → PDF | LGPL [K] | Only if Pillow's PDF output is not good enough |
| **Markdown library** (for example `markdown-it-py`) | Markdown → HTML without Pandoc | MIT [K] | Only if LibreOffice does not cover it |
| **Nuitka** | Alternative to PyInstaller | Apache-2.0 [K] | Evaluate in the spike for start-up time and antivirus behaviour |
| **Native shell shim toolchain** (Rust `windows` crate, or C++/WinRT, or C#) | Windows 11 top-level menu | Various | Separate gated track (TD-04). Visual Studio Community covers C++/C# |

---

## 4. Open dependency question that could shrink the list

LibreOffice 26.2 introduced native Markdown import and export. [V] If real-world quality is acceptable (headings, lists, tables, links, code blocks, images), then:

- DOCX → Markdown, Markdown → DOCX and Markdown → PDF could all go through **one** external tool (LibreOffice), and
- **Pandoc can be dropped entirely**, leaving the whole product with one external dependency.

This is a Phase 0 spike item: convert five representative documents both ways and compare the output by eye and by round-trip. Do not decide from reading release notes alone.

---

## 5. Licence summary and compliance checklist

| Component | Obligation | Action |
|---|---|---|
| PySide6 (LGPL-3.0) | Allow users to replace the Qt libraries; include licence text and notices; do not statically link | Ship onedir (libraries remain separate DLLs). Include `THIRD_PARTY_NOTICES` |
| Pillow, pypdf | Include copyright and licence text | Generated notices file |
| LibreOffice, Pandoc, FFmpeg | Not linked or bundled in MVP, only run as separate programs | State requirements in the README; no redistribution duty while detect-only |
| PyInstaller bootloader | Exception permits bundling | Include notice if required |
| ConvertX itself | Your choice (TD-20) | An OSI licence such as MIT or Apache-2.0 keeps the SignPath Foundation route open |

Release gate: a script lists every installed distribution with its licence metadata (`pip-licenses` or equivalent, free) and fails the release if an unreviewed or copyleft-incompatible licence appears.

---

## 6. Version pinning and update policy

- Pin **exact** versions in `requirements.txt`, and use hash-checked installs (`pip install --require-hashes`) for release builds.
- Separate `requirements-dev.txt`.
- Run `pip-audit` before each release. Update dependencies deliberately, not automatically, with the start-up benchmark and the full test suite rerun after every PySide6 or Pillow bump.
- Record the pinned versions, Python version and build date in the release notes.
- Keep a `DEPENDENCY_LOG` section in `TECHNICAL_DECISIONS.md` for additions and removals, with the justification.

---

## 7. Free tooling inventory and what is *not* free

| Item | Cost to you | Notes |
|---|---|---|
| Python, PySide6, Pillow, pypdf, PyInstaller, Inno Setup, pytest, ruff | Free | |
| LibreOffice (user-installed) | Free | |
| GitHub Pro and Actions minutes | Free while the student pack is verified | Verification renews roughly every two years [V] |
| JetBrains tools | Free via Student Pack | Optional (for example PyCharm) [V] |
| Azure credit in the Student Pack | Free credit | **Not useful for signing**: Azure Artifact Signing rejects free and sponsored subscriptions [V] |
| GitHub Copilot Student | Sign-ups reported **paused** in 2026 | Do not depend on it [V] |
| Google Antigravity with Google AI Pro | Already available to you | Treat as the agent host; limits were not verified this session |
| Visual Studio | Free Community edition for individual use [K] | Only needed for the native shim track |
| Microsoft Store developer account (individual) | **Free** worldwide; Store-signed MSIX | Requires full MSIX packaging and certification review [V] |
| SignPath Foundation code signing | **Free** for qualifying open-source projects | Needs OSI licence and a public, maintained project; approval not guaranteed [V] |
| Azure Artifact Signing | About US$10/month | **Unavailable to you** as an individual outside the USA and Canada [V] |
| Commercial OV code-signing certificate | Paid (open-source tiers start near €49) | Outside the zero-cost budget [V] |

---

## 8. Dependency risk notes

- **PySide6 size and start-up:** the largest contributor to bundle size and cold start. Mitigate with Essentials-only, PyInstaller excludes, and lazy imports. Measure.
- **Pillow decoder vulnerabilities:** image decoders are a frequent source of security fixes. Keep Pillow current, enforce pixel limits, plan optional worker-process isolation.
- **LibreOffice version drift:** headless flags and Markdown filters change between releases. Pin a tested range, check the installed version at probe time, and show a clear message outside the range.
- **Antivirus false positives:** PyInstaller bootloaders are commonly flagged. See `RISK_REGISTER.md` (R-05).
