# 211 – Čisté jmění: aktiva bez tržní ceny, hypotéka a Fingood

**Status:** přijato, 18. 10. 2026

**Kontext:** F-INV-2 chce celkový majetek. Zatím ho tvoří hotovost na účtech a cenné papíry z ADR 209. Uživatel chce vidět celé jmění: hypotéku na vlastní nemovitost, P2P platformy (Fingood, Investown) a další druhy aktiv, které nemají tržní cenu. Ověřeno na skutečném exportu Fingoodu, struktura je v `importer-formaty.md`:
- Export „Moje investice“ je **přehled pozic k okamžiku exportu**: jeden řádek na úvěr. Pohyby (vklady, splátky, úroky po datech) v něm nejsou.
- Úroky, jistina a poplatky jsou kumulativní součty od začátku investice.
- Datum exportu soubor neobsahuje v listu, jen v metadatech (`docProps/core.xml`, `dcterms:created`, UTC).
- Volné prostředky na účtu u platformy v exportu nejsou.
- Pohyby na účtu Fingood exportovat neumí, nabízí jen roční výpis.

**Rozhodnutí:**
- **Způsob ocenění** plyne z druhu účtu (`account.Kind.Valuation()`), v tabulce se neukládá:
  - `ledger`: počáteční zůstatek + transakce. Běžné a spořicí účty, jako dosud.
  - `market`: kusy × cena (ADR 209). Broker.
  - `snapshot`: hodnoty k datu. Mezi nimi platí poslední známá, před první se účet do jmění nepočítá. P2P platformy, penzijní spoření, nemovitost, ostatní aktiva.
  - `schedule`: zůstatek ze splátkového kalendáře. Hypotéka a úvěry.
- **Nové druhy účtů:** `p2p`, `pension`, `property`, `other_asset` (aktiva) a `mortgage`, `loan` (závazky, do jmění se počítají záporně). Nové instituce `fingood` a `investown`. CHECK na `account.kind` a `account.institution` se mění s nimi.
- **`account_valuation`** (účet, den, hodnota v minor jednotkách v měně účtu, poznámka) jsou hodnoty zadané ručně. U úvěru je to dlužný zůstatek, kladně. Má RLS jako všechna data uživatele (ADR 208) a zápis navíc ověřuje, že účet je vidět, protože cizí klíč RLS obchází. Druhé ocenění téhož účtu ke stejnému dni nahradí první. Importovaný snímek P2P se do ní nezapisuje: hodnotou je součet `p2p_position` za den snímku. Ruční hodnota ze stejného dne má přednost.
- **Čisté jmění k dni** = součet ocenění všech nearchivovaných účtů přepočtený kurzem ČNB platným k tomu dni (ADR 209). Den před prvním staženým kurzem vezme první kurz. Počítá se, neukládá. Historie má jeden bod na konci každého měsíce, u aktuálního měsíce k dnešku. Pozice brokera se oceňují kusy drženými ke dni a poslední zavírací cenou do toho dne. Záporný zůstatek běžného účtu je dluh.
- **Vklady a výběry** na platformu jsou vlastní převody podle ADR 206: protiúčet platformy nebo zapamatovaná protistrana. U Fingoodu jsou jediným zdrojem vkladů a výběrů, protože pohyby platforma neexportuje. Z nich a z ocenění se počítá **výnos XIRR** a porovnává se s úrokem, který platforma uvádí.

**Fingood:**
- Parser `fingood-investments` pro XLSX „Moje investice“. Pozná ho podle hlavičky (`Číslo úvěru`, `Název investice`, `Původní jistina`…), sloupce čte podle názvu.
- Import uloží **snímek pozic** `p2p_position` (účet, den snímku, číslo úvěru, název, způsob splácení, nesplacená jistina, úrok p.a., příští a poslední splátka, stav, stav splácení, zpoždění, po splatnosti, kumulativní úroky, sankční úroky, poplatky). Tabulka má RLS a `import_batch_id`, takže snímek pokryje i rollback importu.
- **Den snímku** je `dcterms:created` převedené na datum v Praze. Soubor bez něj se odmítne, den importu by snímek posunul. Náhled ho ukáže.
- **Nesplacená jistina** = `Původní jistina` + `Nakoupeno jistiny na Marketu` − `Prodáno jistiny na Marketu` − `Vráceno jistiny`. Součet za snímek je ocenění účtu.
- Úvěry po splatnosti nebo se `Stav splácení` jiným než „V pořádku“ se ukážou jako **ohrožená jistina**. Aplikace jejich hodnotu sama nesnižuje, odhad vymožení nedělá.
- **Výnos za období** = rozdíl kumulativních úroků a sankčních úroků minus poplatků mezi dvěma snímky. První snímek dává součty za celou dobu.
- **Splatnostní žebříček:** kolik jistiny se vrátí v kterém měsíci podle `Termín poslední splátky`. Balonové úvěry vracejí jistinu až na konci, takže je to skutečná likvidita platformy.
- Idempotence: klíč pozice je (účet, den snímku, číslo úvěru). Stejný soubor podruhé nic nezaloží.

