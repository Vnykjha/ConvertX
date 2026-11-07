# ConvertX — Risk Register

**Status:** Proposed. Review at the end of every milestone and update the *Status* column.
**Scoring:** Likelihood (L) and Impact (I) from 1 (low) to 5 (high). Score = L × I. **≥ 15 = critical**, 8–14 = high, 4–7 = medium, ≤ 3 = low.
**Evidence tags:** [V] verified this session, [K] background knowledge, [S] unknown until the spike.

---

## 1. Top risks at a glance

| Rank | ID | Risk | Score |
|---|---|---|---|
| 1 | R-01 | Windows 11 hides the classic verb, so the headline UX is weaker than advertised | 20 |
| 2 | R-02 | Python + PySide6 start-up too slow for a context-menu tool | 16 |
| 3 | R-03 | Code signing unavailable at zero cost, so SmartScreen and antivirus friction | 16 |
| 4 | R-04 | Frameless/translucent window fails to get focus reliably when launched from Explorer | 15 |
| 5 | R-06 | LibreOffice cold start, hangs, orphan processes and version drift | 15 |
| 6 | R-09 | Network access cannot be fully blocked inside LibreOffice | 12 |
| 7 | R-12 | Scope creep: radial polish and format count outrun the working pipeline | 15 |

---

## 2. Full register

### A. Windows integration

| ID | Risk | L | I | Score | Mitigation | Early signal / trigger | Phase | Status |
|---|---|---|---|---|---|---|---|---|
| **R-01** | Classic registry verbs appear only under "Show more options" on Windows 11. Top-level placement needs `IExplorerCommand` plus package identity. [V] | 5 | 4 | **20** | Accept it for MVP; drop the "two clicks" claim; build cascading and list fallbacks; make the modern menu a gated track (TD-04) | Spike confirms placement | 0 | Open, accepted |
| R-34 | Modern-menu route needs signing, Developer Mode or elevation; reports conflict. [V]/[S] | 4 | 3 | 12 | Treat as a spike on your own machine; decide Store vs SignPath after MVP | Registration fails without Dev Mode | Post-MVP | Open |
| R-35 | Native shell DLL running inside Explorer can destabilise it | 2 | 5 | 10 | Keep it tiny: only launch the exe, no heavy work, no Python in-process; follow Microsoft's guidance to avoid expensive work on the Explorer UI path [V] | Explorer crashes in testing | Post-MVP | Open |
| R-13 | Registry changes damage the user's setup (wrong keys, leftovers, wrong default app) | 2 | 5 | 10 | Per-user `SystemFileAssociations` only; manifest-driven uninstall; dry-run first; injectable test root; agent needs your approval before real writes | Uninstall leaves keys behind | 4 | Open |
| R-14 | Multi-select launches many processes | 3 | 3 | 9 | `MultiSelectModel=Single` [K]; single-instance guard; verify in spike | Several menus appear | 0 / 2 | Open |
| R-15 | Menu positioning is wrong for keyboard-invoked menus; no click coordinates from Explorer | 4 | 2 | 8 | Cursor-at-launch approximation; clamp to work area; centre on the cursor's screen; document the limit | Menu appears far from the file | 5 | Open, accepted |
| R-16 | Path argument arrives as a short name, with odd characters, or very long | 3 | 3 | 9 | Quote `"%1"`; normalise with `GetLongPathNameW`; test Unicode, spaces, 260+ characters, UNC | Wrong file opened | 0 / 2 | Open |

### B. Performance and UI

