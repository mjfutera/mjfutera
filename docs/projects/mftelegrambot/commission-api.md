# MF Telegram Bot - Commission API

## Overview

API do naliczania, korygowania i anulowania prowizji dla partnerów. Każde zapytanie musi mieć poprawny klucz API w nagłówku.

Base URL:
- https://tgbotapi.michalfutera.pro

Autentykacja:
- Nagłówek: `Authorization: Bearer <API_SECRET>`
- `Content-Type: application/json`

Kwoty:
- W zapytaniu podaje się kwotę **w centach** (integer). Przykład: `2500` = $25.00.
- W odpowiedzi kwoty są zwracane jako **USD decimal** (np. `25.0`).

---

## Kiedy używać którego endpointu?

| Sytuacja | Endpoint |
|---|---|
| Zamówienie opłacone — nalicz prowizję | `POST /api/commission` |
| Prowizja była błędna — popraw kwotę lub zmień partnerów | `POST /api/commission/edit` |
| Zamówienie anulowane lub zwrot — cofnij prowizję | `POST /api/commission/delete` |
| Chcesz sprawdzić co się stanie zanim to zrobisz | Dodaj `"dry_run": true` do dowolnego zapytania |

---

## Tryb testowy (dry_run)

Każdy endpoint obsługuje pole `"dry_run": true`. Gdy jest ustawione:

- Cała walidacja jest wykonywana normalnie (sprawdzane są projekty, partnerzy, duplikaty, kwoty)
- **Nic nie jest zapisywane do bazy** — salda nie ulegają zmianie
- **Żadne powiadomienia** nie są wysyłane na Telegramie
- Odpowiedź jest identyczna jak przy prawdziwym żądaniu — z jedną różnicą: `"dry_run": true` w odpowiedzi i `transaction_id: null` zamiast prawdziwych ID

Gdy jesteś gotowy do produkcji, po prostu usuń pole `dry_run` z zapytania. Kod po stronie integracji nie wymaga żadnych innych zmian — struktury odpowiedzi są identyczne.

---

## Endpointy

### 1. Nalicz prowizję — `POST /api/commission`

**Kiedy używać:** Zamówienie zostało opłacone. Chcesz dodać prowizję dla jednego lub wielu partnerów powiązanych z tym zamówieniem.

**Co robi:**
- Tworzy nowe wpisy transakcji typu `commission` w bazie
- Zwiększa saldo każdego partnera o podaną kwotę
- Wysyła powiadomienie na Telegramie do każdego partnera
- Blokuje duplikaty — nie można dwa razy naliczyć prowizji dla tego samego `source_ref`

**Wymagane pola:**
- `source_project` — slug projektu (musi istnieć w bazie)
- `source_ref` — unikalny identyfikator zamówienia/zdarzenia (np. ID zamówienia ze sklepu)
- `commissions` — tablica partnerów z kwotami

**Elementy tablicy `commissions`:**
- `user_id` — ID partnera (musi być aktywnym partnerem)
- `amount` — kwota w centach, liczba całkowita > 0
- `message` — opcjonalna notatka widoczna w historii transakcji

Zapytanie:
```json
{
  "source_project": "moj-sklep",
  "source_ref": "order_2026_0001",
  "commissions": [
    { "user_id": 123456789, "amount": 2500, "message": "Sprzedaż #0001" },
    { "user_id": 100200300, "amount": 900 }
  ]
}
```

Odpowiedź sukces (200):
```json
{
  "dry_run": false,
  "status": "ok",
  "processed": 2,
  "failed": 0,
  "results": [
    { "user_id": 123456789, "transaction_id": 101, "amount": 25, "new_balance": 4275.12 },
    { "user_id": 100200300, "transaction_id": 102, "amount": 9, "new_balance": 177.55 }
  ],
  "errors": []
}
```

Odpowiedź częściowa — gdy część partnerów jest nieprawidłowa (200):
```json
{
  "dry_run": false,
  "status": "partial",
  "processed": 1,
  "failed": 1,
  "results": [
    { "user_id": 123456789, "transaction_id": 103, "amount": 25, "new_balance": 4300.12 }
  ],
  "errors": [
    { "user_id": 999999999, "error": "User is not a partner" }
  ]
}
```

---

### 2. Popraw prowizję — `POST /api/commission/edit`

**Kiedy używać:** Prowizja została już naliczona, ale zawiera błąd — zła kwota, pominięty partner, nadmiarowy partner. Używasz tego samego `source_ref` co przy tworzeniu.

