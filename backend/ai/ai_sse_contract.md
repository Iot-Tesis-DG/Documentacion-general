# Contrato SSE — campos reales vs esperados

## `LecturaResponse` real (`schemas.py`)

Confirmado por grep: contiene `nivel_riesgo: NivelRiesgo | None`, además de los campos de la lectura (device_id, timestamp, temperaturas, humedad, apertura, conectividad). **No incluye `confianza`/`confidence` ni `model_version`.**

## Comparación contra el contrato sugerido por el prompt (sección 19)

| Campo sugerido | ¿Existe en el contrato real? | Nombre real equivalente |
|---|---|---|
| `reading_id` | Sí | `id` (dentro de `LecturaResponse`) |
| `device_id` | Sí | `device_id` |
| `temperature` | Sí (dos variantes) | `temperatura_interna`, `temperatura_ambiental` |
| `humidity` | Sí | `humedad_ambiental` |
| `risk_level` | Sí | `nivel_riesgo` (puede ser `null`) |
| `confidence` | **No existe** | — |
| `model_version` | **No existe** | — |
| `timestamp` | Sí | `timestamp` (ISO 8601, confirmado por el tipo `datetime` de Pydantic) |

## Manejo de `risk_level=null` (sensor sin dato) — verificado end-to-end

1. Backend: `nivel_riesgo: NivelRiesgo | None` en el schema, correctamente tipado como opcional.
2. Frontend (`LecturaTermica.ts`, auditado en Fase 2): `nivel_riesgo: NivelRiesgo | null` — **ya compatible**, el frontend siempre trató este campo como nullable incluso antes de la corrección P0 del backend (`RiskBadge` ya maneja `nivel: NivelRiesgo | null` mostrando `—` cuando es `null`).
3. **Conclusión: no hay incompatibilidad de contrato que requiera cambios en el frontend** para soportar la corrección del hallazgo B-05 (sensor `None`) — el tipo ya era opcional del lado cliente, coincidencia afortunada de diseño previo, no una casualidad forzada por esta sesión.

## Recomendación (Etapa B, no ejecutada aún)

Agregar `confianza` y `model_version` a `LecturaResponse` sería un cambio de contrato aditivo (nuevos campos opcionales), no rompería al frontend actual (que ya ignora campos no reconocidos al hacer `JSON.parse` + acceso por nombre de propiedad). Se documenta aquí como cambio propuesto, **no se implementa en Etapa A**.

## Eventos de alerta vs eventos de lectura

Confirmado: el `SSEBroadcaster` solo difunde eventos de tipo lectura (`lectura_to_response(...)`) — no hay un evento SSE diferenciado para "nueva alerta generada". El frontend descubre alertas nuevas re-consultando `GET /api/alertas` (polling manual al navegar a esa pantalla), no vía push en tiempo real. Esto ya estaba fuera del alcance específico de esta auditoría de IA (es un patrón de la capa SSE general, no de la integración del modelo), se menciona por completitud.
