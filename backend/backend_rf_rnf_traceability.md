# Matriz RF/RNF/HU — consolidado con evidencia de backend

| RF/RNF/HU | Requisito | Evidencia backend | Evidencia frontend | Estado consolidado | Observación |
|---|---|---|---|---|---|
| RF-01 | Captura SHT31 | `LecturaPayload`, `thermal_readings.temperatura_ambiental/humedad_ambiental` | Consume vía API | **Implementado solo en backend** (captura real es firmware, fuera de repo) | Backend recibe y persiste correctamente |
| RF-02 | Captura DS18B20 | ídem `temperatura_interna` | ídem | **Implementado solo en backend** | — |
| RF-03 | Reed switch MC-38 | `apertura_refrigerador` persistido y usado como feature | Visualizado (ícono puerta) | **Implementado integralmente** | — |
| RF-04 | Payload JSON | `LecturaPayload` (Pydantic) | N/A | **Implementado solo en backend** | — |
| RF-05 | MQTT/TLS | `mqtt_client.py`, TLS forzado en producción | N/A | **Implementado solo en backend** | — |
| RF-06 | Buffer offline LittleFS | Firmware ESP32, **fuera de este repositorio** (no verificable) | N/A | **No evaluable** | Ningún código de firmware C++/ESP-IDF existe en los repos auditados |
| RF-07 | Persistencia `thermal_readings` | Tabla real, migración+ORM coincidentes | Consume vía `/api/lecturas` | **Implementado integralmente** | — |
| RF-08 | Clasificación Random Forest 3 estados | Modelo real entrenado, F1=0.9659, salvaguarda de regla | Muestra `nivel_riesgo` sin calcularlo | **Implementado integralmente** | Ver matiz regla+IA en `backend_ai_validation.md` |
| RF-09 | Alertas por riesgo | `GenerarAlertaUseCase`, tabla `thermal_alerts` | `AlertasPage` | **Implementado integralmente** | — |
| RF-10 | Acciones correctivas | Endpoint+caso de uso+auditoría+trazabilidad completos | `AlertasPage` (diálogo) | **Implementado integralmente** | — |
| RF-11 | Dashboard tiempo real SSE | `sse_router.py`, ticket de un solo uso, broadcaster real | `sseClient.ts` EventSource real | **Implementado integralmente** | Con asimetría: SSE solo se emite en camino MQTT, no en ingesta HTTP |
| RF-12 | Filtro historial | Backend soporta `device_id, nivel_riesgo, estado_conectividad, desde, hasta` | Frontend solo usa 4 de 5 (falta `estado_conectividad`) | **Implementado en backend, parcial en frontend** | Backend más completo que lo que el frontend expone |
| RF-13 | Reportes exportables BPA | Datos reales vía `/api/reportes/bpa`, auditado; **sin generación de PDF** | CSV/JSON reales, sin PDF | **Implementado parcialmente (sin PDF) en ambos** | Confirmado ausente en ambos lados |
| RF-14 | Hash SHA-256 encadenado | `HashEncadenado`, estructura exacta confirmada, tests unitarios reales | Visualiza hashes truncados, solo lectura | **Implementado integralmente**, con defecto de concurrencia | Ver `backend_hash_chain_analysis.md` |
| RF-15 | Endpoint verificación integridad | `VerificarIntegridadRegistroUseCase`, O(n), detecta alteración real | Botón conectado, muestra resultado | **Implementado integralmente**, con el mismo defecto de concurrencia que puede generar falso positivo | — |
| RF-16 | `audit_logs` | 8 tipos de acción auditados, tabla real | `AuditoriaPage` | **Implementado integralmente** | — |
| RF-17 | JWT + RBAC 3 niveles | JWT completo (iss/aud/exp/jti), revocación real, RBAC server-side verificado en cada endpoint sensible | JWT en memoria, RBAC de rutas real | **Implementado integralmente** | Robusto en ambos lados, coincidencia exacta de roles |
| RF-18 | Estado conectividad por dispositivo | Campo persistido y **filtrable** en `/api/lecturas` | **No renderizado en ninguna pantalla** | **Implementado solo en backend** | Confirma que Fase 2 detectó correctamente una omisión exclusiva de frontend |
| RNF-01 | Latencia ≤5s captura→persistencia | Pipeline sin operaciones bloqueantes evidentes; no medido con instrumentación real | N/A | **No evaluable** (requiere prueba de carga real) | — |
| RNF-02 | Disponibilidad ≥95% | Infraestructura (Railway), no medible desde código | N/A | **No evaluable** | — |
| RNF-03 | Integridad 100% trazabilidad | Mecanismo real, **pero con condición de carrera confirmada que puede producir falsos negativos de integridad bajo concurrencia** | Expone verificación correctamente | **Implementado con defecto confirmado** | Hallazgo crítico — ver `backend_hash_chain_analysis.md` |
| RNF-04 | F1≥0.85 modelo IA | **Cumplido y demostrado: F1=0.9659, con build que falla si no se alcanza el umbral** | Sin superficie de UI (endpoint existe, no consumido) | **Implementado integralmente en backend, ausente en frontend** | El backend resuelve por completo el gap que Fase 2 detectó — solo falta exponerlo visualmente |
| RNF-05 | TLS + credenciales no embebidas | Validado por configuración forzada en producción | N/A (frontend no maneja MQTT) | **Implementado integralmente** | — |
| RNF-06 | Ningún endpoint sin JWT salvo auth | Confirmado: todos los endpoints mutantes/sensibles requieren `CurrentUserDep`/`require_roles` | Envía Bearer correctamente | **Implementado integralmente** | — |
| RNF-07 | Sync buffer ≤30s | Firmware, fuera de repo | N/A | **No evaluable** | — |
| RNF-08 | DDD escalable (dominio) | Confirmado: arquitectura DDD real, dirección de dependencias correcta | Réplica el mismo patrón de capas | **Implementado integralmente** | — |
| RNF-09 | Repositorio sustituible | Interfaces de dominio + implementaciones SQLAlchemy separadas; también soporta SQLite en tests sin cambiar lógica | N/A | **Implementado integralmente** | — |
| RNF-10 | Carga dashboard ≤3s | N/A backend | No medido en navegador real | **No evaluable** | — |
| HU-37 (checklist BPA) | Persistencia backend firmada JWT | **Ausente — cero código relacionado** | Solo `localStorage` | **Ausente en ambos componentes** | Corrige la clasificación preliminar de Fase 2 |
| HU-38 (export PDF) | PDF con hashes visibles | **Ausente — sin librería PDF, campo `archivo_url` sin poblar** | Solo CSV/JSON | **Ausente en ambos componentes** | Corrige la clasificación preliminar de Fase 2 |
| HU-43 (huérfana, gestión dispositivos) | Baja/reemplazo de dispositivo IoT | Sin router de gestión CRUD de `devices` (solo alta automática interna) | Sin ruta ni componente | **Ausente en ambos componentes** | Coincide con el hallazgo de backlog corrupto de Fase 1 |
