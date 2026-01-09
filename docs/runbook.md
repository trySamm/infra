# Loman AI - Operations Runbook

## Table of Contents
1. [Local Development](#local-development)
2. [Production Deployment](#production-deployment)
3. [Troubleshooting](#troubleshooting)
4. [Common Operations](#common-operations)

## Local Development

### Prerequisites
- Docker Desktop 4.0+
- Docker Compose 2.0+
- ngrok (for Twilio webhooks)
- Node.js 18+ (for dashboard development)
- Rust 1.75+ (for call gateway development)
- Python 3.11+ (for API development)

### First-Time Setup

```bash
# Clone repository
git clone https://github.com/your-org/loman-ai.git
cd loman-ai

# Copy environment file
cp env.example .env

# Edit .env with your API keys
# Required: TWILIO_*, DEEPGRAM_API_KEY, ELEVENLABS_API_KEY
# At least one LLM provider: OPENAI_API_KEY or ANTHROPIC_API_KEY

# Start all services
docker-compose up -d

# Run database migrations
docker-compose exec api alembic upgrade head

# Seed demo data
docker-compose exec api python scripts/seed_demo.py
```

### Exposing Webhooks

Twilio needs to reach your local services:

```bash
# Terminal 1: API webhooks
ngrok http 8000
# Note the URL: https://xxxx.ngrok.io

# Terminal 2: Media stream (optional, can use same ngrok)
ngrok http 8080
# Note the URL: https://yyyy.ngrok.io
```

### Configuring Twilio

1. Go to [Twilio Console](https://console.twilio.com)
2. Navigate to Phone Numbers > Manage > Active Numbers
3. Click your phone number
4. Under Voice Configuration:
   - Webhook URL: `https://xxxx.ngrok.io/webhooks/twilio/voice`
   - HTTP Method: POST
   - Status Callback URL: `https://xxxx.ngrok.io/webhooks/twilio/status`
5. Save changes

### Development Workflows

#### API Development (Python)
```bash
# Run API with hot reload (outside Docker)
cd services/api-python
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

#### Call Gateway Development (Rust)
```bash
# Run gateway with hot reload
cd services/call-gateway-rust
cargo watch -x run
```

#### Dashboard Development (React)
```bash
# Run dashboard with hot reload
cd services/dashboard-react
npm install
npm run dev
```

## Production Deployment

### Kubernetes Deployment

#### Prerequisites
- Kubernetes cluster (1.25+)
- kubectl configured
- Helm 3.0+
- Container registry access

#### Deploy with Helm

```bash
# Add secrets
kubectl create secret generic loman-secrets \
  --from-env-file=.env.production

# Deploy PostgreSQL (or use managed)
helm install postgres bitnami/postgresql \
  --set auth.postgresPassword=xxx \
  --set primary.persistence.size=100Gi

# Deploy Redis (or use managed)
helm install redis bitnami/redis \
  --set auth.password=xxx

# Deploy Loman AI
helm install loman-ai ./charts/loman-ai \
  --values values.production.yaml
```

#### Sample values.production.yaml

```yaml
api:
  replicas: 3
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 2000m
      memory: 2Gi
  
callGateway:
  replicas: 5
  resources:
    requests:
      cpu: 1000m
      memory: 256Mi
    limits:
      cpu: 4000m
      memory: 1Gi

dashboard:
  replicas: 2
  
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - api.loman.ai
    - app.loman.ai
    - ws.loman.ai
```

### SSL/TLS Configuration

```yaml
# Ingress TLS configuration
tls:
  - secretName: loman-tls
    hosts:
      - api.loman.ai
      - app.loman.ai
      - ws.loman.ai
```

### Database Migrations (Production)

```bash
# Run migrations
kubectl exec -it deploy/loman-api -- alembic upgrade head

# Rollback if needed
kubectl exec -it deploy/loman-api -- alembic downgrade -1
```

## Troubleshooting

### Common Issues

#### 1. Calls Not Connecting

**Symptoms:** Twilio reports webhook errors

**Diagnosis:**
```bash
# Check API logs
docker-compose logs api | grep webhook

# Verify ngrok is running
curl https://your-ngrok-url.ngrok.io/health
```

**Solutions:**
- Verify ngrok URL in Twilio console matches running ngrok
- Check API is running: `docker-compose ps`
- Verify Twilio credentials in .env

#### 2. No Audio / Silent Calls

**Symptoms:** Call connects but no audio

**Diagnosis:**
```bash
# Check call gateway logs
docker-compose logs call-gateway

# Check STT connection
docker-compose logs call-gateway | grep deepgram
```

**Solutions:**
- Verify DEEPGRAM_API_KEY is valid
- Check WebSocket connection to call gateway
- Verify Twilio media stream configuration

#### 3. LLM Errors

**Symptoms:** Agent doesn't respond intelligently

**Diagnosis:**
```bash
# Check LLM adapter logs
docker-compose logs api | grep llm

# Test LLM directly
curl -X POST http://localhost:8000/llm/generate \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Hello"}],"provider":"openai"}'
```

**Solutions:**
- Verify LLM API keys
- Check rate limits on LLM provider
- Verify fallback provider is configured

#### 4. Database Connection Issues

**Symptoms:** API fails to start

**Diagnosis:**
```bash
# Check PostgreSQL
docker-compose logs postgres

# Test connection
docker-compose exec postgres psql -U loman -d loman_ai -c "SELECT 1"
```

**Solutions:**
- Wait for PostgreSQL to be ready
- Verify DATABASE_URL in .env
- Check PostgreSQL disk space

### Log Locations

| Service | Docker Logs | File Logs |
|---------|-------------|-----------|
| API | `docker-compose logs api` | `/var/log/loman/api.log` |
| Call Gateway | `docker-compose logs call-gateway` | `/var/log/loman/gateway.log` |
| Worker | `docker-compose logs worker` | `/var/log/loman/worker.log` |

### Health Checks

```bash
# API health
curl http://localhost:8000/health

# Call Gateway health
curl http://localhost:8080/health

# Full readiness
curl http://localhost:8000/health/ready
```

## Common Operations

### Adding a New Tenant

```bash
# Via API
curl -X POST http://localhost:8000/tenants \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Pizza Palace",
    "timezone": "America/New_York"
  }'

# Via seed script
docker-compose exec api python scripts/add_tenant.py "Pizza Palace"
```

### Importing Menu from CSV

```bash
# Via API
curl -X POST http://localhost:8000/tenants/{tenant_id}/menu_items/import_csv \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -F "file=@menu.csv"
```

CSV format:
```csv
name,description,price_cents,category,is_active
Margherita Pizza,Classic tomato and mozzarella,1499,Pizza,true
Pepperoni Pizza,Pepperoni with mozzarella,1699,Pizza,true
```

### Viewing Call Transcripts

```bash
# Get recent calls
curl http://localhost:8000/tenants/{tenant_id}/calls \
  -H "Authorization: Bearer $TOKEN"

# Get specific transcript
curl http://localhost:8000/tenants/{tenant_id}/calls/{call_id} \
  -H "Authorization: Bearer $TOKEN"
```

### Changing LLM Provider

```bash
# Update tenant LLM config
curl -X PUT http://localhost:8000/tenants/{tenant_id}/llm_config \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "anthropic",
    "model": "claude-3-sonnet-20240229",
    "fallback_provider": "openai",
    "fallback_model": "gpt-4-turbo"
  }'
```

### Database Backup

```bash
# Manual backup
docker-compose exec postgres pg_dump -U loman loman_ai > backup.sql

# Restore
docker-compose exec -T postgres psql -U loman loman_ai < backup.sql
```

### Scaling Services

```bash
# Scale call gateway
docker-compose up -d --scale call-gateway=3

# In Kubernetes
kubectl scale deployment loman-call-gateway --replicas=5
```

## Monitoring Alerts

### Critical Alerts
- API error rate > 1%
- Call Gateway error rate > 0.1%
- P95 latency > 2s
- Database connection pool exhausted
- Redis connection failed

### Warning Alerts
- LLM fallback triggered
- STT/TTS provider errors
- Call abandonment rate > 10%
- Queue depth > 100 jobs

