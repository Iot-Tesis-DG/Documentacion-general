# 08 — Integración FastAPI de extremo a extremo (reconstruida, verificada por lectura)

```
MQTT (farmacias/{device_id}/lecturas)
  → LecturaPayload.model_validate_json()          [main.py::_procesar_mensaje_mqtt] — Pydantic, rangos físicos
  → anti-suplantación device_id vs topic            [main.py] — descarta si no coincide, sin persistir
  → RegistrarLecturaTermicaUseCase.execute()        [registrar_lectura_termica.py]
      1. _autorizar_dispositivo()                    → DispositivoNoAutorizadoError si estricto y no registrado
      2. es_lectura_valida()                         → LecturaInvalidaError si rango físico imposible
      3. obtener_por_device_y_timestamp()            → dedup: si existe, retorna sin re-procesar (P0)
      4. listar_recientes_por_device(20)             → historial real para features
      5. ClasificarRiesgoTermicoUseCase.execute()    [clasificar_riesgo_termico.py]
           - guard: temperatura_interna is None → ORIGEN_SIN_DATO, nivel=None (P0)
           - _construir_features()                    → único constructor de features
           - RandomForestRiesgoService.inferir()      [random_forest_service.py]
               a. clasificar_por_regla()               siempre calculada
               b. modelo.predict_proba()                Random Forest real (v2 oficial)
               c. salvaguarda: severidad(regla) > severidad(modelo) → prevalece regla
      6. lectura.nivel_riesgo / modelo_version / confianza_ia / origen_clasificacion = ...
      7. lectura_repository.agregar()                 → INSERT (UNIQUE constraint P0 a nivel de BD también)
      8. RegistrarHashEncadenadoUseCase.execute()     → hash SHA-256 encadenado (candado de proceso P0)
      9. si nivel_riesgo not in (None, NORMAL): GenerarAlertaUseCase + segundo hash
  → broadcaster.publicar(lectura_to_response(...))     → SSE, SOLO camino MQTT (no camino HTTP)
```

## Verificaciones puntuales de la Fase 10 (evidencia, no inferencia)

| Verificación | Resultado | Evidencia |
|---|---|---|
| Una sola inferencia por lectura | **Confirmado** | El guard de deduplicación (paso 3) ocurre **antes** de `ClasificarRiesgoTermicoUseCase.execute()` (paso 5) — un duplicado nunca llega a clasificarse dos veces |
| Reenvío MQTT no produce otra inferencia | **Confirmado por prueba real** | `test_reenvio_mqtt_con_mismo_device_y_timestamp_no_duplica` (pasa) |
| Duplicado no genera otra alerta | **Confirmado** | El `return existente` del guard de dedup ocurre antes del paso 9 (generación de alerta) |
| Duplicado no genera otro hash | **Confirmado** | Ídem, antes del paso 8 |
| Duplicado no genera otro evento SSE lógico | **Confirmado por diseño** | El `broadcaster.publicar()` solo se alcanza si el pipeline completo se ejecutó hasta el final; un duplicado retorna en el paso 3 y nunca llega a esa línea |
| Sensor `None` no llega al modelo | **Confirmado por prueba real** | `test_sensor_temperatura_interna_none_no_se_convierte_en_cero` (pasa) — el guard en `ClasificarRiesgoTermicoUseCase.execute()` corta ANTES de `_construir_features()` |
| Ambos sensores ausentes bloquean inferencia | **Confirmado** | El guard verifica específicamente `lectura.temperatura_interna is None` (el sensor crítico para la regla); si `temperatura_ambiental` también es `None`, `_construir_features` lo sustituye por `0.0` vía `or 0.0` — **esto NO está completamente corregido para el sensor ambiental**, ver hallazgo en `14_findings.md` |
| `0.0°C` real sigue siendo válido | **Confirmado** | El guard verifica `is None`, no `== 0.0` — una lectura real de 0.0°C (fuera del rango 2-8°C, correctamente clasificable como excursión) pasa el guard y se clasifica normalmente, sin confundirse con "sensor caído" |
| Excepción del modelo no se ignora silenciosamente | **Parcialmente corregido** | `mqtt_client.py::consumir_mensajes` ahora usa `logger.exception(...)` antes del `continue` (corrección de esta serie de sesiones) — ya no es 100% silencioso, pero sigue sin distinguir "fallo esperado" de "bug de programación" con acciones distintas |

## Transacción y atomicidad

Toda la secuencia (persistir lectura + hash de lectura + alerta + hash de alerta) ocurre dentro de la **misma sesión SQLAlchemy** pasada al caso de uso — un solo `session.commit()` la ejecuta el router/handler MQTT después de que `execute()` retorna, confirmando atomicidad real (si algo falla antes del commit, nada se persiste).

## Camino HTTP (`POST /api/lecturas`) vs camino MQTT

Ambos ejecutan el mismo `RegistrarLecturaTermicaUseCase`, pero **solo el camino MQTT emite el evento SSE** (`lecturas_router.py` no llama a `broadcaster.publicar(...)` tras persistir) — asimetría ya documentada en auditorías previas (hallazgo B-06 de Fase 3), no corregida en esta serie de sesiones porque no formaba parte del alcance de IA.
