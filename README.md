# 📅 CalendlyProject

> A production-grade scheduling & availability API inspired by Calendly — built with TypeScript, Express, Prisma ORM, PostgreSQL, Temporal, and Google Calendar integration.

[![TypeScript](https://img.shields.io/badge/TypeScript-6.0-blue?logo=typescript)](https://typescriptlang.org)
[![Node.js](https://img.shields.io/badge/Node.js-20+-green?logo=nodedotjs)](https://nodejs.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-blue?logo=postgresql)](https://postgresql.org)
[![Prisma](https://img.shields.io/badge/Prisma-7.8-darkblue?logo=prisma)](https://prisma.io)
[![Temporal](https://img.shields.io/badge/Temporal-1.20-purple)](https://temporal.io)

---

## ✨ Features

- **User onboarding** — register, set availability windows, and generate a personal booking page slug
- **Timezone-aware scheduling** — availability engine built with Luxon for accurate cross-timezone slot resolution
- **Google Calendar OAuth2** — real-time sync of booked meetings to the host's Google Calendar
- **Conflict detection** — atomic double-booking prevention via PostgreSQL transactions
- **Automated email notifications** — booking confirmations and cancellation emails via Nodemailer
- **Background workflows** — durable async jobs (reminders, cleanup) powered by Temporal
- **Request validation** — input schemas enforced with Zod throughout the API

---

## 🏗️ Architecture

```
CalendlyProject/
├── prisma/
│   ├── schema.prisma          # PostgreSQL data models
│   └── migrations/            # Prisma migration history
├── src/
│   ├── server.ts              # Express app entry point
│   ├── routes/                # API route definitions
│   ├── controllers/           # Request handlers
│   ├── services/              # Business logic layer
│   ├── middlewares/           # Auth, validation, error handling
│   ├── utils/                 # Luxon helpers, Google OAuth, mailer
│   └── temporal/
│       ├── worker.ts          # Temporal worker entry point
│       ├── workflows/         # Durable workflow definitions
│       └── activities/        # Temporal activity implementations
├── prisma.config.ts           # Prisma datasource configuration
├── tsconfig.json
└── package.json
```

---

## ⚙️ Tech Stack

| Layer | Technology |
|---|---|
| Language | TypeScript 6 (ESM, strict mode) |
| Framework | Express 5 |
| Database | PostgreSQL 16 |
| ORM | Prisma 7 + `@prisma/adapter-pg` |
| Background Jobs | Temporal (`@temporalio/*`) |
| Calendar Integration | Google Calendar API (`googleapis`) |
| Email | Nodemailer |
| Date/Timezone | Luxon |
| Validation | Zod 4 |
| Package Manager | pnpm |
| Dev tooling | tsx + nodemon |

---

## 🚀 Getting Started

### Prerequisites

- Node.js 20+
- PostgreSQL 16+
- pnpm (`npm install -g pnpm`)
- Temporal server ([quick start](https://docs.temporal.io/application-development/foundations#run-a-development-cluster))
- Google Cloud project with Calendar API enabled

### 1. Clone the repo

```bash
git clone https://github.com/roshnithakor07/calendly
cd calendly
```

### 2. Install dependencies

```bash
pnpm install
```

### 3. Configure environment variables

Create a `.env` file in the project root:

```env
# PostgreSQL
DATABASE_URL=postgresql://user:password@localhost:5432/calendly

# Google OAuth2
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GOOGLE_REDIRECT_URI=http://localhost:3000/auth/google/callback

# Email (SMTP)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password

# Temporal
TEMPORAL_ADDRESS=localhost:7233
TEMPORAL_NAMESPACE=default

# App
PORT=3000
```

### 4. Set up the database

```bash
# Run migrations and generate Prisma client
pnpm run prisma:all
```

Or step by step:

```bash
pnpm run prisma:format    # Format schema
pnpm run prisma:migrate   # Run migrations
pnpm run prisma:generate  # Generate Prisma client
```

### 5. Start the development servers

**API server** (Terminal 1):
```bash
pnpm run dev
```

**Temporal worker** (Terminal 2):
```bash
pnpm run dev:worker
```

The API will be available at `http://localhost:3000`.

---

## 📡 API Reference

### Auth & Users

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/auth/register` | Register a new user |
| `POST` | `/api/auth/login` | Login and receive JWT |
| `GET` | `/api/auth/google` | Initiate Google OAuth2 flow |
| `GET` | `/api/auth/google/callback` | Google OAuth2 callback |

### Availability

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/availability` | Set weekly availability windows |
| `GET` | `/api/availability` | Get your availability |
| `GET` | `/api/availability/:slug` | Get a user's available slots by slug |

### Bookings

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/bookings` | Book a meeting slot |
| `GET` | `/api/bookings` | Get your upcoming bookings |
| `DELETE` | `/api/bookings/:id` | Cancel a booking |

### Example — Book a slot

```json
POST /api/bookings
{
  "hostSlug": "roshni-thakor",
  "startTime": "2025-02-15T10:00:00.000Z",
  "endTime": "2025-02-15T10:30:00.000Z",
  "guestName": "Roshni Thakor",
  "guestEmail": "roshni@example.com",
  "timezone": "Asia/Kolkata"
}
```

---

## 🗄️ Database Schema

### Core models

```prisma
model User {
  id           String        @id @default(uuid())
  name         String
  email        String        @unique
  slug         String        @unique   // e.g. "roshni-thakor"
  googleTokens Json?                   // OAuth2 tokens
  availability Availability[]
  bookings     Booking[]     @relation("HostBookings")
  createdAt    DateTime      @default(now())
}

model Availability {
  id        String   @id @default(uuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  dayOfWeek Int      // 0 = Sunday … 6 = Saturday
  startTime String   // "09:00"
  endTime   String   // "17:00"
  timezone  String
}

model Booking {
  id          String   @id @default(uuid())
  hostId      String
  host        User     @relation("HostBookings", fields: [hostId], references: [id])
  guestName   String
  guestEmail  String
  startTime   DateTime
  endTime     DateTime
  timezone    String
  calEventId  String?  // Google Calendar event ID
  status      String   @default("confirmed")
  createdAt   DateTime @default(now())
}
```

---

## ⚡ Background Jobs (Temporal)

The project uses **Temporal** for durable, fault-tolerant workflows:

- **Booking reminder workflow** — sends email reminders to host and guest 1 hour before the meeting
- **Cancellation workflow** — handles cleanup on booking cancellation (removes Google Calendar event, notifies both parties)
- **Retry logic** — automatic retries with exponential backoff on transient failures (email delivery, Google API rate limits)

Start the Temporal worker separately:

```bash
pnpm run dev:worker
```

> You need a running Temporal server. The easiest way locally is via Docker:
> ```bash
> docker run --rm -p 7233:7233 temporalio/auto-setup:latest
> ```

---

## 🔧 Prisma Commands

```bash
pnpm run prisma:format    # Format schema.prisma
pnpm run prisma:migrate   # Run pending migrations
pnpm run prisma:generate  # Regenerate Prisma client
pnpm run prisma:all       # Run all three above in sequence
pnpm run prisma:studio    # Open Prisma Studio (DB GUI) at localhost:5555
```

---

## 🌐 Google Calendar Integration

1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create a project and enable the **Google Calendar API**
3. Create OAuth2 credentials (Web application type)
4. Set the redirect URI to `http://localhost:3000/auth/google/callback`
5. Copy the Client ID and Secret into your `.env`

When a user connects their Google account, booked meetings are automatically created as Google Calendar events, and cancelled bookings remove the event.

---

## 📁 Environment Variables Reference

| Variable | Description | Required |
|---|---|---|
| `DATABASE_URL` | PostgreSQL connection string | ✅ |
| `GOOGLE_CLIENT_ID` | Google OAuth2 client ID | ✅ |
| `GOOGLE_CLIENT_SECRET` | Google OAuth2 client secret | ✅ |
| `GOOGLE_REDIRECT_URI` | OAuth2 redirect URI | ✅ |
| `SMTP_HOST` | SMTP server host | ✅ |
| `SMTP_PORT` | SMTP port (587 for TLS) | ✅ |
| `SMTP_USER` | SMTP username / email | ✅ |
| `SMTP_PASS` | SMTP password / app password | ✅ |
| `TEMPORAL_ADDRESS` | Temporal server address | ✅ |
| `TEMPORAL_NAMESPACE` | Temporal namespace | ✅ |
| `PORT` | HTTP server port (default: 3000) | ❌ |

---

## 🗺️ Roadmap

- [ ] Frontend booking page (Next.js)
- [ ] Multiple event types (15 min, 30 min, 1 hour)
- [ ] Buffer time between meetings
- [ ] Zoom / Google Meet link generation
- [ ] iCal / `.ics` file export
- [ ] Rate limiting and API keys

---

## 📄 License

MIT — built by [Roshni Thakor](https://github.com/roshnithakor07)
