# CLAUDE.md

Guidance for AI agents working in this repository.

## What this repo is

A curated library of **distilled, source-verified reference guides** for popular Python packages, designed to be dropped into other projects so AI coding agents can code against accurate, current APIs instead of guessing from stale training data.

This is **not** an application or library — there is no source code, no tests, no build. The product is the markdown files in [package-guides/](package-guides/). Treat every change as a documentation change.

The README at [README.md](README.md) is the public catalog. Anything that changes the set of guides must also update the README.

## Repo layout

```
package-guides/        The product. One markdown file per package.
projects_temp/         Scratch space for package sources/docs while authoring a guide. Gitignored.
README.md              Public catalog. Table is ordered by package popularity (not alphabetical).
LICENSE                MIT.
.claude/settings.json  Pre-approved Bash permissions for this repo.
```

`projects_temp/` is where you place the target package's source code and official docs while researching a new guide. It is gitignored — never commit anything from it. Clean it up after the guide is done if it bloats.

## Workflow: adding or updating a guide

Follow this order. The cross-check step is what makes these guides valuable — skipping it defeats the purpose of the repo.

1. **Acquire the source.** Drop the latest source code (and ideally the official docs) for the target package into [projects_temp/](projects_temp/). Prefer a release tag over `main`/`master` so the version in the preamble is meaningful.
2. **Read both.** Read the official documentation *and* the source. Note the version you're working from.
3. **Draft the guide** in [package-guides/](package-guides/) using the conventions below.
4. **Cross-check every signature, default, and behavior against the source code.** When the docs and the source disagree, **the source wins** — and consider noting the discrepancy in the guide if it's likely to bite users.
5. **Update [README.md](README.md)**: add (or update) the row in the table, insert it in **popularity order** (not alphabetical), and refresh the approximate line count.
6. **Commit.** Follow the existing commit style — short, lowercase-ish, present tense (`Add markdown2`, `adds granian and docker`, `Add discord.py, HTTPX, and listmonk package guides`).

## Guide format conventions

Every guide follows the same shape. Match it.

### Preamble
- `# <Package> — Comprehensive API Reference` (or `Comprehensive Reference`) as the H1.
- A one-paragraph blockquote or sentence summarizing what the package is.
- A version line such as `> Version: 0.7.3 | License: MIT | Python: 3.9+` or a sentence like *"Built from the source code of the HTTPX 0.28.1 repository."* The verified version must appear somewhere in the preamble.
- Horizontal rule (`---`).

### Table of contents
- Markdown TOC with anchor links to every major section.
- For long guides, group with sub-bullets (see [loguru_reference.md](package-guides/loguru_reference.md)).

### Body sections
- Lead with **Installation** and **Quick Start**, then move section-by-section through the API surface.
- For each function/class/method, show the **full signature** in a fenced ` ```python ` block — keyword args, type hints, and defaults included.
- Follow signatures with a **parameter table** (`| Parameter | Type | Default | Description |`).
- Cite the source module path when helpful (e.g., `**Source:** httpx/_api.py`).
- Include short, runnable code examples — not toy snippets, but the shape of real usage.
- End with **Common Patterns** / recipes that show idiomatic, multi-step usage.

### Style rules
- Plain markdown — no HTML, no emojis, no MDX.
- Be exhaustive about API surface, but don't pad. If a parameter is rarely used, mention it briefly; if it's load-bearing, give it space.
- Document deprecated APIs only if users are still likely to encounter them. Skip private/internal modules unless they leak into public usage.
- Show defaults as actual values, not prose ("`timeout: float = 5.0`", not "defaults to five seconds").
- When the source and docs disagree, document what the source actually does.

## Naming

- File pattern: `<package>_reference.md` in snake_case, matching how the package is commonly imported or spelled.
- Examples: `flask_reference.md`, `pymongo_reference.md`, `discordpy_reference.md` (for `discord.py`), `dataclasswizard_reference.md`.
- For already-hyphenated package names, hyphens are fine: `chameleon-flask_reference.md`, `chameleon-partials_reference.md`, `content-types_reference.md`.
- One outlier exists (`robyn-reference.md`); don't replicate that pattern for new guides.

## README maintenance

The table in [README.md](README.md) is **ordered by package popularity, not alphabetically**. When adding a new guide:

- Insert the row at the position matching its rough popularity relative to neighbors.
- Fill all four columns: Package (linked to the guide), Version, Lines (approx, rounded to nearest ~10), Description.
- Get the line count with `wc -l package-guides/<file>` (this Bash invocation is pre-approved in [.claude/settings.json](.claude/settings.json)).
- Round line counts to the nearest 10 and prefix with `~` to match the existing column style.

## What not to do

- Don't invent APIs or signatures. If you can't verify it from the source, don't include it.
- Don't commit anything from `projects_temp/`.
- Don't reorder the README table alphabetically — popularity ordering is intentional.
- Don't add tooling (linters, CI, build scripts) unless asked. This repo is deliberately just markdown + LICENSE + README.
- Don't add a guide for a package you haven't actually pulled the source for.
