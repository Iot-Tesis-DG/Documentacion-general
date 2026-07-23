# FASE 2 — FRONTEND: Informe de cierre

## 1. Archivos encontrados
67 archivos totales detectados por `find` (incluyendo `.vercel/`, lockfile, configs). 55 son código/configuración propios; el resto son `node_modules` (excluido), `dist` (excluido), `.DS_Store` (excluidos).

## 2. Archivos revisados
**Todos los 55 archivos propios fueron leídos** (39 archivos `.ts`/`.tsx` de código fuente al 100%, más 9 configs, más `.env.example`/`.env.demo`, `vercel.json`, `.gitignore`, favicon). 3719 líneas de código TypeScript/TSX en total, todas revisadas.

## 3. Archivos excluidos
`node_modules/` (dependencias de terceros), `dist/` (build generado), `.DS_Store` ×2, `.vercel/` (solo metadata de despliegue, sí se leyó `project.json` y `README.txt`).

## 4. Archivos no accesibles
Ninguno. `package-lock.json` se confirmó existente (npm) pero no se leyó línea por línea (no aporta valor de auditoría más allá de confirmar el gestor de paquetes usado).

## 5. Arquitectura real
DDD limpio y consistentemente aplicado: `domain/` (entidades + value objects sin dependencias externas) → `application/` (hooks + store Zustand) → `infrastructure/` (API, auth, SSE, charts, demo, i18n) → `presentation/` (páginas, layout, componentes). Dirección de dependencias correcta. Ver `frontend_architecture.md`.

## 6. Rutas y páginas
10 rutas (login + 9 protegidas), todas mapeadas contra RF/backlog en `frontend_routes.md`. Todas las pantallas esperadas por TI/backlog existen, excepto una pantalla de "gestión de dispositivos" (coincide con la historia huérfana HU-43 de Fase 1) y una pantalla genérica de "configuración" (no exigida explícitamente por TI).

## 7. Funcionalidades completas
Login/JWT/RBAC, Dashboard con SSE real, Historial con filtros, Alertas con acciones correctivas, Trazabilidad con verificación de integridad, Auditoría, Usuarios (alta + listado).

## 8. Funcionalidades parciales
- Reportes BPA: datos reales del backend, pero exportación solo CSV/JSON (falta PDF, HU-38).
- Filtro de historial por conectividad: campo mostrado pero no filtrable.
- RF-18 (estado de conectividad): campo existe en el modelo, no se renderiza en ninguna pantalla.

## 9. Funcionalidades simuladas
- **Checklist BPA: 100% simulado/local** (`localStorage`, cero llamadas API) — contradice HU-37.
- Todo el "Modo Demo" (build Vercel público): simulación completa e intencional, correctamente aislada y documentada, no engañosa para quien lea el código, pero **la URL pública en producción nunca toca el backend real**.

## 10. Datos hardcodeados
Ninguno en el camino de producción real (el clasificador de riesgo del modo demo replica reglas simples solo para la simulación, explícitamente documentado como tal, y aislado del código real vía el flag `MODO_DEMO`). Las 4 cuentas demo y sus roles están hardcodeadas en `datosDemo.ts`, activas solo bajo `MODO_DEMO=true`.

## 11. Endpoints consumidos
13 endpoints reales identificados y documentados en `frontend_api_map.md` (login, sse-ticket, lecturas, alertas ×3, trazabilidad ×2, reportes/bpa, usuarios ×2, auditoría). Checklist BPA y exportación PDF: **sin endpoint correspondiente consumido**.

## 12. Autenticación y RBAC
JWT en memoria (nunca localStorage en producción real) — verificado en código, no solo declarado. RBAC de 3 roles (administrador/farmacéutico/técnico) con bloqueo real de rutas (no solo visual). Ver `frontend_auth_rbac.md`.

## 13. Mecanismo real de tiempo real
SSE genuino vía `EventSource` nativo, con esquema de ticket de autenticación efímero. Coincide con lo declarado en TI. Ver `frontend_realtime_analysis.md`.

## 14. Estado de IA en frontend
El frontend **nunca calcula clasificación de riesgo por sí mismo** en el camino de producción — siempre consume `nivel_riesgo` ya calculado del backend. No se repite el overclaim "Exactitud >96%" detectado en la presentación PPTX (Fase 1). Ausencia total de UI de métricas del modelo (F1, precisión, recall) — refuerza el gap ya detectado en el backlog (RNF-04 sin historia de medición).

## 15. Estado de trazabilidad
UI de solo lectura (correcto — respeta inmutabilidad), muestra `previous_hash`/`hash_actual` truncados, botón de verificación de integridad conectado a un endpoint real. Terminología correcta (nunca usa "blockchain"). No se puede confirmar aún si el hash real es criptográficamente válido (responsabilidad de backend, Fase 3).

