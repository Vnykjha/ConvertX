# ConvertX — Implementation Plan

**Status:** Proposed. **Implementation will not begin until you approve** the decisions in §9.
**Baseline:** `Project_Descritpion.md` Revision 2.
**Companion docs:** `ARCHITECTURE.md`, `TECHNICAL_DECISIONS.md` (TD-nn), `RISK_REGISTER.md` (R-nn), `DEPENDENCIES.md`.

---

## 1. Product analysis

### 1.1 What the product really is

A privacy-first, offline "right-click → convert" utility for Windows whose value is **trust**: it converts the common formats correctly, never harms your files, never touches the network, and uninstalls cleanly. The radial menu is the memorable front end. It is not the product.

### 1.2 Missing requirements (not in the spec)

| Gap | Proposed requirement |
|---|---|
| Supported Windows versions | Windows 11 and Windows 10 22H2, x64 only for MVP. Windows 10 gets the entry in the main menu, so it is a *better* experience |
| Install scope | Per-user install, no administrator rights |
| First-run and missing-tool guidance | A clear in-app message when LibreOffice is missing, with a link; images and PDF→TXT work with zero installs |
| Update strategy | Manual in MVP (no network code, so no update checks). Version shown in the status window; releases on GitHub |
| Exit codes and CLI contract | Defined in `ARCHITECTURE.md` §5.3 and TD-03; needed for testing and for the cascading mode |
| High-contrast and reduced-motion | Detect Windows high-contrast mode and force list mode; honour reduced-motion by skipping animation |
| Output location rules | Beside the original; collision-safe names; failure message if unwritable; fallback folder later |
| Localisation | English only for MVP; keep strings in one place to allow translation later |
| Network paths, UNC, OneDrive placeholders | Defined behaviour (TD-15), not an afterthought |
| Large files and limits | Explicit size, pixel and time limits with clear messages |
| Uninstall data cleanup | Uninstaller removes registry entries and optionally user data (logs, settings) |
| Name and trademark check | "ConvertX" resembles existing products. Check Microsoft Store name availability and trademarks before any public release [K] |
| Privacy statement | One paragraph in the README: no network, no telemetry, what is logged |
| Crash handling | Catch-all that shows a friendly error and writes a local log; no upload |

### 1.3 Unrealistic or expensive expectations in the spec

| Expectation | Reality | Resolution |
|---|---|---|
| "No more than two clicks" | Windows 11 hides classic verbs under "Show more options" [V] | Dropped for MVP; modern-menu track after MVP |
| "Universal file converter" | MVP covers a small, reliable set | Rename the claim: "Convert common files locally" |
| Radial menu "near the invocation cursor" | Classic verbs get no click coordinates | Cursor-at-launch approximation, clamped to the work area |
| "Launch quickly" | Python + Qt + PyInstaller start-up is unmeasured and likely the weakest point | Budget and Phase 0 measurement; fallbacks defined |
| "Never advertise perfect fidelity" | Correct | DOCX output is LibreOffice-rendered; say so |
| "Cancellation where supported" | Subprocess trees survive naive cancel | Job Objects (TD-10) |
| "Eventually" keyboard navigation | Accessibility cannot be bolted on | MVP requirement |
| Zero-cost *and* native-feel on Windows 11 | Needs identity packaging and signing; no free path fits an individual in India except Store or SignPath Foundation [V] | Decision TD-19/TD-20 |
| PDF → DOCX | Poor results; PyMuPDF licence issue | Post-MVP, clearly labelled best-effort |
| Many PDF-output conversions | Toolchain undefined | Only DOCX→PDF and image→PDF are committed |

### 1.4 Features worth adding (high value, low cost)

| Feature | Why | Cost | When |
|---|---|---|---|
| **Image → PDF** | Pillow does it natively; very common need | S | v0.1/v0.2 |
| **HEIC → JPG** (`pillow-heif`) | Phone photos; likely the single most useful conversion | M | Post-MVP, first item |
| **List popup mode** | Fast, accessible, the real safety net | S | v0.1 |
| **Drop window / CLI** | Works without Explorer integration, great for testing | S | v0.1 |
| **JPG quality default (about 90) with one setting** | Avoids bad defaults | S | v0.1 |
| **"Show in folder" and "Open file" after success** | Completes the flow | S | v0.1 |
| **"Copy details" on errors** | Better bug reports, no telemetry | S | v0.1 |
| **Dry-run installer output** | Safety and trust | S | v0.1 |
| Image resize / compress | Easy with Pillow, popular | M | Post-MVP |
| PDF merge / split | Easy with `pypdf` | M | Post-MVP |

### 1.5 Features to remove or postpone from the MVP

