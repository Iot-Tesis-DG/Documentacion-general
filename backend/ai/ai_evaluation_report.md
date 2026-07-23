# Reproducción de métricas — ejecutada realmente

## Comando y entorno

```
$ cd backend && .venv312/bin/python -m src.infrastructure.ai.train_model
```
Python 3.12.13, scikit-learn 1.9.0 (idénticos a los registrados originalmente en `training_metrics.json::metadata`).

## Procedimiento

1. Se respaldó el artefacto original (`random_forest_termico.pkl`, sha256 `ca8e3efe...`) y `training_metrics.json` fuera del repositorio.
2. Se ejecutó `train_model.py` desde cero (dataset sintético regenerado en runtime, `random_state=42`, `N_SAMPLES=8000`).
3. Se comparó el resultado contra el original.
4. Se restauraron los archivos originales al finalizar (Etapa A es auditoría, no modificación).

## Resultado de la reproducción

```
Exactitud: 0.9656
F1 ponderado: 0.9659 (RNF-04 >= 0.85: cumplido)
CV 5-fold f1_weighted: 0.9615 ± 0.0031
```

**Diff de `training_metrics.json` (excluyendo el campo `metadata.trained_at`, que naturalmente cambia con cada ejecución) entre el original y el reproducido: vacío.** `accuracy`, `f1_weighted`, `classification_report` completo (precision/recall/F1 por clase), `confusion_matrix`, `cross_validation.scores` (los 5 folds individuales) y `feature_importances` son **byte-idénticos**.

## Conclusión de reproducibilidad

**F1 weighted = 0.9659 y accuracy = 96.56% se reproducen exactamente**, con semilla fija (`random_state=42`) determinando de forma 100% determinista tanto la generación del dataset como el entrenamiento. No hay componente aleatorio no controlado en el pipeline (el `n_jobs=-1` de `RandomForestClassifier` no introduce no-determinismo en el resultado final con `random_state` fijo). **Esto confirma que el número no fue inventado ni corresponde a una ejecución distinta de la documentada** — es exactamente lo que el código produce, de forma verificable por cualquier tercero con el mismo entorno.

## Pero la reproducibilidad no valida la metodología

Ver `ai_leakage_analysis.md` (hallazgo AI-01, CRÍTICO): el número es reproducible pero está inflado por fuga parcial de features. **Reproducible ≠ válido.** Ambas cosas deben reportarse juntas en la tesis: sí se puede reproducir el 0.9659, pero ese 0.9659 no representa limpiamente el desempeño esperado en producción.

## Diferenciación de métricas (exigida por el prompt, sección 10)

| Métrica | Valor exacto | Tipo |
|---|---|---|
| Accuracy | 0.965625 (96.5625%) | Global, no diferenciada por clase |
| F1 macro | 0.9534579151311545 | Promedio simple entre las 3 clases (sin ponderar por soporte) |
| F1 weighted | 0.9658930392248002 | Promedio ponderado por soporte de cada clase — **esta es la métrica oficial de RNF-04** |
| F1 `excursion_critica` | 0.9814585908529048 | Por clase (soporte=816) |
| F1 `riesgo_preventivo` | 0.9563145353455124 | Por clase (soporte=630) |
| F1 `normal` | 0.9226006191950464 | Por clase (soporte=154, la clase minoritaria y con peor F1) |
| Recall `excursion_critica` | 0.9730392156862745 | — |
| Precision `excursion_critica` | 0.9900249376558603 | — |
| CV 5-fold (f1_weighted) | media=0.9614753, std=0.0030760 | Validación cruzada sobre el conjunto de entrenamiento, no el de prueba |

**Nunca se debe decir "exactitud de la IA es 0.9659"** — ese valor es F1 weighted, no accuracy (0.9656). Son métricas numéricamente parecidas mas conceptualmente distintas. TI (según lo auditado en Fase 1) no especifica explícitamente qué tipo de promediado de F1 usa — se recomienda fijar formalmente "F1 ponderado (weighted)" como la métrica oficial de RNF-04, ya que es la que el propio código usa como gate (`scoring="f1_weighted"` en `cross_val_score`, línea 122, y `reporte["weighted avg"]["f1-score"]`, línea 131).

## Recall de `excursion_critica` (pregunta explícita del prompt)

**0.9730** (97.30%) — de 816 casos reales de excursión crítica en el conjunto de prueba, el modelo identificó correctamente 794 (ver matriz de confusión: fila `excursion_critica` = `[794, 0, 22]`, 22 falsos negativos clasificados como `riesgo_preventivo`, **cero** falsos negativos hacia `normal`). Esto es clínicamente relevante: el modelo nunca confunde una excursión crítica real con "normal" en el conjunto de prueba — el peor error posible (pasar por alto completamente un riesgo grave) no ocurrió en esta evaluación, aunque de nuevo, bajo el matiz de que 3 de las features que facilitan esta detección llegan sin ruido de estimación (hallazgo AI-01).

## Comparación con baseline (sección 12) — no ejecutada en Etapa A

Pendiente para Etapa B (requiere código nuevo, fuera del alcance de "auditar sin modificar").
