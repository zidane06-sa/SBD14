# Reservo 🍽️

A full-stack restaurant table reservation system built as a final project for the Database Systems (SBD) practicum. Reservo allows customers to discover restaurants, browse available tables, and book sessions — while giving admins full control over restaurant management, table availability, and reservation approvals.

---

# Tech Stack

## Frontend

| Tech | Role |
|------|------|
| React 18 + TypeScript | UI framework |
| Vite | Build tool & dev server |
| Tailwind CSS | Styling |
| Axios | HTTP client (`api/client.ts`) |
| Lucide React | Icon library |

## Backend

| Tech | Role |
|------|------|
| Node.js + Express | REST API server |
| Sequelize ORM | Database abstraction |
| PostgreSQL (Neon DB) | Cloud database |
| bcryptjs | Password hashing |
| jsonwebtoken (JWT) | Auth token generation & validation |
| dotenv | Environment variable management |
| nodemon | Dev server auto-restart |

---

# Features

## Customer

- **Discover** — Browse and search restaurants by name or cuisine type
- **Session-based Booking** — Each restaurant has auto-generated 90-minute sessions based on its opening/closing hours. Customers pick a date, select consecutive sessions, and choose an available table
- **Smart Availability** — Sessions already booked by others appear grayed out and are unselectable

### Booking Rules
- Booking **H-2 or earlier** → auto-confirmed immediately, table marked as reserved
- Booking **H or H-1** → sent as pending request, requires admin approval

### Additional Features
- **My Reservations** — Track all past and upcoming bookings with full status visibility
- **Rejection Reason** — If a reservation is rejected, the admin's reason is shown in the customer's dashboard
- **Cancel** — Customers can cancel pending or confirmed reservations
- **Favorites** — Save restaurants for quick access

---

## Admin

- **Add Restaurant** — Create new restaurants with full details (name, cuisine, address, city, hours, phone)
- **Add Table** — Assign tables to restaurants with capacity, location, and initial status
- **Schedule View** — Per-restaurant table grid showing availability per session per date, color-coded:
  - Green = available
  - Red = booked
  - Gray = maintenance
- **Table Status Management** — Change individual table status directly from the schedule grid
- **Incoming Requests** — Review pending H/H-1 reservations with customer and booking details
- **Check Availability** — Admin can open the full schedule for the selected restaurant/date before deciding
- **Approve** — Confirm reservation; table status auto-updates to reserved
- **Reject** — Reject reservation with required written reason surfaced to customer
- **Delete Restaurant / Table** — Remove entries with confirmation modal

---

# Project Structure

```txt
reservo/
├── reservo-frontend/
│   └── src/
│       ├── app/
│       │   └── App.tsx
│       └── api/
│           └── client.ts
│
└── reservo-backend/
    ├── app.js
    ├── config/
    │   └── database.js
    ├── models/
    │   ├── index.js
    │   ├── User.js
    │   ├── Restaurants.js
    │   ├── Table.js
    │   └── Reservation.js
    ├── controllers/
    │   ├── userController.js
    │   ├── restaurantController.js
    │   ├── tableController.js
    │   └── reservationController.js
    └── routes/
        ├── userRoutes.js
        ├── restaurantRoutes.js
        ├── tableRoutes.js
        └── reservationRoutes.js
```

---

# Data Models

## User

| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| name | String | |
| email | String | Unique |
| password | String | bcrypt hashed |
| role | Enum | `customer` \| `admin` |

---

## Restaurant

| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| name | String | |
| description | Text | Optional |
| address | String | |
| city | String | |
| cuisine | String | e.g. Indonesian, Japanese |
| phone | String | Optional |
| opening_time | Time | |
| closing_time | Time | |
| is_active | Boolean | Default true |
| owner_id | UUID | FK → User |

---

## Table

| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| restaurant_id | UUID | FK → Restaurant |
| table_number | String | Unique per restaurant |
| capacity | Integer | 1–50 |
| status | Enum | `available` \| `reserved` \| `maintenance` |
| location | String | e.g. Indoor, Window |

---

## Reservation

| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Primary key |
| customer_id | UUID | FK → User |
| restaurant_id | UUID | FK → Restaurant |
| table_id | UUID | FK → Table |
| reservation_date | DateOnly | |
| start_time | Time | First selected session start |
| end_time | Time | Last selected session end |
| guest_count | Integer | |
| status | Enum | `pending` \| `confirmed` \| `rejected` \| `cancelled` \| `completed` |
| special_request | Text | Optional |
| rejection_reason | Text | Filled on reject |
| confirmed_at | DateTime | Auto-set on confirm |

---

# API Endpoints

## Auth — `/api/users`

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/register` | Register new user |
| POST | `/login` | Login and return JWT token |
| GET | `/` | Get all users |
| GET | `/:id` | Get user by ID |
| PUT | `/:id` | Update user |
| DELETE | `/:id` | Delete user |

---

## Restaurants — `/api/restaurants`

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/` | Create restaurant |
| GET | `/` | Get all restaurants |
| GET | `/:id` | Get restaurant by ID |
| GET | `/city/:city` | Get restaurants by city |
| PUT | `/:id` | Update restaurant |
| DELETE | `/:id` | Delete restaurant |

---

## Tables — `/api/tables`

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/` | Create table |
| GET | `/` | Get all tables |
| GET | `/:id` | Get table by ID |
| GET | `/restaurant/:restaurant_id` | Get all tables for a restaurant |
| GET | `/restaurant/:restaurant_id/available` | Get available tables |
| GET | `/restaurant/:restaurant_id/schedule` | Get full session schedule |
| PUT | `/:id` | Update table |
| DELETE | `/:id` | Delete table |
| PATCH | `/:id/status` | Update table status |

---

## Reservations — `/api/reservations`

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/` | Create reservation |
| GET | `/` | Get all reservations |
| GET | `/:id` | Get reservation by ID |
| GET | `/customer/:customer_id` | Get reservations by customer |
| GET | `/restaurant/:restaurant_id` | Get reservations by restaurant |
| PUT | `/:id` | Update reservation |
| DELETE | `/:id` | Delete reservation |
| PATCH | `/:id/confirm` | Confirm reservation |
| PATCH | `/:id/reject` | Reject reservation |
| PATCH | `/:id/cancel` | Cancel reservation |
| PATCH | `/:id/complete` | Mark reservation as completed |

---

# Booking Logic

```txt
Customer selects date
        │
        ├── H-2 or earlier ──→ Auto-confirmed + table marked reserved
        │
        └── H or H-1 ────────→ Status: pending
                                        │
                                Admin reviews request
                                        │
                          ┌─────────────┴──────────────┐
                        Approve                      Reject
                          │                            │
                  Status: confirmed            Status: rejected
                  Table → reserved             rejection_reason
                                               surfaced to customer
```

Sessions are generated automatically from restaurant opening/closing times in 90-minute blocks. Customers can book multiple consecutive sessions stored as a single reservation.

---

# Setup

## Prerequisites

- Node.js v18+
- PostgreSQL database (Neon DB or local)

---

## Backend

```bash
cd reservo-backend
npm install
cp .env.example .env
node app.js
```

---

## Frontend

```bash
cd reservo-frontend
npm install
npm run dev
```

---

## Environment Variables (Backend)

```env
DB_HOST=
DB_PORT=5432
DB_NAME=
DB_USER=
DB_PASSWORD=
JWT_SECRET=
PORT=5000
```

---

# Planned Deployment

- **Frontend** → Vercel
- **Backend** → Railway
- **Database** → Neon DB (PostgreSQL)

---

# Authors

Built for SBD (Sistem Basis Data) Practicum Final Project.
