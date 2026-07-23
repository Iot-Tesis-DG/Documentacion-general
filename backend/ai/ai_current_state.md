# Estado actual del Random Forest — respuesta a las 10 preguntas de la sección 3

1. **¿Existe código de entrenamiento?** Sí — `src/infrastructure/ai/train_model.py`, función `entrenar()` (líneas 103-175).
2. **¿Existe un modelo entrenado?** Sí — `RandomForestClassifier` real de scikit-learn, `n_estimators=200, max_depth=12` (líneas 110-117 de `train_model.py`).
3. **¿Existe un artefacto serializado?** Sí — `src/infrastructure/ai/models/random_forest_termico.pkl`, formato `joblib.dump({"modelo":..., "metadata":...})` (línea 155).
4. **¿El backend carga el artefacto?** Sí — `RandomForestRiesgoService._cargar_modelo()` (`random_forest_service.py` líneas 69-80), carga perezosa una sola vez (singleton vía `get_random_forest_service()`, líneas 119-126), no `joblib.load` por request.
5. **¿El modelo se ejecuta cuando llega una lectura?** Sí — `ClasificarRiesgoTermicoUseCase.execute()` (`clasificar_riesgo_termico.py` línea 84), invocado desde `RegistrarLecturaTermicaUseCase.execute()` en el pipeline MQTT/HTTP.
6. **¿Las predicciones se persisten?** Parcialmente — `nivel_riesgo` se guarda en `thermal_readings.nivel_riesgo` (columna propia); `confianza_ia` y `origen_clasificacion` solo se guardan dentro del payload JSON de `traceability_records`, no en columnas propias de `thermal_readings`. `model_version` **no se persiste en ningún lugar de la base de datos** (ver hallazgo AI-06).
7. **¿Se generan alertas usando la salida del modelo?** Sí — `GenerarAlertaUseCase`, disparado cuando `nivel_riesgo not in (None, NORMAL)` (`registrar_lectura_termica.py`).
8. **¿El resultado llega al frontend mediante SSE?** Sí, la clase (`nivel_riesgo`) sí llega vía `LecturaResponse` en el evento SSE; **la confianza y la versión del modelo NO llegan** (`schemas.py::LecturaResponse` solo declara `nivel_riesgo`, sin `confianza`/`model_version` — ver hallazgo AI-07).
9. **¿Existe versionado del modelo?** Parcial — `metadata.model_version = "2.0.0"` embebido en el artefacto, pero **sin hash/checksum del artefacto ni del dataset** que permita verificar que el modelo cargado en producción es exactamente el que fue evaluado (ver hallazgo AI-05).
10. **¿Las métricas pueden reproducirse?** Sí, con matices — reproducidas en esta sesión (ver `ai_evaluation_report.md`), pero las métricas **están infladas por fuga de datos parcial** en el dataset sintético (ver `ai_leakage_analysis.md`, hallazgo AI-01 CRÍTICO). El número reportado (F1=0.9659) es reproducible, pero no representa fielmente el desempeño esperado en producción.

## Clasificación del estado

**Modelo real e integrado, pero con validación insuficiente — metodología de generación de dataset cuestionable por fuga parcial de features hacia la etiqueta.**

No es "modelo real, reproducible, evaluado e integrado" sin reservas (la reproducibilidad es técnica pero la validez de la métrica está comprometida). No es "reglas presentadas como IA" (hay inferencia estadística real que decide en la mayoría de casos, no solo el fallback). No es "simulado" ni "ausente". El matiz correcto es: existe una IA real, correctamente integrada en la arquitectura (carga única, casos de uso, persistencia parcial, alertas, hash, SSE parcial), pero el **número que la tesis reporta como evidencia de RNF-04 (F1=0.9659) no es una medida limpia del desempeño real del modelo**, porque 3 de sus 10 features de entrada llegan sin ruido y son las mismas que determinan la etiqueta.
