# 201 – Agent access through the API, read-only scopes

**Status:** přijato, 16. 10. 2026

**Kontext:** F-AGT-2 požaduje tokeny s oprávněními a audit každého přístupu. noeto má PAT tokeny bez scopes.

**Rozhodnutí:** MCP server nemá vlastní přístup k databázi; volá API s tokenem `coinbook_pat_…`. Každá operace v API deklaruje úroveň (`AuthSession` nebo `AuthRead` + scope); všechny zápisy jsou `AuthSession`, takže token nic nezapíše. Audit zapisuje middleware u každého požadavku s tokenem včetně odmítnutých.

**Důsledky:** Oprávnění a audit jsou na jednom místě. MCP funguje jen při běžícím API.
