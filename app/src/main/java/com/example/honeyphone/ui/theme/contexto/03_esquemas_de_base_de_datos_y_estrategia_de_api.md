# HoneyPhone: Esquemas de Base de Datos (Local & Remota) y Estrategia de Conexión a API
> **Propósito para Agentes de IA:** Contiene las definiciones DDL definitivas para **SQLite/Room (Móvil)** y **PostgreSQL 16+ (Backend)**, así como el protocolo de sincronización diferida y el contrato de endpoints para la Fase Final.

---

## 1. Base de Datos Móvil Endurecida (Android - Room / SQLite + SQLCipher)

### 1.1. Reglas de Inicialización en el Cliente
Al abrir la base de datos en `HoneyPhoneDatabase`, se deben ejecutar los siguientes `PRAGMA`:
```sql
PRAGMA foreign_keys = ON;
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA secure_delete = ON;
PRAGMA temp_store = MEMORY;
```

### 1.2. Esquema DDL Local (SQLite)
```sql
-- 1. Revisiones de red Wi-Fi con máquina de estados de sincronización
CREATE TABLE network_checks (
    check_id TEXT PRIMARY KEY NOT NULL,                 -- UUID v7 generado localmente
    timestamp_utc INTEGER NOT NULL,                     -- Epoch ms
    network_pseudonym TEXT NOT NULL
        CHECK(length(network_pseudonym) = 64),          -- HMAC-SHA256 del BSSID vía Keystore
    security_type TEXT NOT NULL
        CHECK(security_type IN ('OPEN', 'WEP', 'WPA_PSK', 'WPA2_PSK', 'WPA3_SAE', 'ENTERPRISE', 'UNKNOWN')),
    captive_portal_status TEXT NOT NULL,
    dns_status TEXT NOT NULL,
    https_status TEXT NOT NULL,
    overall_verdict TEXT NOT NULL
        CHECK(overall_verdict IN ('NO_DISCREPANCIES', 'ATTENTION_REQUIRED', 'INCOMPLETE')),
    sync_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(sync_status IN ('PENDING', 'IN_FLIGHT', 'SYNCED', 'FAILED_PERMANENT')),
    sync_batch_id TEXT NULL,
    sync_attempts INTEGER NOT NULL DEFAULT 0,
    last_sync_attempt_utc INTEGER NULL
);

-- 2. Señales individuales de cada revisión (Relación 1 a N)
CREATE TABLE check_signals (
    signal_id TEXT PRIMARY KEY NOT NULL,                -- UUID v7
    check_id TEXT NOT NULL,
    signal_name TEXT NOT NULL CHECK(length(signal_name) <= 64),
    status_category TEXT NOT NULL,
    explanation_code TEXT NOT NULL CHECK(length(explanation_code) <= 64),
    evidence_payload TEXT NULL CHECK(evidence_payload IS NULL OR length(evidence_payload) <= 2048),
    FOREIGN KEY (check_id) REFERENCES network_checks(check_id) ON DELETE CASCADE
);

-- 3. Sesiones del señuelo HTTP con contadores de protección anti-flood
CREATE TABLE honeypot_sessions (
    session_id TEXT PRIMARY KEY NOT NULL,               -- UUID v7
    started_at_utc INTEGER NOT NULL,
    ended_at_utc INTEGER NULL,
    listen_port INTEGER NOT NULL DEFAULT 8080 CHECK(listen_port BETWEEN 1024 AND 65535),
    unique_events_count INTEGER NOT NULL DEFAULT 0,     -- Límite duro: 200 eventos únicos por sesión
    total_hits_count INTEGER NOT NULL DEFAULT 0,
    dropped_events_count INTEGER NOT NULL DEFAULT 0,
    sync_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(sync_status IN ('PENDING', 'IN_FLIGHT', 'SYNCED', 'FAILED_PERMANENT')),
    sync_attempts INTEGER NOT NULL DEFAULT 0
);

-- 4. Eventos del señuelo con deduplicación en caliente (hit_count)
CREATE TABLE honeypot_events (
    event_id TEXT PRIMARY KEY NOT NULL,                 -- UUID v7
    session_id TEXT NOT NULL,
    idempotency_hash TEXT UNIQUE NOT NULL               -- SHA-256(session_id + src_ip_masked + method + path)
        CHECK(length(idempotency_hash) = 64),
    first_seen_at_utc INTEGER NOT NULL,
    last_seen_at_utc INTEGER NOT NULL,
    hit_count INTEGER NOT NULL DEFAULT 1,
    source_ip_masked TEXT NOT NULL CHECK(length(source_ip_masked) <= 45),
    http_method TEXT NOT NULL CHECK(length(http_method) <= 10),
    request_path TEXT NOT NULL CHECK(length(request_path) <= 256),
    user_agent TEXT NULL CHECK(user_agent IS NULL OR length(user_agent) <= 256),
    content_length INTEGER NOT NULL DEFAULT 0 CHECK(content_length >= 0),
    response_code INTEGER NOT NULL DEFAULT 200,
    sync_status TEXT NOT NULL DEFAULT 'PENDING'
        CHECK(sync_status IN ('PENDING', 'IN_FLIGHT', 'SYNCED', 'FAILED_PERMANENT')),
    FOREIGN KEY (session_id) REFERENCES honeypot_sessions(session_id) ON DELETE CASCADE
);

CREATE INDEX idx_local_checks_sync ON network_checks(sync_status, timestamp_utc ASC) WHERE sync_status IN ('PENDING', 'IN_FLIGHT');
CREATE INDEX idx_local_events_sync ON honeypot_events(session_id, sync_status);
CREATE INDEX idx_local_checks_history ON network_checks(timestamp_utc DESC);
```

