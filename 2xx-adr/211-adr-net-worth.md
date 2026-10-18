# 211 – Čisté jmění: aktiva bez tržní ceny, hypotéka a Fingood

**Status:** přijato, 18. 10. 2026

**Kontext:** F-INV-2 chce celkový majetek. Zatím ho tvoří hotovost na účtech a cenné papíry z ADR 209. Uživatel chce vidět celé jmění: hypotéku na vlastní nemovitost, P2P platformy (Fingood, Investown) a další druhy aktiv, které nemají tržní cenu. Ověřeno na skutečném exportu Fingoodu, struktura je v `importer-formaty.md`:
- Export „Moje investice“ je **přehled pozic k okamžiku exportu**: jeden řádek na úvěr. Pohyby (vklady, splátky, úroky po datech) v něm nejsou.
- Úroky, jistina a poplatky jsou kumulativní součty od začátku investice.
- Datum exportu soubor neobsahuje v listu, jen v metadatech (`docProps/core.xml`, `dcterms:created`, UTC).
- Volné prostředky na účtu u platformy v exportu nejsou.
- Pohyby na účtu Fingood exportovat neumí, nabízí jen roční výpis.

**Rozhodnutí:**
- **Způsob ocenění** je vlastnost účtu (`account.valuation`):
  - `ledger`: počáteční zůstatek + transakce. Běžné a spořicí účty, jako dosud.
  - `market`: kusy × cena (ADR 209). Broker.
  - `snapshot`: hodnoty k datu. Mezi nimi platí poslední známá, před první se účet do jmění nepočítá. P2P platformy, penzijní spoření, nemovitost, ostatní aktiva.
  - `schedule`: zůstatek ze splátkového kalendáře. Hypotéka a úvěry.
- **Nové druhy účtů:** `p2p`, `pension`, `property`, `other_asset` (aktiva) a `mortgage`, `loan` (závazky, do jmění se počítají záporně). Nové instituce `fingood` a `investown`. CHECK na `account.kind` a `account.institution` se mění s nimi.
- **`account_valuation`** (účet, den, hodnota v minor jednotkách, měna, zdroj `manual` nebo `import`, `import_batch_id`). Má RLS jako všechna data uživatele (ADR 208). Druhé ocenění téhož účtu ke stejnému dni nahradí první.
- **Čisté jmění k dni** = součet ocenění všech účtů přepočtený kurzem ČNB k tomu dni (ADR 209). Počítá se, neukládá. Řada pro graf se skládá po dnech z ocenění a transakcí.
- **Vklady a výběry** na platformu jsou vlastní převody podle ADR 206: protiúčet platformy nebo zapamatovaná protistrana. U Fingoodu jsou jediným zdrojem vkladů a výběrů, protože pohyby platforma neexportuje. Z nich a z ocenění se počítá **výnos XIRR** a porovnává se s úrokem, který platforma uvádí.

**Fingood:**
- Parser `fingood-investments` pro XLSX „Moje investice“. Pozná ho podle hlavičky (`Číslo úvěru`, `Název investice`, `Původní jistina`…), sloupce čte podle názvu.
- Import uloží **snímek pozic** `p2p_position` (účet, den snímku, číslo úvěru, název, způsob splácení, nesplacená jistina, úrok p.a., příští a poslední splátka, stav, stav splácení, zpoždění, po splatnosti, kumulativní úroky, sankční úroky, poplatky). Tabulka má RLS a `import_batch_id`, takže snímek pokryje i rollback importu.
- **Den snímku** je `dcterms:created` převedené na datum v Praze. Když chybí, použije se den importu. Náhled importu ho ukáže k potvrzení.
- **Nesplacená jistina** = `Původní jistina` + `Nakoupeno jistiny na Marketu` − `Prodáno jistiny na Marketu` − `Vráceno jistiny`. Součet za snímek je ocenění účtu (`import`).
- Úvěry po splatnosti nebo se `Stav splácení` jiným než „V pořádku“ se ukážou jako **ohrožená jistina**. Aplikace jejich hodnotu sama nesnižuje, odhad vymožení nedělá.
- **Výnos za období** = rozdíl kumulativních úroků a sankčních úroků minus poplatků mezi dvěma snímky. První snímek dává součty za celou dobu.
- **Splatnostní žebříček:** kolik jistiny se vrátí v kterém měsíci podle `Termín poslední splátky`. Balonové úvěry vracejí jistinu až na konci, takže je to skutečná likvidita platformy.
- Idempotence: klíč pozice je (účet, den snímku, číslo úvěru). Stejný soubor podruhé nic nezaloží.

**Hypotéka:**
- **`loan_terms`** (účet, platí od, jistina nebo zůstatek k datu, sazba p.a., splátka, den splátky, splatnost, konec fixace). Nová sazba po fixaci je nový řádek s novým „platí od“.
- Zůstatek se počítá z kalendáře od posledního známého bodu. Ruční ocenění (`account_valuation`, například zůstatek z ročního výpisu banky) kalendář opraví a počítá se dál od něj.
- **Splátka z banky** (už importovaná transakce) se označí jako splátka úvěru (`loan_payment`: transakce, účet, úrok, jistina). Rozdělení navrhne kalendář a uživatel ho může opravit. Úrok je výdaj v kategorii „Úroky z hypotéky“. Jistina je převod, který snižuje dluh, a do výdajů se nepočítá.
- **Nemovitost** je účet `property` s ručním oceněním, první hodnota je kupní cena. Přehled ukáže vlastní kapitál = hodnota nemovitosti − zůstatek hypotéky.
- Odpočet do konce fixace a simulace mimořádné splátky (o kolik klesne splátka nebo se zkrátí doba splácení).
- **Daňový přehled úroků:** úroky zaplacené za kalendářní rok a limit odpočtu 150 000 Kč. Nemovitost je vlastní bydlení, takže odpočet se na ni vztahuje. Pronajímanou nemovitost aplikace neřeší. Je to přehled, ne daňové přiznání.

**Přehled jmění** (web `/majetek`, MCP jen ke čtení):
- čisté jmění v čase, aktiva a závazky,
- rozložení podle druhu aktiva, likvidity (hned, do roka, zamčené) a měny,
- rezerva v měsících = likvidní hotovost / průměrné měsíční výdaje za 12 měsíců,
- měsíční přírůstek jmění rozdělený na vlastní vklady a výnos.

**Mimo rozsah:** roční výpis Fingoodu, importér Investownu, dokud nebude ověřený export (do té doby ruční ocenění), automatické ocenění nemovitosti podle cenového indexu, krypto a komodity (dál podle ADR 209).

**Důsledky:**
- Pohled `transfer_leg` a výdajové přehledy musí u splátky úvěru brát jen část jistiny jako převod a úrok jako výdaj. Dosud byla transakce převodem celá, nebo vůbec.
- Ocenění Fingoodu nezahrnuje volné prostředky na platformě. Kdo je tam drží, založí si pro ně samostatný účet s ručním oceněním.
- Sloupce Marketu jsou v ověřeném exportu všude nulové, takže vzorec nesplacené jistiny pro nákupy a prodeje na Marketu není ověřený. Stejně tak význam sloupce `Přislíbeno`. **[OVĚŘIT na exportu s obchodem na Marketu]**
- `Stav` a `Stav splácení` jsou číselníky Fingoodu a známe jen jejich část. Neznámá hodnota se uloží beze změny a počítá se jako ohrožená.
- Hodnota mezi snímky je schodovitá. Jmění u P2P je tak přesné, jak často uživatel exportuje. Přehled ukáže stáří posledního ocenění.

