# 01 — Control de repositorio e inventario

## Fase 1 — control del repositorio

```
$ pwd
/Volumes/Universidad/Tesis Code/backend
$ git branch --show-current
feat/random-forest-pipeline
$ git branch -a
* feat/random-forest-pipeline
  main
```

Rama `feat/random-forest-pipeline` **confirmada existente y activa**. No se ejecutó ningún comando destructivo (`reset`, `checkout --`, `clean`). No se hizo commit ni push en esta verificación.

## Clasificación de cambios locales (54 entradas, `git status --short`)

| Categoría | Archivos | Origen |
|---|---|---|
| **Pre-existentes (anteriores a toda intervención de esta sesión de auditoría)** | `.env.example`, `autenticar_usuario.py`, `verificar_integridad_registro.py`, `registro_trazabilidad.py`, `exceptions.py`, `i_device_repository.py`, `hash_encadenado.py`, `config.py`, `device_repository.py`, `jwt_handler.py`, `auth_router.py`, `deps.py`, `lecturas_router.py`, `sse_router.py`, `main.py`, `security/rate_limiter.py`, `security/revocation_store.py`, `api_protection.py`, `security_headers.py` | Confirmado en auditorías previas: el repositorio tiene un único commit `"Init backend"` desactualizado respecto al árbol de trabajo real — estos archivos ya estaban modificados antes de que empezara cualquier intervención de IA/P0 |
| **P0 (trazabilidad, deduplicación, sensor `None`)** | `registrar_hash_encadenado.py`, `trazabilidad_repository.py`, `i_lectura_repository.py`, `lectura_repository.py`, `models.py` (parcial), `registrar_lectura_termica.py` (parcial), `alembic/versions/0002_thermal_readings_dedup.py`, `tests/integration/test_hash_chain_concurrencia.py`, `tests/integration/test_ingesta_dedup_y_sensor_nulo.py` | Sesión P0 |
| **IA (Etapa A/B de esta serie)** | `clasificar_riesgo_termico.py`, `random_forest_service.py`, `train_model.py` (dirty preexistente, restaurado tras reproducción — ver `06_reproducibility.md`), `train_model_v2.py` (nuevo), `models.py` (columnas IA), `lectura_repository.py` (columnas IA), `lectura_termica.py`, `schemas.py`, `mappers.py`, `mqtt_client.py`, `alembic/versions/0003_lecturas_modelo_ia.py`, `models/random_forest_v1_original.joblib`, `models/random_forest_v2_corrected.joblib`, `models/training_metrics_v1_original.json`, `models/training_metrics_v2_corrected.json`, `models/random_forest_termico.pkl` (contenido = v2), `models/training_metrics.json` (contenido = v2), `tests/unit/test_random_forest_service.py`, `tests/unit/test_random_forest_service_validacion.py`, `tests/integration/test_ai_persistencia_version.py` | Sesión IA |
| **Generados por entorno (no código)** | `.venv312/` (entorno virtual Python 3.12, ignorado del control de versiones en la práctica), `scripts/` (preexistente, `seed_dev.py`) | Entorno de pruebas |
| **No rastreados adicionales sin relación con IA/P0** | `tests/integration/test_seguridad_api.py`, `tests/unit/test_revocation_store.py` | Pre-existentes (parte del estado dirty original) |

Ningún cambio fue descartado ni sobrescrito en esta verificación.

## Inventario de archivos relacionados con IA (búsqueda fresca, no asumida)

```
$ grep -l "RandomForest|random_forest|joblib|model_version|confianza_ia|GroupShuffleSplit|StratifiedGroupKFold" (recursivo, excluyendo .venv*/.git/.pytest_cache)
```

