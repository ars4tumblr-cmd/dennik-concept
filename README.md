# Denník Concept

> **Stav: FUNKČNÝ LOKÁLNY PROTOTYP / NIE PRODUKCIA**
> Verejný opis výrobného denníka a dashboardov pre obrábaciu výrobu. Implementácia, testovacie katalógy a Docker konfigurácia sú vedené oddelene.

## Cieľ a základný princíp

Cieľom je nahradiť stav, keď jeden Excel súbor súčasne slúži ako vstupný formulár, databáza, história, kontrola chýb aj dashboard.

```text
Udalosti = jediný zdroj pravdy
             ↓
     deterministický prepočet
             ↓
┌────────┬────────┬───────────┬─────────────┬────────────┐
│ Denník │ Stroje │ Operátori │ Zoraďovači  │ Súčiastky │
└────────┴────────┴───────────┴─────────────┴────────────┘
             ↓
        Kontrola dát
```

Používateľ nemení dashboard ani aktuálny stav priamo. Zapíše udalosť a všetky pohľady sa vypočítajú z tej istej histórie.

## Overený stav prototypu

Lokálny Frappe prototyp už beží cez Docker a MariaDB so syntetickými dátami. Overený je celý priechod:

```text
dynamický formulár → Udalosť v databáze → pravidlá → Denník a dashboardy
```

Prototyp obsahuje nemenný denník udalostí, dynamický formulár, automatické časové segmenty, päť hlavných pohľadov, korekcie bez straty histórie, kontrolu vlastníctva stroja, prechod cez polnoc a syntetické databázové testy.

## Udalosť a odvodený segment

Udalosť je jeden fakt v konkrétnom čase, napríklad prevzatie stroja, začiatok nevýroby, kontrolný rez alebo zmena výrobného kontextu. Ukladá sa s úplným timestampom, ID požiadavky, typom a štruktúrovanými údajmi.

Časový úsek `Od–Do` nie je primárny ručne upravovaný záznam. Systém ho odvodí z následnosti udalostí. Nová relevantná udalosť uzavrie predchádzajúci segment a otvorí ďalší. Pri rovnakom čase rozhoduje stabilné poradie zápisu.

Interný výpočet používa UTC; používateľ vidí lokálny čas. Nočná zmena preto zostáva jednou súvislou osou aj pri prechode cez polnoc. Stav, ktorý začal pred filtrom, sa prenesie do zobrazenia so zachovaným skutočným začiatkom.

## Dynamický formulár

Prvá voľba určuje jednu zo šiestich vetiev:

1. `Operátor`
2. `Zoraďovač`
3. `Stroj / nevýrobný stav`
4. `Výrobný výsledok`
5. `Predák / riadenie zmeny`
6. `Korekcia`

Formulár zobrazuje iba potrebné polia. Používateľa, čas zápisu, ID a známy kontext doplní systém. Stabilné hodnoty sa vyberajú z riadených zoznamov; voľný text zostáva iba poznámkou. Automatický dôsledok jednej udalosti sa nezapisuje druhýkrát ako ďalšia používateľská udalosť.

## Hlavné pravidlá

### Vlastníctvo stroja

- Jeden stroj môže mať v danom okamihu najviac jedného výhradného vlastníka: operátora alebo zoraďovača.
- Prevzatie zoraďovačom atomicky ukončí priradenie operátora, uzavrie jeho výrobný segment a otvorí nastavovanie.
- Odovzdanie môže priamo určiť ďalšieho operátora alebo zoraďovača bez neplatného medzistavu.
- Výmena skupiny strojov je jedna transakcia: buď prejde celá, alebo sa neuloží nič.
- Fyzický stav stroja a vlastníctvo sú oddelené. Samotné odovzdanie nevymýšľa fyzický prestoj.

### Stav stroja

Najvyššia úroveň má iba `VYRÁBA` alebo `NEVYRÁBA`. Kategória a dôvod nevýroby sú samostatné atribúty. Zmena dôvodu počas nevýroby rozdelí segment, hoci hlavný stav zostáva rovnaký.

