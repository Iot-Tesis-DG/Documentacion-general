# 05 — AIV-02 resuelto: control de episodio de alerta

## Diseño

`GenerarAlertaUseCase` pasó de "una alerta por lectura crítica" a "una alerta por EPISODIO" (device_id + tipo de riesgo):

- **Lectura NORMAL** con episodio abierto → cierra el episodio (evento de recuperación), sin crear fila.
- **Lectura crítica del MISMO tipo** que el episodio abierto → actualiza `lectura_mas_reciente_id`/`ultima_actualizacion` en la MISMA fila, sin insertar.
- **Lectura crítica de OTRO tipo** (escalamiento/desescalamiento, p. ej. `riesgo_preventivo` → `excursion_critica`) → cierra el episodio anterior y abre uno nuevo del tipo entrante.
- **Sin episodio abierto, pero hubo uno cerrado del mismo tipo hace menos de `COOLDOWN_MINUTOS=15`** → se reabre esa misma fila (evita flapping de apertura/cierre por una lectura normal aislada — histéresis) en vez de crear una nueva.
- **Sin episodio abierto ni cooldown aplicable** → crea una alerta nueva.

## Garantía a nivel de base de datos, no solo en memoria

Migración `0004_ia_correcciones_p1.py`: `UniqueConstraint("device_id", "nivel_riesgo", "episodio_abierto")` sobre `thermal_alerts`. `episodio_abierto` es `Integer NULL`, con `1` para episodios abiertos y `NULL` para cerrados — la semántica SQL estándar de NULL≠NULL en restricciones únicas (válida en PostgreSQL y SQLite) permite múltiples filas cerradas del mismo device+tipo sin violar la unicidad, mientras garantiza que **nunca puede existir más de una fila abierta simultánea** para el mismo device+tipo, incluso bajo escritura concurrente — la restricción la aplica el propio motor de base de datos, no solo la lógica de aplicación.

## Campos de episodio

`lectura_inicial_id` (la lectura que originó el episodio), `lectura_mas_reciente_id` (la última que lo actualizó), `ultima_actualizacion`, `cerrada_en` — todos en `ThermalAlertModel`/`AlertaTermica`/`AlertaResponse`.

## Verificación — 4 pruebas nuevas, todas verdes

| Prueba | Qué confirma |
|---|---|
| `test_lecturas_criticas_consecutivas_no_generan_alertas_duplicadas` | 5 lecturas críticas consecutivas → 1 sola fila de alerta, `episodio_abierto=1` |
| `test_recuperacion_a_normal_cierra_el_episodio` (unitaria, repositorio en memoria) | Recuperación a NORMAL cierra el episodio sin crear fila nueva |
| `test_escalamiento_preventivo_a_critico_cierra_y_abre_nuevo_episodio` | Escalamiento cierra la fila anterior (`riesgo_preventivo`) y abre una nueva (`excursion_critica`) — 2 filas, la primera cerrada |
| `test_dos_lecturas_concurrentes_del_mismo_episodio_no_duplican_alerta` | 2 lecturas críticas concurrentes (`asyncio.gather`) del mismo dispositivo → 1 sola fila abierta (garantía de BD, no solo de aplicación) |

**Nota metodológica**: la prueba de recuperación se implementó como prueba UNITARIA de `GenerarAlertaUseCase` con un repositorio en memoria (no a través del pipeline completo con el Random Forest real). Esto se decidió tras observar, en una primera versión de la prueba con el pipeline completo, que el modelo real puede legítimamente seguir clasificando `riesgo_preventivo` en la primera lectura "normal" inmediatamente posterior a una excursión, si el historial reciente (features `frecuencia_desviaciones`/`duracion_fuera_rango`) todavía refleja la inestabilidad — un comportamiento correcto del modelo, no un defecto, pero que hacía la prueba de la regla de negocio (cierre de episodio) no determinista si dependía de que el modelo devolviera exactamente `NORMAL`. Aislar la prueba de la variabilidad legítima del clasificador es la forma correcta de probar la regla de episodio en sí.

## Estado AIV-02: **RESUELTO**