---

## 2. Base de Datos Central Desacoplada (Backend - PostgreSQL 16+)

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TYPE security_protocol_type AS ENUM (
    'OPEN', 'WEP', 'WPA_PSK', 'WPA2_PSK', 'WPA3_SAE', 'ENTERPRISE', 'UNKNOWN'
);

CREATE TYPE check_status_type AS ENUM (
    'MATCH', 'MISMATCH', 'TIMEOUT', 'ERROR', 'NOT_APPLICABLE',
    'NOT_DETECTED', 'PORTAL_DETECTED', 'VALID_CERT', 'INVALID_CERT'
);

CREATE TYPE verdict_type AS ENUM (
    'NO_DISCREPANCIES', 'ATTENTION_REQUIRED', 'INCOMPLETE'
);

CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE devices (
    device_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NULL REFERENCES users(user_id) ON DELETE SET NULL, -- Permite enrolamiento anónimo
    device_token_hash VARCHAR(64) UNIQUE NOT NULL,                  -- SHA-256(X-Device-Token)
    app_platform VARCHAR(16) NOT NULL DEFAULT 'ANDROID',
    client_version VARCHAR(32) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    enrolled_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_seen_at TIMESTAMPTZ NULL,
    revoked_at TIMESTAMPTZ NULL
);

