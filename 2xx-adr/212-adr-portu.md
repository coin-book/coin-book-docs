# 212 – Portu: historie transakcí jako investiční účet

**Status:** přijato, 19. 10. 2026

**Kontext:** Uživatel investuje přes Portu (robo-advisor, ETF a akcie). Portu exportuje „Transakční historii“ v CSV za celou dobu. Ověřeno na skutečném exportu, v repozitáři je jen struktura (`importer-formaty.md`):
- Jeden řádek na pohyb: nákup, prodej, dividenda, vklad, směna měn, poplatek, sleva na poplatku, úrok z hotovosti a převody mezi portfolii.
- Účet má několik portfolií (sloupec `Název`). Mezi nimi se převádějí peníze i kusy.
- Prodeje nejsou po lotech, jen počet kusů a cena.
- Hodnoty jsou jen v měně obchodu. CZK hodnotu neuvádí, ale kurz plyne ze směn, které Portu ten den provedlo (`Forex nákup/prodej`, v roce 2022 `Swap USDCZK`).
- Hotovost v rezervě má pseudo-ISIN `CZK000000001`.

**Rozhodnutí:**
- Parser `portu-transactions` vytvoří jeden účet `broker` (`portu:investments`). Pozice se oceňují jako u Revolutu (ADR 209).
- **Loty párujeme FIFO v rámci portfolia.** Převod kusů do jiného portfolia není prodej, loty si nesou původní datum nákupu (test 3 let). Prodej, který žádný naimportovaný nákup nepokryje, dostane lot koupený v den prodeje: test 3 let nesplní a pozice ukáže neúplnou historii.
- **CZK hodnoty** obchodů a dividend: kurz směn téhož dne (CZK ÷ cizí měna), jinak nejbližšího dne do 7 dnů. Dál zůstane CZK hodnota prázdná.
- **Symbol Yahoo** plyne ze symbolu Portu: `LN` → `.L`, `GY` → `.DE`, `CP` → `.PR`, `US` bez přípony. `statement.Security.YahooSymbol` má přednost před odhadem podle měny.
- **Neimportuje se:** vklady (párují se s bankou jako vlastní převody podle čísla účtu, ADR 206), směny, poplatky, úroky z hotovosti a hotovost v portfoliích. Hodnota účtu jsou jeho pozice.
- Idempotence: klíče obsahují portfolio. FIFO je deterministické, takže pozdější export se stejnou historií dá stejné prodané loty.

**Důsledky:**
- Import potřebuje celou historii. Export za období by FIFO spároval špatně.
- Poplatky Portu nejsou v nákladech lotů ani ve výdajích.
- Párování FIFO nemusí odpovídat tomu, jak loty vybírá Portu. Pro daň je to přehled, ne daňové přiznání.
