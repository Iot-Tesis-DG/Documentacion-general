# 06 — Reproducibilidad (ejecución real, esta sesión)

## Entorno

`backend/.venv312`, Python 3.12.13, sin modificar dependencias.

## Comandos ejecutados y resultados reales

| Comando | Resultado |
|---|---|
| `python --version` | `Python 3.12.13` |
| `python -m compileall src/ -q` | Exit code 0, sin errores de sintaxis |
| `pytest --collect-only -q` | **123 tests recolectados** |
| `pytest -q` | **123 passed** en 25.91s (0 fallos, 0 omitidos) |
| `python -m src.infrastructure.ai.train_model_v2` (repetición 1) | Éxito, F1=0.9744, ver detalle abajo |
| `python -m src.infrastructure.ai.train_model_v2` (repetición 2) | Éxito, F1=0.9712, ver detalle abajo |

## Dataset: byte-idéntico entre ejecuciones (SÍ reproducible)

Verificado con `np.array_equal(x1, x2)` y `np.array_equal(y1, y2)` sobre dos llamadas independientes a `generar_dataset_v2(400, np.random.default_rng(42))`: **`True` en ambos casos** — las 10076 filas de features y etiquetas son exactamente iguales, bit a bit, en cada ejecución.

## Partición y métricas: NO bit-reproducibles entre ejecuciones (hallazgo, ver `05_training_pipeline_verification.md`)

Los `escenario_id` usan `uuid4()` no seedeada, lo que cambia la partición `GroupShuffleSplit` en cada corrida y, por tanto, las métricas de test exactas:

| Ejecución | n_train/n_test | F1 weighted | Accuracy | CV media±std |
|---|---|---:|---:|---|
| Oficial (guardada, la que carga el backend) | 7926/2150 | 0.9748 | 0.9744 | (ver `training_metrics.json`) |
| Repetición 1 | 8054/2022 | 0.9744 | 0.9738 | 0.9689±0.0068 |
| Repetición 2 | 8063/2013 | 0.9712 | 0.9707 | 0.9714±0.0029 |

**No se inventa reproducibilidad donde no la hay**: el número exacto 0.9748 es el resultado de UNA partición particular entre varias posibles con la misma semilla de generación de datos — no es reproducible bit a bit en su tercer/cuarto decimal, aunque sí lo es en orden de magnitud (0.971-0.975, siempre ≥0.85).

## Reproducción de v1 (verificado en sesión anterior, no repetido en esta sesión para no sobrescribir evidencia)

Documentado en `audit-output/backend/ai/ai_evaluation_report.md`: F1=0.9659 y accuracy=96.56% se reprodujeron **exactamente** (diff vacío en toda la matriz de confusión, CV, feature importances) porque v1 usa `train_test_split` aleatorio simple (no agrupado), cuyo `random_state=42` sí determina completamente la partición sin depender de ningún identificador de grupo no seedeado — v1 no tiene el problema de v2 precisamente porque no agrupa por escenario.

## Artefactos oficiales restaurados sin alteración

Antes de las 2 repeticiones de entrenamiento de esta verificación, se respaldaron `random_forest_termico.pkl` y `training_metrics.json`. Al finalizar, se restauraron exactamente:
```
sha256 tras restaurar: 31e9224a201631cdb0072f6c7b7c7af5b45329e0a5686c350f4d2c5097dba656
```
Coincide con el estado del artefacto oficial antes de iniciar esta verificación (no se alteró el estado del repositorio).

## Conclusión

- **Suite de pruebas: 100% reproducible** (123/123, siempre).
- **Dataset (features/etiquetas): 100% reproducible** (byte-idéntico).
- **Partición y métrica de test exacta: NO reproducible bit a bit**, por un bug real y corregible (UUID no seedeado) — impacto acotado (todas las variantes observadas superan 0.85), pero debe corregirse antes de citar una cifra decimal específica como "la" métrica oficial en la tesis.
