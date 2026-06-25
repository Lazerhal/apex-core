# apex-core

Shared library for the APEX project operating system. Contains schemas, markdown tooling, model routing helpers, common data models, connector clients, and reusable prompts.

**Part of the APEX ecosystem — see [apex-control](https://github.com/Lazerhal/apex-control) for the main application.**

## Structure
- `src/apex_core/schemas/` — Pydantic data models and validation
- `src/apex_core/markdown/` — Markdown parser, writer, validator, handoff generator
- `src/apex_core/routing/` — Model router and provider abstractions
- `src/apex_core/models/` — Shared entity definitions
- `src/apex_core/connectors/` — External service clients (GA4, Supabase, Railway, etc.)
- `src/apex_core/prompts/` — Reusable prompt templates
- `docs/` — Schema specs and architecture docs
