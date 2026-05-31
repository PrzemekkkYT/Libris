# Libris

Libris is a lightweight personal library app for physical books, inspired by self-hosted system architectures. It lets you quickly catalog books you own or have read by scanning ISBN barcodes with your phone.

## Key features

- **Mobile ISBN scanner:** Add books with a single camera scan (PWA).
- **Built-in data providers:** Automatically fetch metadata and covers from Google Books and Poland’s National Library.
- **Plugin-based architecture:** Easily integrate external data providers (e.g., dedicated scrapers) as isolated HTTP microservices.
- **Reading status tracking:** Track reading progress (To read, Reading, Finished), ratings, and private notes.
- **Privacy and control:** Built-in authentication and a local database.

## Tech stack

- **Frontend:** Preact + Tailwind CSS (Vite / PWA)
- **Backend:** FastAPI (Python)
- **Database:** SQLite + SQLModel
- **Authentication:** JWT (JSON Web Tokens)

---

## Running locally (development)

### Requirements

- Python 3.10+
- Poetry 2.4.1+
- Node.js 18+

### 1. Clone the repository

```bash
git clone https://github.com/PrzemekkkYT/Libris.git
cd Libris
```

### 2. Start the backend (FastAPI)

```bash
cd backend
python -m venv .venv
source .venv/bin/activate
poetry install
uvicorn main:app --reload
```

On Windows, activate the environment with: `.venv\Scripts\activate`

_Backend will be available at: `http://127.0.0.1:8000`_
_Swagger API docs: `http://127.0.0.1:8000/docs`_

### 3. Start the frontend (Preact)

Open a new terminal in the project root:

```bash
cd frontend
npm install
npm run dev
```

_Frontend will be available at: `http://localhost:5173`_

---

## Architecture and API

A detailed description of the database structure, endpoints, and the contract for external providers can be found in:

- [ARCHITECTURE.md](./ARCHITECTURE.md) – Database architecture, overall stack and api endpoints.

## License

This project is released under the MIT license. See [LICENSE](LICENSE) for details.
