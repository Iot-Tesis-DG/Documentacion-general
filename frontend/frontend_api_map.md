# Mapa de API — endpoints que el frontend intenta consumir

Fuente: inventario cruzado de todos los hooks (`application/hooks/*`) + `demoAdapter.ts` (que replica exactamente el contrato esperado del backend real, dado que "los hooks no cambian en absoluto" entre modo real y demo).

`API_BASE_URL` = `import.meta.env.VITE_API_BASE_URL` (vacío por defecto → mismo origen; en dev, proxy Vite `/api` → `http://localhost:8000`).

| Servicio/archivo | Método | Endpoint | Request | Response esperada | Auth | Componente consumidor | Estado |
|---|---|---|---|---|---|---|---|
| `authService.ts` | POST | `/api/auth/login` | form-urlencoded `username`, `password` | `{access_token, token_type}` | No (endpoint público) | `LoginPage` vía `authStore.login` | **Consumido realmente** |
| `sseClient.ts` | POST | `/api/auth/sse-ticket` | — | `{ticket: string}` | Sí (Bearer) | `DashboardPage` (indirecto vía `useMonitoreoTermico`) | **Consumido realmente** |
| `sseClient.ts` | GET (SSE) | `${VITE_SSE_URL}?ticket=...` | — | stream `event: message`, `data: LecturaTermica` JSON | Ticket en query param | `DashboardPage` | **Consumido realmente**, `EventSource` nativo |
| `useMonitoreoTermico.ts` | GET | `/api/lecturas?limite=60` | query params | `LecturaTermica[]` | Sí | `DashboardPage` | **Consumido realmente** |
| `useHistorial.ts` | GET | `/api/lecturas` | params: `device_id`, `nivel_riesgo`, `desde`, `hasta`, `limite` | `LecturaTermica[]` | Sí | `HistorialPage` | **Consumido realmente** |
| `useAlertas.ts` | GET | `/api/alertas` | params: `revisada`, `limite` | `AlertaTermica[]` | Sí | `AlertasPage` | **Consumido realmente** |
| `useAlertas.ts` | PATCH | `/api/alertas/{id}/revisar` | — | `AlertaTermica` | Sí | `AlertasPage` | **Consumido realmente** |
| `useAlertas.ts` | POST | `/api/alertas/{id}/acciones-correctivas` | `{descripcion}` | `AccionCorrectiva` | Sí | `AlertasPage` | **Consumido realmente** |
| `useTrazabilidad.ts` | GET | `/api/trazabilidad` | params: `tipo_evento`, `limite` | `RegistroTrazabilidad[]` | Sí | `TrazabilidadPage` | **Consumido realmente** |
| `useTrazabilidad.ts` | GET | `/api/trazabilidad/verificar` | — | `VerificacionIntegridad` | Sí | `TrazabilidadPage` | **Consumido realmente** |
| `useReportesBPA.ts` | GET | `/api/reportes/bpa` | params: `fecha_desde`, `fecha_hasta`, `device_id?` | `ReporteBPA` (lecturas+alertas+trazabilidad del período) | Sí | `ReportesPage` | **Consumido realmente** |
| `useUsuarios.ts` | GET | `/api/usuarios` | — | `Usuario[]` | Sí | `UsuariosPage` | **Consumido realmente** |
| `useUsuarios.ts` | POST | `/api/usuarios` | `{nombre, email, password, rol}` | `Usuario` (201) o 409 si duplicado | Sí | `UsuariosPage` | **Consumido realmente** |
| `useAuditoria.ts` | GET | `/api/auditoria?limite=200` | — | `RegistroAuditoria[]` | Sí | `AuditoriaPage` | **Consumido realmente** |
| — | — | Checklist BPA (cualquier endpoint) | — | — | — | `ChecklistBPAPage` | **INEXISTENTE — no se llama ningún endpoint**, todo vía `localStorage` |
| — | — | Export PDF | — | — | — | `ReportesPage` (`useReportesBPA`) | **INEXISTENTE** — solo CSV/JSON generados client-side desde los datos ya obtenidos de `/api/reportes/bpa` |

## Endpoints declarados en TI/backlog sin consumo en frontend

- **Verificación de dispositivos / gestión de altas-bajas de ESP32** (aludido en HU-43 huérfana): no hay ningún endpoint ni pantalla correspondiente.
- **Endpoint de checklist BPA persistente**: TI/HU-37 asumen un endpoint POST de checklist con firma JWT; no existe llamada en el frontend — a verificar si el backend lo expone igualmente sin consumidor, o si tampoco existe (pendiente Fase 3).

## Manejo de códigos HTTP

| Código | Manejo real |
|---|---|
| 401 | Interceptor global en `apiClient.ts`: limpia token, dispara `onSesionExpirada` → logout + redirect a `/login`. Real y correcto. |
| 403 | No hay interceptor global; no se vio manejo explícito en ningún hook (RouteGuards previene UI, pero si el backend retorna 403 a una llamada ya en curso, no hay captura específica — cae al `catch` genérico o revienta la promesa sin mensaje al usuario en varios hooks). |
| 404 | Sin interceptor global. En `useAlertas`/`demoAdapter` se ve manejo puntual (`fallar` en demo), pero en el cliente real no hay lógica específica para 404 fuera de captura genérica. |
| 409 | Manejado explícitamente en `useUsuarios.crear()` (`axios.isAxiosError(error) && error.response?.status === 409` → `'duplicado'`). Único caso de manejo granular de error de negocio. |
| 422 | Sin manejo explícito en ningún hook — errores de validación de Pydantic (esperados del backend) no tienen tratamiento dedicado (aparecerían como error genérico). |
| 429 | Manejado en `LoginPage.enviar()` (mensaje "demasiados intentos"). |
| 500 | Sin manejo específico; cae a mensajes genéricos ("errorServidor" en login, `error` boolean en reportes, ausencia de manejo en otros hooks). |

## Otros elementos técnicos

- **Timeout**: no configurado en la instancia Axios (`apiClient`) — sin límite de tiempo, una petición colgada nunca se cancela.
- **Reintentos**: ninguno a nivel HTTP (sí existe reintento en SSE, ver `frontend_realtime_analysis.md`).
- **Cancelación de solicitudes**: no se usa `AbortController` en ningún hook; el patrón de guardia `let activo = true` en `useEffect` previene actualizar estado tras desmontaje, pero no cancela la petición HTTP en curso.
- **Refresh token**: no existe ningún mecanismo de refresh — el JWT vive en memoria y expira; al recibir 401 se cierra sesión (no hay endpoint `/api/auth/refresh` consumido).
- **Transformación de respuestas / validación de contratos**: no hay validación de esquema en runtime (ej. Zod/Yup) — se confía ciegamente en el tipo TypeScript declarado (`apiClient.get<LecturaTermica[]>(...)`), sin verificación real de que el backend cumpla el contrato. Si el backend cambia un campo, TypeScript no lo detectará en runtime.
