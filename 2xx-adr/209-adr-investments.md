# 209 – Investice: akcie a ETF z Revolutu, ceny z Yahoo, test 3 let

**Status:** přijato, 18. 10. 2026

**Kontext:** Fáze 2 (F-INV-1 až F-INV-3) přidává investice. Prvním zdrojem je konsolidovaný výpis Revolutu, který už importujeme kvůli běžným účtům. Jeho investiční sekce parser zatím přeskakuje. Ověřeno na reálném výpisu, v repozitáři je jen struktura:
- Akciový účet a Robo-Advisor jsou dva účty se stejnými bloky: „Iné transakcie na brokerskom účte“, „Predané jednotky“ a dividendy.
- „Iné transakcie“ nemají typ operace, ale jsou to **nákupy**. Žádný řádek nepřipadá na den prodeje, hodnoty jsou kladné a část řádků sedí na datum nákupu prodaných lotů.
- „Predané jednotky“ jsou prodeje **po lotech**. Každý řádek má datum nákupu i prodeje, počet kusů, ceny a kapitálový zisk. Všechny hodnoty jsou v měně obchodu i v CZK. Výběr lotu dělá Revolut.
- Výpis pokrývá jen zvolené období. Prodané loty můžou pocházet z nákupů mimo něj.
- Robo-Advisor drží zlomky kusů s přesností až na 8 desetinných míst.
- Nákupy a prodeje mají anglický formát čísel (`US$1,234.56`, `+€9.99`), dividendy slovenský (`12,34$`).

Uživatel chce vidět pozice a jejich hodnotu, svíčkové grafy jednotlivých titulů a časový test pro osvobození od daně (držba nad 3 roky). IBKR přijde později přes Flex Query a musí se vejít do stejného modelu.

**Rozhodnutí:**
- **Účty:** investiční účet Revolutu a Robo-Advisor jsou účty typu `broker` (`revolut:broker`, `revolut:robo`). Převody „Na investičný účet“ z běžného účtu jsou tak vlastní převody podle ADR 206. Měnu nese každý investiční záznam sám. Měna účtu je jen výchozí pro zobrazení.
- **Sdílená tržní data bez `user_id`:** `instrument` (ISIN jako klíč, ticker, název, odhad `yahoo_symbol`), `price_series` (symbol, měna kotace, dokdy staženo), `price_bar` (symbol, den, OHLCV) a `fx_rate` (den, měna, kurz ČNB). Jsou veřejná a stejná pro všechny, takže jsou výjimkou z ADR 208 spolu s `agent_token` a `audit_log`. Akceptační test na RLS je vyjmenuje.
- **Ve sdílených datech není nic, co by prozradilo, kdo co drží nebo odkdy.** Registrace je otevřená (ADR 208), takže jiný uživatel může nahrát podvržený výpis.
  - Instrument se zakládá s prvním importem. Další importy jen doplní prázdný ticker nebo název, nic nepřepíšou.
  - ISIN musí mít platnou kontrolní číslici. Ticker a název mají omezenou délku a tvar.
  - Měnu instrumentu si každý uživatel bere ze svých obchodů. Hodnota pozice se počítá v měně kotace.
  - Vlastní symbol Yahoo má každý uživatel zvlášť (`instrument_symbol` s RLS). Sdílený je jen odhad z prvního importu.
  - Nový symbol se stahuje od pevného dne 1. 1. 2015, ne od prvního obchodu, takže délka historie cen nic neprozradí.
  - Zbývá: ticker, název a odhad symbolu určuje první import. Kdo naimportuje ISIN jako první s cizím tickerem, ostatním se titul ocení podle něj, dokud si nenastaví vlastní symbol. Že titul už někdo drží, prozradí jen to, že název nebo ceny jsou tam dřív.
  - Jeden uživatel může nechat stahovat nejvýš 300 symbolů. Kdo si založí víc účtů s vymyšlenými tituly, obnovu zdrží, ale Yahoo se pořád ptáme nejvýš dvakrát za sekundu. Ceny pak zastarávají, nic horšího se nestane.
- **Data uživatele s RLS:**
  - `investment_tx` (`buy` a `dividend`, později z IBKR i `sell`, `fee`, `split`) a `investment_lot_sale` (prodaný lot).
  - Obě tabulky mají `import_batch_id`, takže je pokryje rollback importu.
  - Částky jsou v minor jednotkách v měně obchodu a vedle toho v CZK, pokud je zdroj uvádí. Daňový pohled tak používá kurz z výpisu a nepotřebuje vlastní.
  - Počet kusů je `numeric(28,10)`, v Go `decimal.Decimal`, nikdy `float`.
