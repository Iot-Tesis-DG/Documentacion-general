# 15 — Veredicto final de re-verificación independiente

## Respuestas a las 10 afirmaciones que no debían asumirse como ciertas

1. **¿Existe realmente un módulo Random Forest, no un simulacro?** Sí — confirmado por hash sha256 del artefacto cargado, código de entrenamiento real (`train_model_v2.py`), y predicción real de `sklearn.ensemble.RandomForestClassifier` (`.predict_proba`).
2. **¿Existe un modelo v2 distinto de v1?** Sí, confirmado por hash: v1 `ca8e3efe...` (F1=0.9659), v2/oficial `31e9224a...` (F1=0.9748) — archivos físicamente distintos, no un renombrado.
3. **¿Carga singleton?** Sí, patrón lazy-loading confirmado en `random_forest_service.py` vía `get_random_forest_service()`.
4. **¿Persiste `model_version`/`confianza_ia`?** Sí, confirmado en ORM, migración 0003 aplicada contra PostgreSQL real, y prueba de integración pasando.
5. **¿Existen checksums de dataset y modelo?** Sí (`dataset_hash`, `model_hash` en metadata), con la salvedad documentada en AIV-06 (`model_hash` no es el hash final del archivo por circularidad de auto-referencia).
6. **¿Migración 0003 aplica limpiamente en PostgreSQL real?** Sí, re-ejecutada en esta sesión contra una base nueva y aislada, confirmado por `\d thermal_readings`.
7. **¿Existen escenarios temporales en el dataset?** Sí, 400 escenarios de 15-35 ticks secuenciales con regímenes térmicos distintos.
8. **¿Partición por grupo (no aleatoria simple)?** Sí, `GroupShuffleSplit`/`StratifiedGroupKFold` agrupados por `escenario_id`, con assertion explícita de disjunción train/test — **pero la reproducibilidad de esa partición está rota** (ver AIV-01).
9. **¿La clasificación es aproximación de regla, no IA genuina?** No — confirmado que existe una regla determinista de salvaguarda que puede prevalecer sobre el modelo, pero el modelo Random Forest real ejecuta `predict_proba` y su salida prevalece cuando la regla no la sobrescribe; ambos caminos (`origen_clasificacion="modelo"` vs `"regla_salvaguarda"`) son auditables y distintos, no un disfraz de if/else como IA.
10. **¿123 pruebas pasan realmente?** Sí, re-ejecutado fresco en esta sesión: `123 passed, 2414 warnings in 25.90s`. **Pero pasar no implica cobertura de todos los hallazgos abiertos** (ver `13_tests_execution.md`).

## Corrección P0 — ¿siguen intactas?

Sí. `registrar_hash_encadenado.py` (candado de proceso) y `trazabilidad_repository.py` (`pg_advisory_xact_lock`) no aparecen en la lista de archivos tocados por la serie de sesiones de IA (`01_inventory.md`), y las pruebas de concurrencia/dedup/sensor-`None` de P0 siguen pasando en la ejecución fresca de esta sesión.

## Respuestas a las 30 preguntas obligatorias (agrupadas temáticamente, sin omitir ninguna)

**Arquitectura/dataset (1-5)**: Modularización DDD correcta y confirmada por grep fresco; dataset 10076 muestras/400 escenarios, sin nulos/duplicados; features documentadas y con paridad exacta train/inferencia (una sola función compartida); no hay features de información futura (`_construir_features` usa solo historial pasado respecto al timestamp evaluado); ventanas temporales de orden de magnitud comparable (20 lecturas reales vs 15-35 ticks simulados).

**Etiquetado/circularidad (6-9)**: v1 tenía circularidad real (3 features pasadas sin ruido desde "verdad" a "medido"); v2 la corrige con ruido físicamente justificado (precisión de sensor, pérdida de mensajes MQTT) y reutilización literal del constructor de features de producción; la clasificación por Escenario B (circularidad, no leakage clásico) fue la interpretación correcta, ya documentada en sesión previa; no hay leakage clásico train/test (partición por grupo con assertion de disjunción).