Obnovenie výroby vyžaduje platný výrobný kontext a priradeného operátora. Nastavovanie, blokujúca OTK alebo porucha môžu vytvoriť `NEVYRÁBA` automaticky.

### Výrobný kontext a uzatvorenie segmentu

Výrobný segment patrí kombinácii:

```text
Operátor + Stroj + Zákazka + Súčiastka + Operácia + Norma
```

Segment sa uzavrie pri odovzdaní alebo odobratí stroja, prevzatí zoraďovačom, zmene zákazky/súčiastky/operácie, potvrdení konca aktuálneho úseku alebo na konci zmeny. Zmena kontextu načíta schválenú normu pre `Súčiastka + Operácia`; história si zachová normu použitú pri svojom vzniku.

### Kontrolný rez a plnenie normy

`Kontrolný rez` vytvorí provizórny bod výkonu. Neuzavrie segment ani priradenie a nesmie sa neskôr započítať druhýkrát. V prototype zadáva množstvo od začiatku aktuálneho segmentu; automatické CNC počítadlo a jeho reset/baseline zatiaľ nie sú súčasťou riešenia.

Potvrdenie množstva patrí celému uzavretému výrobnému segmentu:

```text
Plnenie segmentu = OK / (trvanie v hodinách × norma za hodinu) × 100 %
```

Kumulatívne plnenie operátora je:

```text
Σ potvrdených OK kusov / Σ normovaného množstva × 100 %
```

Priemer percent segmentov sa nepoužíva. Orezanie časovým filtrom množstvo pomerne nerozpočítava a nevymýšľa nepotvrdené údaje.

### Zaťaženie operátora

Zaťaženie je nezávislé od výrobnej normy. Je súčtom koeficientov aktuálne pridelených strojov:

- `11,1 %` — plne automatizovaná obsluha;
- `20 %` — automatizovaná obsluha citlivých dielov;
- `100 %` — ručné zakladanie a odoberanie;
- iný koeficient iba ako schválená referenčná hodnota.

Operátor môže mať viac strojov a súčet môže presiahnuť 100 %. Krivka sa mení iba udalosťou priradenia, odovzdania, výmeny skupiny alebo zmeny režimu.

### Zoraďovač, OTK a práca mimo stroja

Zoraďovač môže byť legitímne obsadený nastavovaním, donastavením, OTK laboratóriom, prípravou alebo presunom. Kontrola prvého kusa patrí zoraďovačovi, nie operátorovi.

Blokujúca OTK zastaví výrobu; neblokujúca kontrola môže prebiehať bez zastavenia stroja. Výsledok `OK / NOK / podmienečne OK` určí ďalší krok.

`Dopyt > 0` bez zaznamenanej činnosti vytvára nevysvetlené okno na preverenie, nie automatický rozsudok o chybe človeka. Voľno pri nulovom dopyte je normálny stav.

## Finálne pracovné pohľady

Hlavné pohľady sa na bežnom desktopovom pracovisku zmestia na jednu obrazovku bez horizontálneho posúvania. Pri väčšom počte záznamov roluje iba vnútorná plocha tabuľky.

### Denník

Jeden riadok je odvodený časový segment. Presný poriadok:

```text
Dátum | Zmena | Stroj | Súčiastka | Zákazka | Operácia | Meno |
Od | Do | Stav | Norma/h | Splnené | NOK | Dôvod / Kontext
```

Zmena je `R` alebo `N`; stĺpce `Stroj–Operácia` používajú dohodnuté farebné rozlíšenie. Denník nie je priamo editovateľný zdroj dát.

### Stroje

- tri kompaktné skupiny `FRÉZY / SÚSTRUHY / OSTATNÉ`, pripravené približne na 30 strojov;
- KPI `Výroba v období`, `Nevýroba v období`, `Stav na konci`, `Skutočný začiatok`;
- jedna reálna časová os, modré `VYRÁBA`, červené `NEVYRÁBA`, body udalostí a prerušovaný prenesený stav;
- hover s intervalom, stavom, dôvodom, osobou, kontextom, skutočným začiatkom a zdrojom;
- história `Od / Do / Stav / Dôvod / Osoba / Kontext`.

