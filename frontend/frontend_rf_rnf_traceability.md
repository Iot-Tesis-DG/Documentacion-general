# Matriz Frontend vs Tesis (RF/RNF/HU)

| RF/RNF/HU | Requisito oficial | Evidencia frontend | Estado | Tipo de datos | Observación |
|---|---|---|---|---|---|
| RF-01 | Captura SHT31 (temp+humedad) | `LecturaTermica.temperatura_ambiental`, `.humedad_ambiental` mostrados en Dashboard/Historial | No corresponde al frontend (captura es firmware) | Real (consumido de API) | Frontend correctamente solo consume/muestra |
| RF-02 | Captura DS18B20 | `LecturaTermica.temperatura_interna` | No corresponde al frontend | Real | — |
| RF-03 | Reed switch MC-38 | `LecturaTermica.apertura_refrigerador`, ícono puerta abierta/cerrada | No corresponde al frontend (captura), **sí corresponde la visualización** → Implementado y conectado | Real | — |
| RF-04 | Payload JSON | N/A frontend | No corresponde al frontend | — | — |
| RF-05 | MQTT/TLS | N/A frontend | No corresponde al frontend | — | — |
| RF-06 | Buffer offline LittleFS | N/A frontend | No corresponde al frontend | — | — |
| RF-07 | Persistencia `thermal_readings` | Consumido vía `GET /api/lecturas` | Pendiente de verificar en backend | Real (asumido) | Frontend asume que existe, no lo prueba |
| RF-08 | Clasificación Random Forest 3 estados | `nivel_riesgo` mostrado vía `RiskBadge`, nunca calculado client-side en producción | **Implementado y conectado** (frontend no calcula, solo muestra) | Real (asumido del backend) | Correcto: no hay "IA falsa" en el cliente |
| RF-09 | Alertas por riesgo | `AlertasPage`, filtros pendientes/revisadas/todas | **Implementado y conectado** | Real | — |
| RF-10 | Acciones correctivas | Diálogo en `AlertasPage`, `POST /api/alertas/{id}/acciones-correctivas` | **Implementado y conectado** | Real | — |
| RF-11 | Dashboard tiempo real SSE | `DashboardPage` + `sseClient.ts` (EventSource real) | **Implementado y conectado** | Real | Ver `frontend_realtime_analysis.md` |
| RF-12 | Filtro historial (fecha/dispositivo/conectividad/riesgo) | `HistorialPage`: filtros device_id, nivel_riesgo, desde, hasta | **Implementado parcialmente** | Real | Falta filtro explícito por "estado de conectividad" (solo se muestra la columna, no se filtra por ella) |
| RF-13 | Reportes exportables BPA | `ReportesPage` (CSV/JSON real) + `ChecklistBPAPage` (solo local) | **Implementado parcialmente / contradictorio** | Mixto | Reportes: real. Checklist: simulado/local, sin persistencia backend |
| RF-14 | Hash SHA-256 encadenado (payload+timestamp+previous_hash+hash_actual) | `TrazabilidadPage` muestra `previous_hash`/`hash_actual` truncados | **Implementado y conectado** (visualización); cálculo real es responsabilidad backend | Real (asumido) | UI no permite editar/eliminar registros — correcto, respeta inmutabilidad |
| RF-15 | Endpoint verificación integridad de cadena | Botón "Verificar" → `GET /api/trazabilidad/verificar` → muestra íntegra/rota | **Implementado y conectado** | Real | Pendiente confirmar lógica real en backend (Fase 3) |
| RF-16 | `audit_logs` de acciones críticas | `AuditoriaPage` (`GET /api/auditoria`) | **Implementado y conectado** | Real | — |
| RF-17 | JWT + RBAC 3 niveles | Login, `authStore`, `RouteGuards`, `Rol.ts` | **Implementado y conectado** | Real | Ver `frontend_auth_rbac.md` — robusto |
| RF-18 | Estado de conectividad por dispositivo | `LecturaTermica.estado_conectividad` existe en el tipo, pero **no se renderiza visiblemente en ninguna página revisada** (ni Dashboard ni Historial muestran explícitamente este campo como columna/indicador) | **Implementado parcialmente / posible ausencia visual** | Real (campo existe, uso en UI no confirmado) | Verificar si se omitió por decisión de diseño o descuido |
| RNF-01 | Latencia captura→persistencia ≤5s | N/A frontend (backend/firmware) | No corresponde al frontend | — | — |
| RNF-02 | Disponibilidad ≥95% | N/A frontend (infraestructura backend) | No corresponde al frontend | — | — |
| RNF-03 | Integridad 100% trazabilidad | Verificable vía `/api/trazabilidad/verificar`, consumido correctamente | Pendiente de verificar en backend | — | Frontend expone la UI correctamente |
| RNF-04 | F1≥0.85 modelo IA | **Ninguna pantalla muestra métricas del modelo (F1, precisión, recall, accuracy)** | **Ausente en frontend** | — | Coincide con el gap ya detectado en Fase 1 (backlog sin HU de medición) — el frontend tampoco expone ningún panel de métricas de IA |
| RNF-05 | TLS ESP32↔EMQX, sin credenciales embebidas | N/A frontend | No corresponde al frontend | — | — |
| RNF-06 | Ningún endpoint sin JWT salvo auth | Interceptor Axios inyecta Bearer en todo request si hay token | **Implementado (frontend envía correctamente)** | Real | Verificación de que el backend *rechace* sin JWT es Fase 3 |
| RNF-07 | Sync buffer ≤30s | N/A frontend | No corresponde al frontend | — | — |
| RNF-08 | DDD escalable (dominio) | Arquitectura del frontend replica capas DDD (ver `frontend_architecture.md`) | No corresponde directamente (RNF es sobre backend), pero frontend demuestra la misma disciplina arquitectónica | — | Extra positivo no exigido |
| RNF-09 | Repositorio sustituible sin alterar lógica | N/A frontend | No corresponde al frontend | — | — |
| RNF-10 | Carga inicial dashboard ≤3s | No medido en esta auditoría (requiere prueba de carga real en navegador, no solo build) | **No comprobado** | — | Build compila rápido (7s) pero eso no mide tiempo de carga en navegador real |
| HU-37 (checklist BPA) | Envío a backend firmado con JWT | `ChecklistBPAPage` usa solo `localStorage`, ninguna llamada API | **Contradictorio** | Simulado | Hallazgo alto — ver `frontend_findings.md` |
| HU-38 (export PDF) | Exportar PDF con hashes SHA-256 visibles | Solo CSV/JSON, sin librería PDF en el proyecto | **Implementado parcialmente / contradictorio** | Real (CSV/JSON) pero falta el formato exigido | Hallazgo alto |
| HU-40 (JWT en memoria) | Token nunca en localStorage | Confirmado en código: memoria únicamente en producción | **Implementado y conectado** | Real | — |
| HU-41 (RBAC 3 roles) | Rutas protegidas por rol | Confirmado, ver `frontend_auth_rbac.md` | **Implementado y conectado** | Real | — |
| HU-42 (auditoría inmutable) | Bitácora con hash SHA-256 | `/auditoria` solo lectura, sin edición | **Implementado y conectado** (frontend), cálculo real es backend | Real (asumido) | — |
| HU-43 (huérfana, gestión dispositivos) | Baja/reemplazo de dispositivo IoT | Ninguna ruta ni componente relacionado | **Ausente** | — | Coincide con hallazgo de Fase 1 (historia nunca planificada correctamente) |
