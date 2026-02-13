# OpenMemory Production Deployment

## Prerequisites
- Docker & Docker Compose installed on server
- (Optional) GPU for faster Ollama embeddings
- Domain/subdomain configured (for reverse proxy)

## Deployment Steps

### 1. Clone Your Fork
```bash
git clone https://github.com/jeanpijon/OpenMemory.git
cd OpenMemory
```

### 2. Configure Environment
```bash
cp .env.production.example .env.production
nano .env.production  # Edit with your settings
```

**Required settings:**
- `POSTGRES_PASSWORD` - Strong password for Postgres
- `DOMAIN` - Your domain (e.g., memory.example.com)
- `OM_API_KEY` - Leave empty for VPN/private access, or set for auth

### 3. Start Services
```bash
docker compose -f docker-compose.production.yml --env-file .env.production up -d
```

### 4. Pull Ollama Embedding Model
```bash
# Pull recommended model (1024-dim, good quality)
docker compose -f docker-compose.production.yml exec ollama ollama pull nomic-embed-text

# Or alternative models:
# docker compose -f docker-compose.production.yml exec ollama ollama pull mxbai-embed-large
# docker compose -f docker-compose.production.yml exec ollama ollama pull all-minilm
```

### 5. Verify Deployment
```bash
# Check all services are healthy
docker compose -f docker-compose.production.yml ps

# Test API
curl http://localhost:8080/health

# Test Ollama
docker compose -f docker-compose.production.yml exec ollama ollama list
```

### 6. Access Dashboard
- **Dashboard**: http://your-domain:3000
- **API**: http://your-domain:8080
- **MCP**: http://your-domain:8080/mcp

## Configuration

### DEEP Tier with Ollama
```bash
OM_TIER=deep
OM_EMBEDDINGS=ollama
OM_VEC_DIM=1024  # Matches nomic-embed-text dimensions
```

**Expected Performance:**
- Recall: ~95-100%
- QPS: 350-400
- RAM: ~1.6GB per 10k memories + ~2GB for Ollama model

### Postgres Backend
```bash
OM_METADATA_BACKEND=postgres
OM_VECTOR_BACKEND=postgres
```

**Benefits:**
- Better concurrency (multiple writers)
- ACID guarantees
- Easy backups: `docker compose -f docker-compose.production.yml exec postgres pg_dump -U openmemory_user openmemory > backup.sql`

## MCP Access (Remote)

### Via Tunnel
If using SSH tunnel or Tailscale/WireGuard:
```bash
claude mcp add --transport http openmemory http://your-server:8080/mcp
```

### With Authentication
If `OM_API_KEY` is set:
```bash
claude mcp add --transport http openmemory http://your-server:8080/mcp \
  --header "Authorization: Bearer YOUR_API_KEY"
```

## Maintenance

### View Logs
```bash
docker compose -f docker-compose.production.yml logs -f openmemory
docker compose -f docker-compose.production.yml logs -f ollama
```

### Backup Database
```bash
# Postgres backup
docker compose -f docker-compose.production.yml exec postgres \
  pg_dump -U openmemory_user -Fc openmemory > openmemory_backup.dump

# Restore
docker compose -f docker-compose.production.yml exec -T postgres \
  pg_restore -U openmemory_user -d openmemory < openmemory_backup.dump
```

### Update Deployment
```bash
git pull origin main
docker compose -f docker-compose.production.yml up -d --build
```

## Resource Requirements

**Minimum:**
- 4 CPU cores
- 8GB RAM (2GB Postgres + 2GB Ollama + 2GB OpenMemory + 2GB system)
- 20GB disk

**Recommended:**
- 8+ CPU cores
- 16GB RAM
- 50GB disk (with room for growth)
- GPU (optional, for faster Ollama embeddings)

## Troubleshooting

### Ollama Not Starting
```bash
# Check if GPU is available
docker compose -f docker-compose.production.yml exec ollama nvidia-smi

# If no GPU, remove GPU configuration from docker-compose.production.yml:
# Comment out the deploy.resources.reservations section
```

### Postgres Connection Issues
```bash
# Check Postgres is healthy
docker compose -f docker-compose.production.yml exec postgres pg_isready -U openmemory_user

# Check pgvector extension
docker compose -f docker-compose.production.yml exec postgres \
  psql -U openmemory_user -d openmemory -c "SELECT * FROM pg_extension WHERE extname='vector';"
```

### OpenMemory Not Starting
```bash
# Check logs
docker compose -f docker-compose.production.yml logs openmemory

# Verify Ollama has model pulled
docker compose -f docker-compose.production.yml exec ollama ollama list

# Test embedding generation
docker compose -f docker-compose.production.yml exec ollama \
  ollama run nomic-embed-text "test embedding"
```
