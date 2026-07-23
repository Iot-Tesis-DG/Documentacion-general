# Etapa B — resumen de implementación

## Diagnóstico correcto (no automáticamente "data leakage")

Confirmado como **circularidad entre reglas de etiquetado y variables predictoras**, no fuga clásica train/test (sin duplicados, sin información futura, sin partición contaminada en v1 — ver `ai_diagnosis_v2.md`). Escenario aplicable: **B** (clasificador que aproxima reglas térmicas expertas, no una verdad clínica independiente) — Escenario A (etiquetas independientes) inviable sin datos reales; Escenario C (predicción futura) cambiaría el objetivo oficial de la tesis, no autorizado.

## Corrección implementada

`train_model_v2.py` (nuevo, no sobrescribe `train_model.py`): dataset con **escenarios temporales** (episodios de 15-35 lecturas consecutivas), donde `duracion_fuera_rango`/`frecuencia_desviaciones`/`tendencia_termica` se derivan de una serie **observada** (ruido de sensor + pérdida simulada de mensajes MQTT) usando la **misma función de producción** (`ClasificarRiesgoTermicoUseCase._construir_features`, reutilizada literalmente — single source of truth real entre entrenamiento e inferencia). Partición `GroupShuffleSplit`/`StratifiedGroupKFold` agrupada por `escenario_id`, con `assert` que verifica que ningún escenario se reparte entre train/test.

## Resultado — sin manipular para forzar el número

**v2: F1 weighted 0.975-0.981, accuracy 97.4%, recall excursión crítica 1.0000** — superior a v1 (evaluado justamente: 0.9665), a la regla pura (0.9706), al árbol simple (0.9651), a la regresión logística (0.8838) y al baseline mayoritario (0.6058). No se buscó ni se ajustó ruido para bajar la métrica — el resultado salió más alto de lo esperado inicialmente, y se reporta con transparencia total incluyendo la pequeña discrepancia de reproducibilidad detectada entre dos ejecuciones independientes (0.9748 vs 0.9809, ver `ai_model_comparison.md`).

## v1 preservado íntegro

`random_forest_v1_original.joblib` + `training_metrics_v1_original.json` — copias exactas, hash sha256 verificado, nunca modificadas ni eliminadas.

## Hallazgos medios corregidos

| Hallazgo | Corrección | Archivo |
|---|---|---|
| AI-03 (clase desconocida) | Validado al cargar, no en cada inferencia | `random_forest_service.py::_validar_compatibilidad` |
| AI-04 (sin validar features/clases al cargar) | Ídem — `RuntimeError` claro si el artefacto es incompatible | ídem |
| AI-05 (sin checksum) | `dataset_hash` + `model_hash` embebidos en la metadata | `train_model_v2.py` |
| AI-06 (`model_version` no persistido) | Nuevas columnas `modelo_version`, `confianza_ia`, `origen_clasificacion` en `thermal_readings` | Migración `0003_lecturas_modelo_ia.py` (upgrade+downgrade, no toca migraciones previas) |
| AI-07 (SSE sin confianza/versión) | `LecturaResponse` ahora incluye `confianza_ia`, `modelo_version` | `schemas.py`, `mappers.py` |
| AI-02 (manejo de errores demasiado amplio) | El `except Exception` de MQTT ahora audita con `logger.exception` en vez de descartar en silencio | `mqtt_client.py::consumir_mensajes` |

## Verificación real ejecutada (no solo lectura de código)

- Migraciones `0001→0002→0003` corridas contra **PostgreSQL 18 real** (Postgres.app, DB aislada `thermotrace_ai_test`, eliminada al finalizar) — **encontró y corrigió un bug real**: el ID de la migración 0003 excedía `VARCHAR(32)` de `alembic_version` (SQLite no lo habría detectado).
- Suite completa: **123/123 pruebas pasan** (117 anteriores sin regresión + 6 nuevas: 4 de validación de compatibilidad del modelo, 2 de persistencia de versión/confianza).
- Reentrenamiento v1 reproducido exactamente antes de tocar nada (ver `ai_evaluation_report.md`).
- Reentrenamiento v2 ejecutado dos veces (script principal + script de comparación), ambas superan 0.85.

## Preservado sin tocar (confirmado)

- Candado de proceso del hash SHA-256 (P0).
- Deduplicación `(device_id, timestamp)` (P0).
- Tratamiento de sensor `None` (P0, reutilizado y extendido con `modelo_version`/`confianza_ia`).
- Migraciones `0001` y `0002` — no editadas, solo se añadió `0003` encima.
- Las 117 pruebas P0/Fase 3 — todas siguen pasando.

## Pendiente (no bloqueante, documentado)

- Endpoint dedicado `/api/ia/estado` simplificado (existe `/api/ia/modelo` que ya cumple función equivalente, más verboso).
- Redacción de la sección académica de tesis (pendiente, no se toca TI/PPTX en esta sesión por instrucción explícita: "No modifiques todavía la tesis ni la presentación").
- Fijar `n_jobs=1` en el entrenamiento para eliminar la pequeña discrepancia de reproducibilidad detectada.
