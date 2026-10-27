<p align="center">
  <img src="assets/tripalcare-logo.webp" alt="TripalCare" width="420" />
</p>

<p align="center">
  <b>Backend for TripalCare (TMA): medical help for travellers, over WhatsApp.</b>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/NestJS-11-E0234E?logo=nestjs&logoColor=white" alt="NestJS 11" />
  <img src="https://img.shields.io/badge/TypeScript-5-3178C6?logo=typescript&logoColor=white" alt="TypeScript" />
  <img src="https://img.shields.io/badge/Prisma-7-2D3748?logo=prisma&logoColor=white" alt="Prisma 7" />
  <img src="https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white" alt="PostgreSQL" />
  <img src="https://img.shields.io/badge/Redis-7-DC382D?logo=redis&logoColor=white" alt="Redis" />
  <img src="https://img.shields.io/badge/Node-22-339933?logo=node.js&logoColor=white" alt="Node 22" />
</p>

---

## Overview

TripalCare connects tourists who feel unwell with nearby clinics and doctors. The tourist doesn't install an app. They scan a QR code in their hotel room, WhatsApp opens, and the conversation takes it from there.

1. **Scan.** Each hotel room has its own QR code. Scanning it opens WhatsApp with a prefilled `START` message that carries the hotel and room.
2. **Intake.** The bot asks short multiple-choice questions. Tourists can also reply by voice note, which is transcribed to text.
3. **Triage.** Each message is checked for emergencies. Hard body-state keywords escalate straight away. Everything else goes to a context-aware check with Claude Haiku.
4. **Booking.** Non-emergency cases are matched to a clinic slot or an on-call doctor, then paid for. Payment is held in escrow until the visit is proven.

## Features

- **Hotel QR onboarding.** Generate QR codes for every room in a hotel and download them as a printable PDF.
- **WhatsApp Cloud API bot.** Webhook verification, incoming text, buttons and voice notes. The webhook returns `200` immediately and processes in the background, so Meta never retries a slow delivery.
- **Speech-to-text.** Voice notes are transcribed with Google Cloud Speech. If GCP credentials aren't set, OpenAI Whisper is used instead.
- **Two-stage emergency triage.** A keyword gate first, then an LLM check behind a circuit breaker. If the AI provider is down, the tourist is never blocked.
- **Session state in Redis.** Conversation state and message dedup.
- **Bookings, payments and escrow.** Razorpay and Stripe, escrow events, visit proof and disputes.

## Tech stack

| Layer          | Choice                                              |
| -------------- | --------------------------------------------------- |
| Framework      | NestJS 11 on Node.js 22                             |
| Database       | PostgreSQL, accessed through Prisma 7 (`@prisma/adapter-pg`) |
| Cache/session  | Redis 7 (`ioredis`)                                 |
| Messaging      | WhatsApp Cloud API (Meta)                           |
| AI             | Anthropic Claude, Google Cloud Speech, OpenAI Whisper (fallback) |
| Documents      | `qrcode` + `pdfkit`                                 |
| Infrastructure | Docker, AWS ECR + ECS, GitHub Actions               |

## Project structure

```text
src/
├── booking/       # Booking flow and status transitions
├── client/        # Clinic lookup and slot availability
├── database/      # PrismaService / PrismaModule
├── hotel-qr/      # Room QR generation, PDF export, scan → WhatsApp redirect
├── seed/          # Endpoints for seeding demo hotels and clinics
├── session/       # Redis provider and session service
├── triage/        # Intake questions, AI service, emergency detection
├── types/         # Shared types and emergency keyword lists
└── whatsapp/      # Webhook controller, message handling, outbound sends
prisma/
├── schema.prisma  # Data model
└── migrations/    # SQL migrations
```

### Data model

`Tourist`, `Hotel`, `HotelRoom`, `Clinic`, `ClinicSlot`, `OnCallDoctor`, `Hospital`, `Booking`, `Payment`, `EscrowEvent`, `VisitProof`, `Dispute`, `AiSession`, `TriageLog`

## Getting started

### Prerequisites

