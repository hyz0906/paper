# Combined Scheduled Log and File Fetcher System Design

## Analysis of Source Documents

### ChatGPT Design Advantages:
- **Comprehensive error handling** with partial status and detailed retry mechanisms
- **Advanced security** with environment variable interpolation for secrets
- **Robust configuration management** with atomic writes and file locking
- **Sophisticated API design** with validation endpoints and operational controls
- **Memory-efficient streaming** with dedicated psycopg3 connections for Large Objects
- **Production-ready features** like checksums, observability, and retention policies

### ChatGPT Design Disadvantages:
- **Complex implementation** with multiple connection pools and threading
- **Over-engineered** for simple use cases with extensive configuration options
- **Harder to maintain** due to complex error handling and multiple code paths

### Gemini Design Advantages:
- **Clean, simple architecture** that's easy to understand and implement
- **Clear separation of concerns** with straightforward data flow
- **Focused feature set** without unnecessary complexity
- **Good use of PostgreSQL enums** for data integrity
- **Well-structured configuration** with clear examples

### Gemini Design Disadvantages:
- **Limited error handling** with only success/failed states
- **Basic security** with no secret management
- **No retry mechanisms** for failed operations
- **Missing operational features** like health checks and monitoring
- **Less robust file handling** without checksums or partial failure handling

---

## 1. High-Level System Architecture

```mermaid
flowchart LR
    A[Vue.js Frontend<br/>(TypeScript)] <--> B[FastAPI Backend<br/>(REST + WebSocket)]
    B <--> D[(PostgreSQL)]
    B <-- atomic read/write --> E[[config.yml]]
    subgraph "Backend Process"
        B
        C[APScheduler<br/>(AsyncIOScheduler)]
        F[Configuration Service<br/>(File Locking)]
    end
    C -->|cron triggers| B
    B -->|stream via psycopg3| D
    B -->|metadata| D
    A -->|CRUD sources & history| B
    A -->|download files| B
    F -->|manages| E
    F -->|reloads| C
```

**Key Design Principles:**
- **Simplicity**: Easy to understand and maintain core architecture
- **Robustness**: Production-ready error handling and security
- **Scalability**: Memory-efficient streaming for large files
- **Observability**: Comprehensive logging and monitoring capabilities

---

## 2. Enhanced Configuration File Design

### Goals
- Human-readable YAML with comprehensive validation
- Environment-based secret management
- Flexible retry and timeout configurations
- Clear source identification and metadata

### Example `config.yml`

```yaml
version: 1

# Global defaults with sensible overrides
defaults:
  retry:
    max_attempts: 3
    backoff_seconds: 10
  http:
    timeout_seconds: 60
    verify_tls: true
  ssh:
    port: 22
    known_hosts: "strict"  # strict|accept-new|ignore
    connect_timeout_seconds: 20

# Source definitions
sources:
  - name: "prod-auth-logs"
    type: "ssh"
    enabled: true
    schedule: "0 */1 * * *"  # Every hour
    description: "Production authentication logs"
    ssh:
      host: "prod.example.com"
      port: 22
      username: "logfetcher"
      private_key_path: "/etc/app/keys/logfetcher_ed25519"
      # passphrase: "${SSH_KEY_PASSPHRASE}"  # Environment variable
    remote_path: "/var/log/auth.log"
    tags: ["production", "security", "logs"]
    retention_days: 30  # Optional: auto-cleanup after N days

  - name: "weekly-reports"
    type: "http"
    enabled: true
    schedule: "0 18 * * 5"  # Friday 6 PM
    description: "Weekly archived reports"
    http:
      url: "https://reports.example.com/weekly/latest.tar.gz"
      headers:
        Authorization: "Bearer ${REPORTS_API_TOKEN}"
      timeout_seconds: 120
      verify_tls: true
    filename_hint: "weekly_report.tar.gz"
    tags: ["reports", "weekly"]
    retry:
      max_attempts: 5
      backoff_seconds: 30

# Optional global settings
network:
  http_proxy: null
  https_proxy: null

notifications:
  webhook_url: "${WEBHOOK_URL}"  # Optional: notify on failures
```

