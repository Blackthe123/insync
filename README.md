# InSync

> **You picked your degree. You picked your courses. Now pick time for the people that matter.**

InSync is a friendship optimiser for UNSW students. It pulls real timetable data from the DevSoc API, runs a combinatorial optimisation across everyone's course selections, and surfaces the golden windows — shared free gaps where your whole group overlaps. Then it helps you actually use them.

---

## Features

- **Friend groups** — create a group and share an 8-character invite code with your crew
- **Course sync** — search and select your UNSW courses; data is pulled live from the [CSESoc GraphQL API](https://graphql.csesoc.app/v1/graphql)
- **Timetable optimisation** — the algorithm finds the class section combination across all members that maximises shared free time
- **Free window detection** — contiguous gaps of ≥ 1 hour where everyone is simultaneously free are surfaced as "windows"
- **Campus events** — a weekly board of UNSW events (Brekkie Club, Gigs in the Garden, Happy Hour, …) with per-group interest voting
- **Group chat** — a lightweight real-time-ish chat (3 s poll) so the plan doesn't die in someone's DMs
- **Owner controls** — group owners can remove members and delete the group

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML/CSS/JS, served by nginx |
| Backend | Python 3.12, FastAPI, SQLAlchemy (async), asyncpg |
| Database | PostgreSQL 16 |
| Auth | JWT (python-jose) + bcrypt passwords |
| External data | [DevSoc GraphQL API](https://graphql.csesoc.app/v1/graphql) |
| Containerisation | Docker + Docker Compose |

---

## Getting Started

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/) installed
- Ports `80` and `8000` free on your machine

### 1. Clone the repo

```bash
git clone https://github.com/Blackthe123/insync.git
cd insync
```

### 2. Create environment files

**`backend/.env`**

```env
DATABASE_URL=postgresql+asyncpg://mysite:yourpassword@mysite-postgres:5432/todolist
SECRET_KEY=change-me-to-a-long-random-string
```

**`postgres/.env`**

```env
POSTGRES_USER=mysite
POSTGRES_PASSWORD=yourpassword
POSTGRES_DB=todolist
```

> Make sure the credentials in both files match.

### 3. Build and run

```bash
docker compose up --build
```

The first run will build both images and wait for Postgres to become healthy before starting the API.

| Service | URL |
|---|---|
| Frontend | http://localhost |
| Backend API | http://localhost:8000 |
| API docs (Swagger) | http://localhost:8000/docs |

### 4. Stop

```bash
docker compose down          # keep the database volume
docker compose down -v       # also wipe the database
```

---

## Project Structure

```
insync/
├── backend/
│   ├── src/mysite/
│   │   └── main.py          # All FastAPI routes, models, and business logic
│   ├── Dockerfile
│   ├── pyproject.toml
│   └── requirements.txt
│   └── .env                 # git-ignored   
├── frontend/
│   ├── static/
│   │   └── index.html       # Single-page app (HTML + CSS + JS)
│   └── Dockerfile
├── postgres/
│   └── .env                 # Postgres credentials (git-ignored)
├── docker-compose.yml
└── README.md
```

---

## API Overview

All endpoints (except `/auth/register` and `/auth/login`) require a `Bearer` token in the `Authorization` header.

| Method | Path | Description |
|---|---|---|
| `POST` | `/auth/register` | Create an account |
| `POST` | `/auth/login` | Log in, receive JWT |
| `GET` | `/auth/me` | Current user info |
| `GET` | `/groups/me` | List your groups |
| `POST` | `/groups` | Create a group |
| `POST` | `/groups/join` | Join via invite code |
| `GET` | `/groups/{id}` | Group detail + members |
| `DELETE` | `/groups/{id}/leave` | Leave a group |
| `DELETE` | `/groups/{id}` | Delete a group (owner only) |
| `DELETE` | `/groups/{id}/members/{uid}` | Remove a member (owner only) |
| `GET` | `/groups/{id}/messages` | Fetch chat messages |
| `POST` | `/groups/{id}/messages` | Send a chat message |
| `GET` | `/groups/{id}/courses` | All members' course selections |
| `PUT` | `/groups/{id}/my-courses` | Update your course selections |
| `GET` | `/groups/{id}/timetable` | Retrieve saved timetable result |
| `POST` | `/groups/{id}/timetable` | Save a timetable result |
| `GET` | `/groups/{id}/votes` | All campus-event votes for the group |
| `POST` | `/groups/{id}/votes` | Cast a vote on a campus event |
| `DELETE` | `/groups/{id}/votes/{event_id}` | Remove your vote |

Full interactive documentation is available at `http://localhost:8000/docs` when the backend is running.

---

## How the Optimisation Works

1. **Fetch** — for every course code across all group members, InSync queries the DevSoc API for available class sections and their scheduled times. Full/closed sections are automatically excluded.

2. **Encode** — each section's schedule is converted into a set of 30-minute slot indices (Mon–Fri, 9 am – 8 pm → 110 slots total).

3. **Combine** — for each person, every valid combination of class sections (one per activity type per course) is enumerated. Combinations are capped at 2,000 per person to keep runtime bounded.

4. **Optimise** — if the product of all members' combination counts is ≤ 10,000, an exhaustive search finds the globally optimal assignment. Otherwise a greedy per-person pass is used. The objective is to minimise the union of busy slots across the group — i.e. maximise shared free time.

5. **Surface** — contiguous free runs of ≥ 1 hour (2 slots) in the resulting group schedule are extracted and displayed as "windows", alongside a colour-coded timetable grid.

---

## Roadmap

- [ ] Google Calendar / Apple Calendar sync
- [ ] Push notifications for upcoming shared windows
- [ ] Activity suggestions (pool, table tennis, coffee spots) tied to free windows
- [ ] Support for universities beyond UNSW

---

## Contributing

1. Fork the repo and create a feature branch: `git checkout -b feat/my-feature`
2. Make your changes and ensure the app starts cleanly with `docker compose up --build`
3. Open a pull request with a clear description of what you changed and why

---
