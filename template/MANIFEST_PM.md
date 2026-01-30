# Manifest.pm v2 Protocol

This repo uses **Manifest.pm v2**.

Canonical state:
- `tasks/MANIFEST.json`

Generated views (do not edit by hand):
- `tasks/INDEX.json`
- `tasks/BOARD.md`

Claims:
- `execution/ACTIVE.json`

Boot routine:
```bash
git pull
node tools/manifest/validate.mjs
node tools/manifest/render.mjs
```

Read:
- `CLAUDE.md`
