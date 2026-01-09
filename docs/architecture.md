# Loman AI - Architecture Documentation

## Overview

Loman AI is a multi-tenant SaaS platform providing AI-powered phone answering for restaurants. The system is designed with the following principles:

1. **LLM-Agnostic**: A unified adapter interface abstracts all LLM providers
2. **Low Latency**: Rust handles real-time voice processing
3. **Multi-Tenant**: Complete data isolation per restaurant
4. **Pluggable**: STT, TTS, and LLM providers are swappable

## System Components

### 1. Call Gateway (Rust)

The Rust service handles real-time voice processing with sub-100ms latency requirements.

**Responsibilities:**
- Accept Twilio Media Stream WebSocket connections
- Stream audio to STT provider (Deepgram)
- Manage conversation state machine
- Call Python LLM Adapter for AI responses
- Execute tools via Python Tool Executor
- Stream TTS audio back to caller
- Handle barge-in (interrupt TTS when user speaks)
- Manage call timeouts

**Key Components:**
```
call-gateway-rust/
├── src/
│   ├── main.rs              # Entry point
│   ├── server.rs            # Axum HTTP/WS server
│   ├── session/
│   │   ├── mod.rs           # Call session manager
│   │   ├── state.rs         # Conversation state machine
│   │   └── handler.rs       # Message handlers
│   ├── stt/
│   │   ├── mod.rs           # STT adapter interface
│   │   ├── deepgram.rs      # Deepgram implementation
│   │   └── whisper.rs       # Local Whisper implementation
│   ├── tts/
│   │   ├── mod.rs           # TTS adapter interface
│   │   ├── elevenlabs.rs    # ElevenLabs implementation
│   │   └── piper.rs         # Local Piper implementation
│   ├── llm/
│   │   └── client.rs        # Python LLM adapter client
│   ├── tools/
│   │   └── executor.rs      # Python tool executor client
│   └── twilio/
│       ├── media.rs         # Media stream protocol
│       └── audio.rs         # Audio encoding/decoding
```

**State Machine:**
```
┌─────────────┐
│   IDLE      │
└──────┬──────┘
       │ call_started
       ▼
┌─────────────┐
│  GREETING   │──────────────────────┐
└──────┬──────┘                      │
       │ greeting_complete           │
       ▼                             │
┌─────────────┐                      │
│  LISTENING  │◄─────────────────────┤
└──────┬──────┘                      │
       │ user_utterance              │
       ▼                             │
┌─────────────┐                      │
│  THINKING   │                      │
└──────┬──────┘                      │
       │ response_ready              │
       ▼                             │
┌─────────────┐                      │
│  SPEAKING   │──────────────────────┘
└──────┬──────┘
       │ goodbye / timeout / transfer
       ▼
┌─────────────┐
│   ENDED     │
└─────────────┘
```

### 2. API Backend (Python/FastAPI)

The Python service provides the API layer, LLM adapter, and background jobs.

**Responsibilities:**
- REST API for dashboard
- Twilio webhooks (call initiation, status)
- LLM Adapter interface (provider-neutral)
- Tool executor endpoints
- Background jobs (transcripts, SMS, summaries)
- Database operations

**Key Components:**
```
api-python/
├── app/
│   ├── main.py              # FastAPI app
│   ├── config.py            # Configuration
│   ├── database.py          # SQLAlchemy setup
│   ├── models/              # Database models
│   │   ├── tenant.py
│   │   ├── call.py
│   │   ├── order.py
│   │   └── ...
│   ├── schemas/             # Pydantic schemas
│   ├── api/
│   │   ├── auth.py          # Authentication endpoints
│   │   ├── tenants.py       # Tenant management
│   │   ├── calls.py         # Call history
│   │   ├── menu.py          # Menu management
│   │   ├── orders.py        # Order management
│   │   └── reservations.py  # Reservation management
│   ├── webhooks/
│   │   └── twilio.py        # Twilio webhooks
│   ├── llm/
│   │   ├── adapter.py       # Unified LLM interface
│   │   ├── providers/
│   │   │   ├── openai.py
│   │   │   ├── anthropic.py
│   │   │   ├── gemini.py
│   │   │   └── ollama.py
│   │   └── schemas.py       # Provider-neutral schemas
│   ├── tools/
│   │   ├── router.py        # Tool endpoints
│   │   ├── context.py       # GetRestaurantContext
│   │   ├── menu.py          # SearchMenu
│   │   ├── orders.py        # CreateOrder
│   │   ├── reservations.py  # CreateReservation
│   │   └── sms.py           # SendSMS
│   └── jobs/
│       ├── celery_app.py
│       ├── transcripts.py
│       └── notifications.py
├── alembic/                 # Database migrations
└── tests/
```

### 3. Dashboard (React)

The React dashboard provides the admin interface.

**Pages:**
- Login
- Tenants list (SuperAdmin)
- Restaurant settings
- LLM configuration
- Menu manager
- Call history
- Orders
- Reservations

### 4. LLM Adapter Interface

The LLM Adapter provides a unified interface for all LLM providers.

