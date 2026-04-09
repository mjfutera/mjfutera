# Komendy bota (z kodu)

Poniżej znajduje się komplet komend, które bot rozpoznaje w kodzie.

## Komendy ogólne

- `/start`
  - Start bota i wiadomość powitalna.

- `/showcommands`
  - Wyświetla listę dostępnych komend.

- `/projects`
  - Pokazuje listę aktywnych projektów.

- `/projects <slug>`
  - Pokazuje szczegóły konkretnego projektu.

- `/becomepartner <activation_key>`
  - Aktywuje status partnera po podaniu poprawnego klucza.

- `/remindpartnerkey`
  - Pokazuje aktualny klucz aktywacyjny partnera.
  - Dostęp: rola `>= 2` (super admin/owner).

## Komendy partnerskie

Te komendy działają tylko dla użytkowników z `is_partner = 1`:

- `/balance`
  - Pokazuje aktualne saldo partnera.

- `/history`
  - Pokazuje historię transakcji partnera.

- `/partnerpanel`
  - Wysyła link do panelu partnera: `https://app.michalfutera.pro/`.

- `/setwallet <USDT_TRC20_address>`
  - Ustawia adres wypłaty USDT TRC20 (tworzy zmianę do zatwierdzenia).

- `/payout`
  - Tworzy żądanie wypłaty (zgodnie z warunkami w logice bota, m.in. minimalna kwota).

## Uwagi

- Bot rozpoznaje też formę komendy z sufiksem nazwy bota, np. `/start@TwojBot`.
- Dla nierozpoznanej komendy bot zwraca: `No command found.`
