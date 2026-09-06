# Denník Concept

> **Status: CONCEPT / PROTOTYPE**  
> Tento repozitár opisuje návrh lokálneho výrobného denníka a analytických dashboardov pre obrábaciu výrobu. Nejde o hotový produkčný systém.

## Cieľ

Cieľom je nahradiť situáciu, keď jeden Excel súbor súčasne slúži ako vstupný formulár, databáza, história udalostí, kontrola chýb aj dashboard.

Navrhovaný systém používa **jednu konzistentnú databázu udalostí** a nad ňou vytvára viac pohľadov podľa potreby používateľa.

```text
Zápis udalosti
      ↓
Jednotná databáza
      ↓
┌────────────┬──────────┬────────────┬─────────────┐
│ Denník     │ Stroje   │ Operátori  │ Zoraďovači  │
└────────────┴──────────┴────────────┴─────────────┘
      ↓
Kontrola dát / analýza / import externých údajov
```

## Prečo nie iba Excel

Excel zostáva výborným nástrojom pre analýzu a export, ale pri dlhodobom prevádzkovom denníku vznikajú typické problémy:

- rovnaký údaj sa zapisuje na viacerých miestach,
- voľný text vytvára množstvo rôznych pomenovaní tej istej udalosti,
- časové intervaly sa môžu prekrývať,
- história opráv sa ťažko kontroluje,
- dashboardy sú závislé od komplikovaných vzorcov,
- chybný zápis sa často odhalí až spätne,
- zošit sa časom stáva veľkým a krehkým.

Cieľom projektu preto nie je „lepší Excel“, ale **oddeliť dáta od ich zobrazenia**.

## Denník ako používateľský pohľad

Existujúci vizuálny princíp výrobného denníka nemusí zmiznúť.

Používateľ môže naďalej vidieť známy pohľad:

```text
Stroj | Operátor | Zákazka | Súčiastka | Operácia | Stav | Od | Trvanie
```

Rozdiel je v tom, že tento pohľad už nie je primárnym miestom uloženia dát. Generuje sa z udalostí uložených v databáze.

Tak je možné zachovať známy spôsob práce bez toho, aby tabuľka zostala zdrojom všetkých problémov.

## Základný dátový princíp

Jedna udalosť predstavuje jeden fakt v konkrétnom časovom intervale.

Príklady informácií, ktoré môžu byť súčasťou udalosti:

- stroj,
- osoba,
- rola osoby,
- začiatok a koniec,
- výrobná zákazka,
- súčiastka,
- operácia,
- stav stroja,
- kategória udalosti,
- dôvod,
- množstvo,
- doplnkový kontext.

V zdrojových dátach sa jednotlivé atribúty ukladajú samostatne. Kombinovaný text patrí až do dashboardu alebo reportu.

## Model stavu stroja

Na najvyššej úrovni sa používa jednoduchý stav:

- `VYRÁBA`
- `NEVYRÁBA`

Dôvod nevýroby nie je samostatný konkurenčný stav. Je to ďalší atribút udalosti, napríklad:

```text
NEVYRÁBA
Kategória: Nastavovanie
Dôvod: Výmena nástroja
```

Tým zostáva časová os stroja jednoduchá a zároveň sa nestráca podrobnosť.

## Navrhované dashboardy

### Stroje

Pohľad zameraný na priebeh stavu jednotlivých strojov:

- čas výroby,
- čas nevýroby,
- podiel výroby v sledovanom období,
- aktuálny stav,
- odkedy stav trvá,
- hlavné príčiny nevýroby,
- časová chronológia udalostí,
- zákazka / súčiastka / operácia.

### Operátori

Pohľad zameraný na prácu človeka v čase:

- prítomnosť v sledovanom období,
- obsluhované stroje,
- aktuálna činnosť,
- prechody medzi pracoviskami,
- časové prekryvy,
- vyťaženie,
- odchýlky a konflikty dát.

### Zoraďovači

Pohľad na technické zásahy a podporné činnosti:

- stroj,
- typ úlohy,
- začiatok a trvanie,
- počet zásahov,
- priemerné trvanie,
- rozdelenie podľa typu činnosti,
- história práce na strojoch.

### Súčiastky

Pohľad na priebeh výrobnej zákazky a operácie:

- objednané množstvo,
- vyrobené množstvo,
- zostávajúce množstvo,
- aktuálna operácia,
- aktuálny stroj,
- rozpracovanosť,
- blokácie,
- dostupnosť polotovarov.

### Kontrola dát

Samostatná vrstva na vyhľadávanie nekonzistentných údajov:

- prekryv činností jednej osoby,
- konflikt osoba × stroj,
- chýbajúce povinné údaje,
- neplatný časový interval,
- otvorená udalosť bez ukončenia,
- duplicity,
- opravené alebo zneplatnené záznamy.

## Externé dáta

Systém môže porovnávať vlastné výrobné údaje s exportmi z existujúcich podnikových systémov vo formáte XLSX alebo CSV.

Typický príklad:

```text
Výrobný denník       Externý export
        \               /
         \             /
          → Porovnanie ←
              ↓
    rozdiely / sklad / WIP /
    vyrobené / zostáva / chýba
```

Externý export sa chápe ako **snapshot v konkrétnom čase**, nie ako druhý nezávislý zdroj výrobnej reality.

## Technologický smer

Ako základ prototypu je navrhnutý:

- **Frappe Framework**
- MariaDB
- Docker
- lokálne alebo interné sieťové nasadenie
- import / export XLSX a CSV
- vlastné Frappe DocTypes a custom pages

Frappe Framework poskytuje hotové stavebné prvky ako:

- používateľov a autentifikáciu,
- role a oprávnenia,
- formuláre,
- zoznamy,
- reporty,
- auditovateľné záznamy,
- API,
- import/export,
- správu databázových objektov.

Vlastná aplikačná logika zostáva oddelená od samotného frameworku.

## Predpokladané nasadenie prototypu

Prvá verzia má byť spustiteľná lokálne cez Docker:

```text
Windows / Linux
      ↓
Docker
      ↓
Frappe + MariaDB
      ↓
http://localhost:PORT
```

V ďalšom kroku môže byť rovnaká aplikácia dostupná v internej sieti.

## Fázy vývoja

1. Navrhnúť dashboardy a požadované výstupy.
2. Z dashboardov odvodiť minimálny dátový model.
3. Vytvoriť základné DocTypes a validačné pravidlá.
4. Vytvoriť známy pohľad `Denník` nad databázou udalostí.
5. Implementovať dashboardy Stroje, Operátori a Zoraďovači.
6. Pridať kontrolu dát.
7. Pridať import externých XLSX/CSV snapshotov.
8. Overiť prototyp na testovacích dátach.
9. Až potom riešiť produkčné nasadenie a integrácie.

## Bezpečnostný a dátový princíp

Tento verejný repozitár obsahuje iba **popis konceptu**.

Nemá obsahovať:

- reálne výrobné dáta,
- exporty podnikových systémov,
- osobné údaje pracovníkov,
- heslá alebo `.env` súbory,
- databázové zálohy,
- interné adresy a sieťové údaje,
- produkčný zdrojový kód privátneho nasadenia.

## Aktuálny stav

Projekt je vo fáze návrhu dátového modelu a dashboardov. Konkrétne polia, validačné pravidlá a používateľské workflow sa budú počas prototypovania meniť podľa overenia na reálnych výrobných situáciách.

---

**CONCEPT / PROTOTYPE — not production ready**