**Key Features:**
- **Secret Management**: Environment variable interpolation (e.g., `${API_TOKEN}`)
- **Flexible Defaults**: Global defaults with per-source overrides
- **Rich Metadata**: Tags, descriptions, and retention policies
- **Validation**: Schema validation before saving

---

## 3. Enhanced Database Schema

Combines the simplicity of enums with comprehensive metadata tracking:

```sql
-- Create custom enum types for better data integrity
CREATE TYPE fetch_status AS ENUM ('success', 'failed', 'partial', 'retrying');

-- Main table for fetch metadata
CREATE TABLE fetched_files (
    id BIGSERIAL PRIMARY KEY,
    source_name VARCHAR(255) NOT NULL,
    fetched_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ,  -- When the operation finished
    status fetch_status NOT NULL,
    
    -- File information
    original_filename VARCHAR(512),
    file_size_bytes BIGINT,
    
    -- PostgreSQL Large Object reference
    lo_oid OID,
    
    -- Enhanced error handling
    error_message TEXT,
    retry_count INTEGER DEFAULT 0,
    
    -- Data integrity
    checksum_sha256 CHAR(64),  -- Hex-encoded SHA-256
    
    -- Additional metadata
    source_tags TEXT[],  -- Array of tags from config
    extra_metadata JSONB DEFAULT '{}'::jsonb,  -- HTTP headers, SSH details, etc.
    
    -- Audit fields
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Optimized indexes
CREATE INDEX idx_fetched_files_source_time ON fetched_files (source_name, fetched_at DESC);
CREATE INDEX idx_fetched_files_status ON fetched_files (status, fetched_at DESC);
CREATE INDEX idx_fetched_files_tags ON fetched_files USING GIN (source_tags);
CREATE INDEX idx_fetched_files_lo_oid ON fetched_files (lo_oid) WHERE lo_oid IS NOT NULL;

-- Trigger to automatically update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_fetched_files_updated_at 
    BEFORE UPDATE ON fetched_files 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

---

## 4. Comprehensive API Design

Base URL: `/api/v1`

### Source Management
| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/sources` | List all sources with status and next run time |
| `POST` | `/sources` | Create new source (validates and updates config) |
| `GET` | `/sources/{name}` | Get single source details |
| `PUT` | `/sources/{name}` | Update source (atomic config update) |
| `DELETE` | `/sources/{name}` | Remove source and cleanup jobs |
| `POST` | `/sources/{name}/trigger` | Manual trigger (run now) |
| `POST` | `/sources/{name}/test` | Test connection without saving |
| `POST` | `/sources/validate` | Validate source configuration |

### Configuration Management
| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/config` | Get current config (secrets masked) |
| `PUT` | `/config` | Replace entire config (atomic) |
| `POST` | `/config/reload` | Force reload from disk |
| `GET` | `/config/schema` | Get configuration JSON schema |

### Fetch History & Files
| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/fetches` | Paginated fetch history with filters |
| `GET` | `/fetches/{id}` | Get single fetch metadata |
| `GET` | `/fetches/{id}/download` | Stream file download |
| `DELETE` | `/fetches/{id}` | Delete fetch record and file |
| `POST` | `/fetches/{id}/retry` | Retry failed fetch |
| `POST` | `/fetches/{id}/verify` | Recompute checksum |

### System Operations
| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | System health check |
| `GET` | `/health/ready` | Readiness probe |
| `GET` | `/metrics` | Prometheus-style metrics |
| `GET` | `/scheduler/jobs` | Current scheduled jobs |
| `POST` | `/scheduler/reload` | Reload scheduler from config |
| `GET` | `/system/stats` | System statistics |

---

## 5. Robust Backend Implementation

### 5.1 Configuration Service

**Thread-safe configuration management with atomic operations:**

