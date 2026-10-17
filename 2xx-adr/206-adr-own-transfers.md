# 206 – Převody na vlastní účty bez výpisu

**Status:** přijato, 17. 10. 2026

**Kontext:** Párování převodů (ADR 202) potřebuje obě strany, tedy výpis z obou účtů. Uživatel ale často importuje jen běžný účet. Peníze poslané na spoření, k brokerovi nebo na Revolut pak vypadaly jako výdaj nebo jako „Spoření a investice“ a zkreslovaly přehled.

**Rozhodnutí:**
- **Jednostranný převod** (`own_transfer`, migrace 007): transakce, která posílá peníze na vlastní účet nebo z něj, i když druhý účet nemá import. Vlastní účet může být ruční účet jen s číslem.
- **Rozpoznání podle čísla protiúčtu:** `transfer.DetectOwn` porovnává protiúčet s čísly vlastních účtů v jednotném tvaru (`bank.CanonicalNumber`: bez nul na začátku, z IBAN). Běží po každém importu nad všemi transakcemi a po založení účtu, takže platí i zpětně.
- **Ruční označení** (`PUT /transfers/{tx}/own`) slouží pro platby bez čísla účtu, např. dobití kartou. Volitelně si zapamatuje protistranu (`own_transfer_rule`).
- **Vrácení:** zrušení jednostranného převodu se uloží do `not_transfer`, aby ho rozpoznání podle čísla znovu neoznačilo.
- **Doplnění na pár:** když se později naimportuje výpis druhého účtu, automatické párování najde i transakce s jednostranným převodem. Pár jednostranný záznam nahradí.
- Všechna místa, která převody vylučují (výdaje, příjmy, rozpočet, fronta k zařazení, klasifikátor), čtou pohled `transfer_leg`, který spojuje páry a jednostranné převody.
- **Návrhy vlastních účtů:** protiúčty, které banka sama označuje za vlastní (mBank „VLASTNÍ PŘEVOD“, „PŘEVOD NA/Z MSPOŘENÍ“, kategorie „Převod mezi vlastními účty“), a které coin-book ještě nezná.
- **Přehled:** Přehled ukazuje převody na spořicí účty (spoření, kapsa, fond, broker) jako „Převedeno na vlastní účty“, samostatně a mimo výdaje.

**Důsledky:** Vyhledání účtu podle čísla při importu porovnává i jednotný tvar, takže výpis účtu založeného z návrhu se naimportuje do něj a nezaloží duplikát.
