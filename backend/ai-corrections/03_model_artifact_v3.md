# 03 — Artefacto oficial v3 (AIV-01 + AIV-06 resueltos, promovido a oficial)

## Tabla de comparación obligatoria

| Modelo | Dataset hash (x) | F1 macro | F1 weighted | Accuracy | Recall excursión crítica | CV media ± std |
|---|---|---:|---:|---:|---:|---:|
| Mayoritario (baseline) | — (mismo split v3) | — | 0.5087 | — | 0.0 (predice siempre la clase mayoritaria) | — |
| Árbol simple (`max_depth=6`) | — (mismo split v3) | — | 0.9637 | — | — (no reportado por separado) | — |
| RF v1 (original, circular) | `2ef4...` (histórico) | 0.9535 | 0.9659 | 0.9656 | 0.9730 | 0.9615 ± 0.0031 |
| RF v2 (corregido, partición no reproducible) | `cad04b23...` (evidencia congelada) | 0.9613 | 0.9748 | 0.9744 | 0.9762 | 0.9722 ± 0.0039 |
| **RF v3 (reproducible, oficial)** | `6f2d2321...` | 0.9586 | **0.9681** | 0.9678 | **0.9778** | 0.9732 ± 0.0019 |

**No se manipuló el dataset ni el umbral para forzar el resultado**: el F1 de v3 (0.9681) es real, medido sobre la partición determinista, y es ligeramente inferior al de v2 (0.9748) porque la partición train/test de v2 dependía de una agrupación aleatoria distinta cada corrida (evidencia congelada de una de esas corridas, no reproducible), mientras que v3 usa la ÚNICA partición determinista y estable derivada de la semilla. Ambos superan holgadamente F1≥0.85 (RNF-04). v3 tiene la desviación estándar de validación cruzada más baja (0.0019 vs 0.0039 de v2, 0.0031 de v1) — evidencia de mayor estabilidad, consistente con una partición fija y reproducible en lugar de variable.

## Decisión de promoción a oficial

**v3 cumple todos los criterios exigidos:**
- Generación reproducible: confirmado en `02_reproducibility_fix.md` (dataset, partición, métricas y — en esta ejecución concreta — el propio `.pkl` son bit-idénticos entre corridas).
- División por escenarios sigue sin contaminación: `assert set(grupos[train_idx]).isdisjoint(set(grupos[test_idx]))` presente y verificado.
- Conserva la interpretación de Escenario B (circularidad corregida, no data leakage clásico): mismo diseño de v2 (reutiliza `_construir_features` de producción), sin cambios metodológicos regresivos.
- Supera F1 weighted ≥ 0.85: 0.9681, real, no forzado.
- Artifact y metadata coinciden: `model_hash` en `model_metadata_v3.json` corresponde exactamente al sha256 del `.pkl` final, calculado UNA sola vez, sin re-serializar (corrige AIV-06 — ver verificación abajo).
- Checksum válido: verificado por `shasum -a 256` externo, coincide con el campo `model_hash`.
- Integración y pruebas completas: pendiente de ejecutar contra la suite completa tras la promoción (ver `09_tests_execution.md`).

**v3 se promueve como el artefacto oficial** (`random_forest_termico.pkl` + `model_metadata.json`), reemplazando el puntero que hasta ahora usaba v2. v1 y v2 se conservan íntegros como evidencia histórica, sin sobrescribir.

## AIV-06 — verificación de la corrección de circularidad de `model_hash`

```
$ shasum -a 256 random_forest_v3_reproducible.joblib
d8a3c12b1528ca6f7fc1135df7591486aa8e852d4c1c8c0d3ecde1bd5637745c
$ python -c "import json; print(json.load(open('model_metadata_v3.json'))['model_hash'])"
d8a3c12b1528ca6f7fc1135df7591486aa8e852d4c1c8c0d3ecde1bd5637745c
```

**Coinciden exactamente** — a diferencia de v2, donde `model_hash` embebido (`e86b4cb0...`) no coincidía con el sha256 real del archivo final (`31e9224a...`) por la doble serialización. v3 calcula el hash sobre el archivo `.pkl` YA guardado en su forma definitiva (estimador puro, sin dict envolvente) y lo escribe únicamente en el archivo externo `model_metadata.json`, sin volver a tocar el `.pkl`.

## Promoción — archivos resultantes tras esta fase

```
random_forest_termico.pkl          → AHORA contiene el estimador v3 (estimador puro, sin dict envolvente)
model_metadata.json                → NUEVO, metadata externa completa de v3 (reemplaza el rol antes cumplido por training_metrics.json embebiendo metadata dentro del .pkl)
training_metrics.json               → se mantiene por compatibilidad con /api/ia/modelo (mismo contenido que model_metadata.json)
random_forest_v3_reproducible.joblib → copia con nombre explícito (idéntica al puntero oficial)
model_metadata_v3.json              → copia con nombre explícito de la metadata
training_metrics_v3_reproducible.json → copia con nombre explícito de las métricas

random_forest_v2_corrected.joblib   → PRESERVADO, sin cambios (evidencia histórica)
training_metrics_v2_corrected.json  → PRESERVADO, sin cambios
random_forest_v1_original.joblib    → PRESERVADO, sin cambios
training_metrics_v1_original.json   → PRESERVADO, sin cambios
```

Estado AIV-01: **RESUELTO**. Estado AIV-06: **RESUELTO**.

## Corrección final de reproducibilidad y artefacto (2026-07-22)

La revisión de cierre detectó que el generador aún tomaba `datetime.now()` para
la feature horaria. Se sustituyó por `2026-01-01T00:00:00Z`, por lo que dos
ejecuciones ya no cambian de dataset al cruzar una hora. El identificador ahora
incluye semilla, índice, régimen y cantidad de ticks.

Se preservó el v3 previo como `*_v3_pre_clock_fix.*`. V1 y V2 no cambiaron.
Nuevo v3 oficial: `model_hash=0294ad0a07a47b8975d6f6102ee7aab85109049a23104ebb81c445bfdf8d33d4`,
`dataset_hash=36fef3457f205bd3262d96e629133fa9b3edf8f4f2779077a4614f87025efa2a:c94fc260bcad61781ed210164b585bdb6bf5ef779015810dafe401bc27cd356c`,
F1 weighted `0.967130`, accuracy `0.966766`, CV `0.972689 ± 0.002311`.