Audio and video, Pandoc-based conversions, PDF → DOCX, OCR, batch and multi-file, settings window, history, themes, supported-format explorer, plugin SDK, modern Windows 11 menu, advanced radial interactions (rotation, sub-rings), blur and glass effects, a resident background process, auto-update.

---

## 2. Revised MVP scope

Two releases, so something useful exists as early as possible.

### v0.1 — "Useful" (the vertical slice)

- Windows 10/11 classic context-menu entry for PNG, JPG, WEBP, via per-user registry with dry-run, install and uninstall.
- Plain **list popup** (no radial yet), status window with success/failure/Open folder.
- Conversions: PNG ↔ JPG ↔ WEBP (all six directions), plus image → PDF.
- Single-instance guard, collision-safe naming, atomic writes, originals untouched.
- Drop window and CLI.
- Start-up time measured and recorded.

### v0.2 — "MVP" (meets the Definition of Done in the spec, Part 12)

Everything in v0.1, plus:
- DOCX → PDF and DOCX → TXT (LibreOffice, detected), PDF → TXT (`pypdf`).
- Disabled-with-reason menu entries when a tool is missing.
- **Radial menu** with full keyboard support, DPI and multi-monitor handling, automatic fallback to the list.
- Per-user installer (Inno Setup), uninstaller, notices, README.
- Clean-machine install/uninstall verification.

### Explicitly not in the MVP

See §1.5. Each item has a post-MVP track (§7).

---

## 3. Changes to the specification (and why)

| # | Spec said | Plan changes it to | Reason |
|---|---|---|---|
| 1 | Phase 0 produces five docs and no code | Phase 0 produces the docs **and** a small throwaway feasibility spike, with no shipped code | Key risks (start-up, focus, Explorer behaviour) cannot be settled by reading |
| 2 | Radial prototype in Phase 2, before the pipeline | List popup and full vertical slice first; radial after | Prove the product before polishing the signature UI |
| 3 | Windows integration in Phase 4 | Registry install/uninstall built with the vertical slice | It is the riskiest integration and the slice needs it |
| 4 | `pywin32` | Removed | Not needed |
| 5 | PyMuPDF | Removed; `pypdf`, with `pypdfium2` as swap | AGPL |
| 6 | Pandoc in the MVP toolchain | Optional; possibly unnecessary | LibreOffice 26.2+ has Markdown import/export [V]; Pandoc cannot make PDFs alone [K] |
| 7 | MVP format matrix of about 12 document pairs | About 9 conversions (6 image, image→PDF, DOCX→PDF, DOCX→TXT) plus PDF→TXT | Reliability over breadth |
| 8 | `scripts/*.ps1` own registry logic | Python `windows/` module with `--install --dry-run`; scripts are thin wrappers | One tested implementation |
| 9 | `launcher.py` + `main.py` | One entry point `main.py`, launcher as a module | Clarity and start-up |
| 10 | `core` / `ui` split | Adds `app/` composition root and an `import-linter` contract | Enforce independence mechanically |
| 11 | Modern menu deferred (no detail) | Gated post-MVP track with cost and routes documented | Decision-ready |
| 12 | Branch model with `develop` | `main` plus short-lived feature branches | Solo project |

Anything in the spec not mentioned here is unchanged.

---

## 4. Milestones

Effort is in **focused working days with AI assistance**, rough (±50%). Each milestone ends with a short report, a commit tag, and a risk-register review. Nothing is "done" until its acceptance checks pass and the output has been opened and verified.

### M0 — Feasibility spike (2–3 days) — *throwaway code allowed, in `spike/`*

**Goal:** settle the assumptions that can sink the plan. Writes `SPIKE_REPORT.md`.

Tasks:
1. With your approval, register a per-user verb for `.png` under `SystemFileAssociations`, using a dry-run first; confirm where it appears on Windows 11 and whether `MultiSelectModel=Single` suppresses multi-select; test `"%1"` with spaces, Unicode, long paths.
2. Minimal PySide6 window at the cursor from that verb; measure warm and cold start from **source**, **PyInstaller onedir**, and optionally **Nuitka**, with and without trimmed Qt modules.
3. Focus test: Escape, click-outside (deactivate), foreground behaviour; `ctypes` helpers if needed.
4. DPI test on at least two scale factors (and mixed-DPI if you have two monitors).
5. One real PNG → JPG with Pillow; verify output opens.
6. LibreOffice: probe, convert DOCX → PDF and → TXT with a dedicated profile; record cold and warm times; check macro handling; check leftover `soffice.bin` after kill (Job Object).
7. LibreOffice Markdown test: five sample documents DOCX ↔ MD and MD → PDF/DOCX; judge quality.
8. `pypdf` vs `pypdfium2` text extraction on three PDFs (clean, multi-column, scanned).
9. Single-instance prototype with a stdlib named pipe.
10. Cascading submenu prototype for one extension (checks `HKCU` behaviour).