| Ruta | Tipo | Propósito | Revisado | Observación |
|---|---|---|---|---|
| `src/infrastructure/ai/features.py` | Código | Definición de 10 features (`FEATURE_NAMES`, `to_array()`) | Sí | Única fuente de verdad del orden — sin cambios |
| `src/infrastructure/ai/reglas_riesgo.py` | Código | Regla determinista de etiquetado/salvaguarda | Sí | Sin cambios respecto a auditoría previa |
| `src/infrastructure/ai/train_model.py` | Código | Script de entrenamiento **v1** (original, preservado como generador de evidencia histórica) | Sí | No se ejecuta en producción; solo referencia |
| `src/infrastructure/ai/train_model_v2.py` | Código | Script de entrenamiento **v2** (oficial, corrige circularidad) | Sí | Nuevo, esta serie de sesiones |
| `src/infrastructure/ai/random_forest_service.py` | Código | Carga singleton + inferencia + validación de compatibilidad | Sí | Modificado (P0 + IA) |
| `src/infrastructure/ai/models/random_forest_termico.pkl` | Artefacto binario | **Puntero oficial** que carga el servicio | Sí (metadata) | Contenido = v2 |
| `src/infrastructure/ai/models/training_metrics.json` | Métricas | Métricas del artefacto oficial | Sí | Contenido = v2 |
| `src/infrastructure/ai/models/random_forest_v1_original.joblib` | Artefacto binario | v1 preservado como evidencia | Sí (hash) | No se carga en producción |
| `src/infrastructure/ai/models/training_metrics_v1_original.json` | Métricas | Métricas v1 preservadas | Sí | — |
| `src/infrastructure/ai/models/random_forest_v2_corrected.joblib` | Artefacto binario | Copia explícita de v2 (idéntica al puntero oficial) | Sí (hash) | — |
| `src/infrastructure/ai/models/training_metrics_v2_corrected.json` | Métricas | Métricas v2 (idénticas a `training_metrics.json`) | Sí | — |
| `src/application/use_cases/clasificar_riesgo_termico.py` | Código | Único caso de uso que construye features de producción | Sí | Guard sensor `None` (P0) |
| `src/application/use_cases/registrar_lectura_termica.py` | Código | Orquesta pipeline completo | Sí | Persistencia de `modelo_version`/`confianza_ia` (IA) |
| `src/domain/entities/lectura_termica.py` | Código | Entidad de dominio | Sí | Campos `modelo_version`/`confianza_ia`/`origen_clasificacion` añadidos |
| `src/infrastructure/database/models.py` | ORM | `ThermalReadingModel` | Sí | Columnas IA + `UniqueConstraint` P0 |
| `src/infrastructure/database/repositories/lectura_repository.py` | Repositorio | Mapeo ORM↔dominio | Sí | Incluye nuevas columnas |
| `src/interface/api/schemas.py` | Pydantic | `LecturaResponse` | Sí | `confianza_ia`/`modelo_version` añadidos |
| `src/interface/api/mappers.py` | Código | Dominio→schema | Sí | Actualizado |
| `src/interface/api/ia_router.py` | Router | `/api/ia/modelo`, `/api/ia/clasificar` | Sí | Sin cambios en esta serie de sesiones |
| `src/interface/main.py` | Código | Pipeline MQTT | Sí | Sin cambios relacionados con IA en esta sesión |
| `src/infrastructure/mqtt/mqtt_client.py` | Código | Consumidor MQTT | Sí | Logging de errores añadido (P0/IA) |
| `alembic/versions/0003_lecturas_modelo_ia.py` | Migración | Columnas IA en `thermal_readings` | Sí | Nueva |
| `tests/unit/test_random_forest_service.py` | Pruebas | 8 pruebas del servicio IA | Sí | 1 assertion actualizada (versión) |
| `tests/unit/test_random_forest_service_validacion.py` | Pruebas | 4 pruebas de validación de compatibilidad | Sí | Nuevo |
| `tests/integration/test_ia_api.py` | Pruebas | 5 pruebas de endpoints IA | Sí | Sin cambios |
| `tests/integration/test_ai_persistencia_version.py` | Pruebas | 2 pruebas de persistencia de versión/confianza | Sí | Nuevo |

Excluidos de lectura línea por línea (existencia registrada): `.venv/`, `.venv312/`, `.git/`, `__pycache__/`, `.pytest_cache/`.
