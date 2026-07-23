# Base de datos y migraciones

Única migración (`0001_initial_schema.py`, `down_revision=None`) — sin historial de evolución de esquema (proyecto nuevo, coherente). ORM (`src/infrastructure/database/models.py`) y migración Alembic **verificados idénticos campo por campo** (mismas tablas, mismos tipos, mismas FKs, mismos índices) — no se detectó divergencia ORM↔Alembic.

## Tablas

| Tabla | Campos clave | Relaciones | Índices | Migración/ORM | Inconsistencia |
|---|---|---|---|---|---|
| `devices` | `id` (String PK, no UUID — es el `device_id` de negocio tipo `FARM-01-CDL`), `estado_conectividad`, `created_at` | 1→N con `thermal_readings` | PK implícito | Coincide | Ninguna |
| `roles` | `id` (UUID PK), `nombre` (unique) | 1→N con `users` | unique en `nombre` | Coincide | Ninguna |
| `users` | `id` (UUID PK), `email` (unique+index), `password_hash`, `rol_id` (FK) | N→1 con `roles` | `ix_users_email` | Coincide | Ninguna |
| `thermal_readings` | `id` (UUID PK), `device_id` (FK), `timestamp`, `temperatura_*`, `humedad_ambiental`, `apertura_refrigerador`, `nivel_riesgo`, `estado_conectividad`, `payload` (JSONB) | N→1 `devices`, 1→N `thermal_alerts` | `ix_thermal_readings_device_id`, `ix_thermal_readings_timestamp` | Coincide | **Sin índice compuesto (device_id, timestamp)** — ver hallazgo |
| `thermal_alerts` | `id` (UUID PK), `reading_id` (FK), `device_id` (FK), `nivel_riesgo`, `revisada`, `revisada_por` (FK a users) | N→1 `thermal_readings`, 1→N `corrective_actions` | index en `device_id` | Coincide | Ninguna |
| `corrective_actions` | `id` (UUID PK), `alert_id` (FK), `usuario_id` (FK), `descripcion` | N→1 `thermal_alerts` | — | Coincide | Sin índice en `alert_id` (FK sin índice explícito) |
| `traceability_records` | `id` (UUID PK), `tipo_evento`, `device_id` (nullable), `usuario_id` (FK nullable), `payload` (JSONB), `timestamp`, `previous_hash`, `hash_actual` (**unique**), `created_at` | FK opcional a `users` | unique en `hash_actual`; **sin índice en `created_at`** (usado para ordenar la cadena) | Coincide | Ver hallazgo de concurrencia — `hash_actual` único previene colisión exacta pero no previene bifurcación de la cadena (dos registros con el mismo `previous_hash`) |
| `audit_logs` | `id` (UUID PK), `usuario_id` (FK nullable), `accion`, `recurso`, `detalle` (JSONB nullable), `ip_origen`, `created_at` | FK opcional a `users` | — | Coincide | Sin índice en `created_at` para paginación eficiente a escala |
| `report_exports` | `id` (UUID PK), `usuario_id` (FK), `tipo_reporte`, `fecha_desde`, `fecha_hasta`, `archivo_url` (nullable, **nunca poblado**) | FK a `users` | — | Coincide | `archivo_url` es un campo muerto en la práctica (siempre `None` — confirma ausencia de generación de PDF/archivo real) |

## Verificaciones solicitadas

- **Orden de migraciones**: solo hay una, trivialmente ordenada.
- **Reversibilidad**: la migración define `downgrade()` (no reproducido aquí por brevedad, pero confirmado presente en el archivo) — sí es reversible.
- **`create_all` vs Alembic**: el proyecto usa Alembic real, no `Base.metadata.create_all()` en producción (aunque `tests/conftest.py` sí podría usar `create_all` para tests — patrón común y aceptable en testing).
- **Timestamps sin zona horaria**: todos los campos de fecha usan `DateTime(timezone=True)` — correctos. El comentario en el código (`_created_at_column`) documenta explícitamente por qué se usa un default Python (`_utcnow`) además del `server_default=func.now()`: SQLite (usado en tests) tiene resolución de 1 segundo en `CURRENT_TIMESTAMP`, lo que rompería el orden estable que necesita la verificación de la cadena hash. **Detalle técnico correcto y bien razonado.**
- **UUID**: uso consistente vía `Uuid(as_uuid=True)` de SQLAlchemy 2.0 (nativo en PostgreSQL, portable en otros dialectos) — sin mezclas de string-uuid vs uuid-nativo.
- **Tipos numéricos de temperatura**: `Float` en todos los campos térmicos — coherente con la precisión de sensores (±0.2-0.5°C), no se sobre-especifica con `Numeric`/`Decimal` innecesariamente.
- **Duplicidad de lecturas**: **no hay ninguna restricción única sobre `(device_id, timestamp)`** en `thermal_readings` — si el firmware reenvía una lectura ya persistida (reintento QoS1 tras PUBACK perdido, escenario 2 de HU-11), se insertaría un duplicado exacto sin que la BD lo impida. Ver hallazgo.
- **Eliminaciones en cascada**: no se detectaron `ondelete="CASCADE"` en ninguna FK — por defecto SQLAlchemy/PostgreSQL usa `RESTRICT` (no se puede borrar un `device` con lecturas asociadas sin borrar antes las lecturas) — es una postura conservadora y seguro por defecto, coherente con el requisito de inmutabilidad de trazabilidad (nada debería poder eliminarse en cascada silenciosamente).
- **Soft delete**: no implementado en ninguna tabla — todas las eliminaciones serían físicas si se ejecutaran (aunque no se expone ningún endpoint DELETE en toda la API, ver `backend_endpoint_map.md` — de hecho, **no existe ningún endpoint DELETE en el sistema**, lo cual es coherente con el principio de inmutabilidad de trazabilidad/auditoría que TI declara).

## SQLite local (`dev.db`) — verificación sin tocar producción

Se confirmó que el proyecto soporta explícitamente SQLite para desarrollo/pruebas (mencionado en comentarios de `models.py`, `aiosqlite` en `requirements-dev.txt`, `pytest-asyncio`). El `JSONVariant` (`JSON().with_variant(JSONB(), "postgresql")`) y el `Uuid` genérico están diseñados deliberadamente para ser portables entre PostgreSQL (producción) y SQLite (tests) — **no es una compatibilidad forzada de forma incorrecta; es un patrón explícito y documentado**. No se modificó `dev.db` en esta auditoría (solo se leyó metadata de archivo, no se ejecutaron queries contra él para evitar cualquier alteración accidental de estado).

## Hallazgos de esquema

1. **[MEDIO]** Sin restricción única `(device_id, timestamp)` en `thermal_readings` → posibilidad de lecturas duplicadas por reenvío MQTT.
2. **[MEDIO]** Sin índice compuesto `(device_id, timestamp)` — las consultas de historial filtradas por ambos campos (patrón más común, ver `useHistorial`) dependen de dos índices simples en vez de uno compuesto, con posible impacto de rendimiento a escala (bajo impacto real dado el volumen de un prototipo de una sola farmacia).
3. **[MEDIO]** `traceability_records.previous_hash` no tiene ninguna restricción (ni única, ni FK a `hash_actual` de otro registro) que impida a dos transacciones concurrentes insertar registros con el mismo `previous_hash` — ver `backend_hash_chain_analysis.md` para el análisis de concurrencia completo.
4. **[BAJO]** `report_exports.archivo_url` es un campo sin uso real (siempre `NULL`) — confirma la ausencia de generación de archivo (PDF) documentada en `backend_reports_bpa.md`.