**Acceptance:** `SPIKE_REPORT.md` contains measurements and a go/no-go table answering every open item in `RISK_REGISTER.md` §3; owner signs off on the resulting architecture adjustments.

### M1 — Repository and foundations (1–2 days)

Git repo, `pyproject.toml`, pinned dependencies, `ruff`, `pytest`, `import-linter` contracts, GitHub Actions (Windows runner), logging setup, settings service, README skeleton, folder layout from `ARCHITECTURE.md` §4.

**Acceptance:** CI green; `python main.py --version` works; layering contract passes and **fails** when a deliberate violation is added (proves it works); no network imports (enforced by test).

### M2 — Core domain (2–3 days)

`models.py`, `errors.py`, `file_detector.py` (extension, signatures, placeholder and long-path flags, mismatch handling), `format_registry.py` (image specs only), `tool_probe.py`. CLI helpers `--probe` and a debug `--describe <file>` printing JSON.

**Acceptance:** unit tests for each detector rule; PNG/JPG/WEBP identified and mismatches reported; registry returns correct options per file; probe states work (simulated missing tool); invalid paths return clean errors, no tracebacks.

### M3 — Safe file layer, process layer, launcher (2–3 days)

`file_service` (naming, temp directories, atomic no-overwrite publish, stale-temp sweep), `process_service` (argument lists, no-window, Job Object, timeout, output capture), `launcher` (arg parse, normalisation, single-instance forward).

**Acceptance:** collision test (50 parallel publishes of the same name produce 50 distinct files, none overwritten); kill-tree test with a child process that spawns a grandchild; timeout test; second launch forwards and exits without importing Qt; original-hash-unchanged helper test.

### M4 — Image converters and direct mode (2 days)

`converters/images.py`, `conversion_manager.py`, CLI `--convert <file> --to <fmt>`, image → PDF.

**Acceptance:** all six image pairs plus image→PDF produce valid files that reopen in Pillow and an external viewer; transparent PNG → JPG uses the configured background; EXIF orientation respected; animated GIF handled with a notice; decompression-bomb fixture rejected cleanly; corrupt file reported; no partial outputs; originals byte-identical.

### M5 — Vertical slice UI and Explorer integration (3–5 days) → **release v0.1-alpha**

`list_menu`, `status_window`, `drop_window`, `app/` controllers and workers, `windows/` install plan, dry-run, real install (with your approval), uninstall, `docs/REGISTRY_CHANGES.md` generated from code.

**Acceptance (manual on Windows 11, recorded in `tests/manual/CHECKLIST.md`):**
- Right-click a PNG → Show more options → ConvertX → list → JPG; output appears beside the original; status shows success and "Show in folder".
- Works for names with spaces, Unicode, and long paths.
- Rapid double invocation shows one window.
- Uninstall restores the registry (before/after export diff is clean); unrelated keys untouched.
- Start-up timings recorded against the budget.
- UI stays responsive during a large image conversion.

### M6 — Documents and PDF (3–5 days)

`converters/documents.py` (LibreOffice), `converters/pdf.py` (`pypdf`), disabled-with-reason states, error messages, first-run guidance.

**Acceptance:** DOCX → PDF and TXT open correctly in a PDF viewer and Notepad; password-protected, corrupt and empty files give correct messages; LibreOffice missing → entries shown disabled with a reason; timeout and cancel leave **no** orphan `soffice` processes and no temp files; user's own LibreOffice session is unaffected; PDF → TXT works on text PDFs and says "may be scanned" on image PDFs; encrypted PDF reported.

### M7 — Radial menu (5–8 days) → part of **v0.2**

`radial_geometry.py` (pure, exhaustively tested), `radial_menu.py`, animation, hover and click, **keyboard (arrows, numbers, Enter, Esc)**, disabled segments, positioning and clamping, DPI handling, automatic fallback to the list, setting to choose mode.

**Acceptance:** geometry tests cover N = 1 to 8 and boundaries; manual matrix: 100/125/150/200% scaling, two monitors, mixed DPI, cursor at screen edges and corners; Escape and click-outside work from an Explorer launch; menu is interactive from the first frame; open animation about 150 ms; start-up still within the budget or the deviation is documented; high-contrast mode shows the list.

### M8 — Cascading submenu mode (2–3 days; **may move earlier** if the spike favours it)

Plan generator for per-extension submenus, direct-convert execution, tool re-probe with clear errors.