**Entrenamiento/reproducibilidad (10-14)**: hiperparámetros documentados y estables; `x`/`y` son reproducibles bit a bit con semilla fija; **la partición NO es reproducible** por `uuid4()` no sembrado (AIV-01); las métricas varían en el tercer/cuarto decimal entre corridas pero siempre superan F1≥0.85; no se infló ni manipuló el dataset para forzar el umbral (confirmado por diseño del generador, sin post-procesamiento de métricas).

**Artefacto oficial/integración (15-19)**: el archivo que carga FastAPI es v2, confirmado por hash, no por nombre; existe validación de compatibilidad de artefacto que rechaza modelos incompatibles con `RuntimeError`, probada con 4 casos reales; el pipeline MQTT→clasificación→persistencia→hash→alerta→SSE fue reconstruido y verificado paso a paso; el camino HTTP no emite SSE (asimetría pre-existente, no introducida por esta serie de sesiones).

**Persistencia/trazabilidad/alertas (20-24)**: columnas IA persisten correctamente en PostgreSQL real; modificar `nivel_riesgo` invalidaría la cadena de hash (mecanismo intacto); un duplicado MQTT no genera nueva entrada, alerta ni hash (confirmado por prueba real); **no hay deduplicación de alertas — tormenta de alertas es un riesgo real y confirmado (AIV-02)**; el payload de trazabilidad de `ALERTA_TERMICA` no incluye `modelo_version`/`confianza_ia` (solo el de `LECTURA_TERMICA` los incluye).

**Contrato SSE/frontend (25-27)**: el backend expone `confianza_ia`/`modelo_version` en el schema; el frontend NO los tipa ni consume en ningún componente de UI real; `origen_clasificacion` no llega ni siquiera al schema Pydantic (AIV-04), agravando la ambigüedad de `confianza_ia=0.0`.

**Pruebas/documentación (28-30)**: 123/123 pasan en ejecución fresca de esta sesión; existen gaps de cobertura reales y explícitos (SSE end-to-end, tormenta de alertas, reproducibilidad de entrenamiento, migraciones contra PostgreSQL real dentro de pytest, asimetría HTTP/SSE) — ninguno de estos gaps está oculto por el resultado verde del suite; la documentación académica (tesis/presentación) **no fue modificada en ninguna sesión de esta serie**, tal como se instruyó explícitamente.

## Hallazgos críticos abiertos

**Ninguno de severidad Crítica.** Los 7 hallazgos de `14_findings.md` son de severidad Media (3), Baja (2) e Informativa (2). Ninguno compromete la integridad de la cadena de hash, la autenticidad del modelo Random Forest, ni introduce datos falsificados o afirmaciones no sustentadas.

## Veredicto

**Modelo de IA real e integrado técnicamente, con hallazgos metodológicos y de cobertura abiertos, ninguno de severidad crítica.**

La circularidad de etiquetado de v1 fue corregida genuinamente en v2 mediante una metodología defendible (partición por grupo, ruido físicamente justificado, reutilización literal del constructor de features de producción, comparación contra líneas base). Sin embargo, persisten brechas reales que impiden un veredicto plenamente favorable: la partición de entrenamiento no es bit-reproducible (AIV-01), no existe protección contra tormentas de alertas (AIV-02), el guard de sensor ausente cubre solo `temperatura_interna` y no `temperatura_ambiental` (AIV-03), y el contrato público de la API omite `origen_clasificacion`, dato necesario para desambiguar `confianza_ia=0.0` (AIV-04/AIV-07). Estas brechas son consistentes con el veredicto consolidado ya fijado en fases anteriores de esta auditoría ("Parcialmente implementado") y no lo contradicen — el trabajo de la serie de sesiones de IA es real, verificable y mejora sustancialmente el estado anterior, pero no cierra por completo la distancia entre el alcance técnico implementado y el alcance oficialmente declarado de la tesis.

No se modificó código, no se hicieron commits, push, despliegues, ni conexiones a Railway o EMQX de producción durante esta verificación.
