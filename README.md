# ChartDB - Database Design with Sync

A fork of ChartDB with backend synchronization support. Design your database schemas with a modern visual editor and sync across devices.

## Features

- **Visual Database Designer** - Modern React UI for designing database schemas
- **Local-First** - IndexedDB for instant local storage and offline support
- **Backend Sync** - Rust backend for data persistence and multi-device sync
- **Real-time** - Automatic synchronization with 2-second debounce
- **Multiple Databases** - Support for PostgreSQL, MySQL, SQLite, and more

## Architecture

```mermaid
graph TB
    subgraph "Frontend (React + Vite)"
        A[UI Components]
        B[IndexedDB<br/>Local Cache]
    end
    
    subgraph "Backend (Rust + Axum)"
        C[REST API]
        D[PostgreSQL<br/>Persistent Storage]
    end
    
    A <-->|Auto Sync| B
    B <-->|HTTP| C
    C <-->|SQL| D
    
    style A fill:#e1f5ff
    style B fill:#fff4e1
    style C fill:#ffe1e1
    style D fill:#e1f5ff
```

## Quick Start

### Option 1: Docker Compose (Recommended)

```bash
# Clone the repository
git clone <your-repo-url>
cd chartdb

# Start all services (frontend, backend, database)
docker-compose up -d

# Access the application
open http://localhost:8080
```

### Option 2: Local Development

**Prerequisites:**
- Node.js 18+
- Rust 1.86+
- PostgreSQL 15+

**Start Backend:**
```bash
cd chartdb-backend

# Setup database
createdb chartdb

# Run migrations
cargo install sqlx-cli
sqlx migrate run

# Start server
cargo run --release
```

**Start Frontend:**
```bash
cd chartdb-frontend

# Install dependencies
npm install

# Start dev server
npm run dev
```

Access: http://localhost:5173

## Sync Architecture

### Push Flow (Frontend → Backend)

```mermaid
sequenceDiagram
    participant F as Frontend<br/>(React)
    participant IDB as IndexedDB<br/>(Local)
    participant B as Backend<br/>(Rust)
    participant PG as PostgreSQL

    F->>IDB: User edits → Save immediately
    F->>F: Debounce 2s
    F->>B: POST /api/sync/push<br/>{ diagram, tables, relationships }
    B->>PG: BEGIN TRANSACTION
    B->>PG: DELETE old data
    B->>PG: INSERT new data
    B->>PG: COMMIT
    B-->>F: { success: true }
```

### Pull Flow (Backend → Frontend)

```mermaid
sequenceDiagram
    participant F as Frontend<br/>(React)
    participant B as Backend<br/>(Rust)
    participant PG as PostgreSQL
    participant IDB as IndexedDB<br/>(Local)

    F->>B: GET /api/sync/pull/:id
    
    alt Diagram exists on server
        B->>PG: SELECT diagram + tables + relationships
        PG-->>B: Full diagram data
        B-->>F: Diagram JSON
        F->>IDB: Update local cache
    else Not found on server
        B-->>F: 404 Not Found
        F->>IDB: Load from local
    end
    
    F-->>F: Display diagram
```

### Architecture Overview

```mermaid
graph LR
    A[Frontend<br/>React] -->|HTTP POST| B[Backend<br/>Rust/Axum]
    A -->|HTTP GET| B
    B -->|SQL| C[(PostgreSQL)]
    
    A -->|Local Cache| D[(IndexedDB)]
    D -.->|Sync| B
    
    style A fill:#e1f5ff
    style B fill:#fff4e1
    style C fill:#e1f5ff
    style D fill:#e1f5ff
```

## API Endpoints

### Backend REST API

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/sync/push` | Push diagram to server |
| `GET` | `/api/sync/pull/:id` | Pull diagram from server |
| `GET` | `/api/sync/diagrams` | List all diagrams (metadata only) |
| `GET` | `/health` | Health check |

### Request/Response Examples

**Push Diagram:**
```bash
curl -X POST http://localhost:3000/api/sync/push \
  -H "Content-Type: application/json" \
  -d '{
    "diagram": {
      "id": "abc123",
      "name": "My Database",
      "databaseType": "postgresql"
    },
    "tables": [...],
    "relationships": [...]
  }'
