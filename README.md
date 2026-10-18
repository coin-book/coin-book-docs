# coin-book-docs

[![ADR](https://img.shields.io/badge/decisions-ADR-blueviolet)](2xx-adr/)
![language](https://img.shields.io/badge/language-%C4%8De%C5%A1tina-lightgrey)

Architecture notes and decisions of coin-book.

- `1xx-architektura/` – architecture notes.
- `2xx-adr/` – decisions, numbered, never rewritten; a reversed decision gets a new ADR that supersedes it.

The requirements (`zadanie-aplikacie.md`), the technical specification (`technicka-specifikace.md`) and the
bank format notes (`importer-formaty.md`) live in the umbrella folder next to the repositories.

| Repository | What |
|---|---|
| [coin-book-api](https://github.com/coin-book/coin-book-api) | Go REST API, PostgreSQL |
| [coin-book-web](https://github.com/coin-book/coin-book-web) | SvelteKit web client |
| [coin-book-mcp](https://github.com/coin-book/coin-book-mcp) | read-only MCP server for AI agents |
| [coin-book-local](https://github.com/coin-book/coin-book-local) | Docker Compose, TrueNAS deployment, backups |