```python
import asyncio
import filelock
import tempfile
import os
from pathlib import Path
from typing import Dict, Any
import yaml
from pydantic import BaseModel, ValidationError

class ConfigurationService:
    def __init__(self, config_path: str):
        self.config_path = Path(config_path)
        self.lock_path = f"{config_path}.lock"
        self._lock = filelock.FileLock(self.lock_path)
        
    async def read_config(self) -> Dict[str, Any]:
        """Thread-safe config reading"""
        def _read():
            with self._lock:
                return yaml.safe_load(self.config_path.read_text())
        return await asyncio.get_event_loop().run_in_executor(None, _read)
    
    async def write_config(self, config: Dict[str, Any]) -> None:
        """Atomic config writing with validation"""
        def _write():
            # Validate first
            validated_config = ConfigSchema(**config)
            
            # Atomic write using temp file
            with self._lock:
                temp_path = self.config_path.with_suffix('.tmp')
                temp_path.write_text(yaml.dump(validated_config.dict()))
                temp_path.replace(self.config_path)
                
        await asyncio.get_event_loop().run_in_executor(None, _write)
        
        # Notify scheduler to reload
        await self._notify_scheduler_reload()
```

### 5.2 Enhanced Scheduler Service

**APScheduler with robust job management:**

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
from apscheduler.jobstores.sqlalchemy import SQLAlchemyJobStore
import random

class SchedulerService:
    def __init__(self, config_service: ConfigurationService):
        self.config_service = config_service
        self.scheduler = AsyncIOScheduler(
            jobstores={'default': SQLAlchemyJobStore(url=DATABASE_URL)},
            job_defaults={'coalesce': True, 'max_instances': 1}
        )
        
    async def start(self):
        """Initialize and start scheduler"""
        await self.reload_jobs()
        self.scheduler.start()
        
    async def reload_jobs(self):
        """Reload all jobs from configuration"""
        config = await self.config_service.read_config()
        
        # Clear existing jobs
        self.scheduler.remove_all_jobs()
        
        # Add jobs for enabled sources
        for source in config.get('sources', []):
            if source.get('enabled', False):
                await self._add_job(source)
                
    async def _add_job(self, source: Dict[str, Any]):
        """Add a single job with jitter"""
        # Add jitter to prevent thundering herd
        jitter_seconds = random.randint(0, 30)
        
        trigger = CronTrigger.from_crontab(
            source['schedule'], 
            timezone='UTC',
            jitter=jitter_seconds
        )
        
        self.scheduler.add_job(
            func=self._execute_fetch_job,
            trigger=trigger,
            id=source['name'],
            kwargs={'source_config': source},
            misfire_grace_time=300
        )
```

### 5.3 Memory-Efficient File Fetching

**Streaming implementation with dual connection approach:**

```python
import hashlib
import psycopg
from typing import AsyncGenerator, Callable
import asyncssh
import httpx

