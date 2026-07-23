# Ejecución de pruebas y validaciones locales — ACTUALIZADO (ejecución real)

Esta versión reemplaza el informe anterior, que documentaba un bloqueo de entorno (Python 3.12 no disponible). Ese bloqueo fue resuelto por instrucción explícita del usuario. Ver procedimiento y resultados reales abajo.

## Instalación de Python 3.12 (aislada y reversible)

Condiciones exigidas y cumplidas:
- No se usó/instaló Homebrew (no disponible en la máquina; se evitó instalarlo para no tocar el sistema).
- Se usó **pyenv**, instalado de forma aislada vía `git clone` en `~/.pyenv` (no requiere sudo, no modifica `/usr/bin/python3` del sistema).
- Versión instalada: **Python 3.12.13** (la más reciente estable de la serie 3.12, y la misma versión exacta declarada en `training_metrics.json::metadata.python_version`).
- Entorno virtual nuevo y exclusivo: **`backend/.venv312`** — el `.venv` original (con el symlink roto a un Python 3.12 inexistente en esta máquina) **no se tocó ni se eliminó**.
- Dependencias instaladas: únicamente `pip install -r requirements-dev.txt` (que a su vez incluye `requirements.txt` vía `-r`). No se modificó `requirements.txt`, no se actualizó ninguna versión declarada, no se tocó ningún lockfile (el proyecto no usa lockfile, solo rangos `>=` en requirements).

| Paso | Comando | Resultado |
|---|---|---|
| Clonar pyenv | `git clone --depth 1 https://github.com/pyenv/pyenv.git ~/.pyenv` | Éxito |
| Instalar Python 3.12.13 | `pyenv install 3.12.13` | Éxito (advertencia no crítica: módulo `_lzma` no compilado — no usado por este proyecto) |
| Verificar versión | `.venv312/bin/python -V` | `Python 3.12.13` |
| Verificar pip | `.venv312/bin/pip -V` | `pip 25.0.1` (versión de pip que trae la instalación limpia de 3.12.13; no se actualizó) |
| Instalar dependencias | `.venv312/bin/pip install -r requirements-dev.txt` | Éxito, exit 0, sin errores de resolución de dependencias |

## Recolección de pruebas

```
$ .venv312/bin/python -m pytest --collect-only -q
```
**Antes de los cambios de código (baseline)**: 111 tests recolectados.
**Después de agregar las pruebas nuevas de P0** (concurrencia, deduplicación, sensor sin dato): **117 tests recolectados**.

## Ejecución de la suite completa

### Baseline (código sin modificar, tal como estaba al cierre de Fase 3)

```
$ .venv312/bin/python -m pytest -q
...
111 passed, 1612 warnings in 64.99s
```

**111/111 pruebas pasan realmente**, confirmando con evidencia de ejecución (no solo lectura estática) que el código previo a las correcciones P0 ya era funcionalmente correcto para todo lo que esas 111 pruebas cubren (JWT, RBAC, hash SHA-256 en frío, Random Forest, reglas de riesgo, rango térmico, revocación de tokens, y las 8 suites de integración de endpoints).

### Tras aplicar las correcciones P0 (atomicidad de hash, deduplicación MQTT, sensor `None`)

```
$ .venv312/bin/python -m pytest -q
...
117 passed, 1612 warnings in 25.79s
```

**117/117 pruebas pasan** (111 preexistentes, sin ninguna regresión, + 6 pruebas nuevas escritas específicamente para demostrar las correcciones). Ninguna prueba fue modificada para forzar que pasara — las 111 originales pasan sin ningún cambio en su código.

## Pruebas nuevas y qué demuestran exactamente

