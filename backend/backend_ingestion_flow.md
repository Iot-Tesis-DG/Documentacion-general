# Flujo de ingesta de lecturas — real, paso a paso

Reconstruido leyendo `interface/main.py::_procesar_mensaje_mqtt` (vía MQTT) y `interface/api/lecturas_router.py::ingestar_lectura` (vía HTTP) + `RegistrarLecturaTermicaUseCase.execute()` (compartido por ambos caminos).

```
1. RECEPCIÓN
   ├─ Vía MQTT: mensaje en farmacias/{device_id}/lecturas, deserializado con
   │  LecturaPayload.model_validate_json() (Pydantic v2, valida tipos y rangos
   │  físicos en el borde de entrada)
   └─ Vía HTTP: POST /api/lecturas, body validado por LecturaIngestRequest
      (mismo esquema Pydantic esencialmente), requiere JWT + rol técnico/farmacéutico

2. ANTI-SUPLANTACIÓN (solo camino MQTT)
   └─ device_id del payload debe coincidir con el segmento del topic;
      si no, se descarta silenciosamente (log warning, sin persistir)

3. AUTORIZACIÓN DE DISPOSITIVO (ambos caminos, RegistrarLecturaTermicaUseCase)
   ├─ Modo estricto (producción recomendada): device_id debe existir en `devices`;
   │  si no, DispositivoNoAutorizadoError → se audita "DISPOSITIVO_RECHAZADO" → descarta
   └─ Modo no estricto (desarrollo): se crea el device automáticamente (obtener_o_crear)

4. VALIDACIÓN DE RANGO FÍSICO (dominio)
   └─ LecturaTermica.es_lectura_valida(): temperatura_ambiental en [-40,125]°C,
      temperatura_interna en [-55,125]°C, humedad en [0,100]% — si falla,
      LecturaInvalidaError → se descarta (MQTT: log warning; HTTP: 422)

5. CONSTRUCCIÓN DE FEATURES (ClasificarRiesgoTermicoUseCase)
   └─ Recupera las últimas 20 lecturas del dispositivo (listar_recientes_por_device),
      calcula: diferencia_sensores, duración fuera de rango (mirando hacia atrás en
      el historial), frecuencia de desviaciones, tendencia térmica (regresión lineal
      con np.polyfit sobre las temperaturas previas), hora del evento, estado conectividad

6. CLASIFICACIÓN DE RIESGO (RandomForestRiesgoService.inferir)
   ├─ Calcula la clase por la regla determinista BPA 2-8°C (clasificar_por_regla)
   ├─ Si el modelo .pkl está disponible: predict_proba(), toma la clase de mayor
   │  probabilidad y su confianza
   ├─ SALVAGUARDA: si la regla determinista indica MÁS severidad que el modelo,
   │  prevalece la regla (origen="regla_salvaguarda", confianza=1.0) — el modelo
   │  nunca puede rebajar un riesgo que la regla detecta
   └─ Si el modelo no está disponible: fallback total a la regla (origen="regla_fallback")

7. PERSISTENCIA DE LA LECTURA
   └─ lectura.nivel_riesgo = clasificación; INSERT en thermal_readings (UUID generado,
      created_at con default Python de microsegundos para orden estable)

8. TRAZABILIDAD DE LA LECTURA (RegistrarHashEncadenadoUseCase)
   └─ Lee el último hash_actual global (SELECT ... ORDER BY created_at DESC LIMIT 1,
      SIN LOCK — ver riesgo de concurrencia en backend_hash_chain_analysis.md),
      calcula SHA-256(previous_hash + timestamp_canónico + payload_json_ordenado),
      inserta en traceability_records con tipo_evento="LECTURA_TERMICA", incluyendo
      en el payload la confianza_ia y el origen_clasificacion como evidencia auditable

9. GENERACIÓN DE ALERTA (si nivel_riesgo != normal)
   └─ GenerarAlertaUseCase: INSERT en thermal_alerts con mensaje según nivel

10. TRAZABILIDAD DE LA ALERTA (si se generó)
    └─ Segundo eslabón de hash encadenado, tipo_evento="ALERTA_TERMICA"

11. COMMIT
    └─ Una única transacción de sesión para todo el pipeline (lectura + alerta +
       2 registros de trazabilidad) — atomicidad real: si algo falla antes del
       commit, nada se persiste

12. EMISIÓN SSE (solo camino MQTT)
    └─ broadcaster.publicar(lectura_to_response(...).model_dump()) — difunde a
       todos los EventSource conectados. NOTA: el camino HTTP (POST /api/lecturas)
       NO emite el evento SSE — ver hallazgo

13. MANEJO DE ERRORES
    └─ MQTT: excepciones específicas de dominio se capturan y auditan/loguean sin
       tumbar el consumidor; excepciones no anticipadas se capturan genéricamente en
       consumir_mensajes() (continue). HTTP: excepciones de dominio se traducen a
       códigos HTTP apropiados (403, 422) vía manejo explícito en el router.
```

