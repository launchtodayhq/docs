# Launch Docs

These docs are part of the Launch product. The main goal is **trust + learning**: every page should match the codebase and teach production patterns clearly.

## Local preview

1. Install Mintlify CLI:

```bash
npm i -g mint
```

2. Run the docs locally from `Launch/docs/`:

```bash
cd Launch/docs
mint dev
```

## Authoring rules (important)

- **Docs must match code**: if you change code paths, update the docs in the same PR.
- **Avoid fake file paths**: reference real paths like `apps/api/src/app.ts` and `apps/mobile/app/_layout.tsx`.
- **Explain removal**: for every major feature, include a “disable vs delete” note and a removal checklist.
- **Be explicit about setup**: call out required env vars and the “enabled but misconfigured” failure mode.

## Updating navigation

Navigation is controlled by `docs.json`. If you add a new page, add it to the appropriate group in `docs.json` so it shows up in the sidebar.

