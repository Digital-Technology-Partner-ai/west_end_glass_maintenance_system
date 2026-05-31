# West End Glass Backend API

The FastAPI backend for the West End Glass Maintenance System handles message processing, ticket routing, Claude AI integration, and webhook handling for Meta's WhatsApp Cloud API.

## Quick Start

### 1. Install Dependencies

```bash
cd backend_api
pip install -r requirements.txt
```

### 2. Configure Environment

Copy `.env` template and add your credentials:

```bash
cp .env.example .env  # or manually create with the variables below
```

Required environment variables:

```env
# Database
MONGODB_URL=mongodb://localhost:27017
MONGODB_DB_NAME=maintenance_system

# Admin account (auto-created on startup)
ADMIN_USERNAME=admin
ADMIN_PASSWORD=your_secure_password

# WhatsApp / Meta Cloud API
META_WHATSAPP_TOKEN=your_permanent_access_token_here
META_PHONE_NUMBER_ID=your_phone_number_id_here
META_VERIFY_TOKEN=your_webhook_verify_token_here

# Claude AI
ANTHROPIC_API_KEY=your_anthropic_api_key_here

# CORS
CORS_ORIGINS=http://localhost:3030,http://localhost:3000

# Feature flags
SIMULATE_MODE_ENABLED=true
```

### 3. Run with Docker Compose

```bash
# From project root
docker compose up -d
```

### 4. Access the API

- **OpenAPI Docs:** http://localhost:8000/docs
- **Health Check:** http://localhost:8000/health
- **API Base:** http://localhost:8000

## API Endpoints

### Authentication

- **POST** `/auth/login` — Admin login (username + password)
- **POST** `/auth/technician/login` — Technician login (phone + PIN)
- **GET** `/auth/me` — Get current user info

### Tickets (Require Admin Auth)

- **GET** `/tickets` — List all tickets (filter by status, machine, technician)
- **POST** `/tickets` — Create new ticket
- **GET** `/tickets/{id}` — Get ticket details
- **PATCH** `/tickets/{id}` — Update ticket
- **GET** `/tickets/{id}/messages` — Get conversation history
- **GET** `/tickets/{id}/photos` — Get attached photos

### Users (Require Admin Auth)

- **GET** `/users` — List all users
- **POST** `/users` — Create new user
- **PATCH** `/users/{phone}` — Update user
- **DELETE** `/users/{phone}` — Deactivate user
- **POST** `/users/{phone}/set-pin` — Set technician PIN

### Machines (Require Admin Auth)

- **GET** `/machines` — List all machines
- **POST** `/machines` — Create new machine
- **PATCH** `/machines/{id}` — Update machine
- **DELETE** `/machines/{id}` — Delete machine

### Messages (Require Admin Auth)

- **GET** `/messages` — List all messages
- **GET** `/messages?ticket_id={id}` — Get messages for ticket

### Audit Log (Require Admin Auth)

- **GET** `/audit` — List all audit events (paginated)

### Webhooks (No Auth)

- **POST** `/webhook` — Meta Cloud API webhook handler
- **GET** `/webhook` — Webhook verification endpoint

### Simulator (No Auth - Same as Live WhatsApp)

- **GET** `/simulate/users` — Get list of active technicians
- **POST** `/simulate/message` — Send simulated WhatsApp message

## Testing

### CLI Simulator (Recommended for Development)

Test the WhatsApp bot without needing a real WhatsApp account:

```bash
# Interactive mode
python ../tools/simulate_whatsapp_chat.py

# One-shot test
python ../tools/simulate_whatsapp_chat.py --phone +15551234567 --message "WEG-MACHINE-0042"
```

See [../tools/README.md](../tools/README.md) for complete documentation.

### Unit Tests

```bash
# Run all tests
pytest

# Run specific test file
pytest tests/test_tickets.py -v

# Run with coverage
pytest --cov=app tests/
```

### Integration Tests

```bash
# Requires running backend
docker compose up -d
python ../tools/simulate_whatsapp_chat.py --phone +15551234567 --message "WEG-MACHINE-0042"
```

## Architecture

### Message Processing Pipeline

```
[Inbound WhatsApp Message or Simulated Message]
        │
        ▼
[FastAPI Endpoint (/webhook/whatsapp or /simulate/message)]
        │
        ▼
[message_processor.process_inbound_message()]
        │
        ├─ Auth Check (phone whitelist)
        ├─ Last Activity Update
        ├─ Message Persistence
        ├─ Ticket Routing (machine ID or active ticket)
        ├─ Conversation History Loading
        ├─ Claude Agent Loop (run_agent_loop)
        ├─ Response Message Persistence
        ├─ Audit Log Entry
        └─ Return Bot Response
        │
        ▼
[Meta Cloud API sends to WhatsApp] OR [CLI displays response]
```

### Directory Structure

