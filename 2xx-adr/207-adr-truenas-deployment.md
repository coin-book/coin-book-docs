# 207 – Nasazení na TrueNAS SCALE jako custom app

**Status:** přijato, 17. 10. 2026

**Kontext:** Aplikace má běžet trvale na domácím NAS (TrueNAS SCALE, od verze 24.10 nad Dockerem), ne jen na pracovním počítači. ADR 200 počítalo s `git pull` + `make app` na stroji s Go a pnpm; NAS nemá ani jedno a `make app` tam není. Vlastní katalog aplikací (train s `questions.yaml`) je pro jednoho uživatele zbytečná režie.

**Rozhodnutí:** Obrazy `ghcr.io/coin-book/coin-book-api` a `ghcr.io/coin-book/coin-book-web` publikuje GitHub Actions při pushi do `main` (`latest`, `sha-…`) a při tagu `v*`. Web image je SvelteKit build servírovaný Caddy včetně Caddyfile, takže `coin-book-local` a NAS běží stejný obraz. Na NAS se aplikace instaluje jako custom app (Apps → Install via YAML) z YAML, který vygeneruje `make truenas HOST=…` v `coin-book-local`: doplní tajemství a hostnames NAS do `COINBOOK_SERVER_ALLOWED_HOSTS`. Postgres a importy leží v Docker volumes aplikace, ne na vlastních datasetech; ven je vystaven jen web na portu 8090. Vlastní katalog s `questions.yaml` TrueNAS od 24.10 pro cizí aplikace nenabízí.

**Důsledky:** Aktualizace = nový pull obrazů v TrueNAS; migrace proběhnou při startu API. Data zůstávají v domácí síti (N-1), ale jsou dostupná z LAN po HTTP; TLS a `COINBOOK_SERVER_COOKIE_SECURE=true` přijdou s reverse proxy, až bude třeba. Docker volumes dostanou vlastníka z obrazu, takže API (uid 65532, distroless bez `chown`) zapisuje bez ručního nastavení práv. Záloha na NAS = snapshot datasetu `ix-apps` nebo `pg_dump` z kontejneru; `make backup` zůstává pro lokální běh. Doplňuje ADR 200: deploy workflow existuje, ale jen pro publikaci obrazů.
