# Análisis Completo del Backend — IoT Cadena de Frío Farmacéutica

**Proyecto:** Sistema de Monitoreo Térmico con IA + Trazabilidad Criptográfica  
**Stack:** Python 3.12+, FastAPI, SQLAlchemy async, PostgreSQL/SQLite, MQTT (aiomqtt), scikit-learn RandomForest, JWT + RBAC, SHA-256 Hash Chain  
**Arquitectura:** Clean Architecture (Domain → Application → Infrastructure → Interface)  
**Tests:** 178 pruebas (59 unitarias + 119 integración) — todas pasan

---

## 1. CAPA DE DOMINIO

### 1.1 Entidades (5)

| Entidad | Archivo | Campos clave | HU |
|---------|---------|-------------|-----|
| `Usuario` | `entities/usuario.py` | nombre, email, password_hash, rol, privacy_accepted, privacy_version_accepted, is_active, motivo_desactivacion, desactivado_por, anonymized_for_gdpr | HU-17 (RBAC), HU-44 (privacidad Ley 29733), HU-45 (desactivación) |
| `LecturaTermica` | `entities/lectura_termica.py` | device_id, timestamp, temp_ambiental, temp_interna, humedad, apertura, nivel_riesgo, confianza_ia, modelo_version, origen_clasificacion, estado_inferencia | RF-07 (ingesta), AIV-03 (guard de sensores), AIV-07 (estado inferencia) |
| `AlertaTermica` | `entities/alerta_termica.py` | reading_id, device_id, nivel_riesgo, mensaje, revisada, episodio_abierto, lectura_inicial_id, lectura_mas_reciente_id | RF-09 (alertas), AIV-02 (episodios, anti-tormenta) |
| `AccionCorrectiva` | `entities/accion_correctiva.py` | alert_id, usuario_id, descripcion | RF-10 (acciones correctivas) |
| `RegistroTrazabilidad` | `entities/registro_trazabilidad.py` | tipo_evento, payload, timestamp, hash_encadenado, device_id, usuario_id | RF-14 (hash chain), HU-24/25 |

### 1.2 Value Objects (4)

| VO | Archivo | Descripción |
|----|---------|-------------|
| `Rol` | `value_objects/rol.py` | Enum: administrador, farmaceutico, tecnico |
| `NivelRiesgo` | `value_objects/nivel_riesgo.py` | Enum: normal, riesgo_preventivo, excursion_critica |
| `RangoTermico` | `value_objects/rango_termico.py` | Rango BPA 2°C–8°C. Métodos: contiene(), distancia_al_limite() |
| `HashEncadenado` | `value_objects/hash_encadenado.py` | SHA-256 encadenado. GENESIS_HASH = "0"×64. Métodos: calcular_hash(), encadenar(), verificar() |

### 1.3 Excepciones

| Excepción | HTTP | Contexto |
|-----------|------|----------|
| `CredencialesInvalidasError` | 401 | Login, JWT decode |
| `PermisoDenegadoError` | 403 | RBAC |
| `RecursoNoEncontradoError` | 404 | Get-by-ID fallido |
| `LecturaInvalidaError` | 422 | Rangos físicos |
| `DispositivoNoAutorizadoError` | 403 | Modo estricto |
| `CadenaTrazabilidadRotaError` | — | Verificación hash |
| `DomainError` (base) | 400/409 | Violaciones de reglas |

### 1.4 Interfaces de Repositorio (10)

| Interfaz | Métodos principales |
|----------|-------------------|
| `IUsuarioRepository` | agregar, obtener_por_email, obtener_por_id, listar, actualizar |
| `ILecturaRepository` | agregar, obtener_por_id, listar, listar_recientes_por_device, obtener_por_device_y_timestamp (dedup) |
| `IAlertaRepository` | agregar, obtener_por_id, listar, actualizar, obtener_episodio_abierto, obtener_ultimo_cerrado |
| `IAccionCorrectivaRepository` | agregar, listar_por_alerta |
| `ITrazabilidadRepository` | agregar, marcar_corrupto, marcar_posteriores_como_afectados, obtener_ultimo_hash, listar, listar_todos_ordenados |
| `IAuditLogRepository` | registrar, listar |
| `IDeviceRepository` | existe, obtener_o_crear, actualizar_estado_conectividad, listar, obtener, dar_de_baja, vincular_reemplazo, actualizar_firmware_version |
| `IFirmwareRepository` | crear_release, obtener_release, listar_releases, crear_despliegue, obtener_despliegue, actualizar_despliegue |
| `ICorrupcionRepository` | cadena_comprometida, marcar_comprometida, marcar_restaurada, guardar_snapshot_forense |
| `IReporteRepository` | registrar_exportacion |

