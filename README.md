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

The runner is developed in [BaudBound/BaudBound](https://github.com/BaudBound/BaudBound).
The visual editor is developed in [BaudBound/editor](https://github.com/BaudBound/editor).
Shared machine-readable contracts are published from
[BaudBound/contracts](https://github.com/BaudBound/contracts).

