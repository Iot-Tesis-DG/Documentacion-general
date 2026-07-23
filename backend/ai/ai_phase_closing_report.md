# Etapa A — Informe de cierre de la auditoría de IA

## Respuestas a las 27 preguntas obligatorias (sección 26)

1. **¿El Random Forest ya existía?** Sí.
2. **¿Dónde está?** `src/infrastructure/ai/` (features, reglas, entrenamiento, servicio de inferencia) + `src/application/use_cases/clasificar_riesgo_termico.py`.
3. **¿Cuál es el archivo del modelo?** `src/infrastructure/ai/models/random_forest_termico.pkl` (sha256 `ca8e3efe06e7a3bc974b488bbca88e0bbd8b2b74bdfa4554991c3b651ff931ea`).
4. **¿Con qué dataset se entrenó?** Dataset 100% sintético, generado en runtime por `train_model.py::generar_dataset()`, sin archivo persistido.
5. **¿Real, sintético o mixto?** Sintético, 100%.
6. **¿Cuántos registros?** 8000 (6400 train / 1600 test).
7. **¿Cuáles son las features?** 10, ver `ai_feature_definition.md` (tabla completa con fórmula, unidad, ventana, origen).
8. **¿Cómo se crean las etiquetas?** Mediante `clasificar_por_regla()`, una regla determinista basada en 4 señales (temperatura, duración fuera de rango, tendencia, frecuencia de desviaciones) — el modelo aprende a aproximar esta regla, no una verdad clínica independiente.
9. **¿Existe fuga de datos?** **Sí, confirmada — hallazgo AI-01 CRÍTICO.** 3 de las 4 señales de la regla llegan al modelo sin ruido, idénticas a lo usado para etiquetar.
10. **¿Cómo se dividió train/test?** Aleatorio estratificado 80/20, `random_state=42` — correcto para este generador (sin estructura de grupo/temporal en los datos sintéticos).
11. **¿Qué validación cruzada?** `StratifiedKFold(5)` sobre el conjunto de entrenamiento, `scoring="f1_weighted"`.
12. **¿Qué hiperparámetros?** `n_estimators=200, max_depth=12, min_samples_leaf=3, class_weight="balanced", random_state=42`; `min_samples_split`, `max_features`, `bootstrap` en sus valores por defecto de scikit-learn (no especificados explícitamente en el código).
13. **¿Cuál es el F1 macro?** 0.9535 (0.9534579151311545).
14. **¿Cuál es el F1 weighted?** 0.9659 (0.9658930392248002) — **esta es la métrica oficial de RNF-04**.
15. **¿Cuál es la accuracy?** 0.9656 (96.5625%) — **distinta de F1 weighted, no intercambiable**.
16. **¿Recall de `excursion_critica`?** 0.9730 (97.30%).
17. **¿Se reproduce F1=0.9659?** **Sí, exactamente**, reproducido en esta sesión con el mismo entorno (Python 3.12.13, scikit-learn 1.9.0, `random_state=42`).
18. **¿Se reproduce accuracy=96.56%?** **Sí, exactamente**, junto con toda la matriz de confusión y validación cruzada (diff vacío contra el original).
19. **¿Integrado realmente en FastAPI?** Sí — pipeline completo verificado (MQTT/HTTP → clasificación → persistencia → alerta → hash → SSE).
20. **¿Se carga una sola vez?** Sí — singleton lazy thread-safe, confirmado en código.
21. **¿La predicción se persiste?** Sí, la clase (`nivel_riesgo`); la confianza y origen solo en el payload de trazabilidad, no en columna propia.
22. **¿La versión del modelo se persiste?** **No, en ningún registro de base de datos** — hallazgo AI-06.
23. **¿Las alertas usan la inferencia?** Sí, directamente (`GenerarAlertaUseCase` disparado por `nivel_riesgo`).
24. **¿La inferencia queda en la cadena SHA-256?** Sí — `confianza_ia` y `origen_clasificacion` viajan dentro del payload hasheado de cada lectura.
25. **¿Llega correctamente por SSE?** Parcialmente — la clase sí llega; confianza y versión del modelo no (hallazgo AI-07).
26. **¿Qué falta para RNF-04 implementado sin reservas?** (a) corregir la fuga de datos y reentrenar reportando el F1 corregido; (b) declarar explícitamente en TI que el dataset es sintético; (c) exponer la métrica en el frontend (ya identificado en Fase 2/3).
27. **¿Qué limitaciones deben declararse en la sustentación?** Dataset 100% sintético (no datos reales de campo); 3 de 10 features del entrenamiento no tienen ruido de estimación realista (fuga parcial); sin comparación con baseline; sin checksum de modelo/dataset; `model_version` no trazable por lectura individual.

## Veredicto de la Etapa A

**Modelo real e integrado con validación parcial** (una de las 7 categorías permitidas de la sección 27, la más precisa dado que: el modelo es genuinamente real, entrenado con metodología mayormente correcta —validación cruzada, semilla fija, gate de umbral—, e integrado de forma arquitectónicamente sólida en el pipeline completo; pero su validación tiene un defecto metodológico concreto y confirmado —fuga parcial de 3 features— que compromete la limpieza del número reportado como evidencia de RNF-04).

No es "reglas presentadas como IA" (hay inferencia estadística real, con salvaguarda determinista claramente diferenciada, no oculta). No es "simulado" ni "ausente". No es "modelo real, reproducible, evaluado e integrado" sin reservas, porque "evaluado" con una fuga de datos no es una evaluación limpia — el adjetivo correcto es "con validación parcial".

## Siguiente paso — Etapa B (no iniciada, requiere autorización explícita)

Por instrucción directa del prompt ("No reentrenes ni reemplaces el modelo hasta confirmar exactamente qué existe"), Etapa A se detiene aquí. El hallazgo AI-01 (crítico) cambia el cálculo de lo que Etapa B debería hacer: **reentrenar corrigiendo la fuga producirá un número de F1 distinto al 0.9659 actualmente citado en la tesis** — esto es una decisión que debe tomarse con el usuario antes de ejecutar, no unilateralmente, dado que afecta un resultado ya reportado en el documento oficial de tesis.

Recomendación concreta para Etapa B, pendiente de aprobación:
1. Modificar `generar_dataset()` para inyectar ruido/incertidumbre de estimación realista en `duracion_fuera_rango`, `frecuencia_desviaciones`, `tendencia_termica` (simulando ventanas de historial imperfectas, análogas al ruido ya aplicado a temperatura/humedad).
2. Reentrenar y reportar el F1 corregido, documentando ambos números (original y corregido) con transparencia total.
3. Agregar comparación con baseline (regresión logística simple).
4. Agregar `dataset_hash`/`model_hash` a la metadata.
5. Agregar validación de compatibilidad de features/clases al cargar el modelo.
6. Persistir `model_version` en `thermal_readings` (requiere migración Alembic nueva, `upgrade`+`downgrade`, sin tocar migraciones anteriores).
7. Agregar `confianza`/`model_version` al contrato SSE (`LecturaResponse`).
8. Acotar el `except Exception` genérico de `consumir_mensajes()` para distinguir fallo de inferencia de error de programación, con auditoría del descarte.
9. Redactar la sección técnica académica (sección 23 del prompt) para la tesis, con el número corregido.

**No se ejecuta nada de lo anterior sin autorización explícita del usuario**, dado el peso de los cambios 1-2 sobre un resultado ya citado en el documento oficial.
