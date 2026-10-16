# 200 – Multi-repo, self-hosted on Docker Compose

**Status:** přijato, 16. 10. 2026

**Kontext:** Aplikace má jednoho uživatele a data nesmí opustit počítač (N-1). Vzorem jsou edumeal a noeto: Go API, SvelteKit web, samostatný MCP server, oddělené repozitáře.

**Rozhodnutí:** Stejné rozdělení repozitářů jako vzory. Místo AWS a Terraformu běží vše přes Docker Compose v `coin-book-local`, porty jen na 127.0.0.1, Caddy servíruje web i API z jednoho originu.

**Důsledky:** Žádný `-infra` repozitář ani deploy workflow; aktualizace = `git pull` + `make app`. Záloha je `pg_dump` + importované soubory jedním příkazem.
