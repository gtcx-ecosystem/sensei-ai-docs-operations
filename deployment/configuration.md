# Configuration

## Configuring Sensei for Your Environment

Sensei is configured through a combination of configuration files, environment variables, and runtime options. This guide covers all configuration methods and common configurations.

---

## Configuration Hierarchy

Configuration is loaded in order of precedence (later overrides earlier):

1. **Default values** — Built-in sensible defaults
2. **Configuration files** — `sensei.yaml` or `sensei.json`
3. **Environment variables** — Prefixed with `SENSEI_`
4. **Command-line flags** — Highest precedence

---

## Configuration File

### Location

Sensei looks for configuration files in this order:

1. Path specified with `--config` flag
2. `./sensei.yaml` (current directory)
3. `~/.sensei/config.yaml` (user home)
4. `/etc/sensei/config.yaml` (system-wide)

### Complete Configuration Reference

```yaml
# sensei.yaml - Complete configuration reference

# =============================================================================
# API Configuration
# =============================================================================
api:
  # API key for authentication
  key: ${SENSEI_API_KEY} # Use environment variable

  # API endpoint (SaaS default, override for self-hosted)
  endpoint: https://api.sensei.ai

  # Request timeout
  timeout: 30s

  # Retry configuration
  retry:
    max_attempts: 3
    initial_backoff: 1s
    max_backoff: 30s
    backoff_multiplier: 2

# =============================================================================
# Agent Configuration
# =============================================================================
agents:
  # Cognitive tier settings
  cognitive:
    # Model selection for Scout agents
    scout_model: gpt-4-turbo

    # Model selection for Architect agents
    architect_model: gpt-4-turbo

    # Maximum tokens per request
    max_tokens: 4096

    # Temperature for model responses
    temperature: 0.1

    # Timeout for LLM calls
    timeout: 60s

  # Execution tier settings
  execution:
    # Model for Worker agents
    worker_model: llama-3-8b

    # Number of worker agents per migration
    workers_per_migration: 8

    # Batch size for processing
    batch_size: 10000

    # Memory limit per worker
    memory_limit: 4Gi

  # Meta tier settings
  meta:
    # Enable cross-migration learning
    learning_enabled: true

    # Privacy budget (epsilon for differential privacy)
    privacy_epsilon: 1.0

    # Pattern library update frequency
    pattern_sync_interval: 1h

# =============================================================================
# Database Configuration
# =============================================================================
database:
  # Internal metadata database
  metadata:
    driver: postgresql
    host: ${SENSEI_DB_HOST}
    port: 5432
    database: sensei_metadata
    username: ${SENSEI_DB_USER}
    password: ${SENSEI_DB_PASSWORD}
    ssl_mode: require

    # Connection pool settings
    pool:
      max_connections: 20
      min_connections: 5
      max_idle_time: 10m

  # Vector database for pattern library
  vector:
    provider: pinecone # or: weaviate, qdrant

    pinecone:
      api_key: ${PINECONE_API_KEY}
      environment: us-east-1-aws
      index: sensei-patterns

    weaviate:
      host: weaviate.sensei.internal
      scheme: https
      api_key: ${WEAVIATE_API_KEY}

  # Graph database for knowledge graph
  graph:
    provider: neo4j
    uri: bolt://neo4j.sensei.internal:7687
    username: ${NEO4J_USER}
    password: ${NEO4J_PASSWORD}

# =============================================================================
# Storage Configuration
# =============================================================================
storage:
  # Temporary storage for migration artifacts
  temp:
    provider: s3
    bucket: sensei-temp
    region: us-east-1
    prefix: migrations/

    # Cleanup policy
    retention: 7d

  # Long-term storage for audit logs
  audit:
    provider: s3
    bucket: sensei-audit
    region: us-east-1
    prefix: audit/

    # Compliance retention
    retention: 7y

  # Local cache settings
  cache:
    path: /var/cache/sensei
    max_size: 50Gi

# =============================================================================
# Compute Configuration (Ray)
# =============================================================================
compute:
  # Ray cluster settings
  ray:
    head_node:
      cpu: 4
      memory: 16Gi

    worker_nodes:
      min: 2
      max: 32
      cpu: 8
      memory: 32Gi
      gpu: 0 # Set >0 for local model inference

    autoscaling:
      enabled: true
      scale_up_threshold: 0.8
      scale_down_threshold: 0.2
      cooldown_period: 5m

  # Resource limits per migration
  limits:
    max_concurrent_migrations: 10
    max_workers_per_migration: 16
    max_memory_per_migration: 128Gi

# =============================================================================
# Monitoring & Observability
# =============================================================================
monitoring:
  # Metrics export
  metrics:
    enabled: true
    provider: prometheus
    port: 9090
    path: /metrics

  # Tracing
  tracing:
    enabled: true
    provider: jaeger
    endpoint: http://jaeger.sensei.internal:14268/api/traces
    sample_rate: 0.1

  # Logging
  logging:
    level: info # debug, info, warn, error
    format: json # json, text
    output: stdout # stdout, file
    file_path: /var/log/sensei/sensei.log
    max_size: 100Mi
    max_age: 30d
    max_backups: 10

# =============================================================================
# Security Configuration
# =============================================================================
security:
  # TLS settings
  tls:
    enabled: true
    cert_path: /etc/sensei/tls/cert.pem
    key_path: /etc/sensei/tls/key.pem
    ca_path: /etc/sensei/tls/ca.pem
    min_version: '1.2'

  # Encryption at rest
  encryption:
    enabled: true
    provider: aws-kms # or: vault, local
    key_id: ${KMS_KEY_ID}

  # Secrets management
  secrets:
    provider: aws-secrets-manager # or: vault, env
    region: us-east-1

  # Authentication
  auth:
    # API key authentication
    api_keys:
      enabled: true

    # OAuth/OIDC
    oidc:
      enabled: false
      issuer: https://auth.example.com
      client_id: ${OIDC_CLIENT_ID}
      client_secret: ${OIDC_CLIENT_SECRET}

# =============================================================================
# MABA Configuration
# =============================================================================
maba:
  # Ingestion settings
  ingestion:
    # Maximum file size to process
    max_file_size: 10Gi

    # Parallel file processing
    parallel_files: 4

    # Encoding detection confidence threshold
    encoding_confidence: 0.9

  # Schema analysis
  schema_analysis:
    # Sample size for statistical profiling
    sample_size: 10000

    # Confidence threshold for auto-mapping
    auto_map_confidence: 0.92

    # Maximum LLM calls per table
    max_llm_calls_per_table: 10

  # Transformation
  transformation:
    # Default batch size
    batch_size: 10000

    # Parallel transformation threads
    parallel_transforms: 8

    # Memory limit for transformations
    memory_limit: 8Gi

# =============================================================================
# KORA Configuration
# =============================================================================
kora:
  # Validation settings
  validation:
    # Enable Time-Travel Testing
    time_travel_enabled: true

    # Number of temporal checkpoints
    checkpoint_count: 5

    # Checkpoint interval
    checkpoint_interval: auto # or: 1h, 1d, etc.

    # Replay query limit per checkpoint
    replay_limit: 1000

  # Comparison settings
  comparison:
    # Numeric tolerance (epsilon)
    numeric_epsilon: 0.0001

    # String comparison mode
    string_mode: normalized # exact, normalized, semantic

    # Timestamp tolerance
    timestamp_tolerance: 1s

  # Certificate generation
  certificates:
    # Include detailed evidence
    include_evidence: true

    # Signing key
    signing_key_id: ${KORA_SIGNING_KEY}

# =============================================================================
# AMANI Configuration
# =============================================================================
amani:
  # Channel configuration
  channels:
    web:
      enabled: true

    whatsapp:
      enabled: true
      business_account_id: ${WHATSAPP_ACCOUNT_ID}
      api_token: ${WHATSAPP_TOKEN}

    sms:
      enabled: false
      provider: twilio
      account_sid: ${TWILIO_SID}
      auth_token: ${TWILIO_TOKEN}

    ussd:
      enabled: false

    voice:
      enabled: false

  # Language settings
  language:
    # Default language
    default: en

    # Auto-detect language
    auto_detect: true

    # Supported languages (empty = all)
    supported: []

  # Offline settings
  offline:
    # Enable offline mode
    enabled: true

    # Maximum offline duration
    max_offline_days: 30

    # Local cache size
    cache_size: 1Gi

# =============================================================================
# Feature Flags
# =============================================================================
features:
  # Experimental features
  experimental:
    natural_language_config: false
    predictive_analytics: false
    custom_connector_generation: false

  # Beta features
  beta:
    parallel_validation: true
    incremental_time_travel: true
```

