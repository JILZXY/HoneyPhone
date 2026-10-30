# HoneyPhone: Especificación Arquitectónica (Android & Backend API)
> **Propósito para Agentes de IA:** Define los patrones arquitectónicos seleccionados mediante *Tree of Thoughts (ToT)* y *Chain of Verification (CoVE)*, la estructura exacta de paquetes y las responsabilidades de cada componente.

---

## 1. Arquitectura de la Aplicación Móvil (Android / Kotlin)

### 1.1. Patrón Seleccionado: Feature-First + UDF (MVI) + Motores Desacoplados vía Room SSOT
* **Interfaz de Usuario (Jetpack Compose + MVI):** Cada pantalla consume un único `StateFlow<UiState>` inmutable, emite intenciones (`UiIntent`) al `ViewModel` y recibe eventos de un solo disparo (`UiEffect`).
* **Motores de Ejecución Independientes:**
  * `WifiCheckerEngine`: Se ejecuta en un ámbito de corrutinas inyectado, atado a la interfaz `Network` Wi-Fi activa.
  * `HoneypotService` (`ForegroundService`): Mantiene vivo el `TcpDecoyServer` en el puerto `8080` independientemente de si la pantalla se rota o la aplicación pasa a segundo plano.
* **Room como Única Fuente de Verdad (*Single Source of Truth - SSOT*):** Los motores escriben sus hallazgos directamente en Room (cifrado con SQLCipher). La UI **nunca** interroga directamente al socket del señuelo; observa flujos reactivos (`Flow<List<...>>`) emitidos por los DAOs de Room.

### 1.2. Árbol de Directorios (Android)
```text
honeyphone-android/
└── app/src/main/java/com/upchiapas/honeyphone/
    ├── HoneyPhoneApp.kt                     # Hilt Application + Configuración de WorkManager
    ├── MainActivity.kt                      # Host de navegación Jetpack Compose
    ├── core/
    │   ├── database/                        # SSOT: Room + SQLCipher
    │   │   ├── HoneyPhoneDatabase.kt        # PRAGMAs: WAL, secure_delete=ON, foreign_keys=ON
    │   │   ├── dao/
    │   │   │   ├── NetworkCheckDao.kt
    │   │   │   └── HoneypotDao.kt
    │   │   └── entity/
    │   │       ├── NetworkCheckEntity.kt
    │   │       ├── CheckSignalEntity.kt
    │   │       ├── HoneypotSessionEntity.kt
    │   │       └── HoneypotEventEntity.kt
    │   ├── security/
    │   │   ├── KeystoreManager.kt           # Llave HMAC protegida en Android Keystore (TEE)
    │   │   ├── NetworkPseudonymizer.kt      # HMAC-SHA256(BSSID) -> Hex de 64 chars
    │   │   ├── IpMasker.kt                  # Truncado CIDR en memoria (/24 IPv4, /64 IPv6)
    │   │   └── UuidV7Generator.kt           # UUID v7 monotónico con prefijo epoch_ms
    │   ├── network/                         # [FASE FINAL] Cliente de sincronización hacia API
    │   │   ├── TelemetryApiService.kt       # Contrato Retrofit/Ktor
    │   │   ├── CanaryProbeClient.kt         # Sonda hacia GET /api/v1/canary/probe
    │   │   └── dto/
    │   └── sync/                            # [FASE FINAL] Sincronización diferida
    │       ├── TelemetrySyncGateway.kt      # Interfaz desacoplada (Mock/Local en Fases 1-3)
    │       ├── TelemetrySyncWorker.kt       # CoroutineWorker (PENDING -> IN_FLIGHT -> SYNCED)
    │       └── SyncScheduler.kt
    └── features/
        ├── checker/                         # Módulo 1: Diagnóstico Wi-Fi
        │   ├── engine/
        │   │   ├── WifiCheckerEngine.kt     # Orquestador con Network.bindSocket()
        │   │   ├── WifiMetadataReader.kt    # Lectura resiliente de seguridad y Private DNS
        │   │   ├── CaptivePortalProbe.kt    # Verificación NET_CAPABILITY_CAPTIVE_PORTAL y HTTP
        │   │   ├── DnsCanaryProbe.kt        # Comparación UDP/Sistema vs. DoH sobre dominio propio
        │   │   └── TlsCanaryProbe.kt        # Validación estricta X509TrustManager del sistema
        │   └── ui/
        │       ├── CheckerContract.kt       # UiState (Idle, Running, Completed, Interrupted)
        │       ├── CheckerViewModel.kt
        │       └── CheckerScreen.kt
        ├── honeypot/                        # Módulo 2: Señuelo HTTP 8080/TCP
        │   ├── engine/
        │   │   ├── HoneypotService.kt       # ForegroundService + Notificación + AutoStopTimer
        │   │   ├── TcpDecoyServer.kt        # ServerSocket(8080) + Semaphore(3) + SO_TIMEOUT
        │   │   └── HttpParserSanitizer.kt   # Extracción segura de Method, Path y User-Agent
        │   └── ui/
        │       ├── HoneypotContract.kt
        │       ├── HoneypotViewModel.kt
        │       └── HoneypotScreen.kt
        ├── history/                         # Módulo 3: Historial y Exportación CSV/JSON
        │   └── ui/
        └── settings/                        # Módulo 4: Consentimiento de Sync y Enrolamiento
            └── ui/
```

---

## 2. Arquitectura del Servidor Backend (Python / FastAPI)

### 2.1. Patrón Seleccionado: Modular Layered Architecture (Package-by-Feature)
* Dado que la carga de trabajo consiste en **ingesta de telemetría inmutable (*append-only*)**, validación criptográfica de tokens de dispositivo y sanitización en frontera, el backend se divide por módulos de dominio (`devices`, `checks`, `honeypot`, `canary`), cada uno con tres capas concisas: `router.py` -> `service.py` -> `repository.py`.

### 2.2. Árbol de Directorios (Backend)
```text
honeyphone-backend/
├── alembic/                                 # Migraciones de PostgreSQL 16+
├── src/
│   ├── core/
│   │   ├── config.py                        # Variables de entorno (Pydantic Settings)
│   │   ├── database.py                      # AsyncEngine + AsyncSession (SQLAlchemy 2.0 / asyncpg)
│   │   ├── security.py                      # Argon2id (usuarios) y SHA-256 (Device API Keys)
│   │   └── sanitization.py                  # Re-verificación de máscara CIDR y truncado a 256 chars
│   ├── middleware/
│   │   ├── payload_size.py                  # Rechaza cuerpos > 64 KB con HTTP 413
│   │   ├── rate_limit.py                    # Token Bucket por IP y por X-Device-Token
│   │   └── auth_context.py                  # Resuelve device_id a partir de SHA-256(X-Device-Token)
│   ├── modules/
│   │   ├── devices/                         # Enrolamiento (anónimo o vinculado) y revocación
│   │   ├── checks/                          # Ingesta por lotes idempotentes y consulta por cursor
│   │   ├── honeypot/                        # Ingesta en tabla particionada por rango mensual
│   │   └── canary/                          # Endpoint ligero GET /api/v1/canary/probe
│   ├── api_router.py
│   └── main.py
└── tests/
```