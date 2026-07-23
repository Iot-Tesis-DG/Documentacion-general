# Decisión de artefacto oficial — v1 preservado, v2 promovido

## Archivos finales en `backend/src/infrastructure/ai/models/`

| Archivo | Contenido | Rol |
|---|---|---|
| `random_forest_termico.pkl` | **v2 corregido** (copia de `random_forest_v2_corrected.joblib`) | **Oficial** — el que `RandomForestRiesgoService` carga por defecto |
| `training_metrics.json` | Métricas de v2 | **Oficial** |
| `random_forest_v1_original.joblib` | Copia exacta del artefacto v1 original (sha256 `ca8e3efe06e7a3bc974b488bbca88e0bbd8b2b74bdfa4554991c3b651ff931ea`, sin modificar) | **Evidencia preservada** — F1 weighted=0.9659, accuracy=96.56%, reproducido y verificado en esta sesión |
| `training_metrics_v1_original.json` | Métricas originales de v1 | **Evidencia preservada** |
| `random_forest_v2_corrected.joblib` | Copia explícita del v2 (idéntica al contenido de `random_forest_termico.pkl`) | Nombre explícito para trazabilidad de versión |
| `training_metrics_v2_corrected.json` | Métricas de v2 | Nombre explícito para trazabilidad de versión |

## Criterio de decisión aplicado

Según instrucción explícita: *"Si el modelo v2 mantiene F1 weighted ≥ 0.85 y presenta una metodología defendible, intégralo como artefacto oficial."*

- **F1 weighted v2**: 0.9748-0.9809 según la ejecución (ver nota de reproducibilidad en `ai_model_comparison.md`) — **cumple ampliamente** el umbral de 0.85.
- **Metodología defendible**: confirmado — sin circularidad (verificado arquitectónicamente: las 3 features antes "filtradas" ahora se derivan de una serie observada con ruido de sensor y pérdida simulada de mensajes, usando la misma función de producción), partición agrupada por escenario (`GroupShuffleSplit`/`StratifiedGroupKFold`, verificado que ningún escenario aparece en ambos conjuntos), y **v2 supera tanto a la regla determinista pura como a v1 evaluado de forma justa** en las 4 métricas comparadas.

**Decisión: v2 se integra como artefacto oficial.** v1 permanece disponible íntegro y sin modificar como evidencia reproducible del estado original — nunca se sobrescribió ni se eliminó, solo se copió bajo un nombre explícito antes de reemplazar el puntero que carga el backend.

## Metadata del artefacto oficial (v2)

```json
{
  "model_name": "random_forest_thermal_risk",
  "model_version": "2.0.0-corrected",
  "n_samples": ~10000 (varía según ejecución, ~400 escenarios × 15-35 ticks),
  "particion": "GroupShuffleSplit + StratifiedGroupKFold, agrupado por escenario_id",
  "dataset_hash": "<sha256 del array de features + etiquetas>",
  "model_hash": "<sha256 del archivo .joblib>",
  "correccion_respecto_v1": "duracion_fuera_rango, frecuencia_desviaciones y tendencia_termica ya no son escalares independientes sin incertidumbre..."
}
```

Ambos hashes (`dataset_hash`, `model_hash`) están embebidos en la metadata del artefacto — corrige el hallazgo AI-05.
