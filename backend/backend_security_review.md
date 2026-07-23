# Seguridad — alineamiento técnico (no certificación)

Marco de referencia: ISO/IEC 30141:2024, OWASP IoT Security Testing Guide v1.0.0, OWASP WSTG v4.2 (declarados por TI como alineamiento, no certificación).

| Control | Estado | Evidencia |
|---|---|---|
| Gestión de secretos | **Alineado** | `.env` fuera de git (`.gitignore` confirmado), `Settings` con validador que rechaza secretos por defecto en producción |
| Valores por defecto peligrosos | **Alineado** | JWT secret, MQTT password: ambos bloqueados en producción si quedan en su valor de ejemplo |
| CORS | **Alineado** | Lista explícita de orígenes, `"*"` prohibido en producción |
| Hosts permitidos | **Alineado** | `TrustedHostMiddleware` activo solo en producción, `allowed_hosts` obligatorio |
| HTTPS asumido | **Parcialmente alineado** | El backend no termina TLS él mismo (esperable, lo hace Railway/proxy delante) — HSTS se emite condicionalmente, correcto para ese modelo de despliegue |
| Security headers | **Alineado** | `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`, CSP restrictivo (`default-src 'none'`), `Cache-Control: no-store` en rutas de auth |
| Rate limiting | **Alineado** | Doble capa: login (por fallos) + global por IP (OWASP API4:2023 citado explícitamente en comentario) |
| Validación de entrada | **Alineado** | Pydantic v2 en todos los request bodies/params, rangos físicos explícitos |
| Mass assignment | **Alineado** | Schemas Pydantic explícitos de entrada (`UsuarioCreateRequest`, etc.), no se exponen modelos ORM directamente en las rutas |
| Inyección SQL | **Alineado** | SQLAlchemy ORM con queries parametrizadas (`select(...).where(...)`) en todos los repositorios revisados — sin SQL crudo concatenado en ningún punto leído |
| Path traversal | **No evidenciado** (no aplica — no hay endpoints de manejo de archivos/rutas de sistema) | — |
| SSRF | **No evidenciado** (no aplica — el backend no hace requests salientes a URLs proporcionadas por el usuario) | — |
| Deserialización insegura | **Alineado** | Pydantic (no `pickle`/`eval` sobre datos externos); `joblib.load()` se usa solo para el artefacto `.pkl` propio del repositorio, no para datos de terceros |
| Carga de archivos | **No aplica** (no hay endpoints de upload) | — |
| Logs sensibles | **Parcialmente alineado** | Email se registra en auditoría de login fallido (dato personal, no credencial) — ver `backend_audit_analysis.md` |
| Stack traces expuestos | **Alineado** | FastAPI no expone tracebacks en producción por defecto (sin handlers custom que los filtren, se confía en el comportamiento estándar) — no se detectó ningún `except Exception` que reenvíe `str(exc)` con detalle interno sensible al cliente en los routers revisados |
| Documentación OpenAPI pública | **Alineado** | `/docs`, `/redoc`, `/openapi.json` deshabilitados explícitamente cuando `environment == "production"` |
| Endpoints de administración | **Alineado** | Protegidos con `require_roles(Rol.ADMINISTRADOR)` real (verificado server-side) |
| Dependencias vulnerables | **No evidenciado** (no se ejecutó `pip-audit`/`safety` por falta de entorno Python 3.12 funcional en esta máquina — ver `backend_tests_execution.md`) | — |
| MQTT | **Alineado** | TLS obligatorio y validado en producción, autorización de dispositivo por aplicación (`DEVICE_REGISTRY_ESTRICTO`), anti-suplantación device_id-vs-topic |
| JWT | **Alineado** | Claims completos requeridos, revocación real, tickets SSE de un solo uso con audiencia separada |
| RBAC | **Alineado** | Verificado real server-side en cada endpoint sensible (ver `backend_rbac_matrix.md`) |
| SSE | **Alineado** | Autenticación por ticket efímero de un solo uso, sin exponer el JWT principal en la URL |
| Contraseñas | **Alineado** | bcrypt vía passlib, `password_min_length` configurable (10 por defecto en `Settings`, aunque **no se verificó que este mínimo se aplique realmente en la validación del schema `UsuarioCreateRequest`** — ver hallazgo) |
| Configuración de producción | **Alineado** | Múltiples validadores duros que impiden arrancar con configuración insegura |

## Hallazgo: `password_min_length` declarado pero no confirmado aplicado

`Settings.password_min_length: int = 10` existe como configuración, pero no se verificó (en el tiempo disponible de esta auditoría) que `UsuarioCreateRequest` o el caso de uso `CrearUsuarioUseCase` efectivamente rechacen contraseñas más cortas que ese valor — es una configuración declarada cuyo cumplimiento real no fue confirmado con evidencia de código directa (`schemas.py` no fue leído en el detalle de sus validadores Pydantic específicos por límite de tiempo). Marcado como **pendiente de verificación**, no como defecto confirmado.

## Concurrencia de trazabilidad — clasificado aquí también como hallazgo de seguridad

La condición de carrera documentada en `backend_hash_chain_analysis.md` tiene una dimensión de seguridad: un atacante que pudiera provocar escrituras concurrentes deliberadas (p. ej. flood de lecturas MQTT simultáneas) podría inducir una bifurcación de la cadena y, en el peor caso, una falsa alarma de "cadena rota" que erosione la confianza en el mecanismo de integridad — o, especulativamente, buscar una ventana para insertar un registro fraudulento en una rama que después quede "huérfana" y nunca se verifique como parte de la cadena principal. No se profundizó en explotabilidad activa (fuera de alcance de esta auditoría de solo lectura), pero el riesgo estructural es real.

## Alcance de esta revisión de seguridad

Esta es una revisión de **alineamiento técnico basada en lectura de código**, no un pentest activo ni un escaneo de dependencias (no se pudo ejecutar `pip-audit` por falta de entorno Python 3.12 operativo en esta máquina). No se declara cumplimiento normativo ni certificación de ningún estándar.
