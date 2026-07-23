# Mapa completo de endpoints

| Método | Ruta | Router | Caso de uso | Auth | Roles | Tablas | Consumido por frontend | Estado |
|---|---|---|---|---|---|---|---|---|
| POST | `/api/auth/login` | auth_router | `AutenticarUsuarioUseCase` | Público | — | `users`, `roles`, `audit_logs` | Sí (`authService.login`) | **Implementado y operativo** |
| POST | `/api/auth/sse-ticket` | auth_router | (directo, `JWTHandler.crear_ticket_sse`) | JWT | cualquiera autenticado | — | Sí (`sseClient.ts`) | **Implementado y operativo** |
| POST | `/api/auth/logout` | auth_router | (directo, revocación jti) | JWT | cualquiera autenticado | `audit_logs` | **No consumido por frontend** (frontend solo limpia el token en memoria, nunca llama a `/logout`) | **Implementado, no utilizado por frontend** |
| POST | `/api/usuarios` | usuarios_router | `CrearUsuarioUseCase` | JWT | administrador | `users`, `audit_logs` | Sí (`useUsuarios.crear`) | **Implementado y operativo** |
| GET | `/api/usuarios` | usuarios_router | `ListarUsuariosUseCase` | JWT | administrador | `users` | Sí (`useUsuarios`) | **Implementado y operativo** |
| POST | `/api/lecturas` | lecturas_router | `RegistrarLecturaTermicaUseCase` | JWT | técnico, farmacéutico (+admin) | `thermal_readings`, `thermal_alerts`, `traceability_records`, `devices` | **No consumido por frontend** (el frontend nunca publica lecturas vía HTTP — solo lee; la ingesta real ocurre por MQTT) | **Implementado y operativo** (vía HTTP como ruta alterna de ingesta, probablemente para pruebas/backfill) |
| GET | `/api/lecturas` | lecturas_router | `ConsultarHistorialTermicoUseCase` | JWT | técnico, farmacéutico (+admin) | `thermal_readings` | Sí (`useHistorial`, `useMonitoreoTermico`) | **Implementado y operativo** |
| GET | `/api/lecturas/{id}` | lecturas_router | (directo, repositorio) | JWT | técnico, farmacéutico (+admin) | `thermal_readings` | **No consumido por frontend** | **Implementado, no utilizado por frontend** |
| GET | `/api/alertas` | alertas_router | `ConsultarAlertasUseCase` | JWT | técnico, farmacéutico (+admin) | `thermal_alerts` | Sí (`useAlertas`) | **Implementado y operativo** |
| PATCH | `/api/alertas/{id}/revisar` | alertas_router | `MarcarAlertaRevisadaUseCase` | JWT | **solo farmacéutico** (+admin) | `thermal_alerts`, `audit_logs` | Sí (`useAlertas.marcarRevisada`) | **Implementado y operativo** — coincide exactamente con el `puedeRevisar` del frontend |
| POST | `/api/alertas/{id}/acciones-correctivas` | alertas_router | `RegistrarAccionCorrectivaUseCase` | JWT | técnico, farmacéutico (+admin) | `corrective_actions`, `traceability_records`, `audit_logs` | Sí (`useAlertas.registrarAccionCorrectiva`) | **Implementado y operativo** |
| GET | `/api/trazabilidad` | trazabilidad_router | (directo, repositorio) | JWT | técnico, farmacéutico (+admin) | `traceability_records` | Sí (`useTrazabilidad`) | **Implementado y operativo** |
| GET | `/api/trazabilidad/verificar` | trazabilidad_router | `VerificarIntegridadRegistroUseCase` | JWT | técnico, farmacéutico (+admin) | `traceability_records` | Sí (`useTrazabilidad.verificarIntegridad`) | **Implementado y operativo** |
| GET | `/api/reportes/bpa` | reportes_router | `ExportarReporteBPAUseCase` | JWT | **solo farmacéutico** (+admin) | `thermal_readings`, `thermal_alerts`, `traceability_records`, `report_exports`, `audit_logs` | Sí (`useReportesBPA`) | **Implementado y operativo** (JSON únicamente; sin generación de PDF — ver `backend_reports_bpa.md`) |
| GET | `/api/auditoria` | auditoria_router | (directo, repositorio) | JWT | **solo administrador** | `audit_logs` | Sí (`useAuditoria`) | **Implementado y operativo** |
| GET | `/api/ia/modelo` | ia_router | (directo, `RandomForestRiesgoService.metricas_entrenamiento`) | JWT | **solo farmacéutico** (+admin) | — (lee `training_metrics.json`) | **No consumido por frontend** — endpoint existe con evidencia real de RNF-04, pero ninguna pantalla lo llama (hallazgo cruzado con F-03 de Fase 2) | **Implementado, no utilizado por frontend** |
| POST | `/api/ia/clasificar` | ia_router | (directo, `RandomForestRiesgoService.inferir`) | JWT | **solo farmacéutico** (+admin) | — | **No consumido por frontend** | **Implementado, no utilizado por frontend** |
| GET | `/api/sse/lecturas?ticket=` | sse_router | (directo, `SSEBroadcaster`) | Ticket SSE (no Bearer) | cualquiera con ticket válido | — | Sí (`sseClient.ts` EventSource) | **Implementado y operativo** |
| GET | `/health` | main.py | (directo) | Público | — | — | No consumido por frontend (probe de infraestructura) | **Implementado y operativo** |
| GET | `/docs`, `/redoc`, `/openapi.json` | FastAPI auto | — | Público **solo si no es producción** | — | — | No | **Implementado, deshabilitado correctamente en producción** |

## Endpoints esperados por TI/backlog que NO existen

| Endpoint esperado | Origen de la expectativa | Estado |
|---|---|---|
| Checklist BPA (cualquier método) | HU-37 | **Ausente** — confirmado por `grep` exhaustivo de "checklist" en todo `src/`: cero resultados |
| Exportación PDF (`/api/reportes/bpa?formato=pdf` o similar) | HU-38, TI línea 647 | **Ausente** — cero librerías PDF en `requirements.txt`, `ReportExportModel.archivo_url` existe en el esquema pero nunca se puebla (siempre `None`) |
| Gestión de dispositivos (alta/baja/reemplazo) | HU-43 (huérfana) | **Ausente** — no hay router `dispositivos_router.py` ni endpoints CRUD sobre `devices` más allá del alta automática/estricta interna del pipeline de ingesta |
| Refresh token | Buena práctica JWT esperable | **Ausente** — no hay endpoint de refresh; el modelo es JWT de vida corta (60 min por defecto) sin renovación, coherente con "JWT en memoria" del frontend (al expirar, se relogea) |

## Nota sobre RF-18 (estado de conectividad)

El backend **sí expone** `estado_conectividad` como campo filtrable en `GET /api/lecturas` (parámetro `estado_conectividad`) y como columna en `thermal_readings`/`devices`. El campo existe end-to-end en el backend. La ausencia detectada en Fase 2 (RF-18 no renderizado) es, con esta evidencia, **confirmada como una omisión exclusivamente del frontend** — el dato está disponible y filtrable, simplemente no se usa en ninguna pantalla.
