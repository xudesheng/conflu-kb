# conflu-kb

Knowledge-base digests for [Conflu](https://github.com/xudesheng/conflu)'s
external KB. One directory per family; each is a complete external KB
directory (`kb.toml` + Markdown) as defined by Conflu's external KB contract,
produced under the convention in Conflu's `docs/core/kb-digest.md`.

## Use

Copy a family directory into one of Conflu's KB roots — nothing else:

```bash
cp -R cachething ~/.conflu/kb/            # global: every repository on this machine
cp -R cachething <repo>/.conflu/kb/       # workspace: one Conflu repository (wins over global)
```

Conflu reads the roots fresh on every `twx.kb.*` call; no restart.

## Families

| Directory | What | Source |
|---|---|---|
| `cachething/` | ThingWorx 10 Cache Thing guidance — concepts, patterns, worked examples, observability | the "Cachething guidance" mdBook (provenance line in its README) |

Each family's `README.md` states what was included, what was left out, and
the versions it describes. Licenses travel with their families (`cachething`
carries the PTC Proprietary Freeware License in `src/license.md`); this
repository's own `LICENSE` covers only the repository scaffolding.
