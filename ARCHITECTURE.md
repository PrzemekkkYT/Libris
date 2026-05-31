# Specyfikacja Projektu i Architektury "Libris"

Libris to lekka, osobista biblioteka książek przeczytanych lub posiadanych, oferująca funkcję szybkiego dodawania pozycji poprzez skanowanie kodów kreskowych ISBN za pomocą smartfona. Architektura wspiera rozszerzalność o zewnętrzne źródła danych poprzez bezpieczne, odizolowane mikro-usługi API.

---

## 1. Stack Technologiczny

### Frontend (Klient Mobilny / PWA)

- **Framework:** Preact (superlekki zamiennik Reacta, ~4 KB)
- **Stylizowanie:** Tailwind CSS (własne, lekkie komponenty UI)
- **Skaner kodów:** `html5-qrcode` lub `quagga2` (dostęp do aparatu w przeglądarce przez `useRef`)
- **Architektura:** PWA (Progressive Web App) budowane za pomocą Vite

### Backend (API & Logika)

- **Framework:** FastAPI (Python)
- **Zalety:** Pełna asynchroniczność (`async/await`), automatyczna walidacja danych (Pydantic), automatyczna dokumentacja interaktywna pod adresem `/docs`.

### Baza Danych & ORM

- **Baza danych:** SQLite (plikowa, bezobsługowa, idealna do małych projektów)
- **ORM:** SQLModel (hybryda SQLAlchemy i Pydantica – jedna klasa definiuje tabelę oraz schemat API)

### Bezpieczeństwo & Autentykacja

- **Mechanizm:** Własny system logowania oparty o **JWT (JSON Web Tokens)**
- **Szyfrowanie haseł:** `bcrypt` lub `argon2` na poziomie backendu

---

## 2. Model Bazy Danych (Format DBML)

