# Docker Deployment

## Container Images and Docker Compose

For development, testing, or small deployments, Sensei can run via Docker Compose. For production, we recommend Kubernetes.

---

### Container Images

All images are available from the Sensei registry:

| Image                                    | Description              | Size    |
| ---------------------------------------- | ------------------------ | ------- |
| `registry.sensei.ai/sensei/api`          | API server               | ~500 MB |
| `registry.sensei.ai/sensei/dashboard`    | Web UI                   | ~200 MB |
| `registry.sensei.ai/sensei/orchestrator` | Workflow engine          | ~400 MB |
| `registry.sensei.ai/sensei/maba`         | Transformation engine    | ~1.2 GB |
| `registry.sensei.ai/sensei/kora`         | Validation engine        | ~800 MB |
| `registry.sensei.ai/sensei/amani`        | Conversational interface | ~600 MB |
| `registry.sensei.ai/sensei/worker`       | Worker agent             | ~400 MB |
| `registry.sensei.ai/sensei/agent`        | Generic agent            | ~300 MB |

---

### Authentication

```bash
# Login to Sensei registry
docker login registry.sensei.ai \
  --username ${SENSEI_REGISTRY_USER} \
  --password ${SENSEI_REGISTRY_TOKEN}
```

---

### Docker Compose (Development)

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  # Dependencies
  postgresql:
    image: postgres:16
    environment:
      POSTGRES_DB: sensei
      POSTGRES_USER: sensei
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U sensei']
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 5s
      timeout: 5s
      retries: 5

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_PASSWORD}
    volumes:
      - minio_data:/data
    ports:
      - '9000:9000'
      - '9001:9001'

  weaviate:
    image: semitechnologies/weaviate:1.24
    environment:
      QUERY_DEFAULTS_LIMIT: 25
      AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: 'true'
      PERSISTENCE_DATA_PATH: '/var/lib/weaviate'
    volumes:
      - weaviate_data:/var/lib/weaviate

  # Sensei Services
  api:
    image: registry.sensei.ai/sensei/api:latest
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://sensei:${POSTGRES_PASSWORD}@postgresql:5432/sensei
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      LICENSE_KEY: ${SENSEI_LICENSE_KEY}
      LLM_PROVIDER: anthropic
      LLM_API_KEY: ${ANTHROPIC_API_KEY}
    ports:
      - '8080:8080'

  dashboard:
    image: registry.sensei.ai/sensei/dashboard:latest
    depends_on:
      - api
    environment:
      API_URL: http://api:8080
    ports:
      - '3000:3000'

  orchestrator:
    image: registry.sensei.ai/sensei/orchestrator:latest
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://sensei:${POSTGRES_PASSWORD}@postgresql:5432/sensei
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379

  maba:
    image: registry.sensei.ai/sensei/maba:latest
    depends_on:
      - redis
      - weaviate
    environment:
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      VECTOR_DB_URL: http://weaviate:8080
      LLM_PROVIDER: anthropic
      LLM_API_KEY: ${ANTHROPIC_API_KEY}

  kora:
    image: registry.sensei.ai/sensei/kora:latest
    depends_on:
      - redis
    environment:
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379

  amani:
    image: registry.sensei.ai/sensei/amani:latest
    depends_on:
      - redis
      - api
    environment:
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      API_URL: http://api:8080
      LLM_PROVIDER: anthropic
      LLM_API_KEY: ${ANTHROPIC_API_KEY}

  worker:
    image: registry.sensei.ai/sensei/worker:latest
    depends_on:
      - redis
      - maba
      - kora
    environment:
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
      MABA_URL: http://maba:8081
      KORA_URL: http://kora:8082
    deploy:
      replicas: 4

volumes:
  postgres_data:
  redis_data:
  minio_data:
  weaviate_data:
```

**.env:**

```bash
POSTGRES_PASSWORD=your-secure-password
REDIS_PASSWORD=your-secure-password
MINIO_USER=minioadmin
MINIO_PASSWORD=your-secure-password
SENSEI_LICENSE_KEY=your-license-key
ANTHROPIC_API_KEY=your-api-key
```

---

### Running

```bash
# Start all services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f api

# Scale workers
docker-compose up -d --scale worker=8

# Stop
docker-compose down

# Stop and remove volumes (data loss)
docker-compose down -v
```

---

### Resource Limits

Add resource limits for stability:

```yaml
services:
  maba:
    image: registry.sensei.ai/sensei/maba:latest
    deploy:
      resources:
        limits:
          cpus: '8'
          memory: 16G
        reservations:
          cpus: '4'
          memory: 8G
```

---

### Health Checks

```yaml
services:
  api:
    healthcheck:
      test: ['CMD', 'curl', '-f', 'http://localhost:8080/health']
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

---

### GPU Support (Local LLM)

```yaml
services:
  llm:
    image: registry.sensei.ai/sensei/llm-inference:latest
    environment:
      MODEL: llama-3-8b
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

Requires NVIDIA Container Toolkit installed.

---

### Production Considerations

Docker Compose is **not recommended for production** because:

- No automatic failover
- Limited scaling capabilities
- Manual update process
- No built-in service mesh

For production, use [Kubernetes](kubernetes.md).

---

### Offline / Air-Gapped

```bash
# On connected machine: save images
docker save registry.sensei.ai/sensei/api:latest | gzip > api.tar.gz
docker save registry.sensei.ai/sensei/maba:latest | gzip > maba.tar.gz
# ... repeat for all images

# Transfer to air-gapped machine

# Load images
gunzip -c api.tar.gz | docker load
gunzip -c maba.tar.gz | docker load

# Update docker-compose.yml to use local images
# image: sensei/api:latest (without registry prefix)
```

→ [Kubernetes](kubernetes.md)
→ [Infrastructure Requirements](infrastructure.md)
→ [Deployment Overview](README.md)
