# Inventario del módulo de IA

| Archivo | Propósito | Estado | Observación |
|---|---|---|---|
| `src/infrastructure/ai/features.py` (45 líneas) | Define `FeaturesRiesgoTermico` (10 variables) y `to_array()` — única fuente de verdad del orden de features | Real, usado por entrenamiento e inferencia | Sin cambios respecto a lo auditado en Fase 3 |
| `src/infrastructure/ai/reglas_riesgo.py` (35 líneas) | `clasificar_por_regla()`: regla determinista 2-8°C, usada (a) para etiquetar el dataset sintético y (b) como salvaguarda de producción | Real | **Doble uso es la raíz del hallazgo crítico de fuga, ver `ai_leakage_analysis.md`** |
| `src/infrastructure/ai/train_model.py` (187 líneas) | Script de entrenamiento independiente (`python -m src.infrastructure.ai.train_model`), genera dataset sintético, entrena, valida, guarda artefacto+métricas | Real, ejecutable, no se dispara desde FastAPI | Sin dataset hash, sin model hash, sin comparación con baseline |
| `src/infrastructure/ai/random_forest_service.py` (127 líneas) | Carga perezosa (singleton lazy) del `.pkl`, inferencia con salvaguarda determinista | Real, singleton correcto (no `joblib.load` por request) | Sin validación de compatibilidad de features/clases al cargar |
| `src/infrastructure/ai/models/random_forest_termico.pkl` (binario, ~4.4MB) | Artefacto serializado (`joblib.dump({"modelo":..., "metadata":...})`) | Real, existe, se carga | Sin checksum embebido |
| `src/infrastructure/ai/models/training_metrics.json` (112 líneas) | Métricas del último entrenamiento ejecutado | Real, `trained_at: 2026-07-11T11:39:03Z` | Reproducido en esta sesión, ver `ai_evaluation_report.md` |
| `src/application/use_cases/clasificar_riesgo_termico.py` (86 líneas) | Caso de uso: construye features desde historial real + ejecuta inferencia | Real, única función de construcción de features en producción (`_construir_features`) | Modificado en P0 (guard de sensor `None`) |
| `src/interface/api/ia_router.py` (66 líneas) | Endpoints `GET /api/ia/modelo` (metadata+métricas) y `POST /api/ia/clasificar` (inferencia bajo demanda) | Real, protegidos con RBAC (`Rol.FARMACEUTICO`) | Ninguno consumido por el frontend (hallazgo ya reportado en Fase 3) |
| `tests/unit/test_random_forest_service.py` (147 líneas) | 8 pruebas: disponibilidad, clasificación estable/crítica, fallback sin modelo, confianza+origen, salvaguarda, metadata v2 | Real, pasa | No prueba fuga de datos ni reproducibilidad del entrenamiento |
| `tests/integration/test_ia_api.py` (77 líneas) | Pruebas de los endpoints `/api/ia/modelo` y `/api/ia/clasificar` vía HTTP | Real, pasa | — |
| `tests/integration/test_ingesta_dedup_y_sensor_nulo.py` | Prueba (agregada en P0) que sensor `None` no dispara alerta falsa | Real, pasa | Cubre parcialmente sección 8 de este prompt |

## No existe

- Dataset persistido como archivo (CSV/parquet) — se genera en runtime, no hay artefacto de datos versionado por separado del código.
- Notebook Jupyter.
- Script de evaluación independiente del script de entrenamiento (están fusionados en `train_model.py::entrenar()`).
- Hash/checksum del dataset o del modelo en la metadata.
- Endpoint de "estado" simplificado (`/api/ia/estado`) — existe `/api/ia/modelo` que cumple función similar pero devuelve el JSON completo de métricas, no un resumen acotado.
- Comparación con baseline (regla mayoritaria, árbol simple, regresión logística).
- Migración Alembic para persistir `model_version`/`confianza` en `thermal_readings` (actualmente solo viven en el payload JSON de `traceability_records`, no en columnas propias de la lectura).
