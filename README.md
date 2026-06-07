<div align="center">

# 🚌 Havan Bus Booking Engine

### *The infrastructure that moves people — reliably, at scale.*

[![Django](https://img.shields.io/badge/Django-REST_Framework-092E20?style=for-the-badge&logo=django&logoColor=white)](https://www.django-rest-framework.org/)
[![React](https://img.shields.io/badge/React-18-61DAFB?style=for-the-badge&logo=react&logoColor=black)](https://reactjs.org/)
[![PostgreSQL](https://img.shields.io/badge/Neon-PostgreSQL-00E5FF?style=for-the-badge&logo=postgresql&logoColor=white)](https://neon.tech/)
[![Vercel](https://img.shields.io/badge/Vercel-Deployed-000000?style=for-the-badge&logo=vercel&logoColor=white)](https://vercel.com/)
[![Render](https://img.shields.io/badge/Render-API_Host-46E3B7?style=for-the-badge&logo=render&logoColor=white)](https://render.com/)
[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-Automation-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)](https://github.com/features/actions)

---

> **Havan** is a production-grade, multi-tenant bus ticket reservation platform engineered for **high-concurrency**, **zero inventory drift**, and **operator-grade reliability**. Built on a modern decoupled architecture, it handles everything from seat locking to PDF ticket generation — automatically.

</div>

---

## 📌 Table of Contents

- [Value Proposition](#-value-proposition)
- [Tech Stack](#-tech-stack)
- [Architecture](#-architecture)
- [Key Features](#-key-features)
- [Security Design](#-security-design)
- [Getting Started](#-getting-started)
- [Environment Variables](#-environment-variables)
- [API Reference](#-api-reference)
- [Contributing](#-contributing)

---

## 💡 Value Proposition

Most bus booking systems fail under pressure — **double bookings** when traffic spikes, **stale inventory** from manual ops, and **brittle monoliths** that collapse when one service sneezes.

**Havan is designed differently:**

| Problem | Havan's Answer |
|---|---|
| Double bookings under load | Row-level locking via `select_for_update()` |
| Stale or missing daily inventory | Self-healing automation via GitHub Actions CRON |
| Slow PDF generation blocking UX | Async Celery + Redis background task pipeline |
| Security gaps in internal APIs | Bearer Token auth + strict environment variable isolation |
| Vendor lock-in | Decoupled frontend/backend — swap either independently |

---

## 🛠️ Tech Stack

### Frontend

| Layer | Technology | Hosting | Notes |
|---|---|---|---|
| **UI Framework** | React 18 | Vercel | Component-driven SPA |
| **Routing** | React Router v6 | — | Client-side navigation |
| **State Management** | Context API / Hooks | — | Lightweight, no Redux overhead |
| **HTTP Client** | Axios | — | Interceptors for auth headers |
| **Styling** | Tailwind CSS | — | Utility-first, zero bloat |

### Backend

| Layer | Technology | Hosting | Notes |
|---|---|---|---|
| **API Framework** | Django REST Framework | Render | Hypermedia-ready REST API |
| **Task Queue** | Celery | Render (Worker) | Async job processing |
| **Message Broker** | Redis | Render | Celery broker + result backend |
| **ORM** | Django ORM | — | `select_for_update()` for atomicity |
| **Auth** | Bearer Token (DRF TokenAuth) | — | Stateless API security |
| **PDF Generation** | ReportLab / WeasyPrint | — | Ticket rendering in background tasks |

### Data & DevOps

| Layer | Technology | Notes |
|---|---|---|
| **Database** | Neon (Serverless PostgreSQL) | Auto-suspend, branching-ready |
| **CI/CD & Automation** | GitHub Actions | Daily inventory CRON + deployment hooks |
| **Secrets Management** | GitHub Secrets + Render Env Vars | Zero plaintext credentials in code |
| **Monitoring** | Render Logs + GitHub Actions audit trail | Operational observability |

---

## 🏗️ Architecture

### System Overview

Havan follows a **strict three-tier decoupled architecture**. The frontend never touches the database — every operation flows through the authenticated REST API.

```
┌──────────────────────────────────────────────────────────────────┐
│                         USER BROWSER                             │
│                    React SPA  ·  Vercel CDN                      │
└─────────────────────────────┬────────────────────────────────────┘
                              │  HTTPS + Bearer Token
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                      DJANGO REST API                             │
│                   DRF · Render · Gunicorn                        │
│                                                                  │
│   ┌─────────────────┐    ┌──────────────────────────────────┐   │
│   │  Booking Views  │    │        Celery Worker             │   │
│   │ select_for_     │    │  PDF Generation · Email Tasks    │   │
│   │ update() lock   │    └──────────────┬───────────────────┘   │
│   └────────┬────────┘                  │                        │
└────────────┼───────────────────────────┼────────────────────────┘
             │  Django ORM               │  Task Queue
             ▼                           ▼
┌─────────────────────────┐   ┌──────────────────────┐
│   Neon PostgreSQL DB    │   │   Redis (Broker)     │
│   (Serverless · ACID)   │   │   Render Managed     │
└─────────────────────────┘   └──────────────────────┘
             ▲
             │  Automated Inventory Writes
┌────────────┴────────────────────────────────────────────────────┐
│                       GITHUB ACTIONS                            │
│        CRON: Daily Inventory Generation  ·  00:00 UTC           │
│        Calls secured internal API endpoint with Bearer Token     │
└─────────────────────────────────────────────────────────────────┘
```

### 🔄 High-Concurrency Seat Booking — Sequence Flow

```mermaid
sequenceDiagram
    participant U as 🧑 User (React)
    participant A as ⚡ Django API (Render)
    participant DB as 🗄️ Neon PostgreSQL
    participant W as 🔧 Celery Worker
    participant R as 📮 Redis

    U->>A: POST /api/bookings/ {seat_id, trip_id}
    A->>A: Validate Bearer Token
    A->>DB: BEGIN TRANSACTION
    A->>DB: SELECT * FROM seats WHERE id=? FOR UPDATE
    Note over DB: Row-level lock acquired.<br/>Concurrent requests queue here.
    DB-->>A: Seat row (locked)
    A->>A: Check availability
    alt Seat Available
        A->>DB: UPDATE seat SET is_booked=True
        A->>DB: INSERT INTO bookings (...)
        A->>DB: COMMIT
        DB-->>A: ✅ Booking confirmed
        A->>R: Enqueue PDF generation task
        R->>W: Dispatch task
        W->>W: Generate PDF ticket
        W-->>U: 📧 Email with PDF ticket (async)
        A-->>U: 200 OK {booking_id, status: "confirmed"}
    else Seat Already Taken
        A->>DB: ROLLBACK
        A-->>U: 409 Conflict {error: "Seat no longer available"}
    end
```

### 🔁 Self-Healing Inventory — GitHub Actions Flow

```mermaid
flowchart TD
    CRON["⏰ GitHub Actions CRON\n0 0 * * * UTC"] --> RUN["▶️ Trigger Workflow\ninventory_generation.yml"]
    RUN --> AUTH["🔐 Load INVENTORY_API_TOKEN\nfrom GitHub Secrets"]
    AUTH --> CALL["📡 POST /api/internal/generate-inventory/\nAuthorization: Bearer $TOKEN"]
    CALL --> API["⚡ Django API on Render\nValidates token · Runs business logic"]
    API --> DB["🗄️ Neon PostgreSQL\nINSERT daily trips + seat rows"]
    DB --> OK{"✅ Success?"}
    OK -- Yes --> LOG["📋 GitHub Actions Log:\ninventory_generated: true"]
    OK -- No --> FAIL["❌ Workflow fails\nGitHub notifies via email"]
    FAIL --> RETRY["🔄 Manual re-run available\nvia GitHub UI"]
```

---

## ✨ Key Features

### 🔒 Concurrency-Safe Booking Engine

> **Feature:** Row-level pessimistic locking using Django ORM's `select_for_update()`
> 
> **Benefit:** Eliminates the race condition where two users simultaneously book the last seat. The database itself serialises concurrent requests — no application-level mutex, no overselling.

```python
# The core locking pattern
with transaction.atomic():
    seat = Seat.objects.select_for_update().get(id=seat_id, trip=trip)
    if seat.is_booked:
        raise SeatUnavailableError("Seat already taken.")
    seat.is_booked = True
    seat.save()
    Booking.objects.create(user=user, seat=seat, ...)
```

---

### 🤖 Self-Healing Daily Inventory

> **Feature:** GitHub Actions CRON workflow runs at midnight UTC, calling a secured internal endpoint to generate the next day's trip inventory.
>
> **Benefit:** Operators never manually create trips. The system provisions itself. Even after a server restart on Render's free tier, inventory is always fresh. *Zero-ops inventory management.*

```yaml
# .github/workflows/generate_inventory.yml (excerpt)
on:
  schedule:
    - cron: '0 0 * * *'   # Daily at 00:00 UTC
  workflow_dispatch:        # Manual trigger available

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Inventory API
        run: |
          curl -X POST ${{ secrets.API_BASE_URL }}/api/internal/generate-inventory/ \
            -H "Authorization: Bearer ${{ secrets.INVENTORY_API_TOKEN }}" \
            -H "Content-Type: application/json"
```

---

### 📄 Async PDF Ticket Generation

> **Feature:** PDF ticket rendering is offloaded to a Celery worker via a Redis task queue — completely decoupled from the HTTP request cycle.
>
> **Benefit:** API response time stays under 200ms regardless of PDF complexity. Users get instant booking confirmation; their ticket arrives by email moments later, without the booking endpoint ever blocking.

```
Booking Request → API Response (fast) → Redis Queue → Celery Worker → PDF → Email
       ↑                   ↑
   < 200ms           Immediate 200 OK
```

---

### 🏢 Multi-Tenant Architecture

> **Feature:** Operator-scoped data isolation built into the ORM layer via tenant-aware querysets.
>
> **Benefit:** Multiple bus operators can run on the same platform with zero data leakage. One operator's routes, seats, and bookings are structurally invisible to another.

---

### 📱 Decoupled React Frontend

> **Feature:** The React SPA is deployed independently on Vercel's global CDN edge network.
>
> **Benefit:** Frontend and backend scale, deploy, and fail independently. A backend deployment doesn't flash the UI. Frontend updates ship in seconds with zero downtime.

---

## 🛡️ Security Design

Havan is engineered with a **security-first posture** at every layer of the stack.

### Authentication Model

| Endpoint Type | Auth Mechanism | Used By |
|---|---|---|
| User-facing booking endpoints | `Authorization: Bearer <token>` (DRF Token Auth) | React frontend, authenticated users |
| Internal automation endpoints | `Authorization: Bearer <INVENTORY_API_TOKEN>` (GitHub Secret) | GitHub Actions CRON only |
| Admin panel | Django session auth + superuser flag | Ops team |

> [!IMPORTANT]
> **Internal endpoints are never exposed to the frontend.** The `/api/internal/` namespace is protected by a separate, long-lived secret token stored exclusively in GitHub Secrets and Render environment variables. It is **never** committed to source code, never logged, and never returned in API responses.

### Environment Variable Isolation

```
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│   GitHub Secrets    │     │   Render Env Vars   │     │   Vercel Env Vars   │
│                     │     │                     │     │                     │
│ INVENTORY_API_TOKEN │     │ SECRET_KEY          │     │ VITE_API_BASE_URL   │
│ API_BASE_URL        │     │ DATABASE_URL        │     │                     │
│                     │     │ REDIS_URL           │     │                     │
│                     │     │ INVENTORY_API_TOKEN │     │                     │
│                     │     │ ALLOWED_HOSTS       │     │                     │
└─────────────────────┘     └─────────────────────┘     └─────────────────────┘
         │                           │                           │
         └───────────────────────────┴───────────────────────────┘
                    ✅ Zero plaintext secrets in source code
```

> [!NOTE]
> **Pro-Tip:** Rotate `INVENTORY_API_TOKEN` by updating it simultaneously in GitHub Secrets and Render's environment variables. The next CRON run automatically uses the new token — no code changes, no redeployment required.

### Additional Security Measures

- ✅ **CORS**: Strict origin whitelist — only the Vercel domain is allowed on the Django API
- ✅ **`DEBUG=False`** enforced in all Render deployments via environment variable
- ✅ **`ALLOWED_HOSTS`** explicitly set — no wildcard hosts in production
- ✅ **Database**: Neon's serverless PostgreSQL uses TLS-only connections; no public IP exposure
- ✅ **Celery tasks**: Workers run in an isolated Render service with no public-facing ports

---

## 🚀 Getting Started

### Prerequisites

Ensure the following are installed before proceeding:

- **Python** `>= 3.11`
- **Node.js** `>= 18.x`
- **Redis** (local instance or Docker)
- **PostgreSQL** (local) *or* a [Neon](https://neon.tech) account for cloud DB

---

### 1️⃣ Clone the Repository

```bash
git clone https://github.com/your-username/havan-bus-booking.git
cd havan-bus-booking
```

---

### 2️⃣ Backend Setup (Django)

```bash
# Navigate to the backend directory
cd backend

# Create and activate a virtual environment
python -m venv venv
source venv/bin/activate        # macOS/Linux
# venv\Scripts\activate         # Windows

# Install dependencies
pip install -r requirements.txt

# Copy the environment template
cp .env.example .env
```

**Edit `.env`** with your values (see [Environment Variables](#-environment-variables) below), then:

```bash
# Apply database migrations
python manage.py migrate

# Create a superuser for the admin panel
python manage.py createsuperuser

# Start the development server
python manage.py runserver
```

**Start the Celery worker** (in a separate terminal):

```bash
cd backend
source venv/bin/activate
celery -A havan worker --loglevel=info
```

---

### 3️⃣ Frontend Setup (React)

```bash
# Navigate to the frontend directory
cd ../frontend

# Install Node dependencies
npm install

# Copy the environment template
cp .env.example .env.local
```

Set `VITE_API_BASE_URL=http://localhost:8000` in `.env.local`, then:

```bash
# Start the development server
npm run dev
```

The React app will be available at `http://localhost:5173`.

---

### 4️⃣ Test the Inventory Automation Locally

```bash
# Simulate what GitHub Actions does every midnight
curl -X POST http://localhost:8000/api/internal/generate-inventory/ \
  -H "Authorization: Bearer your_local_dev_token" \
  -H "Content-Type: application/json"
```

---

## 🔑 Environment Variables

### Backend (`backend/.env`)

| Variable | Required | Description |
|---|---|---|
| `SECRET_KEY` | ✅ | Django secret key — generate with `python -c "import secrets; print(secrets.token_urlsafe(50))"` |
| `DEBUG` | ✅ | `True` for local dev, **`False` in production** |
| `DATABASE_URL` | ✅ | PostgreSQL connection string (Neon format: `postgresql://user:pass@host/db?sslmode=require`) |
| `REDIS_URL` | ✅ | Redis connection string (`redis://localhost:6379/0` locally) |
| `ALLOWED_HOSTS` | ✅ | Comma-separated allowed hosts (e.g., `localhost,your-app.onrender.com`) |
| `CORS_ALLOWED_ORIGINS` | ✅ | Frontend origin (e.g., `https://your-app.vercel.app`) |
| `INVENTORY_API_TOKEN` | ✅ | Long, random secret used by GitHub Actions to authenticate inventory calls |
| `EMAIL_HOST_USER` | ⚡ | SMTP user for ticket email delivery |
| `EMAIL_HOST_PASSWORD` | ⚡ | SMTP password |

### Frontend (`frontend/.env.local`)

| Variable | Required | Description |
|---|---|---|
| `VITE_API_BASE_URL` | ✅ | Base URL of the Django API (e.g., `https://your-api.onrender.com`) |

> [!WARNING]
> **Never commit `.env` or `.env.local` files.** Both are already in `.gitignore`. Use your hosting platform's environment variable management (Render Dashboard / Vercel Project Settings) for production values.

---

## 📡 API Reference

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `POST` | `/api/auth/login/` | None | Obtain user auth token |
| `POST` | `/api/auth/register/` | None | Register a new user |
| `GET` | `/api/trips/` | Bearer Token | List available trips |
| `GET` | `/api/trips/{id}/seats/` | Bearer Token | Get seat map for a trip |
| `POST` | `/api/bookings/` | Bearer Token | Create a booking (concurrency-safe) |
| `GET` | `/api/bookings/{id}/` | Bearer Token | Retrieve booking details |
| `GET` | `/api/bookings/{id}/ticket/` | Bearer Token | Download PDF ticket |
| `POST` | `/api/internal/generate-inventory/` | Internal Token | **Automation only** — generate daily inventory |

---

## 🤝 Contributing

Contributions are welcome. Please follow this workflow:

1. **Fork** the repository
2. **Create a feature branch**: `git checkout -b feature/your-feature-name`
3. **Commit** with conventional commit messages: `feat: add seat map component`
4. **Push** to your fork: `git push origin feature/your-feature-name`
5. **Open a Pull Request** against `main`

> [!NOTE]
> All PRs require passing CI checks. Run `python manage.py test` and `npm run lint` before submitting.

---

<div align="center">

**Built with precision by the Havan team.**

*If this project helped you, consider starring the repository ⭐*

</div>