```

**Pull Diagram:**
```bash
curl http://localhost:3000/api/sync/pull/abc123
```

**List Diagrams:**
```bash
curl http://localhost:3000/api/sync/diagrams
```

## Configuration

### Backend Environment Variables

Create `.env` file in `chartdb-backend/`:

```env
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/chartdb
PORT=3000
RUST_LOG=info
```

### Frontend Environment Variables

Create `.env` file in `chartdb-frontend/`:

```env
VITE_SYNC_API_URL=http://localhost:3000
VITE_OPENAI_API_KEY=your_key_here  # Optional
```

## Project Structure

```
chartdb/
├── chartdb-backend/          # Rust backend API
│   ├── src/
│   │   ├── main.rs          # Entry point
│   │   ├── handlers.rs      # API handlers
│   │   ├── models.rs        # Data models
│   │   └── routes.rs        # Route definitions
│   ├── migrations/          # SQL migrations
│   ├── Cargo.toml           # Rust dependencies
│   └── Dockerfile
│
├── chartdb-frontend/         # React frontend
│   ├── src/
│   │   ├── context/         # React contexts
│   │   ├── components/      # UI components
│   │   ├── lib/             # Business logic
│   │   └── pages/           # Page components
│   ├── package.json         # Node dependencies
│   └── Dockerfile
│
├── docker-compose.yml        # All services orchestration
├── DOCKER_SETUP.md          # Detailed Docker guide
└── README.md                # This file
```

## Development

### Frontend Development

```bash
cd chartdb-frontend

# Run dev server
npm run dev

# Run tests
npm test

# Build for production
npm run build

# Lint
npm run lint
```

### Backend Development

```bash
cd chartdb-backend

# Run in debug mode
cargo run

# Run in release mode
cargo run --release

# Run tests
cargo test

# Format code
cargo fmt

# Check code
cargo clippy
```

## Database Schema

The backend uses PostgreSQL with the following main tables:

- `diagrams` - Diagram metadata (id, name, database type, timestamps)
- `tables` - Database tables in diagrams
- `relationships` - Foreign key relationships between tables
- `dependencies` - Table dependencies
- `areas` - Visual grouping areas on canvas
- `custom_types` - Custom data types (enums, composites)
- `notes` - Diagram annotations and notes

All tables include `diagram_id` for multi-tenant support.

## Sync Behavior

### When Sync Happens

1. **Push (Frontend → Backend)**
   - Automatically after 2 seconds of inactivity
   - On page unload (using `navigator.sendBeacon`)
   - Only when changes are detected

2. **Pull (Backend → Frontend)**
   - When opening a diagram (loads latest from server)
   - On initial app load (syncs diagram list)
   - Manual refresh

### Conflict Resolution

Currently uses "last write wins" strategy:
- Frontend always pushes complete diagram state
- Backend replaces all diagram data on push
- No merge conflicts (full replace)

Future improvements could include:
- Operational Transformation (OT)
- Conflict-free Replicated Data Types (CRDTs)
- Manual conflict resolution UI

## Troubleshooting

### Backend won't start
- Check PostgreSQL is running: `pg_isready`
- Verify DATABASE_URL in `.env`
- Run migrations: `sqlx migrate run`
- Check port 3000 is free: `lsof -i :3000`

### Frontend can't connect to backend
- Check backend is running: `curl http://localhost:3000/health`
- Verify VITE_SYNC_API_URL in `.env`
- Check browser console for CORS errors
- Ensure backend URL is correct

### Data not syncing
- Open browser DevTools → Console
- Look for sync errors (🔄 or ❌ emoji logs)
- Check backend logs: `docker-compose logs backend`
- Verify network tab shows API calls

### Docker issues
- Clean restart: `docker-compose down -v && docker-compose up -d`
- Check logs: `docker-compose logs -f`
- Verify ports are free: `lsof -i :3000,5432,8080`
- Remove volumes: `docker volume prune`

### Database issues
- Reset database: `dropdb chartdb && createdb chartdb`
- Rerun migrations: `sqlx migrate run`
- Check PostgreSQL logs: `docker-compose logs postgres`

## Performance

- **Local-first**: IndexedDB provides instant UI updates
- **Debounced sync**: Reduces server load with 2-second debounce
- **Efficient queries**: Bulk operations with transactions
- **Connection pooling**: SQLx handles concurrent requests

## Security Considerations

**Current Implementation** (Development):
- No authentication (open access)
- No authorization (any user can modify any diagram)
- No data encryption in transit (HTTP)
- No rate limiting

**Production Recommendations**:
- Add user authentication (OAuth, JWT)
- Implement per-diagram permissions
- Use HTTPS/TLS for all connections
- Add rate limiting and API keys
- Implement data validation and sanitization
- Add audit logging

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Run tests and linting
5. Commit your changes (`git commit -m 'Add amazing feature'`)
6. Push to the branch (`git push origin feature/amazing-feature`)
7. Open a Pull Request

## License

Same as the original ChartDB project.

## Credits

This is a fork of [ChartDB](https://github.com/chartdb/chartdb) with added backend synchronization support.

Original ChartDB by the ChartDB team.
Backend sync implementation by [Your Name/Team].

## Additional Documentation

This project includes additional documentation:
- **DOCKER_SETUP.md** - Detailed Docker deployment guide with architecture diagrams
- **chartdb-backend/README.md** - Backend-specific development guide
- **chartdb-frontend/README.md** - Original ChartDB frontend documentation