---

## 2. CAPA DE APLICACIÓN (16 Use Cases)

### 2.1 Autenticación y Usuarios

| Use Case | Archivo | Flujo | HU |
|----------|---------|-------|-----|
| `AutenticarUsuarioUseCase` | `autenticar_usuario.py` | Busca usuario, compara hash bcrypt con señuelo anti-enumeración, verifica is_active, genera JWT con flag require_privacy_consent | RF-17, HU-44, HU-45 |
| `CrearUsuarioUseCase` | `gestionar_usuarios.py` | Valida unicidad, hashea contraseña, crea usuario | RF-17 |
| `ListarUsuariosUseCase` | `gestionar_usuarios.py` | Lista todos | RF-17 |
| `DesactivarUsuarioUseCase` | `gestionar_usuarios.py` | Valida motivo (renuncia/despido/jubilación/otros), pone is_active=False, registra metadatos GDPR | HU-45 |

### 2.2 Ingesta y Clasificación

| Use Case | Archivo | Flujo | HU |
|----------|---------|-------|-----|
| `RegistrarLecturaTermicaUseCase` | `registrar_lectura_termica.py` | Pipeline completo: autorizar dispositivo → validar rangos → deduplicar (device_id+timestamp) → clasificar riesgo (IA + salvaguarda) → persistir con evidencia IA → registrar hash chain (LECTURA_TERMICA) → evaluar episodio de alerta → registrar hash chain (ALERTA_TERMICA) | RF-07, RF-09, RF-14, AIV-02, AIV-03, B-04 |
| `ClasificarRiesgoTermicoUseCase` | `clasificar_riesgo_termico.py` | Extrae 10 features de lectura + historial (20 últimas). Guard de sensores: None/NaN/inf en sensor crítico → sin inferencia. Random Forest + regla BPA determinista. Si regla > modelo, gana la regla (previene falsos negativos) | RF-08, AIV-03 |
| `GenerarAlertaUseCase` | `generar_alerta.py` | Anti-tormenta: NORMAL cierra episodio. Mismo device+riesgo actualiza episodio (no duplica). Distinto tipo cierra y abre nuevo. Cooldown 15min anti-flapping | RF-09, AIV-02 |

### 2.3 Trazabilidad y Seguridad

| Use Case | Archivo | Flujo | HU |
|----------|---------|-------|-----|
| `RegistrarHashEncadenadoUseCase` | `registrar_hash_encadenado.py` | Cadena GLOBAL única. Lock asyncio (proceso) + pg_advisory_xact_lock (multi-worker). previous_hash_forzado para anclaje de emergencia (HU-47) | RF-14, HU-24/25, HU-47 |
| `VerificarIntegridadRegistroUseCase` | `verificar_integridad_registro.py` | Verificación O(n). Al detectar corrupción: activa flag global, guarda snapshot forense, inserta evento CORRUPCION_CADENA_DETECTADA anclado al último hash íntegro | RF-15, HU-26, HU-47 |
| `AislarCorrupcionUseCase` | `gestionar_corrupcion_cadena.py` | Marca registro corrupto + posteriores como afectados, inicia nueva cadena génesis, restaura flag global | HU-47 |

### 2.4 Dispositivos, Firmware y Privacidad

| Use Case | Archivo | Flujo | HU |
|----------|---------|-------|-----|
| `DarDeBajaDispositivoUseCase` | `gestionar_dispositivos.py` | Valida motivo (falla_hardware/mantenimiento/reemplazo/fin_de_servicio), desactiva, vincula reemplazo, registra BAJA_HARDWARE en hash chain | HU-43 |
| `PrepararFirmwareUseCase` | `gestionar_firmware.py` | Valida versión semántica, rechaza duplicados, registra FIRMWARE_PREPARACION en cadena | HU-46 |
| `ProgramarDespliegueUseCase` | `gestionar_firmware.py` | Rechaza downgrade (compara tuplas de versión), valida dispositivo y release | HU-46 |
| `EjecutarDespliegueUseCase` | `gestionar_firmware.py` | State machine: programado → exitoso/fallido/rollback. Doble check anti-downgrade. Actualiza firmware_version. Registra FIRMWARE_ACTUALIZADO/ROLLBACK en cadena | HU-46 |
| `AceptarPrivacidadUseCase` | `gestionar_privacidad.py` | Marca privacy_accepted=True, timestamps, registra ACEPTACION_PRIVACIDAD en hash chain | HU-44 |
| `RechazarPrivacidadUseCase` | `gestionar_privacidad.py` | Registra RECHAZO_PRIVACIDAD en hash chain | HU-44 |