```
backend_api/
├── app/
│   ├── main.py                 # FastAPI app, router registration
│   ├── auth.py                 # JWT, password hashing, auth dependencies
│   ├── config.py               # Settings, environment variables
│   ├── database.py             # MongoDB connection
│   ├── models/                 # Pydantic models
│   │   ├── ticket.py
│   │   ├── user.py
│   │   ├── message.py
│   │   ├── admin.py
│   │   ├── machine.py
│   │   └── audit.py
│   ├── routers/                # API endpoints
│   │   ├── admin_auth.py       # /auth/login
│   │   ├── tech_auth.py        # /auth/technician/login
│   │   ├── webhook.py          # /webhook/whatsapp
│   │   ├── simulate.py         # /simulate/message
│   │   ├── tickets.py          # /tickets
│   │   ├── users.py            # /users
│   │   ├── machines.py         # /machines
│   │   ├── messages.py         # /messages
│   │   ├── photos.py           # /photos
│   │   ├── audit.py            # /audit
│   │   ├── tech_tickets.py     # /tech/my-tickets
│   │   ├── tech_auth.py        # technician auth
│   │   ├── admins.py           # /admins
│   │   └── dashboard.py        # /dashboard
│   ├── services/               # Business logic
│   │   ├── message_processor.py    # Core pipeline
│   │   ├── claude_agent.py         # Claude AI integration
│   │   ├── ticket_service.py       # Ticket operations
│   │   ├── audit_service.py        # Logging
│   │   ├── whatsapp.py             # Meta API integration
│   │   └── message_processor.py    # Message processing
│   └── utils/
│       └── webhook_verify.py   # Meta signature verification
├── tests/
│   ├── conftest.py             # Pytest fixtures
│   ├── test_auth.py
│   ├── test_tickets.py
│   ├── test_users.py
│   ├── test_machines.py
│   └── test_messages.py
├── requirements.txt
├── Dockerfile
└── README.md (this file)
```

## Development Workflow

1. **Start the system:**
   ```bash
   docker compose up -d
   ```

2. **Test bot with simulator:**
   ```bash
   python ../tools/simulate_whatsapp_chat.py
   ```

3. **Monitor logs:**
   ```bash
   docker compose logs -f fastapi-app
   ```

4. **Check database:**
   ```bash
   docker compose exec mongo mongosh
   use maintenance_system
   db.tickets.find()
   ```

5. **Make code changes:** Edit files in `app/`

6. **Backend auto-reloads** (in development mode)

7. **Run tests:**
   ```bash
   pytest tests/
   ```

## Common Tasks

### Create Admin User

The first admin is auto-created on startup with credentials from `.env`:

```env
ADMIN_USERNAME=admin
ADMIN_PASSWORD=your_password
```

Login at: http://localhost:3030

### Create Technician User

Via Admin Dashboard or API:

```bash
curl -X POST http://localhost:8000/users \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "phone_number": "+15551234567",
    "name": "John Smith",
    "active": true,
    "language": "en"
  }'
```

### Create Machine

Via Admin Dashboard or API:

```bash
curl -X POST http://localhost:8000/machines \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "machine_id": "WEG-MACHINE-0042",
    "name": "Test Machine",
    "location": "Field"
  }'
```

### Send Test Message via Simulator

```bash
python ../tools/simulate_whatsapp_chat.py --phone +15551234567 --message "WEG-MACHINE-0042"
```

## Debugging

### Enable Debug Logging

```python
# In app/main.py
import logging
logging.basicConfig(level=logging.DEBUG)
```

### Check Claude API

```bash
# Verify API key is set
docker compose exec fastapi-app printenv | grep ANTHROPIC

# Check Claude calls in logs
docker compose logs fastapi-app | grep -i claude
```

### Monitor Message Flow

```bash
# Watch logs in real-time
docker compose logs -f fastapi-app | grep "message"
```

### Database Queries

```bash
# Connect to MongoDB
docker compose exec mongo mongosh

# Query tickets
use maintenance_system
db.tickets.find({"machine_id": "WEG-MACHINE-0042"})

# Query messages for ticket
db.messages.find({"ticket_id": ObjectId("...")})

# Check audit log
db.audit_logs.find().sort({timestamp: -1}).limit(10)
```

## Deployment

### Production Considerations

1. **Environment Variables:** Use secure secret management (not .env)
2. **Database:** Use managed MongoDB (not local docker)
3. **API Security:** Enable rate limiting, CORS restrictions
4. **Logging:** Stream to centralized logging service
5. **Monitoring:** Set up alerts for errors/timeouts
6. **Claude API:** Monitor costs and usage

### Docker Image Build

```bash
docker build -t west-end-glass-api:latest .
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| MongoDB connection error | Ensure mongo container is running: `docker compose ps` |
| 401 Unauthorized | Check JWT token in Authorization header |
| 500 Claude API error | Verify ANTHROPIC_API_KEY in .env |
| Webhook not firing | Check `META_VERIFY_TOKEN` in `backend_api/.env` matches the Meta webhook verify token |
| Simulator no technicians | Create user in Admin Dashboard with active=true |

## Resources

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Motor (Async MongoDB Driver)](https://motor.readthedocs.io/)
- [Pydantic](https://docs.pydantic.dev/)
- [PyJWT](https://pyjwt.readthedocs.io/)
- [Claude API](https://docs.anthropic.com/)
- [Meta WhatsApp Cloud API](https://developers.facebook.com/docs/whatsapp/cloud-api/)

## Support

For issues or questions:

1. Check the logs: `docker compose logs fastapi-app`
2. Review [../QUICKSTART.md](../QUICKSTART.md) for setup help
3. See [../TESTING.md](../TESTING.md) for testing guide
4. Check [../tools/README.md](../tools/README.md) for simulator docs