## 16. Errores de compilación, lint o pruebas
- Typecheck: **0 errores**.
- Build (real y demo): **0 errores**, ambos compilan.
- Lint: **no configurado** (sin ESLint).
- Pruebas: **cero pruebas existentes**, ningún framework de testing instalado.

## 17. Diferencias frente a TI, backlog y presentación
- Checklist BPA local vs backend exigido en HU-37 (contradicción alta).
- Exportación sin PDF vs HU-38/TI (contradicción alta).
- RF-18 sin representación visual.
- shadcn/ui usado como patrón manual, no como librería instalada vía CLI.
- El overclaim de la presentación (96% exactitud) **no** se propaga al código — el frontend es más honesto que la presentación en este punto.

## 18. Hallazgos priorizados
11 hallazgos documentados en `frontend_findings.md`: 3 ALTO (F-01 checklist local, F-02 sin PDF, F-03 sin métricas IA), 4 MEDIO (F-04 RF-18 ausente, F-05 shadcn parcial, F-06 sin pruebas, F-07 sin lint), 4 BAJO (F-08 demo login sin password check, F-09 sin ErrorBoundary, F-10 manejo de errores HTTP incompleto, F-11 bundle pesado sin code-splitting).

## 19. Preguntas exactas para verificar en Backend (Fase 3)

1. ¿Existe `POST /api/auth/login` con `OAuth2PasswordRequestForm`, devolviendo JWT real firmado (no `alg: none`)?
2. ¿Existe `POST /api/auth/sse-ticket` y el mecanismo de ticket efímero para SSE está realmente implementado del lado servidor?
3. ¿Existe el endpoint SSE real (`/api/sse/lecturas` o el que indique `VITE_SSE_URL`) emitiendo eventos `message` con JSON de `LecturaTermica`?
4. ¿El backend calcula `nivel_riesgo` con el modelo Random Forest real, o hay reglas if/else disfrazadas de IA?
5. ¿Existen migraciones reales para `thermal_readings`, `thermal_alertas` (o `thermal_alerts`), `traceability_records`, `audit_logs`, `usuarios`?
6. ¿El backend rechaza con 403 a un técnico que invoque directamente `/api/usuarios` o `/api/auditoria` sin pasar por el frontend (verificación de RBAC real, no solo confiado en el cliente)?
7. ¿Existe algún endpoint de checklist BPA en el backend, aunque el frontend no lo consuma? (Determinaría si la desconexión es un defecto de frontend, de backend, o de ambos).
8. ¿El hash SHA-256 encadenado (`previous_hash`/`hash_actual`) se calcula realmente con `hashlib` sobre los campos declarados, y el endpoint `/api/trazabilidad/verificar` realmente recalcula y compara la cadena completa (RF-15)?
9. ¿Existen tests de backend que midan F1-score real del modelo Random Forest (RNF-04), y en qué formato se almacenan esas métricas (si existen)?
10. ¿El backend expone algún endpoint de exportación de PDF que el frontend simplemente no esté consumiendo, o el PDF nunca se implementó en ninguna capa?
11. ¿Los 10 RNF (especialmente disponibilidad ≥95%, latencia ≤5s) tienen algún mecanismo de medición/logging en el backend, o son solo aspiracionales?

## 20. Veredicto del frontend

**Parcialmente implementado y técnicamente consistente.**

Corrección de veredicto: el borrador anterior de este informe usaba "Funcional con inconsistencias menores", que subestima el peso real de los hallazgos. Con 3 hallazgos ALTO (checklist BPA sin persistencia backend, ausencia total de exportación PDF exigida por TI/HU-38, RNF-04 sin ninguna superficie de presentación), más RF-18 no renderizado, cero pruebas automatizadas y ausencia de ESLint, el frontend no puede calificarse como "menor". El veredicto correcto reconoce dos planos distintos:

- **Consistencia técnica**: alta. El código compila limpio (typecheck + build exit 0 en ambos modos), JWT vive genuinamente en memoria (no cosmético), las rutas protegidas bloquean render real (no solo ocultan botones), RBAC de interfaz está bien implementado, y SSE usa `EventSource` real con un esquema de ticket de autenticación bien diseñado. No hay fachada ni simulación oculta en el código de producción (el modo demo está honestamente aislado y declarado).
- **Completitud funcional**: parcial. Tres funcionalidades exigidas explícitamente por TI/backlog (persistencia backend del checklist BPA, exportación PDF, superficie de métricas de IA) están ausentes o mal resueltas, y un requisito funcional (RF-18) existe en el modelo de datos pero no se usa en ninguna pantalla.

En consecuencia, la funcionalidad integral del frontend **depende de que el backend implemente correctamente los contratos, la autorización server-side y las fuentes de datos que el frontend da por sentadas** (JWT válido, RBAC replicado en el servidor, SSE real, cálculo real de `nivel_riesgo`) — nada de esto puede confirmarse sin Fase 3. El frontend, por sí solo, es una base sólida pero incompleta frente al alcance declarado en TI y en el Product Backlog.
