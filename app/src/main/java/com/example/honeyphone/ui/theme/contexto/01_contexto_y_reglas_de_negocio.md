# HoneyPhone: Contexto del Proyecto, Reglas de Negocio y Requerimientos
> **Propósito para Agentes de IA:** Este documento establece las restricciones innegociables de dominio, privacidad, ética de ciberseguridad y fases de implementación de **HoneyPhone**. Ninguna generación de código debe violar las reglas de negocio o límites de alcance aquí descritos.

---

## 1. Visión General del Sistema
**HoneyPhone** es una aplicación Android de diagnóstico defensivo para redes Wi-Fi complementada con un señuelo HTTP de baja interacción (*low-interaction honeypot*) y un backend desacoplado para sincronización opcional de telemetría.

### 1.1. Problema que Resuelve
Los sistemas operativos móviles informan si una red Wi-Fi tiene acceso a Internet o requiere un portal cautivo, pero no agrupan señales técnicas verificables ni las explican en lenguaje claro para un usuario no experto. HoneyPhone ejecuta comprobaciones controladas desde el propio dispositivo sin tocar el router ni inspeccionar tráfico ajeno.

---

## 2. Fases de Implementación y Prioridad Operativa (CRÍTICO)
> **REGLA DE ORDEN DE DESARROLLO:** La aplicación móvil Android es **100% Offline-First y autónoma**. La conexión con la API REST es la **última fase** del desarrollo, dado que el servidor backend evolucionará en paralelo.

1. **Fase 1 (Core Local & Diagnóstico Wi-Fi):** Lectura de capacidades de red (`ConnectivityManager` / `WifiInfo`), vinculación de sockets a `TRANSPORT_WIFI`, sonda de portal cautivo, sonda DNS controlada (sistema vs. DoH) y sonda HTTPS contra endpoint canario.
2. **Fase 2 (Persistencia Local Endurecida):** Base de datos Room + SQLCipher como *Single Source of Truth* (SSOT), generación de identificadores UUID v7 y seudonimización criptográfica de red (`HMAC-SHA256` en Android Keystore).
3. **Fase 3 (Señuelo HTTP de Baja Interacción):** `ForegroundService` con notificación persistente, `ServerSocket` en puerto `8080/TCP` protegido por semáforos, timeouts estrictos y deduplicación en caliente contra inundaciones (*anti-flood*).
4. **Fase 4 (Sincronización Diferida con API REST - Fase Final):** Implementación de `WorkManager` y cliente HTTP para enrolamiento anónimo (`X-Device-Token`) y envío por lotes idempotentes hacia FastAPI + PostgreSQL. Mientras la API no esté activa, todos los módulos deben operar mediante una interfaz desacoplada (`TelemetrySyncGateway`) con modo *No-Op / Local-Only*.

---

## 3. Reglas de Negocio Innegociables (Business Rules)

### RN-01: Prohibición de Veredictos Absolutos
* La aplicación **NUNCA** debe mostrar etiquetas como *"Red Segura"*, *"Red Libre de Hackers"* o *"Ataque MitM Confirmado"*.
* Los únicos veredictos globales permitidos son:
  * `NO_DISCREPANCIES`: Las pruebas ejecutadas coincidieron con los valores esperados (no garantiza ausencia total de riesgo).
  * `ATTENTION_REQUIRED`: Se detectó al menos una señal de precaución (ej. red abierta `OPEN`, discrepancia en respuesta DNS TXT o certificado TLS inválido) que requiere contexto.
  * `INCOMPLETE`: No se pudieron completar las pruebas por falta de permisos, ausencia de salida a Internet o caída del servidor de referencia.
* Un portal cautivo por sí solo se reporta como `PORTAL_DETECTED` (Acceso requerido), no como un ataque.

### RN-02: Privacidad por Diseño (Zero PII / Zero Geolocation Leakage)
* **Prohibido almacenar o transmitir en texto claro:** `SSID` (nombre de la red), `BSSID` (dirección MAC del punto de acceso), dirección MAC del teléfono, coordenadas GPS o historial de navegación del usuario.
* Para agrupar revisiones de la misma red localmente y en analítica, se calcula un `network_pseudonym` de 64 caracteres hexadecimales usando `HMAC-SHA256(BSSID, HardwareBacked_Keystore_Secret)`.
* Las direcciones IP de origen que se conecten al señuelo HTTP (`8080`) deben enmascararse inmediatamente en memoria antes de tocar SQLite: IPv4 truncada a `/24` (ej. `192.168.1.0/24`) e IPv6 a `/64`.