---

## 3. CAPA DE INFRAESTRUCTURA

### 3.1 Base de Datos — 12 Modelos SQLAlchemy

| Modelo | Tabla | Columnas destacadas | Constraints |
|--------|-------|-------------------|-------------|
| `DeviceModel` | devices | id PK, nombre, ubicacion, estado_conectividad, firmware_version, activo, motivo_baja, reemplaza_a_device_id | — |
| `RoleModel` | roles | id PK, nombre UNIQUE | — |
| `UserModel` | users | id PK, email UNIQUE INDEXED, password_hash, rol_id FK, privacy_accepted, privacy_version_accepted, is_active, motivo_desactivacion, anonymized_for_gdpr | — |
| `ThermalReadingModel` | thermal_readings | device_id FK INDEXED, timestamp INDEXED, temp_ambiental, temp_interna, humedad, nivel_riesgo, payload JSONB, confianza_ia, modelo_version, origen_clasificacion, estado_inferencia | UNIQUE(device_id, timestamp) — dedup B-04 |
| `ThermalAlertModel` | thermal_alerts | reading_id FK, device_id FK INDEXED, nivel_riesgo, revisada, episodio_abierto (int/NULL), lectura_inicial_id FK, lectura_mas_reciente_id FK | UNIQUE(device_id, nivel_riesgo, episodio_abierto) — AIV-02 |
| `CorrectiveActionModel` | corrective_actions | alert_id FK, usuario_id FK, descripcion | — |
| `TraceabilityRecordModel` | traceability_records | tipo_evento, payload JSONB, previous_hash, hash_actual UNIQUE, is_corrupted, is_after_corruption | — |
| `AuditLogModel` | audit_logs | usuario_id FK, accion, recurso, detalle JSONB, ip_origen | — |
| `ReportExportModel` | report_exports | usuario_id FK, tipo_reporte, fecha_desde, fecha_hasta, archivo_url | — |
| `FirmwareReleaseModel` | firmware_releases | version UNIQUE, hash_sha256, descripcion | — |
| `FirmwareDeploymentModel` | firmware_deployments | device_id FK, version_objetivo, estado, programado_para, resultado, completado_en | — |
| `ForensicSnapshotModel` | forensic_snapshots | registro_id FK, detalle JSONB | — |
| `SystemStateModel` | system_state | id PK=1 (singleton), cadena_comprometida | — |

### 3.2 Inteligencia Artificial

**Entrenamiento (`train_model_v3.py`):**
- 400 escenarios sintéticos (15–35 ticks c/u), 3 regímenes (estable 45%, deriva preventiva 25%, excursión crítica 30%)
- Ground truth etiquetado por regla BPA determinista, entrenado con ruido de sensor realista
- GroupShuffleSplit (80/20), StratifiedGroupKFold 5-fold (sin leakage entre escenarios)
- RandomForest: 200 árboles, max_depth=12, class_weight="balanced"
- RNF-04: F1 ≥ 0.85 obligatorio, si no, error en entrenamiento
- Baselines: DummyClassifier + DecisionTreeClassifier
- AIV-06: modelo guardado como estimator puro, SHA-256 post-guardado, metadata en JSON separado

**Inferencia (`RandomForestRiesgoService`):**
- Carga lazy con validación de compatibilidad (features, clases, orden, checksum SHA-256)
- Guard de sensores: None/NaN/inf en temperatura_interna → sin inferencia (fallo_sensor)
- Salvaguarda determinista: si `regla > modelo`, gana la regla
- Confianza: solo cuando origen=random_forest, nunca 0.0 como centinela
- Versiones: v1 (solo estimator), v2 (dict con metadata embebida), v3 (estimator + metadata.json externo)

**Features (10):** temperatura_ambiental, humedad_ambiental, temperatura_interna, diferencia_sensores, duracion_fuera_rango, frecuencia_desviaciones, tendencia_termica, apertura_refrigerador, hora_evento, estado_conectividad_online

