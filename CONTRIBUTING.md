# Contributing a KB topic

This repository distributes small, useful Conflu external-KB topics. A topic
may be condensed or incomplete; it does not need to become a version-exact
knowledge-management record.

Before opening a pull request:

1. Add one top-level topic directory containing `kb.toml`, `README.md`, and
   useful Markdown pages.
2. Run `conflu digest validate <topic-directory>` and fix structural errors.
3. Make the manifest summary say what the topic covers and which work it helps
   with. Prefer an organization-prefixed id for organization-specific advice.
4. Add short human-readable source attribution and a date to the topic README.
5. Confirm that you are entitled to publish the included material and that the
   topic contains no credentials, private data, or unintended files.
6. Open a pull request describing who should use the topic and one task it
   improves.

Review checks structural validity, useful scope, attribution or licensing,
publication rights, secrets, and unintended id collisions. Approval means the
topic is suitable for distribution through `conflu digest install`; it does
not certify that every statement is complete, current, or correct.
