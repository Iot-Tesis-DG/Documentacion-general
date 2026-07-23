# Auditoría (`audit_logs`)

## Cobertura de eventos auditados (confirmado por código, no inferido)

`LOGIN_EXITOSO`, `LOGIN_FALLIDO`, `LOGOUT`, `CREAR_USUARIO`, `REVISAR_ALERTA`, `REGISTRAR_ACCION_CORRECTIVA`, `EXPORTAR_REPORTE_BPA`, `DISPOSITIVO_RECHAZADO` — 8 tipos de acción confirmados en el código de routers/use cases leído.

Campos por registro (`AuditLogModel`): `usuario_id` (nullable — permite auditar acciones sin usuario, ej. rechazo de dispositivo MQTT), `accion`, `recurso`, `detalle` (JSONB, contexto libre), `ip_origen`, `created_at`. **Coincide exactamente** con lo que el frontend (`AuditoriaPage`) espera mostrar.

## Inmutabilidad

Igual que trazabilidad: **no existe ningún endpoint de modificación/eliminación sobre `audit_logs`** (`auditoria_router.py` solo expone `GET`). La inmutabilidad es de aplicación, no de motor de BD (mismo matiz que en `backend_hash_chain_analysis.md`). A diferencia de `traceability_records`, los `audit_logs` **no están encadenados por hash** — son una tabla de log simple, sin verificación de integridad criptográfica propia. Esto es coherente con TI (que solo exige hash encadenado para trazabilidad de eventos térmicos/operativos, RF-14), no un defecto — pero significa que, a diferencia de la trazabilidad, un administrador con acceso a la BD SÍ podría alterar un `audit_log` sin que ningún mecanismo del sistema lo detecte automáticamente (ni la app, ni una verificación de hash, porque no existe una).

## ¿El mismo usuario puede modificar sus propios logs?

No hay ningún endpoint que lo permita — ni siquiera el administrador tiene una ruta de edición expuesta.

## Datos sensibles en logs de auditoría

`detalle` incluye en algunos casos el email del usuario (`LOGIN_FALLIDO` guarda `{"email": ...}` incluso para intentos con credenciales inexistentes) — dato personal pero no una credencial (no se guarda la contraseña, correctamente). `ip_origen` se registra siempre que esté disponible — dato personal (IP), su retención sin política de expiración visible es una consideración de privacidad a mencionar (no hay TTL/purga automática de `audit_logs` en el código revisado).

## Integración con la cadena hash

Confirmado que las acciones correctivas y lecturas SÍ generan tanto un `audit_log` (bitácora simple) COMO un `traceability_record` (hash encadenado) — son dos mecanismos paralelos y complementarios, no duplicados: `audit_logs` es para trazabilidad operativa general (quién hizo qué), `traceability_records` es específicamente para la cadena criptográfica de eventos térmicos/operativos críticos exigida por RF-14. Ambos existen y ambos se usan.

## Retención

No se encontró ninguna política de purga/retención automática (ni en `config.py` ni en ningún use case) — los logs y la trazabilidad crecen indefinidamente. Aceptable para un prototipo, mencionable como deuda técnica de producción real.