### 3.3 Seguridad

| Componente | Archivo | Función |
|-----------|---------|--------|
| `JWTHandler` | `security/jwt_handler.py` | Crea/decodifica JWT con claims iss, aud, iat, jti. Tickets SSE efímeros (60s, single-use, audience separada) |
| `PasswordHasher` | `security/password_hasher.py` | Bcrypt vía passlib, salt automático |
| `SlidingWindowRateLimiter` | `security/rate_limiter.py` | Ventana deslizante en memoria, 5 intentos/5min por IP para login. LRU eviction (10K entradas) |
| `JtiStore` | `security/revocation_store.py` | Revocación server-side de JWT (logout). Consumo único de tickets SSE. Purga automática por expiración |
| `verificar_permiso()` | `security/rbac.py` | ADMIN acceso implícito a todo. Otros roles requieren permiso explícito |

### 3.4 MQTT

- **Broker:** EMQX Cloud Serverless (MQTT sobre TLS 1.2)
- **Topics:** `farmacias/+/lecturas`, `farmacias/+/eventos`
- **Anti-spoofing:** device_id en payload debe coincidir con segmento del topic
- **Deduplicación:** (device_id, timestamp) único en BD — idempotente ante QoS1 replay (B-04)
- **Degradación graceful:** si MQTT no conecta, backend arranca sin él

---

## 4. CAPA DE INTERFAZ — Catálogo Completo de Endpoints (33)

### 4.1 Auth (`/api/auth`) — 5 endpoints

| Método | Ruta | Auth | Roles | Descripción |
|--------|------|------|-------|-------------|
| POST | `/api/auth/login` | Form OAuth2 | — | Login con rate limiting (5/5min), señuelo anti-enumeración, auditoría, flag require_privacy_consent |
| POST | `/api/auth/sse-ticket` | Bearer JWT | activo | Ticket efímero 60s single-use para SSE |
| POST | `/api/auth/privacidad/aceptar` | Bearer (sin check privacidad) | activo | Acepta política, registra en hash chain |
| POST | `/api/auth/privacidad/rechazar` | Bearer (sin check privacidad) | activo | Rechaza política, revoca token, registra en hash chain |
| POST | `/api/auth/logout` | Bearer JWT | activo | Revoca JWT server-side, audita |

### 4.2 Usuarios (`/api/usuarios`) — 3 endpoints

| Método | Ruta | Auth | Roles | Descripción |
|--------|------|------|-------|-------------|
| POST | `/api/usuarios` | Bearer | admin | Crea usuario (valida contraseña letras+números, min 10 chars) |
| GET | `/api/usuarios` | Bearer | admin | Lista todos |
| PATCH | `/api/usuarios/{id}/desactivar` | Bearer | admin | Desactiva usuario, registra en hash chain |

### 4.3 Lecturas (`/api/lecturas`) — 3 endpoints

| Método | Ruta | Auth | Roles | Descripción |
|--------|------|------|-------|-------------|
| POST | `/api/lecturas` | Bearer | técnico, farmacéutico | Ingesta REST (pipeline completo: autorizar, validar, clasificar, persistir, hash chain, alerta, SSE) |
| GET | `/api/lecturas` | Bearer | técnico, farmacéutico | Historial con filtros (device, riesgo, conectividad, rango fechas, paginación) |
| GET | `/api/lecturas/{id}` | Bearer | técnico, farmacéutico | Lectura por ID |

### 4.4 Alertas (`/api/alertas`) — 3 endpoints

| Método | Ruta | Auth | Roles | Descripción |
|--------|------|------|-------|-------------|
| GET | `/api/alertas` | Bearer | técnico, farmacéutico | Lista alertas (filtros: device, revisada, paginación) |
| PATCH | `/api/alertas/{id}/revisar` | Bearer | farmacéutico | Marca revisada, audita |
| POST | `/api/alertas/{id}/acciones-correctivas` | Bearer | técnico, farmacéutico | Registra acción correctiva, registra en hash chain |

### 4.5 Trazabilidad (`/api/trazabilidad`) — 4 endpoints

