# 210 – Verze a releasy

**Status:** přijato, 18. 10. 2026

**Kontext:** API a web se po každém pushi na `main` sestaví do obrazu `latest` v GHCR. TrueNAS compose (ADR 207) běží právě z `latest`. Aktualizace tak není opakovatelná a nedá se vrátit, a z běžící aplikace se nepozná, co na NAS je. MCP server se vydával ručně se zadanou verzí. Commity už píšeme jako conventional commits se scope.

**Rozhodnutí:**
- **SemVer v každém repozitáři zvlášť**, od `0.1.0`. Před `1.0` zvedá `feat` minor a `fix` patch (`bump-minor-pre-major`). `coin-book-docs` se neverzuje.
- **release-please** v `coin-book-api`, `coin-book-web`, `coin-book-mcp` a `coin-book-local`:
  - Z commitů na `main` udržuje otevřený „release PR“ s novou verzí, `CHANGELOG.md` a `version.txt`.
  - Merge toho PR vytvoří tag `vX.Y.Z` a GitHub Release.
  - Tag vytvořený přes `GITHUB_TOKEN` nespustí další workflow. Obraz (API, web) a GoReleaser (MCP) proto běží v tomtéž workflow hned po vydání.
  - Repozitáře musí mít povoleno „Allow GitHub Actions to create and approve pull requests“.
- **Obrazy:** každý push na `main` dál publikuje `latest` a `sha-…` pro vývoj. Vydání navíc publikuje `X.Y.Z` a `X.Y`.
- **`coin-book-local` je distribuce.** Jeho verze je to, co instaluje uživatel. TrueNAS compose v něm pripína přesné verze obrazů API a webu, nikdy `latest`.
  - Nová verze API nebo webu se do distribuce dostane commitem `feat(local): …`, který posune připnuté verze. release-please pak vydá novou verzi `coin-book-local`.
  - Návrat na předchozí verzi je vygenerování compose ze staršího tagu `coin-book-local`. Databáze se ale nevrací: migrace jdou jen dopředu, takže před aktualizací se dělá záloha (`make backup`).
- **Verze je vidět v aplikaci:** Nastavení ukazují verzi webu (z buildu) a API (`GET /api/v1/version`).

**Důsledky:**
- Kompatibilitu API a webu zaručuje dvojice verzí, kterou připíná `coin-book-local`. Samostatně vydané verze spolu nemusí fungovat.
- První vydání vznikne, až se sloučí první release PR. Do té doby TrueNAS compose odkazuje na obrazy `0.1.0`, které ještě neexistují. Nejdřív se tedy vydá API a web, pak `coin-book-local`.
- Changelog prvního vydání obsahuje celou dosavadní historii.
