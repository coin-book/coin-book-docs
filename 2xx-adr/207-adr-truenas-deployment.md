# 207 – Nasazení na TrueNAS SCALE jako custom app

**Status:** přijato, 17. 10. 2026

**Kontext:** Aplikace má běžet trvale na domácím NAS (TrueNAS SCALE, od verze 24.10 nad Dockerem), ne jen na pracovním počítači. ADR 200 počítalo s `git pull` + `make app` na stroji s Go a pnpm; NAS nemá ani jedno a `make app` tam není. Vlastní katalog aplikací (train s `questions.yaml`) je pro jednoho uživatele zbytečná režie.

**Rozhodnutí:** Obrazy `ghcr.io/coin-book/coin-book-api` a `ghcr.io/coin-book/coin-book-web` publikuje GitHub Actions při pushi do `main` (`latest`, `sha-…`) a při tagu `v*`. Web image je SvelteKit build servírovaný Caddy včetně Caddyfile, takže `coin-book-local` a NAS běží stejný obraz. Na NAS se aplikace instaluje jako custom app z `coin-book-local/truenas/docker-compose.yml` (Apps → Install via YAML): Postgres a importy leží na datasetech, ven je vystaven jen web na portu 8090, hostnames NAS jsou v `COINBOOK_SERVER_ALLOWED_HOSTS`.

**Důsledky:** Aktualizace = nový pull obrazů v TrueNAS; migrace proběhnou při startu API. Data zůstávají v domácí síti (N-1), ale jsou dostupná z LAN po HTTP; TLS a `COINBOOK_SERVER_COOKIE_SECURE=true` přijdou s reverse proxy, až bude třeba. Dataset `imports` musí vlastnit uid 65532, protože API běží bez roota a distroless nemá `chown`. Záloha na NAS = snapshoty datasetů nebo `pg_dump` z kontejneru; `make backup` zůstává pro lokální běh. Doplňuje ADR 200: deploy workflow existuje, ale jen pro publikaci obrazů.
