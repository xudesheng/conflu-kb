_Digested 2026-08-23 from cachething@78c07ea (v1.1.0); full source commit 78c07ea9f7dca6491dee0a8c9c281d21afe26473._

# Cache Thing guidance — KB digest

This directory is a *digest* of the "Cachething guidance" mdBook for use as
an external knowledge base by Conflu (`twx.kb.*`). It is advisory reading
for a coding agent; anything version-sensitive should be checked against the
live ThingWorx system.

## What is here

- `src/` — the book's Markdown chapters, unchanged and at their original
  paths (start at `src/SUMMARY.md`), with the images they reference.
- `project/samples/historicaldatachart/entities/` — the two ThingWorx entity
  exports that `src/practical_examples.md` links to, at their original
  relative paths so the links resolve.
- `src/license.md` — the PTC Proprietary Freeware License the book is
  published under; it applies to this digest as well.

Nothing else from the source repository is included (test scripts, result
archives, environment files, and other samples are deliberately left out).

## Versions

Cache Thing exists from **ThingWorx 10.0.0**; some patterns the book
describes need **10.0.1** (see `src/introduction.md` "Version Availability"
and `src/basic_concepts.md`). On a ThingWorx 9.x system this digest is still
useful as the reason to upgrade, which is why `kb.toml` does not declare a
minimum version: the chapters say which version each pattern needs.
