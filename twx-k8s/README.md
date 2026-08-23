_Digested 2026-08-23 from twx-k8s@12f64a0 (v0.1.3-49-g12f64a0); full source commit 12f64a0d0720f400813edb7607733bd34d6b9e46._

# ThingWorx Containerization Guide — KB digest

A *digest* of the "Thingworx Containerization Guide" mdBook (deploying
ThingWorx on AKS with Helm, nginx or Istio ingress, PostgreSQL Flexible
Server, monitoring, HA, and upgrades) for use as a Conflu external knowledge
base (`twx.kb.*`). Advisory reading for a coding agent.

## What is here

- `src/` — the book's chapters (both the nginx and the Istio ingress paths)
  at their original paths, starting at `src/SUMMARY.md`, plus
  `src/book-index.md`, the architecture figure `src/architect.svg`, the
  small diagrams under `src/_images/`, and the book's license
  (`src/license.md`).

Redaction note: the source's chapter 04 embedded a personal GitHub token in
its `helm repo add` examples; the token has been **revoked** and is replaced
here with `<YOUR_GITHUB_TOKEN>`. The credentials in that command are in fact
unnecessary — the chart site is public — so the command also works with the
`--username/--password` lines simply removed.

Known source inconsistency, inherited: the table of contents
(`src/SUMMARY.md`) has an "Istio Gateway path" section listing four
`…-istio.md` chapter files that do not exist in the source repository
(mdBook creates empty stubs at build time, which hides this). The Istio
material actually lives in `src/08-ingress-overall.md`,
`src/08-istio-dns-zone-and-public-ip.md`, and
`src/08-upgrade-nginx-to-istio.md`.

Not included: the chapters' step-by-step screenshots (`src/docs/`, ~19 MB of
PNG) — image links into `src/docs/` will not resolve; the text carries the
procedure. Also not included: the source repository's `project/` directory
(cluster manifests, docker files, test trees) — the knowledge lives in the
chapters; the samples are illustrations and sometimes range beyond the
book's topic. Chapter text that mentions `project/...` paths refers to the
source repository.

## Versions

This is deployment knowledge; it has no single ThingWorx floor and `kb.toml`
declares none. The procedures were written and exercised against ThingWorx
9.x–10.x releases on AKS with the tool versions each chapter names
(Kubernetes, Helm charts, cert-manager, Istio). Treat every version shown in
a command as an example to re-check against your environment, not a pin.
