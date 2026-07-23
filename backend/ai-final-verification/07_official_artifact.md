# 07 — Artefacto oficial (verificado por hash, no por nombre de archivo)

## Distinción real por hash sha256 (no asumida por el nombre)

| Archivo | sha256 | model_version | F1 weighted | Accuracy | Rol |
|---|---|---|---:|---:|---|
| `random_forest_termico.pkl` | `31e9224a201631cdb0072f6c7b7c7af5b45329e0a5686c350f4d2c5097dba656` | `2.0.0-corrected` | 0.9748 | 0.9744 | **Oficial — el que carga FastAPI** |
| `random_forest_v2_corrected.joblib` | `31e9224a201631cdb0072f6c7b7c7af5b45329e0a5686c350f4d2c5097dba656` | `2.0.0-corrected` | 0.9748 | 0.9744 | **Idéntico byte a byte al oficial** (mismo hash) — copia con nombre explícito |
| `random_forest_v1_original.joblib` | `ca8e3efe06e7a3bc974b488bbca88e0bbd8b2b74bdfa4554991c3b651ff931ea` | `2.0.0` | 0.9659 | 0.9656 | Evidencia histórica preservada, **no se carga en producción** |

**Confirmado**: `random_forest_termico.pkl` y `random_forest_v2_corrected.joblib` tienen el **mismo hash sha256** — son el mismo artefacto, no una coincidencia de nombre. El archivo v1 tiene un hash completamente distinto y una versión distinta (`2.0.0` sin sufijo `-corrected`).

## Confirmación de qué carga FastAPI realmente

`random_forest_service.py`, líneas 18-19:
```python
DEFAULT_MODEL_PATH = Path(__file__).parent / "models" / "random_forest_termico.pkl"
DEFAULT_METRICS_PATH = Path(__file__).parent / "models" / "training_metrics.json"
```
**FastAPI carga `random_forest_termico.pkl` (= v2, por hash) — nunca `random_forest_v1_original.joblib` accidentalmente.** No hay ninguna referencia a `v1_original` ni a `v2_corrected` en el código de producción — esos nombres existen solo como evidencia archivada.

## Rechazo de artefactos incompatibles — verificado con pruebas reales (no solo código)

`tests/unit/test_random_forest_service_validacion.py` (4 pruebas, todas pasan):
- Modelo con número de features incorrecto → `RuntimeError` (mensaje contiene "features").
- Modelo con clases desconocidas → `RuntimeError` (mensaje contiene "clases desconocidas").
- Modelo con orden de features distinto → `RuntimeError` (mensaje contiene "orden de features").
- Modelo oficial real → pasa la validación sin error, `metadata["model_hash"]` presente.

## Checksum del dataset y del modelo — presentes en la metadata oficial

```
dataset_hash: cad04b23818d515aee08029496ad1d5ee54826c58ab78373b97cb5bb9b845cfe:c94fc260bcad61781ed210164b585bdb6bf5ef779015810dafe401bc27cd356c
model_hash:   e86b4cb0132e921b53b8b2e2a1c677ec1d3738c73a1626c5f9c386d1eb21ed32
```

**Nota importante**: el `model_hash` embebido en la metadata (`e86b4cb0...`) es el hash del artefacto calculado **antes** de volver a serializarlo con ese mismo hash embebido dentro (necesariamente, porque el hash no puede incluirse a sí mismo) — es decir, `model_hash` es el hash del archivo en un paso intermedio del proceso de guardado, **no coincide exactamente con el hash sha256 del archivo `.pkl` final en disco** (que es `31e9224a...`, verificado arriba). Esto es una limitación conocida de auto-referencia (calcular el hash de un archivo que contendrá su propio hash es circular por definición) — el `model_hash` de la metadata sirve para detectar si el **estimador entrenado** cambió respecto a cuando se generaron las métricas, no como un checksum de integridad del archivo completo en disco. Se documenta como una precisión necesaria, no como un defecto crítico.

## Python y scikit-learn — coinciden entre entrenamiento y entorno de carga

`metadata.python_version = "3.12.13"`, `metadata.sklearn_version = "1.9.0"` — coinciden exactamente con el entorno `backend/.venv312` usado para cargar y probar el modelo en esta sesión (`python --version` → `3.12.13`).