| Archivo | Prueba | Qué demuestra |
|---|---|---|
| `tests/integration/test_hash_chain_concurrencia.py` | `test_escrituras_concurrentes_no_bifurcan_la_cadena` | 20 escrituras de trazabilidad lanzadas concurrentemente (`asyncio.gather`, cada una con su propia sesión de BD independiente) producen una cadena con `total_registros=20` e `integra=True` — sin bifurcación |
| ídem | `test_escrituras_concurrentes_producen_cadena_estrictamente_lineal` | Verificación más estricta: cada `previous_hash` coincide exactamente con el `hash_actual` del registro inmediatamente anterior en orden de inserción — descarta bifurcaciones que pasarían una verificación agregada más débil |
| `tests/integration/test_ingesta_dedup_y_sensor_nulo.py` | `test_reenvio_mqtt_con_mismo_device_y_timestamp_no_duplica` | Enviar la misma lectura (mismo `device_id`+`timestamp`) dos veces produce un único registro en `thermal_readings` (idempotencia real, no solo declarada) |
| ídem | `test_lectura_con_temperaturas_distintas_mismo_instante_tambien_es_idempotente` | Confirma que el criterio de deduplicación es `(device_id, timestamp)`, no el contenido — el primer registro prevalece |
| ídem | `test_sensor_temperatura_interna_none_no_se_convierte_en_cero` | Una lectura con `temperatura_interna=None` persiste con `nivel_riesgo=None` y **no genera ninguna alerta** — confirma que ya no se trata como 0.0°C (que antes disparaba una excursión crítica falsa) |
| ídem | `test_sensor_temperatura_interna_presente_sigue_clasificando_normalmente` | Control: el camino feliz (temperatura real presente) sigue clasificando correctamente tras la corrección — no se rompió nada |

## Hallazgo real descubierto durante la implementación (y corregido)

La primera versión del fix de concurrencia usaba un `asyncio.Lock()` module-level simple. Al ejecutar la prueba de concurrencia, esto **falló realmente** con `RuntimeError: <Lock> is bound to a different event loop` — porque pytest-asyncio crea un event loop nuevo por test, y un `asyncio.Lock` ordinario queda atado al primer loop que lo usa. Se corrigió con un wrapper (`_CandadoDeProceso`) que rebina el lock automáticamente si el loop en ejecución cambió — seguro tanto en producción (un solo loop durante toda la vida del proceso, nunca cambia) como en la suite de pruebas (un loop nuevo por test). Esto es exactamente el tipo de defecto que solo se descubre ejecutando pruebas reales, no leyendo el código — justifica la instrucción del usuario de no declarar el P0 cerrado sin ejecución real.

## Verificación explícita solicitada por el usuario: ¿las pruebas demuestran atomicidad, ausencia de duplicados y tratamiento correcto de sensores sin lectura?

**Sí, las tres, con pruebas dedicadas y pasando realmente** (no se declara el P0 cerrado solo porque `pytest` en general pasa):
- **Atomicidad del hash**: demostrada por las 2 pruebas de `test_hash_chain_concurrencia.py` bajo concurrencia real (`asyncio.gather` con sesiones de BD independientes, no secuencial).
- **Ausencia de duplicados**: demostrada por 2 pruebas de deduplicación con reenvío simulado.
- **Tratamiento correcto de sensores sin lectura**: demostrado por 2 pruebas (caso `None` y caso de control con dato real).

## Actualización — validación real contra PostgreSQL local (no producción)

Lo que la sección anterior marcaba como "pendiente" fue ejecutado realmente contra una instancia PostgreSQL local aislada, por instrucción explícita del usuario (no Railway, no credenciales de producción).

### Entorno

- **Servidor**: Postgres.app, `SELECT version()` real devuelto: `PostgreSQL 18.4 (Postgres.app) on aarch64-apple-darwin23.6.0, compiled by Apple clang version 15.0.0 (clang-1500.3.9.4), 64-bit`.
- Nota: se pidió inicializar la instancia 16 de Postgres.app, pero el servidor que efectivamente respondía en `localhost:5432` era la instancia 18 (confirmado por incompatibilidad del cliente `psql` v16 contra el catálogo del servidor: `column d.daticulocale does not exist` — columna renombrada a `datlocale` en PG18). Se confirmó con el usuario y se autorizó continuar con la 18 en lugar de forzar el cambio a la 16.
- Base de datos exclusiva: `thermotrace_p0_test`, creada solo para esta verificación.
- Sin conexión a Railway, sin credenciales de producción, sin tocar ninguna otra base local.