| ID | Risk | L | I | Score | Mitigation | Early signal / trigger | Phase | Status |
|---|---|---|---|---|---|---|---|---|
| **R-02** | PySide6 + PyInstaller start-up exceeds the budget (target about 0.5 s warm, 1.5 s cold; unmeasured). [S] | 4 | 4 | **16** | Measure in Phase 0; onedir not onefile; Essentials-only; excludes; lazy imports; try Nuitka; fallback = cascading submenu as primary; last resort = native launcher or resident helper | Spike timings over budget | 0 | Open |
| **R-04** | Focus and dismissal fail: Windows foreground-lock rules block a window launched by another process, breaking Escape and click-outside | 3 | 5 | **15** | `activateWindow`, then `ctypes` foreground helpers; dismiss on deactivate; if unreliable, default to list mode | Escape does nothing after launch | 0 | Open |
| R-36 | Mixed-DPI and multi-monitor glitches (wrong size, wrong monitor, blurry text) | 4 | 3 | 12 | Set screen before show; logical units; SVG icons; test 100/125/150/200% and mixed-DPI pairs | Visual defects on second monitor | 5 | Open |
| R-17 | Translucent window rendering problems (black corners, flicker, poor perf on some GPUs or RDP) | 2 | 3 | 6 | Draw own shadow; avoid blur and graphics effects; test on a VM and remote desktop; software-render fallback to list mode | Black box around menu | 5 | Open |
| R-18 | Radial menu is slower and less usable than a plain list, so users dislike it | 3 | 3 | 9 | Make it optional ("rich mode"); list mode default if spike says so; user-test with a few people | Feedback after v0.1 | 5 | Open |
| R-19 | Accessibility gap: custom-painted widget invisible to screen readers | 4 | 3 | 12 | Keyboard support in MVP; list mode as the guaranteed accessible path; custom accessible interface where feasible | Narrator reads nothing | 5 | Open |

### C. Conversion engine

| ID | Risk | L | I | Score | Mitigation | Early signal / trigger | Phase | Status |
|---|---|---|---|---|---|---|---|---|
| **R-06** | LibreOffice: slow cold start, hangs, profile locks with the user's own session, orphan `soffice.bin`, version drift | 4 | 4 | **16** | Dedicated profile; Job Object with kill-on-close; timeout; serialise jobs; probe version; test 26.2 and 26.8 | Timeouts; stray processes | 4 | Open |
| R-07 | DOCX fidelity complaints (fonts, layout) | 4 | 2 | 8 | Say "uses LibreOffice; layout may differ" in the status message; do not promise fidelity | User reports | 4 | Open, accepted |
| R-08 | PDF → TXT quality with `pypdf` (ordering, ligatures, scanned files) | 3 | 3 | 9 | Detect empty text and say it may be scanned; swap to `pypdfium2` behind the same interface if poor | Garbled output in tests | 4 | Open |
| **R-09** | LibreOffice (or HTML rendering later) may fetch external linked content; network cannot be fully blocked without a firewall rule or sandbox | 3 | 4 | 12 | Zip pre-scan for external relationships; disable link updates; pre-process HTML to strip remote references; document residual risk honestly; do not claim "never touches the network" for document conversions | Network activity in test with a link-bearing DOCX | 4 | Open |
| R-10 | Decompression bombs and malformed images crash or exhaust memory | 3 | 4 | 12 | `MAX_IMAGE_PIXELS` cap, size cap, catch `DecompressionBombError`, short-lived process, optional worker isolation | Memory spike in bomb test | 4 | Open |
| R-11 | Macros or active content in documents executed | 1 | 5 | 5 | Strictest macro level in the owned profile; never use Word automation; test with a macro-bearing file | Macro runs in test | 4 | Open |
| R-20 | Pandoc cannot create PDFs, and LibreOffice Markdown quality is unknown, so PDF output rows stay unbuilt | 4 | 2 | 8 | Do not show unbuilt rows; decide the PDF route by spike (TD-13) | Spike results | 0 | Open |
| R-21 | OneDrive online-only placeholders trigger a surprise download or fail | 3 | 3 | 9 | Detect recall attributes; refuse with a clear message; test with a real OneDrive folder | Slow or failing reads | 2 | Open |
| R-22 | Output folder read-only, full, on a network share, or path too long | 3 | 2 | 6 | Pre-check writability; atomic publish; clear error; later fallback folder setting | Permission errors | 3 | Open |
| R-23 | Partial or corrupt output left behind after failure or cancel | 2 | 4 | 8 | Temp directory plus atomic rename; cleanup in `finally`; stale-temp sweep at start-up | Stray files after kill | 3 | Open |
| R-24 | Original file modified or overwritten | 1 | 5 | 5 | Open inputs read-only; never write beside under the same name; `os.rename` no-overwrite publish; tests assert hash unchanged | Hash mismatch in test | 3 | Open |

### D. Distribution and supply chain