**Hypotéka:**
- **`loan_terms`** (účet, platí od, zůstatek k tomu dni, sazba p.a., splátka, den splátky, splatnost, konec fixace). Nová sazba po fixaci je nový řádek s novým „platí od“ a vlastním zůstatkem.
- Zůstatek se počítá z kalendáře od posledního známého bodu. Ruční ocenění (`account_valuation`, například zůstatek z ročního výpisu banky) kalendář opraví a počítá se dál od něj. Úrok splátky je dvanáctina roční sazby ze zůstatku (30/360). Splátka, která v měsíci splatnosti nepřišla, se do dneška počítá jako zaplacená podle kalendáře, pozdější jsou plán.
- **Splátka z banky** (už importovaná transakce) se stane splátkou úvěru (`loan_payment`: transakce, účet, úrok, jistina, `by_schedule`). Když má úvěr číslo účtu a podmínky, platby na něj rozpozná ADR 206 jako vlastní převod a ty se převedou na splátky samy (`loan.Service.Reconcile` po každé změně převodů, podmínek nebo zůstatků). Platby v měsíci splatnosti platí tu splátku, první nese úrok, další jsou jistina (mimořádná splátka). Uživatel může rozdělení zadat sám. Zrušená splátka se zapíše do `not_transfer`, aby se nevrátila.
- Úrok je výdaj v kategorii „Úroky z hypotéky“ (u spotřebitelského úvěru „Bankovní poplatky a úroky“). Jistina jde do „Splátky úvěrů“ (druh `financial`), takže není výdajem, a transakce přestane být převodem. Řádky kategorií mají zdroj `loan`: klasifikátor se z nich neučí, jinak by úroky z hypotéky navrhoval i jiným platbám.
- **Rezerva hypotéky (offset, např. Chytrá rezerva Air Bank):** účet `loan_reserve` patřící k hypotéce (`parent_id`, nejvýš jeden). Zakládá se na stránce hypotéky (`POST /loans/{id}/reserve`), převezme banku hypotéky. Zůstatek = zůstatek z výpisu (`account_valuation`) + převody do a z rezervy od té doby (způsob ocenění `flows`). Úrok splátky se počítá ze zůstatku dluhu minus rezervy den před splatností, splátka zůstává, takže se víc splácí jistina. Přehled ukazuje, kolik rezerva ušetřila a o kolik dřív bude hypotéka doplacena. Rezerva se v čistém jmění počítá jako peníze dostupné ihned, hypotéka dál jako celý dluh.
- **Rozpoznání podle typu úhrady:** banka, která u splátky neuvádí protiúčet (Air Bank: „Splátka hypotéky“, „Úhrada do rezervy“, „Výběr z rezervy“), se páruje podle typu úhrady na hypotéku nebo rezervu se stejnou bankou (`own_transfer.matched_by = 'operation'`, seznam v `transfer/own.go`). Jiné banky se párují podle čísla účtu hypotéky nebo rezervy, případně ručně.
- **Nemovitost** je účet `property` s ručním oceněním, první hodnota je kupní cena. Přehled ukáže vlastní kapitál = hodnota nemovitosti − zůstatek hypotéky.
- Odpočet do konce fixace a simulace mimořádné splátky (o kolik klesne splátka nebo se zkrátí doba splácení).
- **Daňový přehled úroků:** úroky zaplacené za kalendářní rok a limit odpočtu 150 000 Kč, u bydlení financovaného před rokem 2021 300 000 Kč (§ 15 odst. 3 ZDP). Rozhoduje den prvních zadaných podmínek. Nemovitost je vlastní bydlení, takže odpočet se na ni vztahuje. Pronajímanou nemovitost aplikace neřeší. Je to přehled, ne daňové přiznání.

**Přehled jmění** (web `/net-worth` a detail účtu `/net-worth/{účet}`, v menu skupina Majetek; MCP `get_net_worth`, `get_loan`, `get_p2p_loans`):
- čisté jmění v čase, aktiva a závazky,
- rozložení podle druhu aktiva a likvidity (ihned, do pár dnů, vázané),
- rezerva v měsících = likvidní hotovost / průměrné měsíční výdaje za 12 měsíců,
- měsíční změna jmění rozdělená na úspory (příjmy − výdaje měsíce) a zbytek (výnosy, přecenění, splátky dluhů oceněných kalendářem).

**Mimo rozsah:** roční výpis Fingoodu, importér Investownu, dokud nebude ověřený export (do té doby ruční ocenění), automatické ocenění nemovitosti podle cenového indexu, krypto a komodity (dál podle ADR 209).

**Důsledky:**
- Splátka úvěru není v `transfer_leg`. Výdajové přehledy ji rozdělí podle řádků kategorií jako každou rozdělenou transakci. Detekce vlastních převodů (ADR 206) transakce se splátkou přeskakuje.
- Vklady a výběry pro XIRR jsou jen převody z vlastních účtů. Historie jmění i výnos jsou tak úplné, jak úplné jsou naimportované výpisy.
- Ocenění Fingoodu nezahrnuje volné prostředky na platformě. Kdo je tam drží, založí si pro ně samostatný účet s ručním oceněním.
- Sloupce Marketu jsou v ověřeném exportu všude nulové, takže vzorec nesplacené jistiny pro nákupy a prodeje na Marketu není ověřený. Stejně tak význam sloupce `Přislíbeno`. **[OVĚŘIT na exportu s obchodem na Marketu]**
- `Stav` a `Stav splácení` jsou číselníky Fingoodu a známe jen jejich část. Neznámá hodnota se uloží beze změny a počítá se jako ohrožená.
- Hodnota mezi snímky je schodovitá. Jmění u P2P je tak přesné, jak často uživatel exportuje. Přehled ukáže stáří posledního ocenění.

