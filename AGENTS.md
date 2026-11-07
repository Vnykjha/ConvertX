# Rules for the coding agent

Read docs/ before doing anything. The plan is in docs/IMPLEMENTATION_PLAN.md,
the design in docs/ARCHITECTURE.md, decisions in docs/TECHNICAL_DECISIONS.md and
docs/DECISIONS_APPROVED.md.

1. Work on ONE milestone at a time. Write a plan first and wait for my approval before writing code.
2. Never implement large parts of the app in one go. Small, testable changes.
3. Respect the layer boundaries in docs/ARCHITECTURE.md section 3.
4. Never run registry writes yourself. Generate a dry-run and wait for me.
5. Never install a dependency that is not in docs/DEPENDENCIES.md without asking.
6. Never claim something works without a passing test or a manual check I confirmed.
7. No network code, no telemetry, no paid services, no cloud APIs.
8. Never overwrite or modify a user's original file.
9. Use subprocess argument lists, never shell strings.
10. Report incomplete work honestly. Ask when unsure; do not guess about Windows APIs.