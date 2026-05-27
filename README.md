# PapaParse + Vite v8 Worker Bug Reproduction

Minimal reproduction repository for a bug where PapaParse returns malformed results when using `worker: true` in Vite v8 production builds.

## Observed behavior

In production builds:

- `data` contains only `null` values
- actual row data appears inside `meta.fields`

The issue:
- only happens with `worker: true`
- only happens in production builds
- does not happen in development mode
- did not happen in Vite v7

## Run the reproduction

```bash
npm install
npm run build
python3 -m http.server 3000