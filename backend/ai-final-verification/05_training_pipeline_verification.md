# 05 — Verificación del pipeline de entrenamiento oficial (`train_model_v2.py`)

## Configuración verificada (lectura directa del código)

| Parámetro | Valor |
|---|---|
| Python | 3.12.13 (confirmado, `metadata.python_version`) |
| scikit-learn | 1.9.0 (confirmado, `metadata.sklearn_version`) |
| `random_state` | 42 |
| `n_estimators` | 200 |
| `max_depth` | 12 |
| `min_samples_leaf` | 3 |
| `class_weight` | "balanced" |
| `max_features` | no especificado (default sklearn = "sqrt") |
| `n_jobs` | -1 |
| Validación cruzada | `StratifiedGroupKFold(n_splits=5, shuffle=True, random_state=42)`, agrupado por `escenario_id` |
| Umbral F1 (RNF-04) | 0.85, gate real (`raise SystemExit` si no se cumple, línea de `entrenar()`) |

## Comportamiento de fallo — verificado por lectura

- Si `f1_ponderado < 0.85`: el script escribe las métricas reales (no manipuladas) en `training_metrics_v2_corrected.json` para diagnóstico, y **lanza `SystemExit` sin guardar el artefacto** — el modelo insuficiente nunca se convierte en el "oficial".
- No hay manejo explícito de "columnas faltantes" o "clases no coincidentes" en el propio script de entrenamiento (el dataset se genera internamente, no se carga de una fuente externa que pudiera tener columnas faltantes) — este chequeo de esquema es más relevante para el **servicio de inferencia** (`random_forest_service.py::_validar_compatibilidad`, verificado en `07_official_artifact.md`), no para el entrenamiento en sí.
- No depende de rutas absolutas de una máquina específica (`MODELS_DIR = Path(__file__).parent / "models"`, relativa al propio archivo).
- No se conecta a producción (confirmado: sin imports de `aiomqtt`, `asyncpg`, ni URLs de Railway/EMQX en todo el archivo).
- No entrena al iniciar FastAPI (confirmado en `02_architecture_verification.md`: cero referencias cruzadas).

## HALLAZGO — no-determinismo real en la partición (nuevo, no reportado en auditorías previas)

**Verificado por ejecución real, dos veces, en esta sesión**: el dataset (`x`, `y`) es **byte-idéntico** entre ejecuciones con la misma semilla (confirmado con `np.array_equal`), pero los **identificadores de escenario (`escenario_id`) se generan con `uuid4().hex[:12]`, que NO está seedeado** por el `rng` del script — usa el generador global de UUIDs del sistema operativo, genuinamente aleatorio en cada proceso.

Esto significa que aunque las features y etiquetas son perfectamente reproducibles, el **agrupamiento para `GroupShuffleSplit`/`StratifiedGroupKFold` cambia de ejecución en ejecución** (los IDs de grupo son distintos strings cada vez, lo que altera qué filas caen en train vs test), produciendo métricas de test ligeramente distintas en cada corrida:

| Ejecución | n_train | n_test | Accuracy | F1 weighted |
|---|---:|---:|---:|---:|
| Oficial (guardada) | 7926 | 2150 | 0.9744 | 0.9748 |
| Repetición 1 (esta sesión) | 8054 | 2022 | 0.9738 | 0.9744 |
| Repetición 2 (esta sesión) | 8063 | 2013 | 0.9707 | 0.9712 |

**Todas las ejecuciones superan ampliamente 0.85 (RNF-04 robusto a esta variación)**, y la diferencia entre ejecuciones es pequeña (0.971-0.975), pero **la cifra exacta reportada (0.9748) no es bit-reproducible entre ejecuciones independientes** — solo el dataset subyacente lo es. Ver clasificación de severidad en `14_findings.md`.

**Causa raíz exacta**: `train_model_v2.py`, dentro de `generar_dataset_v2()`, línea `escenario_id = uuid4().hex[:12]` — debería derivarse determinísticamente del `rng` seedeado (p. ej. `f"escenario-{i:04d}"` usando el índice del bucle, o un hex generado con `rng.bytes(6).hex()`) para que la partición también sea reproducible.

## No se modifica nada en esta verificación

Los artefactos oficiales (`random_forest_termico.pkl`, `training_metrics.json`) fueron respaldados antes de las 2 ejecuciones de prueba y **restaurados exactamente** al finalizar (hash sha256 verificado idéntico al de antes de la verificación: `31e9224a201631cdb0072f6c7b7c7af5b45329e0a5686c350f4d2c5097dba656`).
