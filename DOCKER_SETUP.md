# ChartDB Docker Setup

Complete Docker Compose setup for ChartDB with backend synchronization.

## Architecture

```
┌─────────────────┐
│   Frontend      │
│   (Vite/React)  │
│   Port: 8080    │
└────────┬────────┘
         │
         │ HTTP API
         ▼
┌─────────────────┐
│   Backend       │
│   (Rust/Axum)   │
│   Port: 3000    │
└────────┬────────┘
         │
         │ SQL
         ▼
┌─────────────────┐
│   PostgreSQL    │
│   Port: 5432    │
└─────────────────┘
```

## Services

### 1. PostgreSQL Database
- **Image**: `postgres:16-alpine`
- **Port**: `5432`
- **Database**: `chartdb`
- **User/Password**: `postgres/postgres`
- **Volume**: `postgres_data` (persistent storage)

### 2. Backend API (Rust)
- **Build**: `./chartdb-backend/Dockerfile`
- **Port**: `3000`
- **Endpoints**:
  - `POST /api/sync/push` - Push diagram to server
  - `GET /api/sync/pull/:id` - Pull diagram from server
  - `GET /api/sync/diagrams` - List all diagrams

### 3. Frontend (React)
- **Build**: `./chartdb-frontend/Dockerfile`
- **Port**: `8080`
- **Features**:
  - IndexedDB for local caching
  - Auto-sync to backend (2s debounce)
  - Pull on diagram load
  - Sync diagram list on open dialog

## Quick Start

### 1. Start all services

```bash
docker-compose up -d
```

### 2. Check logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f backend
docker-compose logs -f frontend
docker-compose logs -f postgres
```

### 3. Access the application

Open browser: http://localhost:8080

### 4. Stop services

```bash
docker-compose down
```

### 5. Stop and remove volumes (clean restart)

```bash
docker-compose down -v
```

## Database Migrations

Migrations are automatically applied when the backend starts.

Located in: `./chartdb-backend/migrations/`
- `001_init.sql` - Create tables
- `002_change_id_to_text.sql` - Change ID columns to TEXT

## Environment Variables

Create a `.env` file in the root directory (optional):

```bash
# OpenAI Configuration (Optional)
VITE_OPENAI_API_KEY=your_key_here
VITE_OPENAI_API_ENDPOINT=
VITE_LLM_MODEL_NAME=

# ChartDB Configuration
VITE_HIDE_CHARTDB_CLOUD=false
VITE_DISABLE_ANALYTICS=false
```

## Troubleshooting

### Backend fails to connect to database
- Check if PostgreSQL is healthy: `docker-compose ps`
- Check backend logs: `docker-compose logs backend`
- Verify DATABASE_URL in docker-compose.yml

### Frontend cannot reach backend
- Verify backend is running: `curl http://localhost:3000/health`
- Check CORS settings in backend
- Ensure VITE_SYNC_API_URL is set correctly

### Database data is lost after restart
- Check if volume is created: `docker volume ls`
- Data persists in `postgres_data` volume
- To reset: `docker-compose down -v`

## Development vs Production

### Development (Current Setup)
- Frontend: Hot reload with Vite dev server
- Backend: Run with `cargo run --release`
- Database: Local PostgreSQL

### Production (Docker)
- Frontend: Built static files served by Nginx
- Backend: Compiled Rust binary
- Database: PostgreSQL in container
- All services orchestrated by Docker Compose

## Port Summary

| Service    | Port | Description                    |
|------------|------|--------------------------------|
| Frontend   | 8080 | Web UI (Nginx)                 |
| Backend    | 3000 | REST API (Axum)                |
| PostgreSQL | 5432 | Database                       |

## Health Checks

- **PostgreSQL**: `pg_isready` every 5s
- **Backend**: Depends on PostgreSQL health
- **Frontend**: Depends on Backend availability

## Data Flow

1. **Create/Edit Diagram** → Frontend (IndexedDB) → Auto-sync (2s delay) → Backend API → PostgreSQL
2. **Open Diagram** → Backend API → Frontend → IndexedDB (cache)
3. **List Diagrams** → Backend API → Frontend → Display in "Open Database" dialog