| Método | Ruta | Auth | Roles | Descripción |
|--------|------|------|-------|-------------|
| GET | `/api/trazabilidad` | Bearer | técnico, farmacéutico | Lista registros (filtros: tipo_evento, device, paginación) |
| GET | `/api/trazabilidad/verificar` | Bearer | técnico, farmacéutico | Verificación O(n) de cadena completa. Detecta corrupción, activa flag, guarda snapshot, inserta evento emergencia |
| GET | `/api/trazabilidad/estado` | Bearer | técnico, farmacéutico | Estado global del flag cadena_comprometida |
| POST | `/api/trazabilidad/corrupcion/{id}/aislar` | Bearer | admin | Aísla registro corrupto, inicia nueva cadena génesis |

### 4.6 Reportes (`/api/reportes`) — 1 endpoint

| Método | Ruta | Auth | Roles | Descripción |
|--------|------|------|-------|-------------|
| GET | `/api/reportes/bpa` | Bearer | farmacéutico | Exporta reporte BPA (lecturas + alertas + trazabilidad en rango) |

### 4.7 Auditoría (`/api/auditoria`) — 1 endpoint

| Método | Ruta | Auth | Roles | Descripción |
|--------|------|------|-------|-------------|
| GET | `/api/auditoria` | Bearer | admin | Lista audit logs (paginado) |

### 4.8 IA (`/api/ia`) — 2 endpoints

| Método | Ruta | Auth | Roles | Descripción |
|--------|------|------|-------|-------------|
| GET | `/api/ia/modelo` | Bearer | farmacéutico | Metadata del modelo + métricas de entrenamiento (RNF-04) |
| POST | `/api/ia/clasificar` | Bearer | farmacéutico | Clasifica vector de features arbitrario (sin persistencia) |

### 4.9 SSE (`/api/sse`) — 1 endpoint

| Método | Ruta | Auth | Roles | Descripción |
|--------|------|------|-------|-------------|
| GET | `/api/sse/lecturas` | ticket (query param) | single-use | Stream SSE para dashboard. Keep-alive cada 15s. Tipos de evento: lectura, alerta, episodio_actualizado, recuperacion, fallo_sensor, inferencia_omitida |

### 4.10 Dispositivos (`/api/dispositivos`) — 2 endpoints

| Método | Ruta | Auth | Roles | Descripción |
|--------|------|------|-------|-------------|
| GET | `/api/dispositivos` | Bearer | admin | Lista dispositivos |
| POST | `/api/dispositivos/{id}/baja` | Bearer | admin | Da de baja, vincula reemplazo, registra BAJA_HARDWARE en hash chain |

### 4.11 Firmware OTA (`/api/firmware`) — 4 endpoints

| Método | Ruta | Auth | Roles | Descripción |
|--------|------|------|-------|-------------|
| POST | `/api/firmware/releases` | Bearer | admin | Prepara release (versión semántica, no duplicados) |
| GET | `/api/firmware/releases` | Bearer | admin | Lista releases |
| POST | `/api/firmware/despliegues` | Bearer | admin | Programa despliegue (rechaza downgrade) |
| POST | `/api/firmware/despliegues/{id}/ejecutar` | Bearer | admin | Ejecuta despliegue (state machine, doble check downgrade, actualiza firmware_version) |

### 4.12 Health

| Método | Ruta | Auth | Roles | Descripción |
|--------|------|------|-------|-------------|
| GET | `/health` | — | — | Health probe, exento de rate limiting |

---

## 5. ARQUITECTURA DE SEGURIDAD

### 5.1 Middleware de Autenticación (deps.py)

```
get_current_user:
  1. Decodifica JWT (firma, expiración, issuer, audience)
  2. Verifica revocación (jti en JtiStore)
  3. Carga usuario de BD
  4. Verifica is_active (HU-45) → 401 si desactivado
  5. Verifica privacy_accepted (HU-44) → 403 si no aceptado
  
get_current_user_sin_privacidad:  [para endpoints aceptar/rechazar privacidad]
  1-3. Ídem pero SIN check de privacidad (paso 5)
  4. Verifica is_active → 401 si desactivado
```

### 5.2 RBAC (Control de Acceso por Roles)

| Rol | Permisos |
|-----|----------|
| ADMINISTRADOR | Acceso implícito a todo |
| FARMACEUTICO | Lecturas (CRUD), alertas (revisar), acciones correctivas, trazabilidad, reportes BPA, IA |
| TECNICO | Lecturas (ingesta/consulta), alertas (consulta), acciones correctivas, trazabilidad |

### 5.3 Capas de Defensa

