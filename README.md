# BaudBound Documentation

This repository contains the source for the public BaudBound documentation at
[wiki.baudbound.app](https://wiki.baudbound.app).

Wiki pages live in `wiki/`. The standalone publisher in `publisher/` validates
page metadata, links, assets, navigation, and required coverage before it
reconciles managed pages with Wiki.js.

## Validate locally

```powershell
pnpm --dir publisher install --frozen-lockfile
pnpm --dir publisher test
pnpm --dir publisher validate
```

Publishing is handled by the `Wiki Documentation` GitHub Actions workflow.
Production publishing requires the `wiki-production` environment and its
`WIKI_URL` and `WIKI_API_TOKEN` secrets.

## Related repositories

BaudBound is maintained as focused repositories in the
[BaudBound organization](https://github.com/BaudBound). The runner is in
[BaudBound/baudbound](https://github.com/BaudBound/baudbound), the visual editor
is in [BaudBound/editor](https://github.com/BaudBound/editor), and shared
machine-readable contracts are in
[BaudBound/contracts](https://github.com/BaudBound/contracts). Shared local
development and release helpers are in
[BaudBound/tooling](https://github.com/BaudBound/tooling).