---

## Environment Variables

All configuration options can be set via environment variables:

| Config Path                    | Environment Variable                  |
| ------------------------------ | ------------------------------------- |
| `api.key`                      | `SENSEI_API_KEY`                      |
| `api.endpoint`                 | `SENSEI_API_ENDPOINT`                 |
| `agents.cognitive.scout_model` | `SENSEI_AGENTS_COGNITIVE_SCOUT_MODEL` |
| `database.metadata.host`       | `SENSEI_DATABASE_METADATA_HOST`       |
| `monitoring.logging.level`     | `SENSEI_MONITORING_LOGGING_LEVEL`     |

Pattern: Replace `.` with `_` and prefix with `SENSEI_`, uppercase.

---

## Common Configurations

### Development

```yaml
# sensei-dev.yaml
api:
  endpoint: http://localhost:8080

agents:
  execution:
    workers_per_migration: 2
    batch_size: 1000

monitoring:
  logging:
    level: debug
    format: text

features:
  experimental:
    natural_language_config: true
```

### Production SaaS

```yaml
# sensei-prod.yaml
api:
  key: ${SENSEI_API_KEY}
  endpoint: https://api.sensei.ai

monitoring:
  logging:
    level: info
    format: json
```

### Self-Hosted Enterprise

```yaml
# sensei-enterprise.yaml
api:
  endpoint: https://sensei.internal.example.com

database:
  metadata:
    host: postgres.internal.example.com
  vector:
    provider: weaviate
    weaviate:
      host: weaviate.internal.example.com
  graph:
    uri: bolt://neo4j.internal.example.com:7687

security:
  tls:
    enabled: true
  encryption:
    provider: vault
```

---

## Validating Configuration

```bash
# Validate configuration file
sensei config validate --config sensei.yaml

# Show effective configuration (with defaults filled in)
sensei config show --config sensei.yaml

# Check connectivity to configured services
sensei config test --config sensei.yaml
```

→ [Environment Variables Reference](../../developers/reference/environment-variables.md)
→ [Deployment Options](README.md)
→ [Security Architecture](../../developers/architecture/technology-stack/security-architecture.md)