| Capa | Mecanismo | OWASP |
|------|-----------|-------|
| Rate limiting login | 5 intentos/5min por IP, ventana deslizante | API4 |
| Rate limiting global | 240 req/min por IP, health exento | API4 |
| Revocación JWT | JtiStore server-side, logout real | — |
| Anti-enumeración | Hash señuelo en login, tiempo constante | API2 |
| Security headers | X-Content-Type-Options, X-Frame-Options, CSP, HSTS, Referrer-Policy | API7 |
| Body size limit | 64KB máximo, conteo real de bytes (chunked-safe) | API4 |
| CORS restringido | Solo origins explícitos, métodos GET/POST/PATCH/DELETE | — |
| TrustedHost | Solo allowed_hosts en producción | — |
| Validación producción | Rechaza secretos default, CORS wildcard, hosts vacíos, MQTT sin TLS | — |

### 5.4 Eventos Registrados en Hash Chain

LECTURA_TERMICA, ALERTA_TERMICA, ACCION_CORRECTIVA, FIRMWARE_PREPARACION, FIRMWARE_ACTUALIZADO, FIRMWARE_ROLLBACK, ACEPTACION_PRIVACIDAD, RECHAZO_PRIVACIDAD, DESACTIVACION_USUARIO, BAJA_HARDWARE, CORRUPCION_CADENA_DETECTADA, REGISTRO_AISLADO_CORRUPCION

### 5.5 Concurrencia en Hash Chain

- **PostgreSQL:** `pg_advisory_xact_lock` — protege ante múltiples workers/instancias
- **SQLite:** `asyncio.Lock` de proceso — auto-rebind en cambio de event loop
- **Ambos:** sección crítica "leer último hash + insertar siguiente bloque" es atómica

---

## 6. COBERTURA DE PRUEBAS — 178 Tests

### 6.1 Unitarias (59 tests en 13 archivos)

| Archivo | Qué prueba |
|---------|-----------|
| test_hash_encadenado.py | Hash determinista, detección de alteración, independencia de orden de claves |
| test_jwt_handler.py | Creación/decodificación, expiración, secreto incorrecto, token malformado |
| test_password_hasher.py | Bcrypt hash, verificación, unicidad de salt |
| test_random_forest_service.py | Disponibilidad, clasificación normal/crítica, fallback sin modelo, confianza+origen, salvaguarda, metadata v3 |
| test_random_forest_service_validacion.py | Rechazo por feature count, clases desconocidas, orden de features, checksum SHA-256 |
| test_rango_termico.py | Contención, distancias, rechazo de rango inválido |
| test_rate_limiter.py | No crea claves en consulta, LRU eviction |
| test_rbac.py | Admin universal, acceso por rol, denegación |
| test_reglas_riesgo.py | Umbrales normal/preventivo/crítico, tendencia, proximidad |
| test_revocation_store.py | Registro/contención, purga por expiración, consumo único, acotación |
| test_sha256_service.py | Cadena válida, detección de alteración, registro faltante |
| test_api_protection.py | Rechazo de cuerpo chunked sin Content-Length |

### 6.2 Integración (119 tests en 20 archivos)

| Archivo | Escenarios |
|---------|-----------|
| test_auth_api.py | Login éxito/fallo, protección de endpoints |
| test_usuarios_api.py | CRUD usuarios, duplicados, RBAC |
| test_lecturas_api.py | Ingesta normal/crítica, fuera de rango, historial, 404, RBAC |
| test_alertas_api.py | Generación automática, revisión, acción correctiva, 404 |
| test_trazabilidad_api.py | Registro por lectura, cadena vacía/varias lecturas, integridad con críticas |
| test_reportes_api.py | Exportación BPA, RBAC |
| test_auditoria_api.py | Auditoría de creación, RBAC |
| test_ia_api.py | Metadata + métricas RNF-04, clasificación, RBAC |
| test_seguridad_api.py | 26 pruebas: headers, CSP, rate limiting login/global, enumeración, auditoría, claims JWT, tickets SSE, revocación, políticas de contraseña, validación producción, registro estricto, límite cuerpo, Content-Length malformado |
| test_dispositivos_api.py | Listado, baja, reemplazo, motivo inválido, 404/409, preservación histórico |
| test_firmware_api.py | Release CRUD, despliegue, anti-downgrade, ejecución, conflictos, trazabilidad |
| test_privacidad_api.py | Login requiere consentimiento, aceptar sin re-prompt, rechazar revoca token, hash chain |
| test_desactivar_usuario_api.py | Admin desactiva, login bloqueado, token inválido, motivo inválido, 404, doble desactivación, RBAC, trazabilidad |
| test_corrupcion_cadena_api.py | Verificación sin corrupción, detección + notificación, flag global, evento emergencia, aislamiento, RBAC |
| test_hash_chain_concurrencia.py | 20 escrituras concurrentes no bifurcan, verificación lineal estricta |
| test_ingesta_dedup_y_sensor_nulo.py | Dedup MQTT replay, sensor None ≠ 0.0°C, path normal post B-05 |
| test_ai_persistencia_version.py | Confianza + versión persistidas, sin temp interna = NULL confianza |
| test_correcciones_p1.py | AIV-02: 5 críticas = 1 alerta, cierre por normal, escalamiento, concurrencia sin duplicados. AIV-03: NaN/inf rechazados, doble capa guard, ambos sensores ausentes, fallback con historial, 0.0°C real detectado |

