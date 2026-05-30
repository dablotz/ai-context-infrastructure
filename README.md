# AI Context Infrastructure

High-level design for a purpose-built context infrastructure for AI-assisted software development at team scale. Specifies a trust-tiered belief graph with provenance, lifecycle, and human-in-the-loop promotion.

This repository contains a single substantive artifact — `context-infrastructure-hld.md` — and the supporting materials around it. There is no implementation in this repository. The document specifies the architecture; reference implementations may follow in separate repositories.

## What this is

A high-level design document describing how teams using AI-assisted development tools (Claude Code, Cursor, Copilot, and similar) can build, share, and govern accumulated organizational knowledge across sessions and contributors. The design treats organizational AI context as a curated knowledge graph with the engineering rigor that production databases receive — schema, indexes, consistency model, access control, audit trail, retention policy, migration path.

The document is structured in eleven sections covering the five architectural layers (context snapshot, belief graph, merge strategy, provenance, promotion interface), a system-level cost model, security posture, integration points, and explicitly enumerated open questions requiring empirical calibration.

## How to engage

If you have feedback on the architecture, [file an issue](../../issues/new/choose). The repository is set up to receive architectural critique and design questions specifically. See [CONTRIBUTING.md](CONTRIBUTING.md) for what kind of feedback is most useful.

If you want to discuss the broader argument for context infrastructure as a distinct engineering discipline, the two LinkedIn articles below are the public version of that argument:

- [The Case for Purpose-Built Context Infrastructure in AI-Assisted Development](https://www.linkedin.com/pulse/git-built-case-purpose-built-context-infrastructure-greg-blotzer-mv55e)
- [The Asset You Probably Haven't Classified](https://www.linkedin.com/pulse/asset-you-probably-havent-classified-greg-blotzer-o9m4e/)

## Author and license

Authored by Greg Blotzer. Licensed under [CC-BY-SA 4.0](LICENSE). You may share, adapt, and build on this work — including for commercial purposes — provided you give appropriate credit and license your derivative work under the same terms.

## Status

Draft v0.1, published May 2026. See [CHANGELOG.md](CHANGELOG.md) for revision history.