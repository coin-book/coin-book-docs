# 205 – Systémové a uživatelské kategorie

**Status:** přijato, 17. 10. 2026

**Kontext:** Uživatel si potřebuje přidávat vlastní kategorie a měnit vzhled výchozích. Výchozí taxonomie (migrace 002) je ale společný katalog: kódy používají nápovědy bank a coin-book CSV a překlady jsou česky i slovensky. Kdyby ji uživatel přepisoval, nešlo by ji dál aktualizovat migracemi.

**Rozhodnutí:**
- **Systémové kategorie** (`category`) se nemění. Nejde je přejmenovat, změnit jim typ, přesunout ani smazat.
- Barvu, ikonu a rozpočtovou skupinu systémové hlavní kategorie si uživatel nastaví **jen pro sebe** v tabulce `category_preference`. Systémový řádek zůstává tak, jak přišel z migrace.
- **Uživatelské kategorie** (`user_category`, s `user_id`) mají jeden název bez překladů, protože je uživatel pojmenovává ve svém jazyce. Mohou být hlavní, nebo podkategorií systémové i vlastní hlavní kategorie. Mazat jde jen je. Používanou kategorii jde smazat jen sloučením do jiné.
- Tabulky `category_assignment`, `rule` a `budget` odkazují dvěma cizími klíči, `system_category_id` a `user_category_id`, a CHECK hlídá, že je vyplněný právě jeden. Generovaný sloupec `category_id` je jejich `coalesce`, takže čtení pracuje s jedním id. Zapisuje se do obou sloupců podle toho, ve které tabulce id leží.
- Pohled `any_category` spojuje systémové kategorie s uživatelovým vzhledem a uživatelské kategorie. Čte ho API i výpočty. Pohled sám uživatele nefiltruje. Od migrace 008 (ADR 208) běží s právy volajícího (`security_invoker`), takže řádková bezpečnost na `user_category` a `category_preference` mu nechá jen kategorie a vzhled přihlášeného uživatele.
- API vrací jeden seznam kategorií s příznakem `custom`.

**Důsledky:** Výchozí taxonomii jde rozšiřovat dalšími migracemi bez kolize s uživatelskými daty. Za to se platí složitějšími zápisy, které musí id rozdělit do dvou sloupců (`categorization/repo.go`, `taxonomy.go`, `budget/budget.go`).
