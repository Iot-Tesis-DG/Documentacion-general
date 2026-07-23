# 10 — Persistencia y migraciones (verificado contra PostgreSQL real, esta sesión)

## Migración 0003 — ejecutada fresca contra PostgreSQL 18 real en esta verificación

```
$ dropdb/createdb thermotrace_final_verify (aislada, eliminada al finalizar)
$ alembic upgrade head
Running upgrade  -> 0001_initial_schema
Running upgrade 0001_initial_schema -> 0002_thermal_readings_dedup
Running upgrade 0002_thermal_readings_dedup -> 0003_lecturas_modelo_ia
```
**Las 3 migraciones aplican limpiamente, sin error, en un motor PostgreSQL real** (no solo SQLite vía `create_all`).

`\d thermal_readings` confirma las columnas reales en el esquema:
```
modelo_version        | character varying(50)
confianza_ia           | double precision
origen_clasificacion   | character varying(30)
uq_thermal_readings_device_timestamp | UNIQUE CONSTRAINT, btree (device_id, timestamp)  [P0, sigue presente]
```

## No modifica migraciones históricas — confirmado

`0001_initial_schema.py` y `0002_thermal_readings_dedup.py` no se tocaron (verificado por `git diff` — no aparecen como modificados en el estado de esta sesión, solo `0003` es nuevo).

## `upgrade`/`downgrade` — ambos presentes

```python
def upgrade() -> None:
    op.add_column("thermal_readings", sa.Column("modelo_version", sa.String(50), nullable=True))
    op.add_column("thermal_readings", sa.Column("confianza_ia", sa.Float(), nullable=True))
    op.add_column("thermal_readings", sa.Column("origen_clasificacion", sa.String(30), nullable=True))

def downgrade() -> None:
    op.drop_column("thermal_readings", "origen_clasificacion")
    op.drop_column("thermal_readings", "confianza_ia")
    op.drop_column("thermal_readings", "modelo_version")
```
Reversible, orden correcto (downgrade en orden inverso al upgrade).

## ORM y migración — coinciden

`models.py::ThermalReadingModel` declara los mismos 3 campos con los mismos tipos (`String(50)`, `Float`, `String(30)`) — sin divergencia detectada.

## Campos opcionales toleran sensor sin datos

Los 3 campos son `nullable=True` tanto en el ORM como en la migración — una lectura con `nivel_riesgo=None` (sensor caído) persiste con `modelo_version=None`, `confianza_ia=0.0` (no `None` — ver nota), `origen_clasificacion="sin_dato_sensor"`. Verificado por prueba real: `test_lectura_sin_temperatura_interna_no_tiene_confianza_ni_alerta` (pasa).

**Nota de precisión**: `confianza_ia` cuando no hay dato de sensor se persiste como `0.0`, no como `None` — esto es una decisión de diseño consistente con `ResultadoInferencia.confianza: float` (no `Optional[float]`), pero significa que "confianza 0.0" es ambiguo entre "el modelo está seguro de que NO es la clase" (matemáticamente posible) y "no se ejecutó ninguna clasificación". El campo `origen_clasificacion="sin_dato_sensor"` es el que realmente desambigua este caso — un consumidor de la API que solo mire `confianza_ia` sin revisar `origen_clasificacion` podría malinterpretarlo.

## Compatibilidad con lecturas previas a la migración

Al ser columnas `nullable=True` sin `server_default`, las filas insertadas antes de la migración 0003 (si existieran en una base de datos con historial) quedarían con estos 3 campos en `NULL` tras el upgrade — comportamiento correcto y esperado, sin pérdida de datos ni error de migración.