- **Idempotence:** klíč nákupu je (účet, datum, ISIN, kusy, hrubá hodnota). Klíč prodaného lotu je (účet, datum prodeje, datum nákupu, ISIN, kusy, hodnota prodeje). K oběma se přidává pořadí v rámci shodného klíče, stejně jako u transakcí. Překrývající se exporty proto nezakládají duplicity.
- **Pozice a otevřené loty se počítají, neukládají.** Pozice je součet nakoupených kusů minus součet prodaných kusů. Otevřené loty vzniknou z nákupů, od kterých se odečtou prodané loty se stejným ISIN a datem nákupu. Když prodaný lot nemá odpovídající nákup, protože historie začíná později, ukáže se pozice jako „neúplná historie“ s výzvou naimportovat starší výpis.
- **Test 3 let:**
  - Prodaný lot je osvobozený, když mezi datem nákupu a prodeje uplynuly víc než 3 roky. Lhůta končí stejným dnem, v kratším měsíci jeho posledním dnem (§ 605 OZ). Nákup 29. 2. 2020 je tak osvobozený od 1. 3. 2023.
  - Otevřené loty ukazují datum, kdy test splní.
  - Realizované zisky se sčítají po letech prodeje, rozdělené na osvobozené a zdaňované. U zdaňovaných se zobrazí i hrubý příjem z prodejů za rok kvůli limitu 100 000 Kč.
  - Je to přehled, ne daňové přiznání. Limit 40 mil. Kč a další výjimky aplikace neřeší.
- **Ceny z Yahoo** stahuje `internal/platform/marketdata`, jediný balíček, který sahá na síť. Doména `investment` ho volá přes rozhraní:
  - Adaptér je převzatý z `trading3` (`internal/domain/ingest/prices/yahoo.go`). Volá `GET query1.finance.yahoo.com/v8/finance/chart/{symbol}?interval=1d` bez crumb a cookie s vlastním User-Agentem, nejvýš 2 požadavky za sekundu a jedním opakováním s respektováním `Retry-After`. 404 znamená „žádná data“.
  - Bere jen denní svíčky.
  - `yahoo_symbol` se odvodí z tickeru: obchody v USD zůstávají bez přípony, v EUR dostanou příponu Xetry (`LYTR` → `LYTR.DE`). Uživatel si ho může opravit pro sebe. Bez symbolu instrument nemá graf ani aktuální cenu a ukáže se nákupní.
  - Stahují se jen symboly, které některý uživatel drží. Plánovač je čte postupně v rozsahu každého uživatele a ven pustí jen symboly.
  - Stahuje se po importu s novými záznamy, jednou denně po zavření amerického trhu a tlačítkem „Aktualizovat ceny“. Tlačítko do minuty od poslední obnovy nic neudělá, obnova po importu nebo po změně symbolu ano.
  - Plánovač je jedna gorutina v API s `clock.Clock`, bez externího cronu.
  - Kotace v pencích (`GBp`) nemá kód ISO, takže pozici neocení.
- **Kurzy ČNB** (`kurzy.txt`, denně, stejný plánovač) slouží jen k přepočtu aktuální hodnoty portfolia do CZK a pro celkový majetok (F-INV-2).
- **Web:** stránka `/investice` s pozicemi (hodnota, nerealizovaný P/L, dividendy), loty s odpočtem do splnění testu 3 let a realizovanými zisky po letech. Detail titulu má svíčkový graf s vyznačenými nákupy a prodeji. LayerChart svíčky nemá, takže graf je vlastní SVG (`CandleChart.svelte`).
- **Mimo rozsah:** krypto a komodity z výpisu Revolutu. Parser je dál přeskakuje.

**Důsledky:**
- Plný přehled potřebuje jednorázový import výpisu od založení účtu, další importy můžou být průběžné.
- IBKR dává prodeje jako obchody, ne jako loty. Až přijde, aplikace bude loty párovat sama (FIFO) a výsledek zapíše do `investment_lot_sale`, takže zbytek modelu zůstane beze změny.
- Yahoo nemá oficiální API a může se změnit nebo začít blokovat. Výpadek ale ovlivní jen grafy a aktuální hodnotu, ne importovaná data ani daňový pohled.
- Revolut parser dostane převod čísel podle sekce. Formát a pravidla jsou v `importer-formaty.md`.
- Revolut parser má verzi 2. Soubor už naimportovaný verzí 1 jde nahrát znovu, aby se z něj načetly investice. Transakce se podle id neduplikují. Kdo pak vrátí původní import verze 1, smaže i jeho transakce. Soubor zůstane označený jako naimportovaný verzí 2, dokud nevrátí i ten.
- Kotace v jednotkách menších než měna se třemi velkými písmeny (izraelské agoroty `ILA`) by se ocenily stokrát výš. Aplikace je nerozpozná, revolutí tituly je nemají.
- Pozice s loty v různých měnách nemá nákupní cenu v původní měně, jen v CZK.
- Ceny z Yahoo jsou očištěné o splity, počty kusů z výpisu ne. Po splitu nesedí hodnota pozice, dokud nepřibude druh záznamu `split`.
