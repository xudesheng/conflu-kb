_Digested 2026-08-23 from iot_stream@67bc265 (v1.1.0); full source commit 67bc265de96d46dc1af3e3b2184631536d95c78a._

# ThingWorx IoT Stream Practice — KB digest

A *digest* of the "Thingworx IoT Stream Practice" mdBook for use as a Conflu
external knowledge base (`twx.kb.*`). Advisory reading for a coding agent;
anything version-sensitive should be checked against the live ThingWorx
system.

## What is here

- `src/` — the book's ten chapters at their original paths (start at
  `src/SUMMARY.md`), with the figures they reference, and the book's
  license (`src/license.md`).

The source repository's `project/` directory (sample projects, dev compose
files, test scripts) is not included: the knowledge lives in the chapters;
the samples are illustrations and sometimes range beyond the book's topic.
Chapter text that mentions `project/...` paths refers to the source
repository.

## Versions

**ThingWorx IoT Streams** is a platform capability **introduced in ThingWorx
10.0** (Durable Queues likewise); publishing custom payloads by service call
(`WriteJSONToQueue`) starts in **10.0.1**. The chapters state what each
mechanism needs. On an older system this digest still explains what an
upgrade provides, which is why `kb.toml` declares no minimum version.
External systems (Kafka, Azure Event Hub, KEDA) are covered conceptually;
their own versions are whatever your environment runs.
