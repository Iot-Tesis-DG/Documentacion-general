# 07 — AIV-04 resuelto: API/SSE exponen procedencia y estado real

## Cambios en `schemas.py`/`mappers.py`

`LecturaResponse` ahora incluye (además de `confianza_ia`/`modelo_version`, ya presentes desde la sesión de IA anterior):
- `origen_clasificacion: str | None` — antes solo existía en BD, invisible en la API (hallazgo AIV-04 original).
- `estado_inferencia: str | None` — nuevo (AIV-07).
- `motivo_no_inferencia: str | None` — nuevo (AIV-07), texto corto sin trazas de pila ni secretos (verificado: los valores posibles son códigos como `sensor_interno_ausente`, `modelo_no_cargado_en_este_entorno`, nunca una excepción serializada — con la única excepción defensiva de `excepcion_en_inferencia:{TipoExcepcion}` que solo incluye el NOMBRE de la clase de excepción, no el mensaje ni el traceback).

`AlertaResponse` ahora incluye evidencia de episodio (AIV-02): `episodio_abierto`, `lectura_inicial_id`, `lectura_mas_reciente_id`, `ultima_actualizacion`, `cerrada_en`.

## Compatibilidad con clientes anteriores

Todos los campos nuevos son opcionales con default (`None`/`True` según corresponda) en los schemas Pydantic — un cliente que ignore los campos nuevos sigue funcionando exactamente igual que antes (aditivo, no rompe contratos existentes). Confirmado: los tests de contrato HTTP existentes (`test_lecturas_api.py`, `test_alertas_api.py`) siguen pasando sin modificación.

## Verificación de valores reales en la respuesta

Prueba de integración actualizada `test_ai_persistencia_version.py` confirma en una respuesta HTTP real:
```json
{
  "nivel_riesgo": "normal",
  "confianza_ia": 0.9x,
  "modelo_version": "3.0.0-reproducible",
  "origen_clasificacion": "random_forest",
  "estado_inferencia": "completada",
  "motivo_no_inferencia": null
}
```
y, para sensor ausente:
```json
{
  "nivel_riesgo": null,
  "confianza_ia": null,
  "origen_clasificacion": "fallo_sensor",
  "estado_inferencia": "omitida",
  "motivo_no_inferencia": "sensor_interno_ausente"
}
```

## SSE

El evento SSE usa el mismo `LecturaResponse` (vía `lectura_to_response()`), por lo que los campos nuevos llegan automáticamente al mismo payload que consume `sseClient.ts` — sin cambios adicionales de infraestructura SSE necesarios. Confirmado por lectura de `main.py`/`sse_router.py` (sin cambios en esta fase): el broadcaster serializa el mismo `LecturaResponse.model_dump()`.

## Reenvío MQTT — sigue sin generar segundo evento lógico

Sin cambios en la lógica de deduplicación (P0, intacta) — un reenvío retorna en el guard de `obtener_por_device_y_timestamp` antes de llegar a clasificación, persistencia, hash o `broadcaster.publicar()`. Confirmado por prueba existente (`test_reenvio_mqtt_con_mismo_device_y_timestamp_no_duplica`, sigue verde).

## Estado AIV-04: **RESUELTO**

Actualización de cierre: `estado_sensores` se añadió al mismo contrato con
`valido`, `ausente`, `invalido` o `fisicamente_imposible`. Además se publica
`model_version` como alias aditivo de `modelo_version`; ambos entregan misma
versión durante transición de clientes.