**Request Schema:**
```json
{
  "system_prompt": "You are a restaurant receptionist...",
  "messages": [
    {"role": "user", "content": "I'd like to order a pizza"},
    {"role": "assistant", "content": "I'd be happy to help..."}
  ],
  "tools": [
    {
      "name": "search_menu",
      "description": "Search the restaurant menu",
      "parameters": {
        "type": "object",
        "properties": {
          "query": {"type": "string"}
        }
      }
    }
  ],
  "provider": "openai",
  "model": "gpt-4-turbo",
  "temperature": 0.7,
  "tenant_id": "uuid"
}
```

**Response Schema:**
```json
{
  "type": "text|tool_call",
  "content": "Here's what I found...",
  "tool_call": {
    "name": "search_menu",
    "arguments": {"query": "pizza"}
  },
  "usage": {
    "prompt_tokens": 150,
    "completion_tokens": 50
  },
  "confidence": 0.95,
  "provider": "openai",
  "model": "gpt-4-turbo"
}
```

### 5. Tool System

Tools are executed via the Python backend. The Rust gateway calls these endpoints when the LLM requests a tool.

**Available Tools:**

| Tool | Description | Endpoint |
|------|-------------|----------|
| GetRestaurantContext | Get hours, address, policies | POST /tools/get_context |
| SearchMenu | Search menu items | POST /tools/search_menu |
| CreateOrder | Create a takeout order | POST /tools/create_order |
| CreateReservation | Book a reservation | POST /tools/create_reservation |
| GetAvailability | Check table availability | POST /tools/get_availability |
| SendSMS | Send SMS to customer | POST /tools/send_sms |
| TransferCall | Transfer to human | POST /tools/transfer_call |
| CreateTicket | Create support ticket | POST /tools/create_ticket |

## Data Model

```
┌──────────────┐       ┌──────────────────────┐
│   tenants    │───────│  restaurant_settings │
└──────────────┘       └──────────────────────┘
       │
       ├───────────────┬───────────────┬───────────────┐
       │               │               │               │
       ▼               ▼               ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│phone_numbers│ │ menu_items  │ │staff_contacts│ │   calls     │
└─────────────┘ └─────────────┘ └─────────────┘ └──────┬──────┘
                       │                               │
                       ▼                    ┌──────────┼──────────┐
               ┌─────────────┐              │          │          │
               │menu_modifiers│             ▼          ▼          ▼
               └─────────────┘       ┌──────────┐ ┌────────┐ ┌──────────┐
                                     │transcripts│ │ orders │ │reservations│
                                     └──────────┘ └────────┘ └──────────┘
```

## Security

### Authentication
- JWT tokens with RS256 signing
- Access tokens (15 min) + Refresh tokens (7 days)
- Token rotation on refresh

### Authorization
- Role-based access control (RBAC)
- Roles: SuperAdmin, RestaurantAdmin, StaffViewer
- Per-tenant scoping enforced at query level

### Data Protection
- Phone numbers masked in UI (show last 4 digits)
- Recording consent announcement required
- Audit logging for sensitive operations

## Voice Pipeline

### Inbound Call Flow

```
1. Twilio receives call
   └─> POST /webhooks/twilio/voice
       └─> Create call record
       └─> Return TwiML (connect to media stream)

2. Twilio connects media stream
   └─> WebSocket to Rust gateway
       └─> Create call session
       └─> Play greeting via TTS

3. Conversation loop
   └─> Receive audio frames
       └─> Stream to STT
           └─> On utterance complete
               └─> Send to Python /llm/generate
                   └─> If tool_call: execute tool, re-generate
                   └─> If text: send to TTS
                       └─> Stream audio to caller

4. Call ends
   └─> Twilio status callback
       └─> Finalize call record
       └─> Trigger background jobs (transcript, summary)
```

### Barge-In Support

When the system detects user speech during TTS playback:
1. Immediately stop TTS audio stream
2. Buffer incoming audio
3. Resume STT processing
4. Continue conversation flow

### Latency Targets

| Component | Target | Actual |
|-----------|--------|--------|
| STT first partial | < 200ms | ~150ms |
| LLM response | < 1000ms | ~800ms |
| TTS first audio | < 300ms | ~200ms |
| Total user-to-audio | < 1500ms | ~1200ms |

## Observability

### Logging
- Structured JSON logs
- Correlation ID: call_id
- Log levels: DEBUG, INFO, WARN, ERROR

### Metrics
- `calls_total`: Total calls received
- `calls_escalated`: Calls transferred to human
- `orders_created`: Orders placed via phone
- `reservations_created`: Reservations booked
- `avg_call_duration_seconds`: Average call length
- `llm_latency_seconds`: LLM response time

### Health Checks
- `/health`: Basic liveness
- `/health/ready`: Readiness (DB, Redis, LLM)

## Scaling Considerations

### Horizontal Scaling
- Call Gateway: Stateless, scale by CPU
- API Backend: Stateless, scale by request rate
- Workers: Scale by job queue depth

### Vertical Scaling
- PostgreSQL: Consider read replicas for high-traffic
- Redis: Cluster mode for high message volume

### Capacity Planning
- 1 Call Gateway instance: ~100 concurrent calls
- 1 API instance: ~1000 req/s
- Storage: ~10MB per call (recording + transcript)

## Disaster Recovery

### Backup Strategy
- PostgreSQL: Daily full backup, continuous WAL archiving
- Recordings: S3 with versioning, cross-region replication

### Failover
- LLM: Automatic failover to backup provider
- Database: Standby replica with automatic promotion
- Redis: Sentinel for HA