### Migraciones Alembic — ejecutadas realmente por primera vez contra PostgreSQL

```
$ alembic upgrade head
INFO  Running upgrade  -> 0001_initial_schema, ...
INFO  Running upgrade 0001_initial_schema -> 0002_thermal_readings_dedup, ...
```
Resultado: **10 tablas creadas** (9 propias + `alembic_version`), sin errores. `\d thermal_readings` confirma la restricción `uq_thermal_readings_device_timestamp UNIQUE CONSTRAINT, btree (device_id, "timestamp")` realmente presente en el esquema de PostgreSQL (no solo en SQLite vía `create_all`).

### Concurrencia del hash encadenado — script de verificación directo (no pytest, ver justificación abajo)

30 escrituras concurrentes (`asyncio.gather`, sesiones independientes) contra la tabla real de PostgreSQL:
```
total_registros: 30
integra: True
primer_registro_inconsistente: None
cadena_estrictamente_lineal: True
```
**Sin bifurcación bajo concurrencia real contra PostgreSQL** — la misma garantía ya demostrada contra SQLite en la suite pytest, ahora confirmada también contra el motor de base de datos real de producción (aunque en una instancia local, no en Railway). El branch `pg_advisory_xact_lock` (activo solo cuando `dialect.name == "postgresql"`) se ejecutó realmente en cada una de las 30 escrituras sin error — antes solo se ejecutaba con SQLite, donde ese bloque se omite.

### Restricción `UNIQUE(device_id, timestamp)` — rechazo real confirmado

Insertar una segunda lectura con el mismo `device_id`+`timestamp` que una ya persistida produjo `asyncpg.exceptions.UniqueViolationError` capturado como `IntegrityError` de SQLAlchemy:
```
duplicado_rechazado_por_constraint: True
```
Confirma que la deduplicación (hallazgo B-04) no depende únicamente de la comprobación a nivel de aplicación (`obtener_por_device_y_timestamp`) — el motor de base de datos real la refuerza de forma independiente.

### Por qué un script directo y no `pytest` contra Postgres

`tests/conftest.py::db_session_factory` crea el motor con una URL SQLite **hardcodeada literal** (no leída de `Settings`/variable de entorno), por lo que ejecutar la suite pytest completa contra Postgres habría exigido modificar esa fixture — con riesgo de alterar el comportamiento de los 117 tests ya verificados y aprobados contra SQLite. Se optó por un script de verificación aparte (`verificar_p0_postgres.py`, en el scratchpad de la sesión, no incorporado al repositorio) que reutiliza los mismos casos de uso reales (`RegistrarHashEncadenadoUseCase`, `VerificarIntegridadRegistroUseCase`) contra una sesión SQLAlchemy conectada de verdad a PostgreSQL — mismo código de producción, infraestructura de prueba real, sin tocar la suite pytest existente.

### Limpieza

`DROP DATABASE thermotrace_p0_test;` ejecutado al finalizar — confirmado que no queda ninguna base `thermotrace_*` en el servidor. Ninguna otra base local fue tocada.

### Pendiente real remanente

- Concurrencia contra un broker MQTT real (EMQX) — no evaluable sin infraestructura remota, sigue fuera de alcance.
- Migraciones/concurrencia contra la instancia de Railway real — no evaluado ni se intentó (instrucción explícita de no conectar a producción).

## Otras herramientas (mypy, ruff)

No configuradas en el proyecto (sin `mypy.ini`/`ruff.toml` ni declaradas en `requirements-dev.txt`) — no se instalaron ni ejecutaron, coherente con "no instales dependencias sin necesidad" y con no alterar la configuración declarada del proyecto.

## Conclusión

**El P0 de "ejecutar la suite completa con Python 3.12" está cumplido con evidencia real de ejecución**: 117/117 pruebas pasan, incluyendo 6 pruebas nuevas que demuestran específicamente la corrección de los 3 defectos exigidos (concurrencia del hash, deduplicación MQTT, sensor `None`). La verificación bajo PostgreSQL real y bajo un broker MQTT real queda pendiente y está marcada como tal — no se afirma que esté probada.