**Co robi:**
- Porównuje nową tablicę `commissions` z aktywnymi wpisami w bazie dla danego `source_ref`
- Wpisy, które się nie zmieniły (ten sam `user_id` i ta sama kwota) — zostają nienaruszone
- Wpisy, które się zmieniły lub zostały usunięte — stara transakcja jest oznaczana jako `voided`, saldo cofane
- Nowe lub zmienione wpisy — tworzone są nowe transakcje, saldo aktualizowane
- Partnerzy otrzymują powiadomienia na Telegramie o zmianach

**Wymagane pola:**
- `source_project`
- `source_ref` — ten sam co przy `add`
- `commissions` — pełna, docelowa lista partnerów z kwotami

**Opcjonalne pola:**
- `message` — powód korekty (trafia do logów i powiadomień)

Zapytanie (zmiana kwoty, usunięcie jednego partnera, dodanie nowego):
```json
{
  "source_project": "moj-sklep",
  "source_ref": "order_2026_0001",
  "message": "Korekta po weryfikacji zamówienia",
  "commissions": [
    { "user_id": 123456789, "amount": 3000 },
    { "user_id": 556677889, "amount": 1100 }
  ]
}
```

Odpowiedź (200):
```json
{
  "dry_run": false,
  "status": "ok",
  "source_project": "moj-sklep",
  "source_ref": "order_2026_0001",
  "unchanged": 0,
  "voided": 2,
  "added": 2,
  "details": {
    "voided": [
      { "user_id": 123456789, "amount": 25, "transaction_id": 101 },
      { "user_id": 100200300, "amount": 9, "transaction_id": 102 }
    ],
    "added": [
      { "user_id": 123456789, "amount": 30, "transaction_id": 110 },
      { "user_id": 556677889, "amount": 11, "transaction_id": 111 }
    ]
  }
}
```

---

### 3. Anuluj prowizję — `POST /api/commission/delete`

**Kiedy używać:** Zamówienie zostało anulowane, zwrócone lub z innego powodu prowizja nie powinna być wypłacona. Cofa całą partię prowizji dla danego `source_ref`.

**Co robi:**
- Oznacza wszystkie aktywne transakcje dla danego `source_ref` jako `voided`
- Cofa salda wszystkich partnerów powiązanych z tym `source_ref`
- Wysyła powiadomienia na Telegramie do każdego partnera

**Ważne:** Dane nie są fizycznie usuwane z bazy. Wpisy pozostają z oznaczeniem `status = voided` — historia jest zachowana dla audytu.

**Wymagane pola:**
- `source_project`
- `source_ref`

**Opcjonalne pola:**
- `message` — powód anulowania (np. `"Zwrot zamówienia"`, `"Chargeback"`)

Zapytanie:
```json
{
  "source_project": "moj-sklep",
  "source_ref": "order_2026_0001",
  "message": "Zwrot zamówienia"
}
```

Odpowiedź (200):
```json
{
  "dry_run": false,
  "status": "ok",
  "source_project": "moj-sklep",
  "source_ref": "order_2026_0001",
  "voided": 2,
  "details": {
    "voided": [
      { "user_id": 123456789, "amount": 30, "transaction_id": 110 },
      { "user_id": 556677889, "amount": 11, "transaction_id": 111 }
    ]
  }
}
```

---

### 4. Sprawdź listę partnerów — `GET /api/partners`

**Kiedy używać:** Chcesz sprawdzić, którzy użytkownicy mają status partnera, zanim wyślesz zapytanie o prowizję. Przydatne do weryfikacji `user_id` i pobierania danych wyświetlanych w integracji.

**Co zwraca:**
- Listę wszystkich aktywnych partnerów z ich podstawowymi danymi

**Autentykacja:** Tak — nagłówek `Authorization: Bearer <API_SECRET>`

**Parametry:** Brak

Zapytanie:
```bash
GET /api/partners
Authorization: Bearer YOUR_API_SECRET
```

Odpowiedź (200):
```json
{
  "status": "ok",
  "count": 2,
  "data": [
    { "id": 123456789, "first_name": "Jan", "last_name": "Kowalski", "username": "jankowalski" },
    { "id": 100200300, "first_name": "Anna", "last_name": null, "username": "annak" }
  ]
}
```