CREATE TABLE network_checks (
    check_id UUID PRIMARY KEY,                                      -- UUID v7 enviado por el móvil
    device_id UUID NOT NULL REFERENCES devices(device_id) ON DELETE CASCADE,
    network_pseudonym VARCHAR(64) NOT NULL,
    client_timestamp TIMESTAMPTZ NOT NULL,
    security_type security_protocol_type NOT NULL,
    captive_portal_status check_status_type NOT NULL,
    dns_status check_status_type NOT NULL,
    https_status check_status_type NOT NULL,
    overall_verdict verdict_type NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE check_signals (
    signal_id UUID PRIMARY KEY,
    check_id UUID NOT NULL REFERENCES network_checks(check_id) ON DELETE CASCADE,
    signal_name VARCHAR(64) NOT NULL,
    status_category check_status_type NOT NULL,
    explanation_code VARCHAR(64) NOT NULL,
    evidence_payload JSONB NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE honeypot_sessions (
    session_id UUID PRIMARY KEY,
    device_id UUID NOT NULL REFERENCES devices(device_id) ON DELETE CASCADE,
    listen_port INT NOT NULL CHECK (listen_port BETWEEN 1024 AND 65535),
    started_at TIMESTAMPTZ NOT NULL,
    ended_at TIMESTAMPTZ NULL,
    total_events_captured INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE honeypot_events (
    event_id UUID NOT NULL,
    session_id UUID NOT NULL REFERENCES honeypot_sessions(session_id) ON DELETE CASCADE,
    idempotency_hash VARCHAR(64) NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    source_ip_masked VARCHAR(45) NOT NULL,
    http_method VARCHAR(10) NOT NULL,
    request_path VARCHAR(256) NOT NULL,
    user_agent VARCHAR(256) NULL,
    content_length INT NOT NULL DEFAULT 0,
    hit_count INT NOT NULL DEFAULT 1,
    response_code SMALLINT NOT NULL DEFAULT 200,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (occurred_at, event_id),
    CONSTRAINT uq_event_idempotency UNIQUE (occurred_at, idempotency_hash)
) PARTITION BY RANGE (occurred_at);
```

---

## 3. Estrategia de Integración con la API (Fase Final de Desarrollo)

> **NOTA IMPORTANTE PARA LA IMPLEMENTACIÓN:** Dado que el backend se construirá y desplegará en las semanas finales (Semanas 9-10 del plan de trabajo), la aplicación Android debe implementar el patrón **Gateway / Strategy** en la capa de red:
> * `LocalOnlySyncGateway` (Activo por defecto en Fases 1-3): Mantiene los registros en SQLite con estado `PENDING` y permite probar toda la UX, exportación JSON/CSV y diagnóstico sin requerir servidor.
> * `RemoteApiSyncGateway` (Activado en Fase 4): Conecta `TelemetrySyncWorker` con los endpoints reales de FastAPI.

### 3.1. Flujo de la Máquina de Estados de Sincronización (`TelemetrySyncWorker`)
1. **Selección de Lote:** Consulta hasta 50 registros en `network_checks` donde `sync_status = 'PENDING'` (o `IN_FLIGHT` cuyo `last_sync_attempt_utc` sea mayor a 15 minutos).
2. **Reserva Atómica:** Asigna un mismo `sync_batch_id` (UUID v7), cambia `sync_status = 'IN_FLIGHT'` e incrementa `sync_attempts`.
3. **Transmisión Idempotente:** Envía `POST /api/v1/sync/network-checks` con el encabezado `X-Device-Token`.
4. **Resolución:**
   * Si la API responde `200 OK`: actualiza los registros del lote a `sync_status = 'SYNCED'`.
   * Si la API responde `422 Unprocessable Entity` o `400 Bad Request` (error de esquema permanente): marca como `FAILED_PERMANENT` para no bloquear futuros lotes (*Poison Pill isolation*).
   * Si ocurre `IOException` / `5xx`: revierte a `PENDING` y delega el reintento a `WorkManager` con *Exponential Backoff*.

### 3.2. Resumen de Contratos de Endpoints

| Método | Ruta | Header de Auth | Comportamiento de Idempotencia / Seguridad |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/devices/enroll` | Ninguno o Bearer JWT | Genera `device_id` y devuelve `device_api_key` (`hp_dev_...`) una única vez. Guarda `SHA-256` en DB. |
| `POST` | `/api/v1/devices/{id}/revoke` | `X-Device-Token` | Marca `revoked_at = NOW()` e invalida acceso futuro del dispositivo. |
| `POST` | `/api/v1/sync/network-checks` | `X-Device-Token` | Inserta lote de revisiones y señales en una transacción con `ON CONFLICT (check_id) DO NOTHING`. |
| `POST` | `/api/v1/sync/honeypot-events` | `X-Device-Token` | Inserta sesión y eventos con `ON CONFLICT (occurred_at, idempotency_hash) DO NOTHING`. |
| `GET` | `/api/v1/network-checks` | `X-Device-Token` | Paginación por cursor (`cursor=<ISO-8601>&limit=20`) filtrada estrictamente por `device_id`. |
| `GET` | `/api/v1/canary/probe` | Público | Devuelve firma estática sobre HTTPS con certificado TLS válido para la prueba activa de la app. |