**Acceptance:** right-click → ConvertX ▸ PDF runs without showing any menu process; uninstall removes it; no interference with the launch verb.

### M9 — Hardening and MVP verification (3–4 days)

Run the full test list from the spec (Part 10) and the manual checklist; fix defects; review risks; finalise documentation.

**Acceptance:** every item in the spec's Definition of Done (Part 12) is verified and recorded; no open critical risks without a recorded decision.

### M10 — Packaging and release (3–4 days) → **v0.2**

PyInstaller onedir, icon, Inno Setup per-user installer calling `--install` and `--uninstall`, `THIRD_PARTY_NOTICES`, licence scan, CI build in public, GitHub Release, README with SmartScreen guidance.

**Acceptance:** clean Windows 11 VM: install, use, uninstall, registry diff clean; Defender scan result recorded; `pip-audit` clean or exceptions noted; signing decision (TD-19) recorded.

---

## 5. Fastest path to a working MVP

**M0 → M1 → M2 → M3 → M4 → M5 = v0.1 in about 12–18 focused days.** At that point you have a genuinely useful tool with no radial menu and no external dependencies. M6 adds documents; M7 adds the signature UI. If time runs out, v0.1 is still a shippable product, which is the point of the ordering.

---

## 6. Realistic roadmap

Assumptions: AI-assisted development; you work part-time alongside studies; estimates are rough.

| Pace | v0.1 (M0–M5) | v0.2 (M6–M10) |
|---|---|---|
| About 10 hours per week | 7–10 weeks | a further 10–14 weeks |
| About 20 hours per week | 4–6 weeks | a further 5–8 weeks |

Calendar slips most likely in M0 (surprises), M5 (Explorer and focus behaviour), M7 (DPI and polish) and anything involving LibreOffice timing. Re-estimate after M0 with real data.

---

## 7. Post-MVP tracks (priority order, each a separate approval)

1. **HEIC → JPG/PNG** and more image formats (BMP, TIFF, GIF first frame).
2. **Markdown/HTML/TXT conversions** (LibreOffice or Pandoc, per spike), then PDF-output route.
3. **Image resize/compress; PDF merge/split.**
4. **Audio (FFmpeg).**
5. **Batch and multi-file** via the single-instance queue.
6. **Settings window, history, themes.**
7. **Video (FFmpeg).**
8. **Windows 11 top-level menu** (TD-04): native shim, sparse package, signing route. Gate: MVP stable and you have chosen licence and distribution route.
9. **OCR and PDF → DOCX** (decide on AGPL stance first).
10. **Plugin SDK.**

---

## 8. Working with the AI agent

- **One milestone per task.** Start every task with: "Read `ARCHITECTURE.md` and this milestone. Write a plan. Do not write code until I approve."
- **Rules file.** Put spec Part 13 plus these three lines in the agent's project-rules file: *"Respect the layer boundaries. Never run registry writes yourself. Never claim a result without a passing test or a recorded manual check."* (Check the filename your agent tool expects; this was not verified.)
- **Definition of ready:** plan approved, acceptance criteria copied into the task.
- **Definition of done:** tests pass, `ruff` and `import-linter` pass, docs updated, manual checklist items ticked by you, commit tagged.
- **You review diffs of:** anything touching `windows/`, `process_service`, `file_service`, and any new dependency.
- **The agent cannot test** Explorer, focus, DPI or visual quality. You do, on real Windows 11, and record results. A clean VM or Windows Sandbox is used for install and uninstall tests.
- **Spike code is thrown away.** Production code is written fresh from M1 onward.

---

## 9. Decisions I need from you before any implementation

| # | Decision | My recommendation |
|---|---|---|
| D1 | Approve the two-release MVP (v0.1 list-based, v0.2 with radial and documents) | Approve |
| D2 | Approve spike-first ordering (M0 before architecture is frozen) | Approve |
| D3 | Accept "Show more options" on Windows 11 for the MVP | Accept |
| D4 | **Licence of ConvertX:** open source (MIT or Apache-2.0) or closed | Open source: it unlocks free SignPath Foundation signing and avoids later relicensing |
| D5 | LibreOffice as a **detected prerequisite** (not bundled) | Approve |
| D6 | Stack: Python 3.12, PySide6 (Essentials if sufficient), Pillow, `pypdf`; no `pywin32`, no PyMuPDF | Approve |
| D7 | Target Windows 11 **and** Windows 10 22H2, x64 only | Approve |
| D8 | Check the name "ConvertX" for Store and trademark conflicts before public release | Do this during M1 |
| D9 | Permission to run the M0 spike, including a **real registry write** for a `.png` verb on your machine (I will show the dry run first) | Approve when ready |

**I will not start M0 or write any application code until you reply with your decisions.**