- Node.js 22+ and npm
- A PostgreSQL database
- Docker, to run Redis locally
- A Meta WhatsApp Business app with a test phone number
- API keys for Anthropic and OpenAI. Google Cloud credentials too, if you use Google Speech.

### 1. Install

```bash
git clone https://github.com/Arnab2002Pal/tma-backend.git
cd tma-backend
npm install
```

### 2. Configure environment

Create a `.env` file in the project root:

```env
# Server
PORT=3000
BASE_URL=http://localhost:3000        # Public URL encoded into room QR codes

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/tma
DIRECT_URL=postgresql://user:password@localhost:5432/tma

# Redis
REDIS_URL=redis://localhost:6379

# WhatsApp Cloud API
WHATSAPP_ACCESS_TOKEN=
WHATSAPP_PHONE_NUMBER_ID=
WHATSAPP_PHONE_NUMBER=
WHATSAPP_WABA_ID=
WHATSAPP_WEBHOOK_VERIFY_TOKEN=

# AI / speech
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
GOOGLE_CLOUD_PROJECT=
GOOGLE_APPLICATION_CREDENTIALS=./gcp-service-account.json   # optional; enables Google STT
GOOGLE_MAPS_API_KEY=
```

> [!WARNING]
> Never commit `.env` or Google service-account JSON files. Keep them in `.gitignore`.

### 3. Start Redis

```bash
docker compose up -d
```

This starts Redis on `6379` and RedisInsight on [http://localhost:5540](http://localhost:5540).

### 4. Set up the database

```bash
npx prisma migrate dev
npx prisma generate
```

### 5. Run the app

```bash
# development (watch mode)
npm run start:dev

# production
npm run build
npm run start:prod
```

To get a WhatsApp webhook working locally, expose the port with a tunnel such as `ngrok http 3000`. Then point the Meta webhook at `https://<tunnel>/whatsapp/webhook`.

## API

| Method | Path                           | Purpose                                                               |
| ------ | ------------------------------ | --------------------------------------------------------------------- |
| `GET`  | `/`                            | Health check                                                          |
| `GET`  | `/h/:token?room=305`           | Public QR scan target. Validates the hotel token and redirects (302) to WhatsApp. |
| `POST` | `/hotels/:hotelId/qr/generate` | Creates or updates rooms from `{ floors: [{ floorNumber, roomCount }] }` and returns the QR PDF |
| `GET`  | `/hotels/:hotelId/qr/pdf`      | Downloads the QR PDF again for an existing hotel                      |
| `GET`  | `/whatsapp/webhook`            | Meta webhook verification                                             |
| `POST` | `/whatsapp/webhook`            | Incoming WhatsApp messages                                            |
| `POST` | `/seed/hotel`                  | Seed a demo hotel                                                     |
| `POST` | `/seed/clinic`                 | Seed demo clinics                                                     |
| `POST` | `/seed/all`                    | Seed everything                                                       |
| `GET`  | `/seed/status`                 | Show what has been seeded                                             |

> [!NOTE]
> The QR admin and seed endpoints have no authentication yet. Restrict them at the network level until admin auth is added.

## Testing

```bash
npm run test        # unit tests
npm run test:e2e    # end-to-end tests
npm run test:cov    # coverage
```

## Deployment

The app ships as a multi-stage Docker image (`node:22-alpine`). The image generates the Prisma client at build time. On ECS, `docker-entrypoint.sh` decodes `GCP_SERVICE_ACCOUNT_B64` into a key file for Google STT and then starts `dist/main.js`.

On every push to `staging`, the GitHub Actions workflow in `.github/workflows/staging.yml`:

1. builds the image and pushes it to **Amazon ECR**,
2. injects secrets from GitHub Secrets into the ECS task definition,
3. deploys the new revision to **Amazon ECS**.

## Branching

| Branch      | Purpose                                |
| ----------- | -------------------------------------- |
| `main`      | Stable releases                        |
| `staging`   | Auto-deployed to the staging ECS service |
| `dev`       | Integration branch for features        |
| `Feature/*` | Individual features, merged into `dev` via PR |

## License

UNLICENSED. Private project. All rights reserved.
