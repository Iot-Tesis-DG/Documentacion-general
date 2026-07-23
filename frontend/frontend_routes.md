# Mapa de rutas y páginas

Fuente: `App.tsx` (definición de rutas) + `AppLayout.tsx` (definición de menú/roles visibles) — ambos coinciden exactamente (misma matriz de roles).

| Ruta | Página/componente | Acceso | Roles permitidos | Funcionalidad (RF relacionado) | Fuente de datos | Estado real |
|---|---|---|---|---|---|---|
| `/login` | `LoginPage` | Público | — | RF-17 (login JWT) | `POST /api/auth/login` | **Funcional y conectado** (+ atajos demo solo si `MODO_DEMO`) |
| `/dashboard` | `DashboardPage` | Protegido | farmaceutico, tecnico (+admin implícito) | RF-11 (tiempo real), RF-18 (conectividad) | `GET /api/lecturas` + SSE | **Funcional y conectado** |
| `/historial` | `HistorialPage` | Protegido | farmaceutico, tecnico | RF-12 (filtros) | `GET /api/lecturas` con params | **Funcional y conectado** |
| `/alertas` | `AlertasPage` | Protegido | farmaceutico, tecnico | RF-09, RF-10 | `GET/PATCH /api/alertas`, `POST .../acciones-correctivas` | **Funcional y conectado** |
| `/trazabilidad` | `TrazabilidadPage` | Protegido | farmaceutico, tecnico | RF-14, RF-15 | `GET /api/trazabilidad`, `GET /api/trazabilidad/verificar` | **Funcional y conectado** |
| `/checklist-bpa` | `ChecklistBPAPage` | Protegido | **solo farmaceutico** (+admin) | RF-13 (parcial, BPA) | **`localStorage` únicamente — NINGUNA llamada API** | **Contradictorio** — HU-37 exige envío a backend con JWT; implementación real es 100% local |
| `/reportes` | `ReportesPage` | Protegido | **solo farmaceutico** (+admin) | RF-13 (reportes exportables) | `GET /api/reportes/bpa` | **Parcialmente conectado** — datos reales del backend, pero exportación solo CSV/JSON, **sin PDF** (HU-38 exige PDF) |
| `/auditoria` | `AuditoriaPage` | Protegido | **solo administrador** (`roles: []`) | RF-16 | `GET /api/auditoria` | **Funcional y conectado** |
| `/usuarios` | `UsuariosPage` | Protegido | **solo administrador** (`roles: []`) | RF-17 (gestión RBAC) | `GET/POST /api/usuarios` | **Funcional y conectado** |
| `*` (catch-all) | Redirect a `/dashboard` | — | — | — | — | Funcional |

## Verificación contra pantallas esperadas (RF/backlog)

| Pantalla esperada | ¿Existe? | Observación |
|---|---|---|
| Inicio de sesión | Sí | `/login` |
| Dashboard | Sí | `/dashboard` |
| Monitoreo térmico | Sí | Integrado en `/dashboard` (no ruta separada) |
| Historial | Sí | `/historial` |
| Alertas | Sí | `/alertas` |
| Dispositivos (gestión, HU-43) | **No existe** | Ninguna ruta de "gestión de dispositivos" (alta/baja/reemplazo ESP32) — consistente con el hallazgo de Fase 1 de que HU-43 quedó huérfana/sin planificar |
| Establecimientos/farmacias | No existe ruta dedicada — el `device_id` (`FARM-01-CDL`) identifica implícitamente el establecimiento, sin CRUD de farmacias | Fuera de alcance de TI (TI no define un RF de gestión de farmacias — un solo establecimiento es el escenario de validación) — **no corresponde al frontend** |
| Clasificación de riesgo térmico | Sí | Visible en Dashboard/Historial/Alertas vía `RiskBadge`, consumido de `nivel_riesgo` del backend |
| Reportes | Sí | `/reportes` |
| Acciones correctivas | Sí | Integrado en `/alertas` (diálogo modal) |
| Usuarios | Sí | `/usuarios` |
| Roles | Sí (RBAC vía rutas + menú) | — |
| Auditoría | Sí | `/auditoria` |
| Trazabilidad | Sí | `/trazabilidad` |
| Verificación de integridad | Sí | Botón "Verificar" en `/trazabilidad` → `GET /api/trazabilidad/verificar` |
| Configuración (general) | **No existe** | No hay ruta `/configuracion` genérica; TI no exige explícitamente una pantalla de configuración general aparte del checklist BPA — no es una omisión crítica |

## Nota sobre RBAC de rutas

`roles={[]}` en `App.tsx` para `/auditoria` y `/usuarios` no es una lista vacía "sin restricción" — por la lógica de `tienePermiso()` (`rol === 'administrador' || permitidos.includes(rol)`), un array vacío significa que **solo el administrador** pasa. Verificado correcto, no es un bug.
