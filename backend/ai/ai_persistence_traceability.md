# Persistencia y trazabilidad de la inferencia

## Qué se persiste, dónde, y qué falta

| Dato | ¿Se persiste? | Dónde | Observación |
|---|---|---|---|
| ID de lectura | Sí | `thermal_readings.id` (UUID PK) | — |
| Clase (`nivel_riesgo`) | Sí | `thermal_readings.nivel_riesgo` (columna propia) | `None` si sensor sin dato (corrección P0) |
| Probabilidad por clase | **No** | — | Solo se persiste la confianza de la clase elegida, no el vector completo de `predict_proba()` |
| Confianza máxima | Sí, pero solo en trazabilidad | `traceability_records.payload.confianza_ia` (JSON) | No hay columna propia en `thermal_readings` |
| Versión del modelo | **No en ningún lugar de la BD** | — | `metadata.model_version` vive solo en el artefacto `.pkl`/`training_metrics.json`, nunca se copia a un registro de BD por inferencia — imposible determinar retroactivamente con qué versión del modelo se clasificó una lectura antigua si el modelo se reentrena después |
| Hash del modelo | **No existe (ni la metadata lo tiene)** | — | Ver `ai_inventory.md` |
| Features utilizadas / snapshot auditable | Parcial | El payload de trazabilidad incluye `temperatura_ambiental`, `humedad_ambiental`, `temperatura_interna`, `apertura_refrigerador` — pero NO incluye `duracion_fuera_rango`, `frecuencia_desviaciones`, `tendencia_termica`, `hora_evento`, `estado_conectividad_online` (las 3 features derivadas del historial, justamente las del hallazgo de fuga, no quedan trazadas) | `registrar_lectura_termica.py`, payload de `RegistrarHashEncadenadoUseCase.execute()` |
| Timestamp de inferencia | Implícito (`created_at` de `traceability_records`) | — | No hay un timestamp separado "cuándo se ejecutó la inferencia" vs "cuándo se generó la lectura" — se asume simultáneo, razonable |
| Estado de inferencia (ejecutada/fallback/sin-dato) | Sí | `traceability_records.payload.origen_clasificacion` | Valores: `modelo`, `regla_fallback`, `regla_salvaguarda`, `sin_dato_sensor` (corrección P0) |
| Motivo cuando no se ejecuta | Sí | `origen_clasificacion="sin_dato_sensor"` | — |

## Vínculo con la cadena SHA-256

Confirmado (`registrar_lectura_termica.py`): cada lectura genera un eslabón `tipo_evento="LECTURA_TERMICA"` cuyo payload incluye `nivel_riesgo`, `confianza_ia`, `origen_clasificacion` — **la clasificación queda dentro del contenido hasheado**, por lo que **alterar retroactivamente `nivel_riesgo` en `thermal_readings` sin recalcular la cadena completa sería detectado** por `VerificarIntegridadRegistroUseCase` (RF-15), ya que el hash almacenado no coincidiría con el que se recalcula a partir del payload original. Esto cumple la exigencia del prompt: *"una predicción no pueda cambiarse sin invalidar la cadena"*.

## Reenvío MQTT y trazabilidad — corrección P0 confirmada coherente con IA

La deduplicación (`obtener_por_device_y_timestamp`, corrección P0) ocurre **antes** de ejecutar la clasificación y antes de generar el eslabón de hash — un reenvío exacto retorna la lectura ya existente sin volver a clasificar, sin generar una segunda alerta, y sin un segundo eslabón de trazabilidad. Confirmado en `registrar_lectura_termica.py::execute()`, líneas del guard de idempotencia (antes de `historial = await ...`).

## Versión del modelo — trazable solo indirectamente

Aunque `model_version` no se persiste en cada registro, si el modelo nunca se reentrena en producción entre lecturas, todas comparten implícitamente la misma versión (la del artefacto cargado en el proceso). El riesgo aparece únicamente si el modelo se reemplaza en caliente (el prompt exige poder recargarlo) — en ese escenario, **no hay forma de saber, mirando una lectura antigua, con qué versión del modelo fue clasificada**. Ver hallazgo AI-06.
