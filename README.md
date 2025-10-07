# Fiterview

AI‑powered mock interview platform that turns your resume and a target job description into realistic, feedback‑rich interview practice sessions.

**Live**: [https://fiterview.com/](https://fiterview.com/)

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Monorepo Layout](#monorepo-layout)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Environment Variables](#environment-variables)
  - [Local Development](#local-development)
- [Database](#database)
- [Project Scripts](#project-scripts)
- [Coding Standards](#coding-standards)
- [Deployment](#deployment)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

Fiterview helps candidates rehearse interviews end‑to‑end:

1. Upload or paste your **resume**.
2. Paste a **job post** (JD) you’re targeting.
3. Generate a tailored **interview plan** (question bank by topic/seniority, behavioral + technical).
4. Conduct a **live mock** with voice (STT/TTS) or text.
5. Receive **structured feedback** and an **improvement report**.

The app is built with a clean, layered architecture to keep domain logic independent from adapters (UI, APIs, vendors). The App Router organizes routes and server actions, while Prisma + PostgreSQL persist user data and reports.

---

## Key Features

- **JD‑aware question generation** – prompts are conditioned on both the job post and resume to remove generic “one‑size‑fits‑all” questions.
- **Behavioral + technical coverage** – STAR/ABC prompts, system design, React/Next.js/web‑fundamentals tracks.
- **Live or async practice** – answer via microphone (speech‑to‑text) or type; text‑to‑speech for interviewer voice.
- **Rubric‑based scoring** – clarity, correctness, seniority signal, and actionable improvements per answer.
- **Session history** – store attempts, scores, and feedback for longitudinal progress tracking.
- **Privacy‑first design** – user artifacts are scoped to the account; secrets stored in envs.

> Note: Some features may be behind feature flags while the product evolves.

---

## Architecture

```
┌──────────────────────────────┐
│            App UI            │  Next.js (App Router), React, Tailwind
│  (app/*, components, hooks)  │
└──────────────┬───────────────┘
               │ Server Actions / API Routes (app/api/*)
┌──────────────▼───────────────┐
│         Application           │  Use cases / services orchestrating flows
│   (backend/application/*)     │
└──────────────┬───────────────┘
               │ Calls into domain + adapters
┌──────────────▼───────────────┐
│            Domain            │  Pure business entities, value objects
│     (backend/domain/*)       │
└──────────────┬───────────────┘
               │ Port interfaces
┌──────────────▼───────────────┐
│          Infrastructure      │  Prisma repos, LLM/STT/TTS/email adapters
│  (backend/infrastructure/*)  │
└──────────────┬───────────────┘
               │
       ┌───────▼─────────┐     ┌───────────────┐
       │   Prisma ORM    │────▶│  PostgreSQL   │
       │  (prisma/*)     │     │  (DATABASE)   │
       └─────────────────┘     └───────────────┘
```

### Flow (high‑level)

1. User submits resume + JD → server action builds a **PromptContext**.
2. **QuestionGenerator** use case requests questions from the LLM adapter.
3. User answers via text or mic → **EvaluationService** composes scoring rubrics and LLM feedback.
4. Results stored via Prisma repositories.

---

## Tech Stack

- **Web**: Next.js (App Router), React 18, TypeScript, Tailwind CSS
- **Auth**: NextAuth.js (session/callbacks, providers)
- **DB/ORM**: PostgreSQL + Prisma
- **AI**: LLM(s) via provider SDK (configurable); planned multi‑model routing
- **Speech**: STT/TTS adapters (pluggable; e.g., Whisper‑like STT)
- **State**: React Query (server state) + Zustand (client state)
- **Email/Notifs**: Adapter‑based (e.g., Resend or SMTP)
- **Tooling**: ESLint, Prettier, commitlint, Husky

> Exact vendors are abstracted via adapters in `backend/infrastructure/*` to keep the core portable.

---

## Monorepo Layout

```
.
├─ app/                 # Next.js App Router pages, layouts, server actions, API routes
├─ backend/             # Domain / application / infrastructure (Clean Architecture)
├─ prisma/              # Prisma schema, migrations, seed
├─ public/              # Static assets
├─ hooks/ stores/       # Reusable React hooks, Zustand stores
├─ lib/ utils/          # Shared libs, helpers, validators
├─ constants/ types/    # App‑wide constants and TypeScript types
├─ next.config.ts       # Next.js config
├─ middleware.ts        # Auth / routing middleware
└─ package.json         # Scripts and workspaces config (if any)
```

---

## Getting Started

### Prerequisites

- **Node.js**: 18.x or 20.x LTS
- **Yarn** or **pnpm** (project uses Yarn by default)
- **PostgreSQL**: local or cloud instance

### Environment Variables

Create a `.env` file at the project root:

| Name                     | Example                                           | Description                                     |
| ------------------------ | ------------------------------------------------- | ----------------------------------------------- |
| `DATABASE_URL`           | `postgresql://user:pass@localhost:5432/fiterview` | Prisma connection string                        |
| `NEXTAUTH_URL`           | `http://localhost:3000`                           | NextAuth base URL                               |
| `NEXTAUTH_SECRET`        | `...`                                             | NextAuth secret (use `openssl rand -base64 32`) |
| `OPENAI_API_KEY`         | `sk-...`                                          | LLM provider key for question gen & feedback    |
| `SPEECH_TO_TEXT_API_KEY` | `...`                                             | STT provider key (if using speech mode)         |
| `TEXT_TO_SPEECH_API_KEY` | `...`                                             | TTS provider key (if using voice interviewer)   |
| `RESEND_API_KEY`         | `...`                                             | Email adapter key (optional)                    |
| `GOOGLE_CLIENT_ID`       | `...apps.googleusercontent.com`                   | OAuth provider (if enabled)                     |
| `GOOGLE_CLIENT_SECRET`   | `...`                                             | OAuth provider secret                           |

> Some variables are optional depending on which adapters you enable.

### Local Development

```bash
# 1) Install deps
$ yarn install

# 2) Generate Prisma client
$ npx prisma generate

# 3) Provision DB & run migrations
$ npx prisma migrate dev --name init

# 4) (Optional) Seed local data
$ npx prisma db seed

# 5) Start dev server
$ yarn dev
# App runs on http://localhost:3000
```

> If you change the Prisma schema, rerun `prisma generate` and apply a new migration.

---

## Database

- Prisma schema lives in `prisma/schema.prisma`.
- Use `prisma/migrations/*` for versioned changes.
- Seed script (`prisma/seed.ts`) can populate demo users and sample question banks.

Common commands:

```bash
# Inspect database in a browser
npx prisma studio

# Create a new migration from schema changes
npx prisma migrate dev --name <change>

# Push schema without creating a migration (non‑prod)
npx prisma db push
```

---

## Project Scripts

Package scripts (run with `yarn <script>`):

- `dev` – start Next.js in development
- `build` – typecheck and build production assets
- `start` – run the production server
- `lint` – run ESLint
- `format` – run Prettier
- `typecheck` – run `tsc --noEmit`
- `prisma:*` – convenience wrappers around Prisma CLI

> See `package.json` for the authoritative list of scripts.

---

## Coding Standards

- **ESLint** with TypeScript rules; **Prettier** for formatting.
- **Commitlint + Husky** enforce conventional commits (e.g., `feat:`, `fix:`) on commit/push.
- Folder‑by‑layer structure; keep React components presentational where possible and push side‑effects into use cases/adapters.

---

## Deployment

- Recommended: **Vercel** for the Next.js frontend + serverless API routes.
- Ensure all required environment variables are set in the hosting provider.
- Use a managed PostgreSQL (e.g., Supabase, RDS, Neon) for production.
- Run `yarn build && yarn start` for a Node server deploy.


