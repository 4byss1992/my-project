# Security Policy

## About this repository

This repository contains **static analysis write-ups** of third-party Android
applications. It does not ship a product, service, or executable code, so there
is no software here to patch or version for vulnerabilities.

## Reporting an issue with the analysis

If you believe a report contains an **inaccuracy** (a mischaracterized
capability, a wrong endpoint, an incorrect signing detail, etc.), please open a
regular issue with:

- the report and section affected,
- the specific claim,
- the corrected information, and
- the supporting evidence (decompiled class/method, manifest entry, or extracted
  string).

## Reporting sensitive concerns

If you need to raise something **privately** — for example, you are a vendor
named in a report and wish to discuss factual accuracy, or you believe a
document unintentionally discloses sensitive information — please use GitHub's
**private vulnerability reporting** on this repository, or open a minimal issue
asking to be contacted privately rather than posting the details publicly.

We will review good-faith reports and correct confirmed inaccuracies.

## Scope & principles

- Findings are for **defensive, privacy-review, and educational** purposes only.
- Reports contain **no exploit code** and **no working credentials**; embedded
  identifiers found in the binaries are cited as indicators, not for reuse.
- Only software the analyst is authorized to review is examined.
