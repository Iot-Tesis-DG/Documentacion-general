# Hallazgos — módulo de IA (Etapa A)

## AI-01 [CRÍTICO] Fuga de información en 3 de 10 features del dataset sintético

Ver `ai_leakage_analysis.md` para el análisis completo. **Archivo**: `src/infrastructure/ai/train_model.py`, líneas 89-91. **Impacto**: F1=0.9659 y accuracy=96.56% sobrestiman el desempeño real esperable; el propio gate de RNF-04 se calcula sobre datos con esta fuga. **Estado epistemológico**: Verificado en código (comparación línea por línea).

## AI-02 [ALTO] Manejo de errores de inferencia demasiado amplio en el camino MQTT

**Archivo**: `src/infrastructure/mqtt/mqtt_client.py::consumir_mensajes()`. Un error inesperado en `modelo.predict_proba()` (o en cualquier parte del pipeline) se captura con un `except Exception: continue` genérico, sin distinguir fallo de inferencia de un bug de programación, y sin auditar el descarte. **Impacto**: un fallo real del modelo en producción sería invisible (sin log estructurado, sin registro de auditoría), a diferencia de `DispositivoNoAutorizadoError` que sí se audita. **Estado**: Verificado en código.

## AI-03 [MEDIO] Sin manejo explícito de clase desconocida devuelta por el modelo

**Archivo**: `random_forest_service.py` línea 108, `NivelRiesgo(modelo.classes_[indice])` lanzaría `ValueError` no capturado si el modelo alguna vez devolviera una clase fuera de las 3 esperadas (escenario solo posible si el artefacto se corrompe o se reentrena con clases distintas). **Impacto**: excepción no controlada propagándose hasta el manejador genérico de MQTT. **Estado**: Verificado en código (inferido el escenario de fallo, no reproducido en ejecución).

## AI-04 [MEDIO] Sin validación de compatibilidad de features/clases al cargar el modelo

**Archivo**: `random_forest_service.py::_cargar_modelo()`, líneas 69-80. No compara `modelo.n_features_in_` contra `len(FEATURE_NAMES)` ni `modelo.classes_` contra los 3 valores de `NivelRiesgo` al momento de cargar. **Impacto**: si alguien reemplazara el `.pkl` por un artefacto entrenado con un esquema de features distinto, el backend cargaría "exitosamente" y fallaría de forma confusa (o silenciosamente incorrecta) en la primera inferencia real, no al arrancar. **Estado**: Verificado en código.

## AI-05 [MEDIO] Sin checksum/hash del modelo ni del dataset en la metadata

**Archivo**: `train_model.py::entrenar()`, diccionario `metadata` (líneas 139-152) — no incluye `dataset_hash` ni `model_hash`. **Impacto**: no hay forma de verificar criptográficamente que el `.pkl` cargado en producción es exactamente el que fue evaluado y reportado en `training_metrics.json` (podrían divergir silenciosamente si alguien reemplaza el archivo). **Estado**: Verificado en código (ausencia).

## AI-06 [MEDIO] `model_version` no se persiste en ningún registro de base de datos

Ver `ai_persistence_traceability.md`. **Impacto**: imposible determinar retroactivamente con qué versión del modelo se clasificó una lectura histórica si el modelo se reentrena/reemplaza. **Estado**: Verificado en código.

## AI-07 [MEDIO] Contrato SSE no incluye `confianza` ni `model_version`

Ver `ai_sse_contract.md`. **Impacto**: refuerza el hallazgo ya reportado en Fase 2/3 (RNF-04 sin superficie de UI) — aunque el backend calcula la confianza, nunca llega al cliente en tiempo real. **Estado**: Verificado en código.

## AI-08 [BAJO] Sin comparación con modelo baseline

No existe ningún experimento que demuestre cuantitativamente que Random Forest supera a una alternativa simple (regla mayoritaria, árbol único, regresión logística) sobre el mismo dataset. **Impacto**: académico, no funcional — la tesis no puede argumentar "Random Forest aporta valor frente a X" con evidencia propia. **Estado**: Verificado en código (ausencia).

## AI-09 [BAJO] Features derivadas del historial nunca se incluyen en el payload de trazabilidad

**Archivo**: `registrar_lectura_termica.py`, payload de `RegistrarHashEncadenadoUseCase.execute()` — omite `duracion_fuera_rango`, `frecuencia_desviaciones`, `tendencia_termica`, `hora_evento`, `estado_conectividad_online`. **Impacto**: menor auditabilidad retrospectiva del vector completo de entrada al modelo para una lectura específica. **Estado**: Verificado en código.

## No se encontraron (contexto positivo)

- No hay modelo inexistente presentado como real — el modelo existe, es real, se ejecuta.
- No hay dataset inventado presentado como real — es sintético y el código lo declara honestamente en su propio docstring (aunque la tesis, según lo auditado en Fase 1, no lo declara con la misma claridad).
- El modelo cargado en producción **es el mismo que fue evaluado** — no hay evidencia de artefactos duplicados o inconsistentes (aunque la ausencia de checksum, AI-05, significa que esta garantía depende de disciplina operativa, no de una verificación automática).
- El orden de features es consistente entre entrenamiento e inferencia (única función `to_array()`, ver `ai_feature_definition.md`) — no hay bug de "feature en orden incorrecto".
- Las métricas SÍ se reproducen exactamente — no son irreproducibles ni inventadas.