### Operátori

- prepínanie medzi operátormi aktívnymi vo vybranej zmene;
- KPI `Priemerné zaťaženie`, `Kumulatívne plnenie` a potvrdené množstvo;
- jeden graf so stupňovitou krivkou zaťaženia, krivkou plnenia a pásmom `95–105 %`;
- `●` potvrdený segment, `○` kontrolný rez, `▲` zmena priradenia alebo režimu;
- hover zaťaženia ukazuje aktuálne stroje a koeficienty;
- hover plnenia ukazuje uzavreté stroje, individuálne percentá, kumulatívny výsledok a zdroj;
- dole je kompaktná páska udalostí namiesto technickej tabuľky.

Každý bod musí mať konkrétnu udalosť ako zdroj. Bez novej výrobnej informácie sa nový bod výkonu nevytvára.

### Zoraďovači

- prepínanie medzi zoraďovačmi aktívnymi vo vybranej zmene;
- KPI `Obsadenosť`, `Max. dopyt`, `Nevysvetlené okno`;
- jeden spoločný graf `Dopyt na zásah` verzus `Obsadenosť`;
- hover ukazuje konkrétne stroje, činnosť, kontext a zdroj;
- dole je kompaktná páska významných udalostí.

### Súčiastky

Operatívny skutočný stav z udalostí, nie druhý plánovací systém:

```text
Priorita | Aktuálny stroj | Súčiastka | Zákazka | Operácia | Stav |
Objednané | Hotovo | Zostáva | Sklad | Pri stroji | Blokácia / Kontext
```

Prezentačné stavy zahŕňajú `VÝROBA`, `ČAKÁ NA ZORAĎOVAČA`, `ČAKÁ NA MATERIÁL`, `OTK`, `PORUCHA`, `HOTOVO`, `PRIPRAVENÉ`, `PLÁNOVANÉ`. Kým nie sú pripojené skladové pohyby, `Sklad` a `Pri stroji` zostávajú neznáme.

### Udalosti a kontrola dát

Technický pohľad udalostí slúži na audit zdrojov. Kontrola dát ukazuje miesta vyžadujúce potvrdenie alebo vysvetlenie, napríklad nepotvrdený uzavretý výrobný segment.

## Korekcie bez straty histórie

Pôvodný záznam sa neupravuje ani nemaže. Oprava pridá náhradný fakt; storno pridá udalosť, ktorá pôvodný fakt vyradí z efektívneho prepočtu. Potom sa prehrá celá účinná história. Ak by výsledkom bol konflikt alebo neplatný stav, korekcia sa odmietne.

## Technologický základ

- Frappe Framework a vlastná aplikácia;
- MariaDB;
- Docker Compose s trvalými databázovými volumes;
- lokálny prototyp na `localhost`;
- syntetické katalógy a testovacie udalosti.

## Hranice prvého prototypu

Zámerne nie sú zahrnuté TimeLine integrácia, automatické množstvá z CNC, skladové pohyby/WIP, payroll, telemetry/R&D, produkčné zabezpečenie, LAN ani verejné nasadenie.

Tieto oblasti môžu pribudnúť až po overení pravidiel na schválených reálnych katalógoch, normách a reprezentatívnych výrobných scenároch.

## Verejná a privátna časť

Tento verejný repozitár obsahuje iba všeobecnú textovú koncepciu. Nesmie obsahovať reálne výrobné údaje, osobné údaje, exporty, heslá, `.env`, databázy, zálohy ani interné sieťové údaje.

Pracovná implementácia `dennik-app` je samostatný privátny repozitár s Frappe kódom, Docker konfiguráciou, DocTypes, validačnou logikou, syntetickými dátami a technickými testami.

## Ďalší krok

Najbližší krok už nie je návrh základných dashboardov. Funkčný prototyp treba overiť na schválenom zozname strojov, ľudí, zákaziek, operácií, noriem a dôvodov prestojov. Až potom má zmysel plánovať TimeLine integráciu a interné nasadenie.

---

**FUNKČNÝ PROTOTYP — not production ready**
