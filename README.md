# Python Package Guides for AI Agents

Distilled, source-verified documentation for popular Python packages — built to superpower AI coding agents like [Claude Code](https://claude.ai/claude-code) and [Codex](https://openai.com/index/codex/).

**The idea is simple:** browse the guides below, copy the ones relevant to your project into your repo, and point your AI agent at them. Your agent codes against verified, current APIs instead of guessing from stale training data.

## Why this exists

AI coding agents are incredible, but they have a documentation problem:

- **Training data goes stale.** Models are trained on package docs as they existed months ago. APIs change, defaults shift, arguments get renamed.
- **Web search is slow and noisy.** Agents that search the internet mid-task burn tokens, lose context, and sometimes land on outdated blog posts or wrong versions.
- **Hallucinated APIs cause real bugs.** When an agent guesses at a function signature instead of looking it up, you get code that *looks* right but fails at runtime.

These guides solve all three problems. Each one was created by downloading the **latest source code and documentation** for a package, deeply parsing both, cross-referencing the docs against the actual implementation, and distilling the result into a single, agent-friendly markdown file. This has saved us many bugs before we even started coding.

## How to use these guides

This repo is **not** meant to be cloned and used as-is. It's a library you pick from. The workflow:

1. **Identify** which packages your project depends on
2. **Copy** the matching guide(s) into your project — e.g., into a `docs/`, `agent-refs/`, or `ai-context/` directory
3. **Tell your agent** about them so it reads the guides before writing code
4. **Code with confidence** — fewer hallucinated APIs, fewer runtime surprises

The guides live in your repo alongside your code, so your agent always has them in reach without searching the web or relying on training data.

## What's included

| Package | Version | Lines | Description |
|---------|---------|------:|-------------|
| [Django](package-guides/django_reference.md) | 6.1 | ~2,480 | The batteries-included Python web framework |
| [Pydantic](package-guides/pydantic_reference.md) | 2.13.x | ~2,190 | Data validation and settings management using Python type hints |
| [Flask](package-guides/flask_reference.md) | latest | ~2,100 | The classic Python web micro-framework |
| [Quart](package-guides/quart_reference.md) | latest | ~2,160 | Async Flask-compatible web framework |
| [Pyramid](package-guides/pyramid_reference.md) | 2.1 | ~2,480 | Flexible, full-featured Python web framework |
| [Robyn](package-guides/robyn-reference.md) | latest | ~690 | High-performance Python-Rust web framework |
| [PyMongo](package-guides/pymongo_reference.md) | 4.x | ~2,530 | MongoDB driver for Python |
| [MongoEngine](package-guides/mongoengine_reference.md) | 0.29.0 | ~1,600 | Python Object-Document Mapper (ODM) for MongoDB |
| [Tenacity](package-guides/tenacity_reference.md) | latest | ~1,200 | Retry library for Python |
| [DiskCache](package-guides/diskcache_reference.md) | 5.6.3 | ~1,230 | Disk-backed cache using SQLite |
| [Dataclass Wizard](package-guides/dataclasswizard_reference.md) | 0.39.1 | ~1,510 | Dataclass serialization/deserialization |
| [chameleon-flask](package-guides/chameleon-flask_reference.md) | 0.6.0 | ~780 | Chameleon template engine integration for Flask |
| [chameleon_partials](package-guides/chameleon-partials_reference.md) | 0.1.0 | ~770 | Partial template rendering for Chameleon |
| [FastAPI](package-guides/fastapi_reference.md) | latest | ~1,930 | Modern async web framework with automatic OpenAPI docs |
| [Loguru](package-guides/loguru_reference.md) | 0.7.3 | ~1,280 | Enjoyable, no-boilerplate logging for Python |
| [NiceGUI](package-guides/nicegui_reference.md) | latest | ~2,570 | Python UI framework built on FastAPI, Vue 3, and Quasar |
| [content-types](package-guides/content-types_reference.md) | latest | ~690 | MIME type detection and content type mapping |
| [Docker](package-guides/docker_reference.md) | latest | ~2,230 | Container platform for building, shipping, and running applications |
| [Stripe](package-guides/stripe_reference.md) | 15.1.0 | ~1,420 | Official Stripe Python SDK for payments, billing, subscriptions, and Connect |
| [discord.py](package-guides/discordpy_reference.md) | 2.8.0a | ~2,180 | Feature-rich Discord API wrapper for Python |
| [HTTPX](package-guides/httpx_reference.md) | 0.28.1 | ~1,640 | Modern async/sync HTTP client for Python |
| [listmonk](package-guides/listmonk_reference.md) | latest | ~860 | Python client for the self-hosted Listmonk email platform |
| [Umami](package-guides/umami_reference.md) | v2.x/v3.x | ~1,290 | HTTP API for the privacy-focused web analytics platform — stats, reports, events, and ingestion |
| [Granian](package-guides/granian_reference.md) | 2.7.2 | ~1,200 | Rust HTTP server for Python ASGI/RSGI/WSGI applications |
| [markdown2](package-guides/markdown2_reference.md) | latest | ~970 | Fast, complete Python Markdown-to-HTML converter with 30+ extras |
| [Bulma](package-guides/bulma_reference.md) | 1.0.4 | ~1,510 | Modern CSS framework based on Flexbox with built-in dark mode and theming |
| [EasyMDE](package-guides/easymde_reference.md) | 2.21.0 | ~820 | Embeddable JavaScript Markdown editor — drop-in textarea replacement for web app forms |


### Claude Code

Add a reference in your project's `CLAUDE.md` so the agent knows to check the guides automatically:

```markdown
## Package References

When working with Flask, refer to docs/flask_reference.md for the current API.
When working with PyMongo, refer to docs/pymongo_reference.md for the current API.
```

Or point to them directly during a session:

```
Read the flask reference at docs/flask_reference.md before implementing the route.
```

### Codex / Other Agents

Copy the relevant guide(s) into your project and reference them in whatever context file your agent uses (e.g., `AGENTS.md`, `codex.md`, system prompts). The guides are plain markdown — they work with any agent that reads files.

## Requesting a new guide

Want a guide for another Python package? [Open an issue](../../issues) with the package name and we'll consider generating one. Each guide requires downloading the latest source, cross-referencing docs against the implementation, and distilling it — so we keep the scope to packages where this adds real value.

## License

MIT — see [LICENSE](LICENSE) for details.