_Kod poniżej można wkleić bezpośrednio do [dbdiagram.io](https://dbdiagram.io) w celu wygenerowania schematu graficznego._

```dbml
enum book_status {
  to_read
  reading
  read
}

Table users {
  id integer [primary key, increment]
  username varchar [unique, not null]
  email varchar [unique, not null]
  hashed_password varchar [not null]
  created_at timestamp [default: `now()`]
}

Table books {
  id integer [primary key, increment]
  isbn varchar [unique, not null, note: 'Znormalizowany ciąg cyfr']
  title varchar [not null]
  cover_url varchar
  description text
  published_date varchar
  created_at timestamp [default: `now()`]
}

Table authors {
  id integer [primary key, increment]
  name varchar [unique, not null]
}

Table book_authors {
  book_id integer [ref: > books.id, primary key]
  author_id integer [ref: > authors.id, primary key]
}

Table user_books {
  id integer [primary key, increment]
  user_id integer [ref: > users.id, not null]
  book_id integer [ref: > books.id, not null]
  status book_status [default: 'to_read', not null]
  rating integer [note: 'Skala np. 1-5']
  notes text
  added_at timestamp [default: `now()`]
  read_at timestamp
}

Table external_providers {
  id integer [primary key, increment]
  name varchar [not null]
  base_url varchar [not null, note: 'np. http://localhost:5001']
  is_active boolean [default: true, not null]
  created_at timestamp [default: `now()`]
}
```

---

## 3. Specyfikacja API (Endpointy)

> 🔑 **Uwaga:** Endpointy oznaczone jako `[Zabezpieczony]` wymagają nagłówka `Authorization: Bearer <JWT_TOKEN>`.

### Moduł Autentykacji i Użytkowników (`/api/auth`)

#### Rejestracja użytkownika

- **URL:** `/api/auth/register`
- **Metoda:** `POST`
- **Body (JSON):**

```json
{
  "username": "jan_kowalski",
  "email": "jan@example.com",
  "password": "SuperBezpieczneHaslo123!"
}
```

- **Odpowiedź (201 Created):**

```json
{
  "id": 1,
  "username": "jan_kowalski",
  "email": "jan@example.com",
  "created_at": "2026-05-31T21:30:00Z"
}
```

#### Logowanie (Generowanie tokenu)

- **URL:** `/api/auth/login`
- **Metoda:** `POST`
- **Nagłówki:** `Content-Type: application/x-www-form-urlencoded`
- **Body (Form Data):**
  - `username`: `jan_kowalski`
  - `password`: `SuperBezpieczneHaslo123!`
- **Odpowiedź (200 OK):**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer"
}
```

#### Profil aktualnego użytkownika `[Zabezpieczony]`

- **URL:** `/api/auth/me`
- **Metoda:** `GET`
- **Odpowiedź (200 OK):**

```json
{
  "id": 1,
  "username": "jan_kowalski",
  "email": "jan@example.com"
}
```

---

### Moduł Skanowania i Katalogu Globalnego (`/api/books`)

#### Skanowanie kodu ISBN `[Zabezpieczony]`

- **URL:** `/api/books/scan`
- **Metoda:** `POST`
- **Body (JSON):**

```json
{
  "isbn": "9788328312345"
}
```

- **Opis działania:** Backend czyści ciąg znaków ISBN. Najpierw odpytuje lokalną bazę SQLite. Jeśli nie znajdzie książki, odpytuje wbudowane API (np. Google Books) oraz wszystkie aktywne zewnętrzne wtyczki, zapisuje wynik w katalogu globalnym i zwraca obiekt.
- **Odpowiedź (200 OK):**

```json
{
  "id": 42,
  "isbn": "9788328312345",
  "title": "Wiedźmin: Ostatnie życzenie",
  "authors": ["Andrzej Sapkowski"],
  "cover_url": "[https://books.google.com/covers/](https://books.google.com/covers/)...",
  "description": "Zbiór opowiadań o wiedźminie Geralcie.",
  "published_date": "1993"
}
```

---

### Moduł Osobistej Biblioteki Użytkownika (`/api/my-library`)

#### Pobieranie kolekcji użytkownika `[Zabezpieczony]`

- **URL:** `/api/my-library`
- **Metoda:** `GET`
- **Parametry URL (Query params - opcjonalne):**
  - `status`: `to_read` | `reading` | `read`
  - `sort`: `added_at_desc` | `title_asc` | `rating_desc`
- **Odpowiedź (200 OK):**

```json
[
  {
    "user_book_id": 12,
    "status": "read",
    "rating": 5,
    "notes": "Moja ulubiona książka z dzieciństwa.",
    "added_at": "2026-05-31T12:00:00Z",
    "read_at": "2026-05-31T20:00:00Z",
    "book": {
      "id": 42,
      "isbn": "9788328312345",
      "title": "Wiedźmin: Ostatnie życzenie",
      "authors": ["Andrzej Sapkowski"],
      "cover_url": "https://..."
    }
  }
]
```

#### Dodanie książki do własnej biblioteczki `[Zabezpieczony]`

- **URL:** `/api/my-library`
- **Metoda:** `POST`
- **Body (JSON):**

```json
{
  "book_id": 42,
  "status": "to_read"
}
```

- **Odpowiedź (201 Created):**

```json
{
  "id": 13,
  "user_id": 1,
  "book_id": 42,
  "status": "to_read",
  "rating": null,
  "notes": null,
  "added_at": "2026-05-31T21:34:00Z"
}
```

#### Aktualizacja statusu/notatek książki `[Zabezpieczony]`

- **URL:** `/api/my-library/{user_book_id}`
- **Metoda:** `PATCH`
- **Body (JSON - wszystkie pola opcjonalne):**

```json
{
  "status": "read",
  "rating": 5,
  "notes": "Przeczytane jednym tchem!",
  "read_at": "2026-05-31T21:35:00Z"
}
```

- **Odpowiedź (200 OK):** Zwraca zaktualizowany obiekt `user_books`.

#### Usunięcie książki z biblioteczki `[Zabezpieczony]`

- **URL:** `/api/my-library/{user_book_id}`
- **Metoda:** `DELETE`
- **Odpowiedź (204 No Content):** Brak treści (sukces).

---

### Moduł Zarządzania Zewnętrznymi Providerami (`/api/providers`)

#### Lista zarejestrowanych dostawców `[Zabezpieczony]`

- **URL:** `/api/providers`
- **Metoda:** `GET`
- **Odpowiedź (200 OK):**

```json
[
  {
    "id": 1,
    "name": "LubimyCzytac Scraper",
    "base_url": "http://localhost:5001",
    "is_active": true
  }
]
```

#### Rejestracja nowego dostawcy `[Zabezpieczony]`

- **URL:** `/api/providers`
- **Metoda:** `POST`
- **Body (JSON):**

```json
{
  "name": "Custom BN API",
  "base_url": "[http://192.168.1.150:8000](http://192.168.1.150:8000)"
}
```

- **Odpowiedź (201 Created):** Zwraca obiekt nowo utworzonego providera z przypisanym `id`.

#### Włączenie/Wyłączenie providera `[Zabezpieczony]`

- **URL:** `/api/providers/{id}`
- **Metoda:** `PATCH`
- **Body (JSON):**

```json
{
  "is_active": false
}
```

- **Odpowiedź (200 OK):** Zwraca zmodyfikowany obiekt providera.

#### Usunięcie providera z systemu `[Zabezpieczony]`

- **URL:** `/api/providers/{id}`
- **Metoda:** `DELETE`
- **Odpowiedź (204 No Content).**

---

## 4. Kontrakt Integracyjny dla Zewnętrznych Wtyczek (Zewnętrzne API)

Wszystkie zewnętrzne usługi (np. scrapery w osobnym Dockerze) dodawane przez sekcję zarządzania providerami **muszą** implementować poniższy endpoint, aby Libris mógł pobierać z nich dane.

- **URL:** `/api/v1/book/{isbn}` _(wystawiany na porcie zewnętrznego kontenera)_
- **Metoda:** `GET`

#### Odpowiedź (200 OK) - Książka znaleziona w zewnętrznym źródle:

```json
{
  "found": true,
  "title": "Hobbit, czyli tam i z powrotem",
  "authors": ["J.R.R. Tolkien"],
  "cover_url": "http://localhost:5001/images/hobbit.jpg",
  "description": "Opis przygód Bilbo Bagginsa...",
  "published_date": "1937"
}
```

#### Odpowiedź (200 OK) - Brak wyników w zewnętrznym źródle:

```json
{
  "found": false,
  "title": null,
  "authors": [],
  "cover_url": null,
  "description": null,
  "published_date": null
}
```

---

_Dokumentacja stworzona na potrzeby rozwoju projektu Libris._