---

## 7. FLUJO PRINCIPAL: INGESTA DE LECTURA TÉRMICA

```
ESP32 → MQTT (TLS 1.2, QoS 1) → EMQX Cloud → aiomqtt (backend)
                                    ↓
1. Anti-spoofing: device_id en payload == segmento del topic
2. Validación Pydantic: LecturaPayload (extra=forbid, allow_inf_nan=False)
3. Deduplicación: SELECT por (device_id, timestamp) — si existe, ignorar (QoS1 replay)
4. Autorización: si modo estricto, verificar device en tabla; si no, auto-crear
5. Construir LecturaTermica (entidad de dominio)
                                    ↓
6. Clasificar riesgo:
   a. Obtener últimas 20 lecturas del device
   b. Extraer 10 features (sensores + históricas)
   c. Guard: si temp_interna es None/NaN/inf → sin inferencia (fallo_sensor)
   d. Random Forest → probabilidades + clase
   e. Regla BPA determinista → clase
   f. Salvaguarda: si regla > modelo, regla gana
   g. Resultado: nivel_riesgo, confianza_ia, origen, estado_inferencia
                                    ↓
7. Persistir ThermalReadingModel (con evidencia IA)
8. Registrar hash chain: LECTURA_TERMICA
                                    ↓
9. Evaluar alerta (GenerarAlertaUseCase):
   - NORMAL: cierra episodio abierto → SSE: recuperacion
   - Mismo device+riesgo que episodio abierto: actualiza → SSE: episodio_actualizado
   - Distinto tipo: cierra viejo, abre nuevo → SSE: alerta
   - Sin episodio + cooldown >15min: crea nuevo → SSE: alerta
   - Sin episodio + cooldown ≤15min: reabre → SSE: alerta
                                    ↓
10. Si se generó/modificó alerta: registrar hash chain ALERTA_TERMICA
11. SSE broadcast a todos los dashboards conectados
12. Commit transacción
```

---

## 8. ESTRUCTURA DE DIRECTORIOS

```
tesis/
├── .env                          # Variables de entorno
├── requirements.txt              # 18 dependencias producción
├── requirements-dev.txt          # + pytest, httpx, aiosqlite
├── pytest.ini                    # asyncio_mode=auto
├── alembic.ini                   # Migraciones
├── alembic/versions/             # 5 migraciones
├── scripts/seed_dev.py           # Seed de datos desarrollo
│
├── src/
│   ├── domain/
│   │   ├── entities/             # 5 entidades
│   │   ├── value_objects/        # 4 value objects
│   │   ├── repositories/         # 10 interfaces
│   │   └── exceptions.py         # 7 excepciones
│   │
│   ├── application/
│   │   └── use_cases/            # 16 casos de uso
│   │
│   ├── infrastructure/
│   │   ├── ai/                   # Random Forest (4 archivos + models/)
│   │   ├── database/             # 12 modelos + session + 10 repos
│   │   ├── hash/                 # SHA-256 service
│   │   ├── mqtt/                 # Cliente MQTT + schema
│   │   ├── security/             # JWT, bcrypt, RBAC, rate limiter, revocation
│   │   └── config.py             # Settings (98 líneas, validación producción)
│   │
│   └── interface/
│       ├── main.py               # Factory FastAPI, lifespan, MQTT handler
│       └── api/
│           ├── deps.py           # Dependencias FastAPI (auth, RBAC, privacidad)
│           ├── schemas.py        # 25+ schemas Pydantic
│           ├── mappers.py        # Entidad → Response
│           ├── sse_broadcaster.py
│           ├── security_headers.py
│           ├── api_protection.py
│           └── *_router.py       # 11 routers
│
└── tests/
    ├── conftest.py               # Fixtures: SQLite en memoria, TestClient
    ├── unit/                     # 13 archivos (59 tests)
    └── integration/              # 20 archivos (119 tests)
```