### RN-03: Aislamiento de Tráfico en Pruebas Wi-Fi (Prevención de Cellular Fallback)
* Todas las sondas activas (DNS del sistema, DoH y validación TLS) **DEBEN** ejecutarse atadas explícitamente a la interfaz `Network` que posea `NetworkCapabilities.TRANSPORT_WIFI` (`wifiNetwork.bindSocket(...)` / `wifiNetwork.getAllByName(...)`).
* Si el teléfono no tiene una conexión Wi-Fi activa, la revisión se aborta de inmediato informando al usuario (Prueba `T01`). Nunca debe evaluarse por error la red de datos móviles (4G/5G).

### RN-04: Degradación Elegante ante Restricciones de Android
* Si el usuario deniega permisos de ubicación/dispositivos cercanos o la versión de Android oculta el tipo de seguridad Wi-Fi, la app marca `security_type = UNKNOWN` con explicación clara y **continúa** ejecutando las pruebas de portal cautivo, DNS y HTTPS.
* Si se detecta que el usuario tiene activo *DNS Privado* de Android (DNS-over-TLS nativo), la señal DNS debe registrar este contexto en `evidence_payload`, ya que la consulta del sistema habrá evadido el resolvedor DHCP local.

### RN-05: Interrupción por Cambio de Red
* Si durante una revisión en curso (`Revisando`) el sistema operativo reporta desconexión del Wi-Fi o cambio de red activa, la revisión pasa a estado `Interrumpida`, se cancelan las corrutinas en vuelo y **jamás** se guarda ni presenta un resultado parcial como aprobado.

### RN-06: Seguridad y Límites del Señuelo HTTP (`8080/TCP`)
* Apagado por defecto. Solo se activa por acción explícita del usuario mediante un `ForegroundService` con notificación visible y botón de apagado inmediato.
* No simula SSH/FTP, no ejecuta comandos (`shells`) y descarta cualquier cuerpo (`body`) o parámetro de contraseña en el punto de entrada.
* Máximo 3 conexiones concurrentes (`Semaphore(3)`), `SO_TIMEOUT` de `2500 ms` y lectura máxima de `2048 bytes` de encabezados (`Request-Line`, `User-Agent`, `Content-Length`).
* **Anti-Flood en Base de Datos:** Máximo `200` eventos únicos por sesión. Peticiones idénticas desde la misma subred hacia la misma ruta incrementan `hit_count` en lugar de insertar nuevas filas.

---

## 4. Matriz de Requerimientos Funcionales (Móvil)

| ID | Nombre | Descripción Técnica |
| :--- | :--- | :--- |
| **F-01** | Revisar Wi-Fi bajo demanda | Dispara el motor `WifiCheckerEngine` sobre `TRANSPORT_WIFI` sin alterar la configuración del sistema. |
| **F-02** | Mostrar señales desglosadas | Presenta `security_type`, `captive_portal_status`, `dns_status` y `https_status`. |
| **F-03** | Explicar hallazgos y contexto | Vincula cada `explanation_code` con causas benignas probables (filtros, caché, split-DNS, reloj desincronizado) y acciones recomendadas. |
| **F-04** | Consultar historial local | Lista revisiones almacenadas en Room/SQLCipher ordenadas por `timestamp_utc DESC`. |
| **F-05** | Sincronizar informe (Fase Final) | Encola lotes `PENDING` mediante `WorkManager` hacia `/api/v1/sync/network-checks` solo con consentimiento del usuario. |
| **F-06** | Controlar señuelo HTTP | Inicia/detiene `HoneypotService` en el puerto `8080/TCP` con temporizador de auto-apagado. |
| **F-07** | Visualizar sondas recibidas | Muestra en tiempo real (vía `Flow` de Room) hora, subred enmascarada, método HTTP, ruta sanitizada, `User-Agent` y `hit_count`. |
| **F-08** | Exportar resumen local | Genera archivo JSON/CSV local depurado (sin SSID/BSSID ni datos personales). |
| **F-09** | Reportar datos no comprobados | Distingue explícitamente entre prueba superada (`MATCH`), discrepancia (`MISMATCH`) y prueba no ejecutada (`NOT_APPLICABLE` / `ERROR`). |