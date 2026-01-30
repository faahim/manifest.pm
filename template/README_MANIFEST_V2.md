# Manifest.pm v2 (Lean)

Canonical state lives in `tasks/MANIFEST.json`.

Generated artifacts:
- `tasks/INDEX.json` (generated, compatibility)
- `tasks/BOARD.md` (generated, human view)

Tools:
- `node tools/manifest/validate.mjs`
- `node tools/manifest/render.mjs`

Rule of thumb:
- Agents edit MANIFEST.json + ACTIVE.json (+ task spec file if it exists)
- Then run validate + render before final commit