---

## 9. RESUMEN POR HISTORIA DE USUARIO

| HU | Título | Endpoints | Archivos clave |
|----|--------|-----------|---------------|
| HU-01 a HU-08 | Edge/ESP32 (sensores, buffer, MQTT) | — (capa Edge, no backend) | `mqtt_client.py`, `payload_schema.py` |
| HU-09/10 | TLS, autenticación dispositivos | — | `config.py` (MQTT TLS settings) |
| HU-11 | QoS1 | — | `models.py` (dedup UniqueConstraint) |
| HU-12 | aiomqtt suscripción | — | `main.py` (lifespan MQTT) |
| HU-13 | Last Will | — | `mqtt_client.py` |
| HU-14 | Bloqueo puerto 1883 | — | `config.py` (TLS requerido en prod) |
| HU-15 | Validación Pydantic | — | `payload_schema.py`, `schemas.py` |
| HU-16 | Carga Random Forest | — | `random_forest_service.py` |
| HU-17 | Features + clasificación | POST /api/ia/clasificar | `clasificar_riesgo_termico.py`, `features.py`, `ia_router.py` |
| HU-18 | Clasificación 3 niveles | — | `random_forest_service.py`, `reglas_riesgo.py` |
| HU-19 | Dashboard SSE | GET /api/sse/lecturas | `sse_broadcaster.py`, `sse_router.py` |
| HU-20 | Alerta riesgo preventivo | — | `generar_alerta.py` (AIV-02) |
| HU-21 | Excursión crítica 2°C–8°C | — | `rango_termico.py`, `reglas_riesgo.py` |
| HU-22 | JSONB + persistencia async | — | `models.py` (JSONB variant) |
| HU-23 | Notificaciones email/Telegram | — | (no implementado aún) |
| HU-24/25 | Hash SHA-256 + encadenamiento | GET /api/trazabilidad/* | `hash_encadenado.py`, `registrar_hash_encadenado.py`, `trazabilidad_router.py` |
| HU-26 | Verificación en bloque | GET /api/trazabilidad/verificar | `verificar_integridad_registro.py` |
| HU-27/28 | Acciones correctivas + hash | POST /api/alertas/{id}/acciones-correctivas | `registrar_accion_correctiva.py` |
| HU-29 | Backup BD | — | (Railway snapshots automáticos) |
| HU-30 | Calibración sensores | — | (no implementado aún) |
| HU-31/32 | Gráfica + SSE frontend | — | (capa frontend React) |
| HU-33/34/35 | KPIs, semáforo, puerta | — | (capa frontend, backend vía SSE) |
| HU-36/37/38 | Historial, checklist, PDF | GET /api/reportes/bpa | `exportar_reporte_bpa.py`, `reportes_router.py` |
| HU-39/40/41 | Login seguro, JWT memoria, RBAC | POST /api/auth/* | `autenticar_usuario.py`, `jwt_handler.py`, `rbac.py`, `deps.py` |
| HU-42 | Auditoría automatizada | GET /api/auditoria | `auditar_accion_critica.py`, `auditoria_router.py` |
| HU-43 | Ciclo de vida hardware | POST /api/dispositivos/{id}/baja | `gestionar_dispositivos.py`, `dispositivos_router.py` |
| HU-44 | Aceptación privacidad (Ley 29733) | POST /api/auth/privacidad/* | `gestionar_privacidad.py`, `deps.py` |
| HU-45 | Anonimización usuarios | PATCH /api/usuarios/{id}/desactivar | `gestionar_usuarios.py`, `usuarios_router.py` |
| HU-46 | OTA firmware seguro | POST /api/firmware/* | `gestionar_firmware.py`, `firmware_router.py` |
| HU-47 | Corrupción cadena hash | POST /api/trazabilidad/corrupcion/{id}/aislar | `gestionar_corrupcion_cadena.py`, `trazabilidad_router.py` |