| ID | Risk | L | I | Score | Mitigation | Early signal / trigger | Phase | Status |
|---|---|---|---|---|---|---|---|---|
| **R-03** | No zero-cost code signing: Azure Artifact Signing is unavailable to individuals in India and needs a paid subscription; unsigned binaries meet SmartScreen warnings and some organisations block them. [V] | 5 | 3 | **15** | MVP: unsigned with honest instructions; public release: SignPath Foundation (needs OSI licence, public repo) or Microsoft Store (free signing, needs MSIX) | Defender or SmartScreen blocks testers | 9 | Open, decision needed (TD-19/20) |
| R-05 | Antivirus false positives on PyInstaller output | 4 | 3 | 12 | Build in public CI; reproducible builds; submit false-positive reports to Microsoft; evaluate Nuitka; sign when possible | Defender quarantines build | 9 | Open |
| R-25 | SignPath Foundation or Store application rejected or slow | 3 | 2 | 6 | Start with a maintained public repo and tagged releases; keep unsigned path viable | Rejection email | Post-MVP | Open |
| R-26 | Licence contamination (AGPL via PyMuPDF or `pdf2docx`, GPL via bundled tools) | 2 | 4 | 8 | AGPL libraries excluded; detect-only external tools; licence check script as a release gate; decide TD-20 early | Unreviewed licence appears | 1 onward | Open |
| R-27 | Dependency vulnerabilities (Pillow decoders, others) | 3 | 3 | 9 | Pinned versions; `pip-audit` per release; prompt updates with full test run | Advisory published | Ongoing | Open |
| R-28 | Free-tier or perk changes (Student Pack lapse, Copilot paused, tool limits) | 3 | 2 | 6 | Nothing in the product depends on a perk; GitHub Free is sufficient for hosting and basic CI | Pack re-verification due | Ongoing | Open |

### E. Process and project

| ID | Risk | L | I | Score | Mitigation | Early signal / trigger | Phase | Status |
|---|---|---|---|---|---|---|---|---|
| **R-12** | Scope creep: radial polish, extra formats, settings screens before the pipeline is proven | 5 | 3 | **15** | Risk-ordered plan; vertical slice first; "release gates" per milestone; written post-MVP list | Radial work starts before vertical slice passes | All | Open |
| R-29 | AI agent errors: invented capabilities, silent architecture drift, untested claims, risky system changes | 4 | 3 | 12 | Spec rules; one milestone per task; plan-then-approve; tests required; you run registry scripts; review diffs; keep Explorer/DPI checks manual | Failing tests marked done | All | Open |
| R-30 | The agent cannot see or test Explorer integration, focus or DPI, so defects ship unnoticed | 4 | 3 | 12 | Manual checklist (`tests/manual/CHECKLIST.md`) per release; clean-VM install tests | Defects reported by testers | 4–7 | Open |
| R-31 | Competitor already solves the core job (File Converter, GPL-3.0, supports many formats) [V] | 3 | 2 | 6 | Record differentiators in `TECHNICAL_DECISIONS.md`: privacy stance, safety rules, radial UX, clean uninstall | None | 1 | Open, accepted |
| R-32 | Single maintainer, limited time (study commitments) | 4 | 3 | 12 | Small milestones that each ship something; do not start the modern-menu track until the MVP is stable | Milestones slip two weeks | All | Open |
| R-33 | The name "ConvertX" clashes with existing products or trademarks, blocking Store name reservation or inviting disputes [K] | 3 | 3 | 9 | Check Microsoft Store name availability and trademark databases during M1; keep the name easy to change (single constant, no name in registry keys beyond one ID) | Name unavailable in Partner Center | 1 | Open |

---

## 3. Items that kill or reshape the plan

| If the spike shows… | Then… |
|---|---|
| Cold start is far over budget even for the plain list popup | Make the cascading submenu the primary integration; keep Python for the conversion step only; consider a native launcher |
| Focus and Escape are unreliable for the frameless window | List mode becomes default; radial becomes opt-in |
| `MultiSelectModel=Single` does not suppress multi-select | Rely on the single-instance guard and queue paths into one window |
| LibreOffice 26.x Markdown quality is good | Drop Pandoc from the dependency list; reconsider PDF routes |
| LibreOffice cold start is several seconds | Add an honest "starting converter" status; consider keeping a warm LibreOffice process during batch only |
| You cannot (or will not) open-source the project | Remove SignPath Foundation from the plan; evaluate Store distribution or accept unsigned |

---

## 4. Review cadence

- Review this register at the end of each milestone (see `IMPLEMENTATION_PLAN.md`).
- Move closed risks to a "Retired" section with the evidence.
- Add new risks the moment the agent or testing reveals them.