class FileFetcher:
    def __init__(self, db_pool, sync_db_dsn: str):
        self.db_pool = db_pool  # AsyncPG pool
        self.sync_db_dsn = sync_db_dsn  # For psycopg3 LO operations
        
    async def fetch_and_store(self, source_config: Dict[str, Any]) -> int:
        """Main fetch operation with streaming storage"""
        fetch_id = None
        lo_oid = None
        hasher = hashlib.sha256()
        bytes_written = 0
        error_msg = None
        status = 'failed'
        
        try:
            # Create fetch record
            fetch_id = await self._create_fetch_record(source_config)
            
            # Create Large Object in separate thread
            lo_oid = await self._create_large_object()
            
            # Stream fetch with chunked processing
            async for chunk in self._fetch_chunks(source_config):
                hasher.update(chunk)
                await self._write_chunk_to_lo(lo_oid, chunk)
                bytes_written += len(chunk)
                
                # Update progress periodically
                if bytes_written % (10 * 1024 * 1024) == 0:  # Every 10MB
                    await self._update_progress(fetch_id, bytes_written)
            
            # Finalize
            await self._finalize_large_object(lo_oid)
            checksum = hasher.hexdigest()
            status = 'success'
            
        except Exception as e:
            error_msg = str(e)[:4000]
            if lo_oid:
                await self._cleanup_large_object(lo_oid)
                lo_oid = None
                
        finally:
            # Update final record
            await self._finalize_fetch_record(
                fetch_id, status, bytes_written, lo_oid, 
                checksum if status == 'success' else None, 
                error_msg
            )
            
        return fetch_id
        
    async def _fetch_chunks(self, source_config: Dict[str, Any]) -> AsyncGenerator[bytes, None]:
        """Protocol-agnostic chunk generator"""
        if source_config['type'] == 'ssh':
            async for chunk in self._fetch_ssh_chunks(source_config):
                yield chunk
        elif source_config['type'] == 'http':
            async for chunk in self._fetch_http_chunks(source_config):
                yield chunk
                
    async def _write_chunk_to_lo(self, lo_oid: int, chunk: bytes):
        """Write chunk to Large Object in thread"""
        def _write():
            with psycopg.connect(self.sync_db_dsn) as conn:
                with conn.lobject(lo_oid, 'w') as lo:
                    lo.write(chunk)
                    
        await asyncio.get_event_loop().run_in_executor(None, _write)
```

### 5.4 Production Features

**Health checks, metrics, and observability:**

```python
from fastapi import FastAPI, HTTPException, BackgroundTasks
from prometheus_client import Counter, Histogram, Gauge
import structlog

# Metrics
fetch_counter = Counter('file_fetches_total', 'Total file fetches', ['source', 'status'])
fetch_duration = Histogram('file_fetch_duration_seconds', 'Fetch duration')
active_jobs = Gauge('scheduler_jobs_active', 'Active scheduled jobs')

logger = structlog.get_logger()

@app.get("/api/v1/health")
async def health_check():
    """Comprehensive health check"""
    checks = {
        'database': await _check_database(),
        'scheduler': _check_scheduler(),
        'config_file': _check_config_file(),
        'disk_space': _check_disk_space()
    }
    
    all_healthy = all(checks.values())
    status_code = 200 if all_healthy else 503
    
    return {"status": "healthy" if all_healthy else "unhealthy", "checks": checks}

@app.get("/api/v1/metrics")
async def get_metrics():
    """Prometheus metrics endpoint"""
    from prometheus_client import generate_latest, CONTENT_TYPE_LATEST
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)
```

---

## 6. Frontend Integration Points

**Vue.js application with real-time updates:**

- **WebSocket Integration**: Real-time job status updates
- **Progressive Enhancement**: Works without JS for basic functionality
- **Responsive Design**: Mobile-friendly interface
- **Error Boundaries**: Graceful error handling and recovery
- **Offline Support**: Basic caching for configuration management

---

## 7. Security & Compliance

### Secret Management
- Environment variable interpolation for all sensitive data
- No secrets stored in configuration files
- Secure key storage recommendations

### Access Control
- API authentication (JWT/OAuth2 ready)
- Role-based permissions for different operations
- Audit logging for all configuration changes

### Data Protection
- Automatic cleanup of old files based on retention policies
- Encrypted storage options for sensitive files
- Compliance-ready audit trails

---

## 8. Operational Excellence

### Monitoring
- Structured logging with correlation IDs
- Prometheus metrics for all operations
- Health checks for all components
- Performance monitoring and alerting

### Reliability
- Graceful degradation during failures
- Circuit breakers for external dependencies
- Comprehensive retry mechanisms with exponential backoff
- Database connection pooling and management

### Maintenance
- Automatic cleanup jobs for old data
- Configuration validation and migration tools
- Backup and restore procedures
- Update and deployment strategies

---

This combined design provides the simplicity and clarity of the Gemini approach while incorporating the robustness, security, and production-readiness features of the ChatGPT design. The result is a system that is both easy to understand and maintain, while being capable of handling enterprise-level requirements.