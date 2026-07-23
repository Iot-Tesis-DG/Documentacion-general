# Integración con FastAPI — flujo real

## Carga del modelo — singleton confirmado, no por-request

`get_random_forest_service()` (`random_forest_service.py` líneas 122-126): patrón singleton módulo-global, `_cargar_modelo()` usa `Lock()` + doble check (líneas 69-80) para carga perezosa thread-safe una sola vez. **No hay `joblib.load()` repetido por request** — confirmado, cumple la restricción del prompt.

## Flujo real reconstruido (MQTT → SSE)

```
MQTT (farmacias/{device_id}/lecturas)
  → LecturaPayload.model_validate_json() [validación Pydantic, rangos físicos]
  → anti-suplantación device_id vs topic
  → RegistrarLecturaTermicaUseCase.execute():
      1. _autorizar_dispositivo()          [mínimo privilegio, RNF-05]
      2. es_lectura_valida()                [rango físico]
      3. obtener_por_device_y_timestamp()   [dedup, corrección P0]
      4. listar_recientes_por_device(20)    [historial real para features]
      5. ClasificarRiesgoTermicoUseCase.execute():
           - guard sensor None → ORIGEN_SIN_DATO (corrección P0)
           - _construir_features()          [única función, ver ai_feature_definition.md]
           - RandomForestRiesgoService.inferir():
               a. clasificar_por_regla()     [regla determinista, siempre calculada]
               b. modelo.predict_proba()     [Random Forest real]
               c. salvaguarda: máx(regla, modelo) por severidad
      6. lectura.nivel_riesgo = clasificacion.nivel  [None si sin dato]
      7. lectura_repository.agregar()        [persiste, incluye UNIQUE constraint P0]
      8. RegistrarHashEncadenadoUseCase.execute()  [trazabilidad, incluye confianza_ia/origen_clasificacion]
      9. si nivel_riesgo not in (None, NORMAL): GenerarAlertaUseCase + hash de la alerta
  → broadcaster.publicar(lectura_to_response(...))  [SSE, SOLO camino MQTT — ver hallazgo previo B-06]
```

## Comportamiento ante fallos — verificado

| Escenario | Comportamiento real | Evidencia |
|---|---|---|
| Modelo no disponible (.pkl ausente) | Fallback a regla determinista, `origen="regla_fallback"`, `confianza=1.0` | `random_forest_service.py` líneas 103-104 |
| Features insuficientes (sensor `None`) | Corte temprano ANTES de construir features, `origen="sin_dato_sensor"`, `nivel=None`, sin alerta | `clasificar_riesgo_termico.py` líneas 81-82 (corrección P0) |
| Error de inferencia inesperado (excepción en `predict_proba`) | **No capturado explícitamente** — una excepción no anticipada en `modelo.predict_proba()` se propagaría hacia arriba y (en el camino MQTT) sería atrapada por el `except Exception: continue` genérico de `consumir_mensajes()` en `mqtt_client.py`, descartando el mensaje completo silenciosamente sin persistir la lectura ni auditar el fallo | Ver hallazgo AI-02 |
| Clase devuelta desconocida (el modelo predice algo fuera de las 3 clases) | `NivelRiesgo(modelo.classes_[indice])` lanzaría `ValueError` si el string no coincide con el enum — no hay manejo explícito de este caso | Ver hallazgo AI-03 |
| Probabilidades no suman 1 | No verificado explícitamente (se confía en `predict_proba()` de sklearn, que garantiza esto matemáticamente por construcción del estimador) — riesgo teórico, no práctico | — |
| Modelo y backend esperan features distintas (drift de esquema) | **No hay validación al cargar** que compare `modelo.n_features_in_` contra `len(FEATURE_NAMES)`, ni que compare `modelo.classes_` contra los 3 valores de `NivelRiesgo` | Ver hallazgo AI-04 |

## Una inferencia fallida no debe romper silenciosamente toda la ingesta

Parcialmente cumplido: sí existe una capa de protección genérica (`except Exception: continue` en `consumir_mensajes`), pero es **demasiado amplia** — captura y descarta silenciosamente CUALQUIER error (incluyendo bugs de programación no relacionados con el modelo), sin distinguir "falla esperada de inferencia" de "error de programación", y sin auditar el descarte (a diferencia de `DispositivoNoAutorizadoError`, que sí genera un registro de auditoría `DISPOSITIVO_RECHAZADO`). Ver hallazgo AI-02.
