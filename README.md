# conflu-kb

Knowledge-base digests for [Conflu](https://github.com/xudesheng/conflu)'s
external KB. One directory per family; each is a complete external KB
directory (`kb.toml` + Markdown) as defined by Conflu's external KB contract,
produced under the convention in Conflu's `docs/core/kb-digest.md` by the
recipes kept in the private `conflu-digest` repository.

## Use

Install an approved family into the current workspace by default:

```bash
conflu digest install cachething
conflu digest install cachething --global
```

For private or offline use, a family is still just a directory. Copy it to
`<repo>/.conflu/kb/` for one workspace or `~/.conflu/kb/` for global use.
Conflu reads both roots fresh on every `twx.kb.*` call; no restart.

See [CONTRIBUTING.md](CONTRIBUTING.md) to propose a topic for official
installation.

## Families

| Directory | What | Source |
|---|---|---|
| `cachething/` | ThingWorx 10 Cache Thing guidance — concepts, patterns, worked examples, observability | the "Cachething guidance" mdBook (provenance line in its README) |
| `parler/` | Parler Kit — configure, deploy, diagnose and test the Parler AI agent extension with a coding agent; guide pages, three Conflu-facing skills, Parler documents and contracts, the golden configuration example, the workshop course book and exercises | parler, ParlerGuidance, parler-workshop repositories (provenance line in its README) |
| `iot_stream/` | ThingWorx IoT Streams (10.0+) — durable queues, Kafka / Event Hub routing, WriteJSONToQueue, consumer groups and partitions | the "Thingworx IoT Stream Practice" mdBook (provenance line in its README) |
| `twx-k8s/` | Deploying ThingWorx on Azure AKS — Helm, nginx and Istio ingress, PostgreSQL Flexible Server, HA, upgrades | the "Thingworx Containerization Guide" mdBook (provenance line in its README) |

Each family's `README.md` states what was included, what was left out, and
the versions it describes. Licenses travel with their families (`cachething`
carries the PTC Proprietary Freeware License in `src/license.md`; `parler`
carries Parler's `LICENSE` under `parler/parler/LICENSE`; `iot_stream` and `twx-k8s` carry
theirs in `src/license.md`); this
repository's own `LICENSE` covers only the repository scaffolding.
