# 202 – Transfers, currency exchanges and fixed bills

**Status:** přijato, 16. 10. 2026 (řeší F-PRE-4 ze zadání)

**Rozhodnutí:**
- Převod je dvojice v `transfer_pair`, ne příznak na transakci. Automaticky se páruje jen jednoznačná shoda: jediný kandidát bez soupeře, nebo kandidát na účtu, který platba jmenuje jako protiúčet. Ostatní řeší uživatel na stránce K vyřešení.
- Výměna měn v Revolutu (`Zameniť`) se páruje jako převod mezi měnovými účty téhož dne s rozdílnými částkami.
- Úroky z flexibilních fondů Revolutu jsou příjem „Úroky a výnosy“.
- Rozpočtová skupina „Fixní výdaje“ (nájem, energie) nehlásí „rychleji než tempo“: platí se jednou měsíčně, tempo o nich nic neříká.
