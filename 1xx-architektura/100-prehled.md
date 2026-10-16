# 100 – Přehled architektury

```mermaid
flowchart LR
    B[Prohlížeč] -->|127.0.0.1:8090| C[Caddy]
    C -->|/| W[coin-book-web build]
    C -->|/api/v1| A[coin-book-api]
    A --> P[(PostgreSQL 18)]
    M[coin-book-mcp stdio] -->|coinbook_pat_ token| A
```

Import: soubor → detekce formátu → parser → kontrola součtů → deterministická ID → jedna DB transakce (COPY do staging, `INSERT … ON CONFLICT DO NOTHING`, kategorizace pravidly a nápovědou banky, párování převodů). Měřeno: 10 000 řádků za ~0,45 s.
