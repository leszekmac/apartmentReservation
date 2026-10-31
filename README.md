# apartmentReservation

[![CI](https://github.com/leszekmac/apartmentReservation/actions/workflows/ci.yml/badge.svg)](https://github.com/leszekmac/apartmentReservation/actions/workflows/ci.yml)

System rezerwacji apartamentów: logowanie, lista apartamentów, kalendarz zajętości i tworzenie rezerwacji z ochroną przed podwójną rezerwacją tego samego terminu.

Projekt pokazuje warstwową architekturę w Symfony, obsługę współbieżności sprawdzaną testem na równoległych procesach oraz testy backendu i frontendu.

**Stos:** PHP 8.3, Symfony 7.4, Doctrine, SQLite, React 19, TypeScript, Vite, Docker, GitHub Actions, PHPUnit, PHPStan, PHP_CodeSniffer, Vitest

## Uruchomienie

```bash
cp .env.example .env
docker compose up --build
```

| Usługa | Adres |
|---|---|
| Frontend | http://localhost:3100 |
| API | http://localhost:8200 |

Migracje i konto demo tworzą się przy starcie kontenera. Dane logowania demo są w `.env.example`.

## Decyzje techniczne

- **Brak podwójnych rezerwacji:** kontrola kolizji i zapis to jedno polecenie SQL (`INSERT ... SELECT ... WHERE NOT EXISTS`), więc dwa równoległe żądania nie mogą obie zobaczyć wolnego terminu. Sprawdza to test, w którym 6 osobnych procesów PHP rezerwuje ten sam termin: dokładnie jeden wygrywa, a reszta dostaje 409.
- **Przedziały półotwarte:** rezerwacje mają postać `[start, end)`, więc pobyt kończący się o 12:00 nie blokuje kolejnego zaczynającego się o 12:00.
- **Czas:** API wymaga jawnej strefy w datach, a wszystko normalizuje do UTC i pełnych sekund. Porównywane są chwile, a nie godziny na zegarze.
- **Tokeny:** logowanie zwraca losowy token 256-bitowy, a w bazie jest tylko jego skrót SHA-256.
- **Logowanie:** odpowiedź jest identyczna dla nieznanego e-maila i złego hasła, a liczbę prób ogranicza limit.

## Architektura

Backend jest podzielony na warstwy w `backend/src/`:

| Warstwa | Odpowiedzialność |
|---|---|
| `Domain` | Encje, obiekt wartości `BookingPeriod`, polityka okna rezerwacji, wyjątki i interfejsy repozytoriów |
| `Application` | Przypadki użycia, np. `CreateBookingHandler`, `LoginHandler` |
| `Infrastructure` | Repozytoria Doctrine, bezpieczeństwo, polecenia konsoli |
| `UI` | Kontrolery HTTP, walidacja żądań, prezentery, obsługa błędów |

Warstwy `Domain` i `Application` zależą od interfejsów, dlatego testy jednostkowe działają na repozytoriach w pamięci.

## API

Wszystko poza logowaniem wymaga nagłówka `Authorization: Bearer <token>`. Odpowiedzi sukcesu mają kształt `{"data": ...}`, a czasy w odpowiedziach są zawsze w UTC.

| Metoda | Ścieżka |
|---|---|
| POST | `/api/auth/login` |
| GET | `/api/resources` |
| GET | `/api/bookings` |
| POST | `/api/bookings` |

### Błędy

| HTTP | `code` | Kiedy |
|---|---|---|
| 401 | `unauthenticated` | Brak, błędny lub wygasły token |
| 401 | `invalid_credentials` | Złe dane logowania |
| 422 | `validation_failed` | Brakujące lub niepoprawne pola; `errors` mapuje pole na komunikaty |
| 422 | `booking_in_past` | Rezerwacja zaczyna się w przeszłości |
| 422 | `booking_too_far_ahead` | Rezerwacja kończy się później niż za rok |
| 422 | `invalid_period` | Koniec nie jest po początku (po normalizacji do sekund) |
| 404 | `resource_not_found` | `resource_id` jest poprawny, ale taki apartament nie istnieje |
| 409 | `slot_conflict` | Termin nakłada się na istniejącą rezerwację tego apartamentu |
| 429 | `too_many_requests` | Zbyt wiele prób logowania (nagłówek `Retry-After`) |

## Reguły biznesowe

- **Uwierzytelnianie:** token jest ważny domyślnie 12 h (`API_TOKEN_TTL`), a hasła są haszowane algorytmem `auto` Symfony.
- **Limit prób logowania:** 5 na minutę na parę IP i e-mail oraz 30 na minutę na samo IP (sliding window).
- **Nie w przeszłości:** `start_at` nie może być wcześniejszy niż „teraz”.
- **Najwyżej rok do przodu:** `end_at` nie może być późniejszy niż „teraz + `BOOKING_MAX_ADVANCE`” (domyślnie `P1Y`).
- **Kolizje:** liczą się tylko w obrębie jednego apartamentu.

## Testy i analiza statyczna

Backend:

```bash
docker compose exec backend bin/phpunit
docker compose exec backend vendor/bin/phpstan analyse
docker compose exec backend vendor/bin/phpcs
```

Frontend:

```bash
docker compose exec frontend npm test
docker compose exec frontend npm run typecheck
```

Testy backendu używają osobnej bazy (`var/data_test.db`), więc nie ruszają danych deweloperskich. Te same kontrole uruchamia GitHub Actions przy każdej zmianie.
