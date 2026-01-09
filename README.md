# Loman AI - 24/7 AI Phone Answering System for Restaurants

![Loman AI](https://img.shields.io/badge/version-1.0.0-blue.svg)
![License](https://img.shields.io/badge/license-MIT-green.svg)

Loman AI is a production-ready, LLM-agnostic SaaS platform that provides 24/7 AI-powered phone answering for restaurants. Handle takeout orders, reservations, FAQs, and seamlessly escalate to human staff when needed.

## Features

- **🤖 LLM-Agnostic**: Works with OpenAI, Anthropic, Gemini, Azure OpenAI, and local models (Llama/Mistral via Ollama)
- **📞 24/7 Call Handling**: Answer inbound calls, take orders, manage reservations
- **🍔 Takeout Orders**: Create orders with confirmation via SMS and staff notifications
- **📅 Reservations & Waitlist**: Book tables with voice and SMS confirmation
- **❓ FAQ Handling**: Hours, address, directions, menu questions
- **👤 Human Escalation**: Transfer to staff when requested or confidence is low
- **📊 Call Analytics**: Transcripts, recordings, and outcomes in dashboard

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Twilio                                      │
│                    (Inbound Calls / Media Streams)                       │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
        ┌───────────────────┐           ┌───────────────────┐
        │   Call Gateway    │           │   Python API      │
        │      (Rust)       │◄─────────►│   (FastAPI)       │
        │                   │   HTTP    │                   │
        │ • WebSocket Server│           │ • LLM Adapter     │
        │ • STT/TTS Pipeline│           │ • Tool Executor   │
        │ • Conversation SM │           │ • REST API        │
        │ • Barge-in Support│           │ • Webhooks        │
        └───────────────────┘           └───────────────────┘
                                                │
                                    ┌───────────┴───────────┐
                                    ▼                       ▼
                            ┌─────────────┐         ┌─────────────┐
                            │  PostgreSQL │         │    Redis    │
                            │  (Data)     │         │  (Jobs/Cache)│
                            └─────────────┘         └─────────────┘
                                    ▲
                                    │
                            ┌───────┴───────┐
                            │   Dashboard   │
                            │   (React)     │
                            └───────────────┘
```

## Tech Stack

| Component | Technology |
|-----------|------------|
| Call Gateway | Rust (Tokio, Axum, Tungstenite) |
| API Backend | Python (FastAPI, SQLAlchemy, Celery) |
| Dashboard | React + Vite + TypeScript + Tailwind |
| Database | PostgreSQL |
| Queue/Cache | Redis |
| Telephony | Twilio |
| STT | Deepgram (default) / Whisper (optional) |
| TTS | ElevenLabs (default) / Piper (optional) |

## Quick Start

### Prerequisites

- Docker & Docker Compose
- ngrok (for local development with Twilio)
- Twilio account with phone number
- API keys for LLM provider(s)

### 1. Clone and Setup

```bash
cd loman-ai
cp .env.example .env
# Edit .env with your API keys
```

### 2. Start Services

```bash
docker-compose up -d
```

### 3. Run Database Migrations

```bash
docker-compose exec api alembic upgrade head
```

### 4. Seed Demo Data

```bash
docker-compose exec api python scripts/seed_demo.py
```

### 5. Expose Webhooks (Development)

```bash
# Terminal 1: Expose API for Twilio webhooks
ngrok http 8000

# Terminal 2: Expose Rust gateway for media streams
ngrok http 8080
```

### 6. Configure Twilio

1. Go to Twilio Console > Phone Numbers
2. Set Voice Webhook URL: `https://YOUR_NGROK_URL/webhooks/twilio/voice`
3. Set Status Callback URL: `https://YOUR_NGROK_URL/webhooks/twilio/status`

### 7. Access Dashboard

Open http://localhost:3000

Default credentials:
- Email: `admin@loman.ai`
- Password: `admin123`

### 8. Place a Test Call

Call your Twilio phone number and try:
- "I'd like to order a pizza"
- "I want to make a reservation for 4 people tonight at 7pm"
- "What are your hours?"
- "Can I speak to a manager?"

## Project Structure

```
loman-ai/
├── services/
│   ├── call-gateway-rust/     # Real-time call handling
│   ├── api-python/            # Backend API & LLM adapter
│   └── dashboard-react/       # Admin dashboard
├── infra/
│   ├── docker-compose.yml
│   └── nginx.conf
├── docs/
│   ├── architecture.md
│   └── runbook.md
├── .env.example
└── README.md
```

## Configuration

See `.env.example` for all configuration options. Key settings:

| Variable | Description |
|----------|-------------|
| `DATABASE_URL` | PostgreSQL connection string |
| `REDIS_URL` | Redis connection string |
| `TWILIO_ACCOUNT_SID` | Twilio account SID |
| `TWILIO_AUTH_TOKEN` | Twilio auth token |
| `DEEPGRAM_API_KEY` | Deepgram API key for STT |
| `ELEVENLABS_API_KEY` | ElevenLabs API key for TTS |
| `OPENAI_API_KEY` | OpenAI API key |
| `ANTHROPIC_API_KEY` | Anthropic API key |
| `GEMINI_API_KEY` | Google Gemini API key |
| `OLLAMA_BASE_URL` | Ollama server URL for local models |

## LLM Provider Selection

Each tenant can configure their preferred LLM provider in the dashboard:

- **OpenAI**: GPT-4, GPT-3.5-turbo
- **Anthropic**: Claude 3 Opus, Sonnet, Haiku
- **Google**: Gemini Pro, Gemini Flash
- **Azure OpenAI**: Any deployed model
- **Ollama**: Llama 2, Mistral, etc.

Fallback providers are automatically used if the primary fails.

## API Documentation

Once running, access the API docs at:
- Swagger UI: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc

## Testing

```bash
# Python tests
docker-compose exec api pytest

# Rust tests
docker-compose exec call-gateway cargo test

# React tests
docker-compose exec dashboard npm test
```

## Production Deployment

See `docs/runbook.md` for production deployment instructions including:
- Kubernetes manifests
- SSL/TLS configuration
- Monitoring setup
- Scaling considerations

## License

MIT License - see LICENSE file for details.

