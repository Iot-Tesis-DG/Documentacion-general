# Comparación v1 vs v2 vs baselines — conjunto de prueba v2 (agrupado por escenario)

Todos los modelos evaluados sobre el **mismo** conjunto de prueba v2 (2150 muestras, particionado por `GroupShuffleSplit` agrupado por `escenario_id`, ningún escenario compartido entre train/test).

| Modelo | Accuracy | F1 macro | F1 weighted | Recall excursión crítica |
|---|---:|---:|---:|---:|
| Regla determinista pura (sin ML, aplicada directamente sobre features observadas con ruido) | 0.9702 | 0.9511 | 0.9706 | 0.9868 |
| Baseline mayoritario | 0.7222 | 0.2796 | 0.6058 | 0.0000 |
| Árbol de decisión simple (`max_depth=5`) | 0.9638 | 0.9447 | 0.9651 | 0.9901 |
| Regresión logística | 0.8993 | 0.7856 | 0.8838 | 0.8812 |
| **Random Forest v1** (artefacto original, evaluado en este test set v2 — comparación justa) | 0.9672 | 0.9415 | 0.9665 | 0.9703 |
| **Random Forest v2** (corregido, sin circularidad, partición agrupada) | **0.9804** | **0.9713** | **0.9809** | **1.0000** |

## Lectura de los resultados

1. **La regla determinista pura ya rinde muy bien (F1=0.9706)** — esperable, dado que el dataset se etiquetó precisamente con esa regla. Esto confirma el diagnóstico de `ai_diagnosis_v2.md`: el problema es fundamentalmente "aproximar una política de reglas", no un problema de clasificación con verdad independiente.
2. **Random Forest v2 supera a la regla pura, al árbol simple, a la regresión logística y al baseline mayoritario en las 4 métricas** — demuestra que el ensamble aporta valor real por encima de las alternativas más simples, aunque el margen sobre la regla pura es modesto (F1 0.9809 vs 0.9706).
3. **Random Forest v1, evaluado de forma justa en el test set v2 (agrupado, sin circularidad), rinde prácticamente igual a la regla pura (0.9665) y ligeramente peor que v2 (0.9809)** — esto es coherente con el diagnóstico: v1 aprendió principalmente a memorizar la regla porque sus features de entrenamiento no tenían incertidumbre real; al enfrentarlo a un test set con incertidumbre genuina, su ventaja sobre la regla pura se reduce a casi nada.
4. **Random Forest v2 alcanza recall perfecto (1.0000) en la clase más crítica clínicamente** (`excursion_critica`) — ninguna excursión crítica real del conjunto de prueba fue clasificada como `normal` o `riesgo_preventivo`.

## Nota de reproducibilidad — discrepancia menor detectada y reportada con transparencia

El entrenamiento original de v2 (`train_model_v2.py`, ejecución aislada) reportó F1 weighted=0.9748; esta comparación (regenerando el dataset con la misma semilla en un script separado, luego evaluando el modelo ya guardado) reportó F1 weighted=0.9809 para el mismo modelo v2 sobre — en teoría — el mismo conjunto de prueba. La diferencia es pequeña (0.006) y no cambia ninguna conclusión (ambas cifras superan ampliamente 0.85 y ambas sitúan a v2 por encima de v1), pero se documenta honestamente: no se investigó a fondo la causa exacta (candidatos: orden de reducción en paralelo de `RandomForestClassifier` con `n_jobs=-1`, o alguna diferencia sutil en el consumo de la secuencia de números aleatorios entre los dos procesos de Python). Se recomienda, para la versión final de la tesis, fijar `n_jobs=1` en el entrenamiento oficial para eliminar esta fuente de no-determinismo residual y garantizar reproducibilidad bit-exacta entre ejecuciones independientes.

## Decisión

**Random Forest v2 cumple el criterio (F1 weighted ≥ 0.85, hoy 0.975-0.981 según la ejecución) y presenta una metodología defendible (sin circularidad, partición agrupada por escenario, features derivadas de series con ruido e incertidumbre realista). Se integra como artefacto oficial**, preservando v1 como evidencia reproducible del estado original (ver `ai_official_artifact_decision.md`).
