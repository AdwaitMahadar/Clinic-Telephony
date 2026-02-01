# Clinic Telephony Service

An AI-powered after-hours voice agent for clinics using Twilio and OpenAI Realtime API.

## 📋 Features

- Voice conversations in Hindi, Marathi, and English
- Appointment booking with Google Calendar integration
- WhatsApp summaries for patients and staff
- FAQ handling (hours, location, fees, services)
- Callback request recording

## 🛠️ Tech Stack

- **Runtime:** Node.js with TypeScript
- **Server:** Fastify
- **Voice:** Twilio Media Streams
- **AI:** OpenAI Realtime API (gpt-4o)
- **Cache:** Redis (ioredis)
- **Integrations:** Google Calendar, Google Docs, WhatsApp Business API

## 📦 Installation

```bash
# Install dependencies
npm install

# Copy environment template
cp .env.example .env

# Edit .env with your credentials
```

## 🔧 Development

```bash
# Run in development mode (with auto-reload)
npm run dev

# Type check
npm run typecheck

# Build for production
npm run build

# Run production build
npm start

# Run tests
npm test
```

## 📁 Project Structure

```
clinic-telephony/
├── src/
│   ├── index.ts              # Entry point
│   ├── config/               # Environment & logging config
│   ├── services/             # Business logic (Google, WhatsApp, Redis)
│   ├── websocket/            # Twilio & OpenAI WebSocket handlers
│   ├── tools/                # OpenAI Realtime tools
│   ├── models/               # TypeScript types & session management
│   └── utils/                # Helpers (phone, audio, datetime)
├── tests/                    # Unit & integration tests
├── docs/                     # Documentation (PRD, TDD, Implementation Plan)
└── package.json
```

## 📚 Documentation

- [Product Requirements Document (PRD)](./docs/PRD.md)
- [Technical Design Document (TDD)](./docs/TDD.md)
- [Implementation Plan](./docs/IMPLEMENTATION-PLAN.md)

## 🔑 Environment Variables

See `.env.example` for all required configuration. Key variables:

- `TWILIO_*` - Twilio account credentials
- `OPENAI_API_KEY` - OpenAI API key
- `REDIS_URL` - Redis connection string
- `GOOGLE_*` - Google service account credentials
- `WHATSAPP_*` - WhatsApp Business API credentials
- `CLINIC_*` - Clinic-specific configuration

## 🚀 Deployment

The service requires:
- WebSocket support (Railway, Render, DigitalOcean, AWS)
- Redis instance (Upstash, Railway Redis, or self-hosted)
- HTTPS/WSS for production

Configure Twilio webhook to point to: `POST https://your-domain.com/twilio/voice`

## 📝 License

MIT