**Pola danych:**
| Pole | Typ | Opis |
|---|---|---|
| `id` | integer | Telegram user ID — używany jako `user_id` w zapytaniach prowizji |
| `first_name` | string | Imię z profilu Telegram |
| `last_name` | string \| null | Nazwisko z profilu Telegram (może być null) |
| `username` | string \| null | Nazwa użytkownika Telegram (może być null) |

---

### 5. Sprawdź czy użytkownik jest partnerem — `GET /partnercheck/:userId`

**Kiedy używać:** Szybka weryfikacja przed naliczeniem prowizji — sprawdza czy konkretny `user_id` ma aktywny status partnera.

**Autentykacja:** Brak — endpoint jest publiczny.

**Parametry URL:**
- `:userId` — Telegram user ID (integer)

Zapytanie:
```bash
GET /partnercheck/123456789
```

Odpowiedź (200):
```json
true
```

lub gdy użytkownik nie jest partnerem:
```json
false
```

Odpowiedź jest zawsze `200` — wartość `false` nie oznacza błędu, oznacza że użytkownik nie jest partnerem (lub nie istnieje w bazie).

---

## Format błędów

```json
{
  "status": "error",
  "code": "KOD_BLEDU",
  "message": "Opis błędu",
  "details": { "opcjonalny": "kontekst" }
}
```

| Kod | Status HTTP | Kiedy |
|---|---|---|
| `UNAUTHORIZED` | 403 | Brak lub błędny klucz API |
| `INVALID_JSON` | 400 | Body nie jest poprawnym JSON |
| `MISSING_SOURCE_PROJECT` | 400 | Brak pola source_project |
| `MISSING_SOURCE_REF` | 400 | Brak pola source_ref |
| `SOURCE_PROJECT_NOT_FOUND` | 404 | Projekt o podanym slug nie istnieje |
| `DUPLICATE_SOURCE_REF` | 409 | Prowizja dla tego source_ref już istnieje (tylko add) |
| `INVALID_COMMISSIONS` | 400 | Błędna tablica commissions (zły format, duplikat user_id) |
| `USER_NOT_PARTNER` | 400 | Podany użytkownik nie jest partnerem |
| `COMMISSION_BATCH_NOT_FOUND` | 404 | Nie ma aktywnych prowizji dla tego source_ref (edit/delete) |
| `INVALID_MODE` | 400 | Nieprawidłowa wartość pola mode |

---

## Przykłady cURL

### Nalicz prowizję:
```bash
curl -X POST "https://tgbotapi.michalfutera.pro/api/commission" \
  -H "Authorization: Bearer YOUR_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source_project": "moj-sklep",
    "source_ref": "order_2026_0001",
    "commissions": [
      { "user_id": 123456789, "amount": 2500 }
    ]
  }'
```

### Dry_run przed naliczeniem:
```bash
curl -X POST "https://tgbotapi.michalfutera.pro/api/commission" \
  -H "Authorization: Bearer YOUR_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source_project": "moj-sklep",
    "source_ref": "order_2026_0001",
    "dry_run": true,
    "commissions": [
      { "user_id": 123456789, "amount": 2500 }
    ]
  }'
```

### Popraw prowizję:
```bash
curl -X POST "https://tgbotapi.michalfutera.pro/api/commission/edit" \
  -H "Authorization: Bearer YOUR_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source_project": "moj-sklep",
    "source_ref": "order_2026_0001",
    "message": "Korekta kwoty",
    "commissions": [
      { "user_id": 123456789, "amount": 2600 }
    ]
  }'
```

### Anuluj prowizję:
```bash
curl -X POST "https://tgbotapi.michalfutera.pro/api/commission/delete" \
  -H "Authorization: Bearer YOUR_API_SECRET" \
  -H "Content-Type: application/json" \
  -d '{
    "source_project": "moj-sklep",
    "source_ref": "order_2026_0001",
    "message": "Zwrot zamówienia"
  }'
```

---

## Uwagi integracyjne

- `source_ref` powinien być stabilnym zewnętrznym identyfikatorem — najlepiej ID zamówienia z Twojego systemu.
- Nie generuj nowego `source_ref` przy edycji lub anulowaniu — zawsze używaj tego samego co przy tworzeniu.
- Używaj całkowitych centów zamiast liczb dziesiętnych, żeby unikać błędów zaokrąglania.
- Jeśli otrzymasz `409 DUPLICATE_SOURCE_REF` przy tworzeniu, prowizja już istnieje — nie ponawiaj, sprawdź stan.

