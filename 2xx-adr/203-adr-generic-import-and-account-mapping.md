# 203 – Obecný import a mapování účtů

**Status:** přijato, 17. 10. 2026

**Kontext:** Import zakládal účty automaticky podle čísla nebo reference z výpisu. Pro banky bez parseru chyběl vstup a uživatel neměl jak výpis přiřadit k existujícímu účtu ani opravit omylem založený duplicitní účet.

**Rozhodnutí:**
- Formát **coin-book CSV** je třetí parser vedle mBank a Revolutu a mapuje se do stejného normalizovaného výpisu. Stačí CSV, protože ho umí uložit každá tabulka; XLSX se nepodporuje.
- Náhled importu navrhne cíl pro každý účet z výpisu a uživatel ho může změnit. Ruční přiřazení bankovního účtu se uloží jako alias a další importy ho použijí. Soubor bez identity účtu vyžaduje výběr.
- **Sloučení účtů** přesune transakce do cílového účtu stejné měny, sečte počáteční zůstatky, převody mezi oběma účty zruší a smaže zdrojový účet. Identita zdrojového bankovního účtu se uloží jako alias na cíl, takže další importy jdou rovnou tam. Ruční účty alias nezakládají.
- ID transakcí se dál odvozují z identity bankovního účtu, takže přemapování ani sloučení nemění odhalení duplicit.

## Doplněk: číselník bank (17. 10. 2026)

- Tabulka `bank` (migrace 004) s id `cz-<kód>`, `sk-<kód>` a slugy fintechů (`revolut`, `wise`, …). Zdroje: CSV kódů platebního styku ČNB a převodník identifikačních kódů pro tuzemský platobný styk SR platný od 18. 5. 2026.
- Banka se odvozuje z kódu za lomítkem českého čísla účtu nebo z kódu v CZ/SK IBAN; zbytek vybírá uživatel. Logika je v Go (`domain/bank.CandidateID`) a její dvojče ve webu (`taxonomy.bankOfNumber`).
- Číselník se mění jen migrací, aby byl stejný ve všech instalacích a v testech (testy ho při čištění databáze nemažou).
