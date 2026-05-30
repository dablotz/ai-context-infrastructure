# Contributing

This repository contains a design document. The engagement model is different from a code repository, and this file explains what kind of contribution is welcome.

## What this repository is for

The repository exists to make the design publicly available, to receive substantive critique that improves the architecture, and to maintain a versioned record of how the design evolves. It is not currently set up to receive code, evaluation results, or reference implementations.

## What kind of feedback is most useful

**Architectural critique.** Identification of failure modes the design does not address, mechanisms that will not work as specified, assumptions that will not hold in real deployments, or alternatives that would produce better outcomes. The more specific the better — "Section 5.6.1's scoring function does not handle case X" is far more useful than "the conflict resolution feels fragile."

**Identification of related work.** Adjacent systems, papers, or designs that the document should engage with. The Related Work section (1.4) and References appendix are not exhaustive, and the design is strengthened by being explicitly positioned against more of the existing landscape.

**Empirical calibration suggestions.** The document flags many parameters as requiring empirical calibration. If you have deployment data, research results, or domain experience that suggests ranges for these parameters, that's valuable.

**Specific section feedback.** Issues that identify a specific section number, quote the specific text being engaged with, and explain the concern are easier to act on than general impressions.

## What kind of contribution is not currently invited

**Pull requests that rewrite portions of the design.** The document represents specific architectural choices, some of which I'll defend and some of which I'll revise based on feedback — but in either case, the revision should happen as a result of discussion rather than a PR that presents a finished alternative. If you have a substantially different design in mind, the most useful path is to open an issue describing it, or to publish your own design separately.

**Implementation contributions.** This repository does not contain code. Reference implementations may eventually appear in separate repositories.

**Typo and grammar fixes.** Appreciated in principle, but the document is in active revision and editorial fixes are better consolidated into batches rather than submitted as individual PRs. If you spot something egregious, file an issue.

## Issue templates

The repository provides two issue templates: one for architectural critique and one for questions about the design. Using the appropriate template helps me triage and respond.

## Response expectations

This is a solo effort done outside of any institutional context. Response time on issues is best-effort. Substantive critique will receive substantive response; low-effort comments may not.