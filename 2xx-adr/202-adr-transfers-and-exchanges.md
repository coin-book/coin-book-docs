# 202 – Transfers, currency exchanges and fixed bills

**Status:** přijato, 16. 10. 2026 (řeší F-PRE-4 ze zadání)

**Rozhodnutí:**
- Převod je dvojice v `transfer_pair`, ne příznak na transakci. Automaticky se páruje jen jednoznačná shoda: jediný kandidát bez soupeře, nebo kandidát na účtu, který platba jmenuje jako protiúčet. Ostatní řeší uživatel na stránce K vyřešení. Párování běží po každém importu nad všemi nespárovanými transakcemi, nejen nad novými: na pořadí importů ani na později doplněném čísle účtu nezáleží.
- Výměna měn v Revolutu (`Zameniť`) se páruje jako převod mezi měnovými účty téhož dne s rozdílnými částkami.
- Převod do jiné měny mimo výměnu (např. CZK z mBank na eurový účet) se automaticky páruje jen tehdy, když jedna strana jmenuje číslo účtu druhé (v jednotném tvaru `bank.CanonicalNumber`) a částky se po přepočtu kurzem ČNB (`fx_rate`) liší nejvýš o 10 %. Ruční výběr protistrany nabízí pohyby v jiné měně ±7 dní, které kurzem sedí; bez uloženého kurzu nefiltruje.
- Úroky z flexibilních fondů Revolutu jsou příjem „Úroky a výnosy“.
- Rozpočtová skupina „Fixní výdaje“ (nájem, energie) nehlásí „rychleji než tempo“: platí se jednou měsíčně, tempo o nich nic neříká.
