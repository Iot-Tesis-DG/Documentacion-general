# Pipeline de entrenamiento — estado actual (Etapa A, sin modificar)

## Estructura real (no reorganizada — se documenta tal como existe)

```
backend/src/infrastructure/ai/
├── features.py            ← definición de features (fuente única)
├── reglas_riesgo.py        ← regla determinista (etiquetado + salvaguarda)
├── train_model.py          ← dataset sintético + entrenamiento + evaluación + guardado (todo en un archivo)
├── random_forest_service.py ← carga + inferencia en producción
└── models/
    ├── random_forest_termico.pkl
    └── training_metrics.json
```

No existe la estructura `ml/datasets/`, `ml/features/`, `ml/training/`, `ml/evaluation/`, `ml/artifacts/` sugerida por el prompt como ejemplo — **correctamente no se reorganizó**, ya que el prompt explícitamente permite adaptar la estructura y pide no reorganizar el backend solo por imitarla. La separación conceptual sí existe (features.py separado, reglas separadas, entrenamiento separado, servicio de inferencia separado) aunque en menos carpetas.

## Pasos ejecutados por `train_model.py::entrenar()` — verificados contra el prompt

| Paso pedido | ¿Existe? | Evidencia |
|---|---|---|
| 1. Cargar el dataset | Sí (se genera, no se carga de archivo) | `generar_dataset()` |
| 2. Validar el esquema | No explícito | El dataclass `FeaturesRiesgoTermico` impone tipos en construcción, pero no hay una validación posterior separada |
| 3. Limpiar mediante reglas documentadas | No aplica | Dataset sintético sin necesidad de limpieza (sin missing/outliers reales) |
| 4. Separar train/test sin fuga | **Parcial — ver hallazgo AI-01 crítico**: la partición en sí es correcta, el contenido de las features no |
| 5. Entrenar Random Forest | Sí | líneas 110-124 |
| 6. Validación cruzada | Sí | `StratifiedKFold(5)` + `cross_val_score` |
| 7. Métricas | Sí | accuracy, classification_report, confusion_matrix |
| 8. Matriz de confusión | Sí | línea 129 |
| 9. Guardar artefacto | Sí | `joblib.dump` línea 155 |
| 10. Guardar metadata | Sí | dict `metadata`, embebido en el mismo `.pkl` y también en `training_metrics.json` |
| 11. Guardar orden de features | Sí | `metadata["feature_names"] = list(FEATURE_NAMES)` |
| 12. Checksum del modelo | **No existe** | Sin hash del `.pkl` ni del dataset en ningún archivo |
| 13. Fallar si no cumple umbral | Sí | `if f1_ponderado < F1_MINIMO_RNF04: raise SystemExit(...)` líneas 133-137 |

## Hiperparámetros documentados (verificados en código, `train_model.py` líneas 110-117)

| Hiperparámetro | Valor |
|---|---|
| `n_estimators` | 200 |
| `max_depth` | 12 |
| `min_samples_split` | no especificado (default sklearn = 2) |
| `min_samples_leaf` | 3 |
| `max_features` | no especificado (default sklearn = "sqrt" para clasificación) |
| `class_weight` | "balanced" |
| `bootstrap` | no especificado (default sklearn = True) |
| `random_state` | 42 |

No hay evidencia de búsqueda de hiperparámetros (`GridSearchCV`/`RandomizedSearchCV`) — valores fijos, razonables, no optimizados exhaustivamente. Correcto según la restricción del prompt de no perseguir la métrica con tuning agresivo.

## No dispara desde FastAPI

Confirmado: `train_model.py` solo se invoca manualmente (`python -m src.infrastructure.ai.train_model`, línea 14 del docstring) y bajo `if __name__ == "__main__"` (línea 178). Ningún router ni caso de uso lo importa. El backend nunca reentrena por sí mismo en cada petición ni en el arranque — cumple la restricción explícita del prompt.
