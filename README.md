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

| Guide | Package | Version | Lines | Description |
|-------|---------|---------|------:|-------------|
| [django_reference.md](package-guides/django_reference.md) | Django | 6.1 | ~2,480 | The batteries-included Python web framework |
| [pydantic_reference.md](package-guides/pydantic_reference.md) | Pydantic | 2.13.x | ~2,190 | Data validation and settings management using Python type hints |
| [flask_reference.md](package-guides/flask_reference.md) | Flask | latest | ~2,100 | The classic Python web micro-framework |
| [quart_reference.md](package-guides/quart_reference.md) | Quart | latest | ~2,160 | Async Flask-compatible web framework |
| [pyramid_reference.md](package-guides/pyramid_reference.md) | Pyramid | 2.1 | ~2,480 | Flexible, full-featured Python web framework |
| [robyn-reference.md](package-guides/robyn-reference.md) | Robyn | latest | ~690 | High-performance Python-Rust web framework |
| [pymongo_reference.md](package-guides/pymongo_reference.md) | PyMongo | 4.x | ~2,530 | MongoDB driver for Python |
| [mongoengine_reference.md](package-guides/mongoengine_reference.md) | MongoEngine | 0.29.0 | ~1,600 | Python Object-Document Mapper (ODM) for MongoDB |
| [tenacity_reference.md](package-guides/tenacity_reference.md) | Tenacity | latest | ~1,200 | Retry library for Python |
| [diskcache_reference.md](package-guides/diskcache_reference.md) | DiskCache | 5.6.3 | ~1,230 | Disk-backed cache using SQLite |
| [dataclasswizard_reference.md](package-guides/dataclasswizard_reference.md) | Dataclass Wizard | 0.39.1 | ~1,510 | Dataclass serialization/deserialization |
| [chameleon-flask_reference.md](package-guides/chameleon-flask_reference.md) | chameleon-flask | 0.6.0 | ~780 | Chameleon template engine integration for Flask |
| [chameleon-partials_reference.md](package-guides/chameleon-partials_reference.md) | chameleon_partials | 0.1.0 | ~770 | Partial template rendering for Chameleon |
| [fastapi_reference.md](package-guides/fastapi_reference.md) | FastAPI | latest | ~1,930 | Modern async web framework with automatic OpenAPI docs |
| [loguru_reference.md](package-guides/loguru_reference.md) | Loguru | 0.7.3 | ~1,280 | Enjoyable, no-boilerplate logging for Python |
| [nicegui_reference.md](package-guides/nicegui_reference.md) | NiceGUI | latest | ~2,570 | Python UI framework built on FastAPI, Vue 3, and Quasar |
| [content-types_reference.md](package-guides/content-types_reference.md) | content-types | latest | ~690 | MIME type detection and content type mapping |


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
