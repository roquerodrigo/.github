# CLAUDE.md

This repository holds the default community health files for every repository owned by
`roquerodrigo`. It has no code, build, tests or CI.

## Cross-repo impact

- Any file added here applies to every `roquerodrigo` repository that does not define its own
  copy. Treat each change as a change to all of them.
- Fallback is per file and all-or-nothing: if a repository has its own copy of a file (for
  example its own `.github/FUNDING.yml`), that copy replaces the default entirely; nothing is merged.
- `.github/FUNDING.yml` here is what shows the **Sponsor** button on repositories without their
  own. Removing or renaming it removes the button from all of them.

## Gotchas

- Only files GitHub recognises as community health files fall back (e.g. `FUNDING.yml`,
  `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, `SUPPORT.md`, issue and pull request
  templates). Workflows in `.github/workflows/` and `LICENSE` files are never inherited by other
  repositories.
- The fallback only works while this repository stays public.
- Keep `README.md` in sync when adding or removing a default file; it lists what this repository
  currently provides.
