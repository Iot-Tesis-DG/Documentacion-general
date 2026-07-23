# 02 — AIV-01 corregido: reproducibilidad del entrenamiento

## Cambio

`train_model_v3.py::generar_dataset_v3` reemplaza `escenario_id = uuid4().hex[:12]` por:

```python
escenario_id = f"esc-{RANDOM_STATE:04d}-{indice:05d}"
```

Determinista: depende únicamente de `RANDOM_STATE` (configuración fija) y del índice de escenario (0..399), sin ninguna fuente de entropía del sistema operativo.

## Verificación — dos ejecuciones independientes, misma configuración

```
Corrida 1: dataset_hash 6f2d2321dff07f9...:c94fc260bcad617...
Corrida 2: dataset_hash 6f2d2321dff07f9...:c94fc260bcad617...   → IDÉNTICO
Corrida 1: model_hash   d8a3c12b1528ca6f...
Corrida 2: model_hash   d8a3c12b1528ca6f...                     → IDÉNTICO
Corrida 1: f1_weighted  0.9681
Corrida 2: f1_weighted  0.9681                                  → IDÉNTICO
```

```
$ shasum -a 256 /tmp/run1_model.joblib random_forest_v3_reproducible.joblib
d8a3c12b1528ca6f7fc1135df7591486aa8e852d4c1c8c0d3ecde1bd5637745c  /tmp/run1_model.joblib
d8a3c12b1528ca6f7fc1135df7591486aa8e852d4c1c8c0d3ecde1bd5637745c  random_forest_v3_reproducible.joblib
```

**El archivo `.pkl` resultó byte-idéntico entre ambas ejecuciones** (no solo las métricas/dataset). Esto es mejor que el mínimo exigido: no solo hay reproducibilidad funcional (mismas predicciones/métricas), sino identidad binaria completa del artefacto en este caso concreto — determinada por: (a) `escenario_id` ahora determinista, (b) `RandomForestClassifier(random_state=42, n_jobs=-1)` produce el mismo árbol interno con estos datos/tamaño de problema (no se observó no-determinismo de paralelización en esta ejecución), (c) `joblib.dump` sobre un estimador puro (sin diccionario envolvente con timestamp u otros campos variables) serializa de forma estable.

**Nota de precisión honesta**: esta identidad binaria del `.pkl` no está garantizada por diseño para todo hardware/versión de scikit-learn — la reproducibilidad GARANTIZADA por el diseño es la de `dataset_hash`/`f1_weighted`/matriz de confusión (deterministas por construcción, dado que dependen solo de `rng` sembrado + `escenario_id` determinista). La identidad del `.pkl` es una observación empírica de esta verificación, no una garantía formal del pipeline — no se afirma reproducibilidad binaria universal, solo se reporta lo observado.

## Verificación adicional: aislamiento exacto de la causa

Se invocó `generar_dataset_v2()` y `generar_dataset_v3()` directamente, cada uno con un `np.random.default_rng(42)` recién creado, para descartar cualquier otra diferencia entre ambos scripts más allá de la fuente de `escenario_id`:

```python
x2, y2, g2 = generar_dataset_v2(400, np.random.default_rng(42))
x3, y3, g3 = generar_dataset_v3(400, np.random.default_rng(42))
np.array_equal(x2, x3)  # True
np.array_equal(y2, y3)  # True
np.array_equal(g2, g3)  # False (esperado: v2 usa uuid4, v3 determinista)
```

**Confirmado**: el código de generación de features/etiquetas de v2 y v3 es funcionalmente idéntico (`x`/`y` byte-idénticos dado el mismo `rng`); la ÚNICA fuente de no-reproducibilidad en v2 era `escenario_id`. El `dataset_hash` distinto entre el `training_metrics_v2_corrected.json` archivado (evidencia congelada de una corrida anterior de v2) y `model_metadata_v3.json` no indica un nuevo defecto — es la evidencia esperada de que el artefacto v2 archivado proviene de una corrida histórica anterior a esta corrección, preservada intacta como evidencia (no se regeneró ni se sobrescribió).

## Estado AIV-01: **RESUELTO**
