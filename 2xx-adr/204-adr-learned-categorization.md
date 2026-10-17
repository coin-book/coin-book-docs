# 204 – Učící se kategorizace

**Status:** přijato, 17. 10. 2026

**Kontext:** Kategorizace stála na pravidlech, pevném mapování kategorií mBank a četnosti kategorií u přesně stejné protistrany. Nové obchodníky a banky bez vlastních kategorií (Revolut) nepokrývala.

**Rozhodnutí:**
- Lokální multinomický naivní Bayes v Go, žádná data neopouštějí počítač (N-1). Je malý, přetrénování na 5 000 transakcích trvá desetiny sekundy a dá se vysvětlit slovy, která k návrhu vedla.
- Váhy příkladů: ruční zařazení 3, pravidlo 2, kategorie banky 1. Vlastní odhady modelu (`source = 'model'`) se do učení nepočítají, aby se model nepotvrzoval sám.
- Jistota = posteriorní pravděpodobnost × e / (e + 2), kde e je váha slov transakce viděných v dané kategorii. Bez kalibrace dával model 99 % u nového obchodníka jen podle typu operace a částky.
- Automatické zařazení je volitelné (Nastavení, výchozí vypnuto). Zařazené transakce mají zdroj `model`, objeví se v K vyřešení a potvrzují se jednotlivě nebo hromadně. Pravidla mohou automatické zařazení přepsat, ruční volbu ne.
- Po třetím ručním zařazení stejné protistrany do stejné kategorie aplikace nabídne pravidlo.
- Velké jazykové modely se nepoužívají, protože by popisy transakcí odcházely ven.

**Měření** (reálný export mBank, 6 639 řádků, 80/20, štítky z kategorií banky, `TestClassifierOnRealExport`):

| Varianta | Top 1 | Top 3 | Jistota ≥ 90 %: pokrytí / přesnost |
|---|---|---|---|
| s kategorií banky | 97,5 % | 98,1 % | 87,0 % / 97,9 % |
| jen text (jako Revolut) | 89,0 % | 94,1 % | 79,1 % / 96,1 % |
| dřívější přesná shoda protistrany | | | 83,4 % / 94,7 %, nové protistrany 0 % |