## Verificaciones solicitadas

- **Rango físico razonable**: sí, validado (paso 4).
- **Valores nulos**: los campos térmicos son `float | None` — una lectura con sensor fallido (`None`) pasa la validación de rango (el `if value is not None` la excluye del chequeo) y se persiste igual, con `nivel_riesgo` calculado usando `0.0` como valor por defecto en la construcción de features (`temperatura_interna = lectura.temperatura_interna or 0.0`) — **esto es un problema sutil**: una lectura con sensor caído (`None`) se trata como si tuviera 0.0°C para fines de clasificación de riesgo, lo cual **0.0°C está fuera del rango 2-8°C** y produciría una clasificación de riesgo (probablemente crítica) basada en un valor inventado, no en la ausencia real de dato. Ver hallazgo.
- **NaN/infinitos**: Pydantic v2 rechaza NaN/Infinity por defecto en campos `float` a menos que se permita explícitamente (`allow_inf_nan`) — no se vio esa opción activada, por lo que están implícitamente rechazados por el framework antes de llegar al dominio.
- **Timestamps futuros/antiguos**: **no se valida** (ver `backend_mqtt_analysis.md`).
- **Lecturas fuera de orden**: no se reordenan antes de insertar (ver `backend_mqtt_analysis.md`).
- **Duplicados por reenvío**: **no se detectan ni previenen** (sin restricción única en BD).
- **Dos sensores discrepantes**: `diferencia_sensores()` calcula la diferencia entre ambiental e interna y la pasa como feature al modelo (información, no una validación de descarte) — el sistema no rechaza lecturas por discrepancia excesiva entre sensores, solo la usa como señal para el clasificador.
- **Estado del MC-38 (apertura_refrigerador)**: se persiste y se usa como feature (`apertura_refrigerador`) — confirmado que influye en la clasificación (aunque su `feature_importance` real es casi nula: 0.0005, ver `training_metrics.json` — el modelo aprendió que este feature aporta poquísimo poder predictivo, dato honesto reportado en las métricas).
- **Estado de conectividad**: se persiste (`estado_conectividad`) y se usa como feature booleano (`estado_conectividad_online`).
- **Lecturas sin dispositivo registrado**: rechazadas en modo estricto (paso 3), auto-registradas en modo no estricto.
- **Idempotencia**: **no garantizada** — reenviar la misma lectura dos veces produce dos registros distintos con dos eslabones de hash distintos.
- **Atomicidad**: **sí garantizada** dentro de una ejecución del pipeline (una sola transacción/commit), pero no entre ejecuciones concurrentes (ver hallazgo de concurrencia de hash).

## Hallazgo: sensor caído (`None`) se trata como 0.0°C para clasificación

Ubicación: `application/use_cases/clasificar_riesgo_termico.py`, línea `temperatura_interna = lectura.temperatura_interna or 0.0`. Si `temperatura_interna` es `None` (sensor desconectado, según HU-01 Escenario 2 del backlog, que dice que estos casos "se descartan" en el firmware antes de construir el payload), pero si de alguna forma una lectura con `None` llega al backend (p. ej. vía el endpoint HTTP directo sin pasar por el firmware), el modelo clasificaría basándose en 0.0°C — un valor fuera del rango 2-8°C, generando muy probablemente una alerta de excursión crítica **falsa**, basada en la ausencia de dato, no en una lectura real. El firmware (fuera de este repo) mitigaría esto en el mejor caso, pero el backend no tiene una segunda línea de defensa explícita para "sensor caído" como un estado distinto de "0°C real".

## Emisión SSE solo en camino MQTT, no en camino HTTP

`POST /api/lecturas` (ingesta HTTP) ejecuta el mismo `RegistrarLecturaTermicaUseCase` pero el router **no invoca `broadcaster.publicar(...)`** después — solo el manejador MQTT (`_procesar_mensaje_mqtt` en `main.py`) hace la difusión SSE. Esto significa: si una lectura se ingesta vía HTTP en vez de MQTT (p. ej. en pruebas E2E, backfill administrativo, o si el firmware alguna vez usara HTTP como respaldo), **el dashboard no se actualizaría en tiempo real para esa lectura** — solo aparecería al recargar/re-consultar el historial. Es una asimetría real entre los dos caminos de ingesta.
