# 04 — Etiquetado y clasificación precisa de fuga/circularidad

## Cómo se generan las etiquetas (verificado en `train_model_v2.py`, función `_generar_episodio`)

Para cada tick, se construye una `LecturaTermica` con la temperatura **verdadera** (sin ruido) y se calcula `clasificar_por_regla(features_verdaderas)` — donde `features_verdaderas` se deriva con la misma función de producción (`_construir_features`) aplicada sobre el **historial verdadero** (sin ruido, sin gaps). La etiqueta proviene, por tanto, de una **regla térmica determinista**, no de anotación humana, no de expertos, no de eventos futuros, no de datos reales observados.

## Clasificación exacta del fenómeno (los 6 chequeos exigidos)

1. **¿Fuga train/test clásica (duplicados exactos entre partición)?** No — verificado: 0 duplicados de fila en todo el dataset (`03_dataset_verification.md`), y la partición es agrupada por escenario (sin solapamiento, verificado independientemente).
2. **¿Fuga temporal?** No — las features de cada tick solo usan el historial **hasta ese tick** (`historial_verdadero`/`historial_observado` se construyen incrementalmente, `append` ocurre DESPUÉS de calcular las features de ese tick, confirmado en el código: líneas de `_generar_episodio` agregan la lectura al historial al final del bucle, después de haber calculado `features_verdaderas`/`features_observadas`).
3. **¿Uso de información futura?** No — mismo argumento que el punto 2, ninguna feature de un tick t usa datos de ticks > t.
4. **¿Duplicados entre train/test?** No — 0 duplicados exactos en todo el dataset (partición por grupo adicionalmente lo refuerza).
5. **¿Circularidad entre reglas de etiquetado y features?** **Corregida respecto a v1.** En v1, 3 de 10 features llegaban al modelo idénticas a lo usado para la etiqueta. En v2, la etiqueta usa el historial **verdadero** (sin ruido, sin gaps) y las features del modelo usan el historial **observado** (con ruido de sensor + gaps por mensajes perdidos) — son **series de datos distintas**, calculadas con la misma fórmula pero sobre datos de entrada diferentes. No hay ninguna variable que llegue al modelo siendo bit-idéntica a lo que vio la regla.
6. **¿Ausencia total de fuga?** No exactamente — persiste una forma más sutil y **inevitable dado el Escenario B** (ver abajo): la etiqueta sigue derivándose de una regla determinista sobre la MISMA estructura subyacente (temperatura real de un régimen elegido deliberadamente separable) que también determina, indirectamente, cómo se ve la serie observada. Esto no es fuga de datos en sentido técnico (no hay copia de valores), es la naturaleza del Escenario B.

## Interpretación correcta (exigida, no "fuga" genérica)

> "El modelo aprende a aproximar una política de clasificación térmica basada en reglas expertas; sus métricas reflejan concordancia con dichas reglas y no validación clínica independiente."

**Esta limitación SÍ está documentada** en `audit-output/backend/ai/ai_diagnosis_v2.md` (sesión anterior) y en la metadata del artefacto oficial (`training_metrics.json::metadata.correccion_respecto_v1`, que explica textualmente la corrección aplicada). **No está documentada aún en el propio código fuente como un comentario de advertencia visible en `train_model_v2.py` para quien lea solo el código sin consultar `audit-output/`** — verificado por lectura del archivo: el docstring del módulo sí explica la corrección técnica, pero no repite explícitamente la frase de interpretación académica exacta. Se recomienda añadirla como comentario en el propio código fuente para que sobreviva independientemente de la documentación externa.

## Separación por grupos — correcta

`GroupShuffleSplit(test_size=0.2, random_state=42)` + `StratifiedGroupKFold(n_splits=5)`, ambos agrupados por `escenario_id`. Verificado en `03_dataset_verification.md` que la intersección de escenarios train/test es el conjunto vacío.

## Conclusión de esta fase

**No hay fuga clásica, ni temporal, ni de información futura, ni de partición.** La circularidad de v1 (variables copiadas sin ruido) está corregida. La limitación residual (Escenario B: el modelo aproxima una política de reglas, no una verdad clínica) es inherente a no contar con datos reales, está parcialmente documentada, y no invalida el prototipo — pero debe explicitarse mejor en el propio código, no solo en los informes de auditoría.
