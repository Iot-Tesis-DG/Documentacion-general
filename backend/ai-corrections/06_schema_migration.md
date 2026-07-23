# 06 — Migración 0004 verificada contra PostgreSQL 18 real

## Entorno

PostgreSQL 18.4 (Postgres.app), local, `localhost:5432`. Base exclusiva y aislada `thermotrace_p1_correction_test`, creada y eliminada dentro de esta sesión. Sin conexión a Railway ni datos/credenciales de producción.

## Verificaciones ejecutadas

1. **Upgrade completo 0001→0004** contra base recién creada: las 4 migraciones aplican sin error.
2. **Downgrade -1 (0004→0003)**: revierte limpiamente — elimina constraints/FKs/columnas nuevas en el orden inverso correcto.
3. **Upgrade nuevamente (0003→0004)**: reaplica sin error tras el downgrade.
4. **Base recreada desde cero → head**: confirma que la migración no depende de estado intermedio dejado por una corrida anterior.

## Esquema resultante (`\d thermal_alerts`, `\d thermal_readings`)

Confirmado por inspección directa: `thermal_alerts` tiene `episodio_abierto` (integer, nullable), `lectura_inicial_id`/`lectura_mas_reciente_id` (uuid, NOT NULL, FK a `thermal_readings.id`), `ultima_actualizacion` (timestamptz, NOT NULL), `cerrada_en` (timestamptz, nullable), y la restricción `uq_thermal_alerts_episodio_abierto_por_device_y_riesgo` sobre `(device_id, nivel_riesgo, episodio_abierto)`. `thermal_readings` tiene `estado_inferencia`/`motivo_no_inferencia` (nullable). `0001`, `0002`, `0003` no se modificaron.

## Compatibilidad con registros existentes

El `UPDATE ... WHERE lectura_inicial_id IS NULL` backfillea filas de `thermal_alerts` previas a esta migración con `lectura_inicial_id = lectura_mas_reciente_id = reading_id`, `ultima_actualizacion = cerrada_en = created_at` — tratándolas como episodios ya cerrados (`episodio_abierto` queda NULL), evitando que colisionen con la nueva restricción única sobre filas que en el modelo de datos anterior no tenían noción de "episodio". Solo después del backfill se agregan las restricciones NOT NULL/FK/UNIQUE.

## ORM alineado

`ThermalAlertModel`/`ThermalReadingModel` en `models.py` declaran exactamente las mismas columnas que la migración (mismos tipos, mismas nulabilidades) — sin divergencia detectada.

## Estado: migración validada de extremo a extremo contra PostgreSQL real, base de prueba eliminada al finalizar.

## Revalidación de cierre

En esta sesión `pg_isready`/`psql` no están disponibles, por lo que no se
repite la prueba contra PostgreSQL ni se usa producción. Sí se confirmó con el
entorno Python 3.12 que `alembic heads` devuelve
`0004_ia_correcciones_p1 (head)` y que la suite SQLite completa pasa. La
validación PostgreSQL anterior queda como evidencia histórica y debe repetirse
en CI/local con PostgreSQL antes de despliegue.

## Validación PostgreSQL 18 ejecutada — cierre

Postgres.app 18.4 local, `127.0.0.1:5432`, usuario local `diegosoto`. Base
exclusiva y descartable `tesis_code_alembic_validation_20260722`:

1. `upgrade 0001 -> 0003` correcto.
2. `upgrade 0003 -> 0004` correcto; Alembic confirmó head.
3. Confirmadas 5 columnas de episodio, 2 FKs y constraint único.
4. `downgrade 0004 -> 0003` correcto; quedaron 0 columnas P1.
5. `upgrade 0003 -> head` correcto; quedaron 5 columnas P1 y 2 columnas de inferencia.
6. Base eliminada; consulta final en `pg_database` devolvió 0.

Resultado PostgreSQL: **VALIDADO**. Sin Railway, producción ni datos reales.
