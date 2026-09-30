# Contributing

Thanks for your interest in improving this analysis. This repository is a
**static security-research write-up**, so contributions are mostly about
accuracy, clarity, and additional evidence — not code.

## Ways to contribute

- **Corrections** — if a claim in a report doesn't match the decompiled code,
  open an issue (or PR) citing the class/method and what it actually does.
- **Additional depth** — deeper traces of a specific component (e.g. the exact
  exfil payload format, a command-dispatch path) are welcome.
- **New indicators** — additional endpoints, signing details, or version diffs,
  with the source of the observation.
- **Readability** — typos, formatting, tables, and structure.

## Ground rules

- **Defensive scope only.** This project documents capabilities and network
  behavior for privacy review and defense. Do **not** submit exploit code,
  weaponization, deployment/evasion instructions, or working credentials.
- **Cite your evidence.** Every factual claim should be traceable to the
  decompiled source, the manifest, or an extracted string. Reference the file
  and line where possible.
- **Static only.** Keep contributions consistent with the static-analysis
  methodology ([`reports/methodology.md`](reports/methodology.md)). If you run
  dynamic analysis, clearly label it as such and describe your lab setup.
- **No raw binaries.** Do not commit APKs, DEX files, or bulk decompiler output
  (see [`.gitignore`](.gitignore)). Quote only the minimal snippets needed to
  support a finding.
- **Respect authorization.** Only analyze software you are permitted to review.

## Workflow

1. Open an issue describing the correction or addition (use a template).
2. Fork and create a topic branch.
3. Keep changes focused; match the existing tone and Markdown style.
4. Open a pull request referencing the issue and summarizing the evidence.

## Style

- Wrap prose at ~80 columns to keep diffs readable.
- Prefer tables for structured comparisons.
- Use fenced code blocks for endpoints, class names, and snippets.
