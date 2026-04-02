# MichalFutera.pro CSS - AI Guide

## Cel
Ten katalog zawiera kopie stylow z motywu WordPress `MichalFutera.Pro`, aby inna AI mogla je analizowac i modyfikowac bez ryzyka pracy na pliku produkcyjnym.

## Pliki
- `style.css` - kopia glownego arkusza stylow motywu z repo nadrzednego.

## Jak tego uzywac (workflow dla AI)
1. Traktuj `style.css` jako plik roboczy do analizy i propozycji zmian.
2. Najpierw czytaj selektywne zakresy linii (sekcje, ktore chcesz zmienic), potem proponuj minimalne poprawki.
3. Pilnuj zgodnosci z WordPress theme header na poczatku pliku.
4. Nie usuwaj istniejacych klas i selektorow bez wyraznej potrzeby (ryzyko regresji UI).
5. Dla wiekszych zmian dodawaj krotki changelog w commit message.

## Synchronizacja z pliku zrodlowego
Jesli trzeba odswiezyc kopie z glownego motywu, uruchom z katalogu glownego repo (`MichalFutera.Pro`):

```powershell
Copy-Item -Path "style.css" -Destination "mjfutera/docs/MichalFutera.pro/style.css" -Force
```

## Dobre praktyki dla AI
- Zachowuj istniejace nazewnictwo klas.
- Preferuj zmiany lokalne zamiast globalnych resetow.
- Unikaj "!important", chyba ze to konieczne.
- Po zmianach porownaj diff i sprawdz, czy nie ruszono niepowiazanych sekcji.

## Kontekst repozytoriow
- Repo nadrzedne: `MichalFutera.Pro` (motyw WordPress).
- Subrepo: `mjfutera`.
- Katalog dokumentacji: `docs/MichalFutera.pro`.
