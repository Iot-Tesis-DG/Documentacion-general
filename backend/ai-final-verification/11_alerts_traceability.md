# 11 — Alertas y trazabilidad

## Alertas — verificado en `generar_alerta.py`

- Genera alerta cuando `nivel_riesgo != NORMAL` (incluye `riesgo_preventivo` y `excursion_critica`).
- Severidad: mensaje fijo por nivel (`MENSAJES_POR_RIESGO`), sin escalado dinámico.
- Vínculo con lectura: `reading_id` (FK a `thermal_readings`).
- Vínculo con predicción: la alerta se genera a partir de `lectura_guardada.nivel_riesgo`, que ya incluye el resultado de IA+regla+salvaguarda combinados — pero **la alerta en sí NO almacena `modelo_version` ni `confianza_ia`** (esos campos viven en `thermal_readings`, no en `thermal_alerts`) — se puede reconstruir vía JOIN con `reading_id`, pero no está denormalizado en la propia alerta.

## HALLAZGO — sin deduplicación/cooldown/histéresis a nivel de alerta (confirmado, no corregido en ninguna sesión previa)

`GenerarAlertaUseCase.execute()` **no verifica si ya existe una alerta abierta/no revisada para el mismo dispositivo** antes de crear una nueva. Si un dispositivo permanece en `excursion_critica` durante múltiples lecturas consecutivas (cada una con timestamp distinto, por lo que la deduplicación P0 por `(device_id, timestamp)` no aplica aquí — son lecturas genuinamente distintas, no reenvíos), **cada lectura crítica genera una alerta nueva**, sin cooldown ni consolidación. Esto es una "tormenta de alertas" real y confirmada por lectura directa del código, **no corregida por P0 ni por la serie de sesiones de IA** (estaba fuera del alcance declarado de ambas). Ver severidad en `14_findings.md`.

- **Recuperación al rango normal**: no genera ningún evento/alerta de "recuperación" — solo se registra la ausencia de nuevas alertas cuando `nivel_riesgo` vuelve a `NORMAL`. No hay cierre explícito de la alerta anterior vinculado a este evento (el cierre de alertas, según `alertas_router.py` ya auditado en Fase 3, depende de una acción manual del farmacéutico vía `PATCH /api/alertas/{id}/revisar`, no de un mecanismo automático).

## Distinción salida de IA / regla / alerta / fallo de sensor — confirmada clara

| Concepto | Dónde vive | Valor posible |
|---|---|---|
| Salida de IA | `origen_clasificacion="modelo"` | Random Forest decidió sin que la regla lo sobrescribiera |
| Regla operativa (salvaguarda) | `origen_clasificacion="regla_salvaguarda"` | La regla determinista prevaleció sobre el modelo |
| Regla operativa (fallback) | `origen_clasificacion="regla_fallback"` | No hay modelo cargado |
| Fallo/ausencia de sensor | `origen_clasificacion="sin_dato_sensor"` | `temperatura_interna is None` |
| Alerta | Tabla `thermal_alerts`, independiente de `origen_clasificacion` | Se genera igual sin importar si el `nivel_riesgo` vino del modelo o de la regla |

## Trazabilidad SHA-256 — vínculo con la inferencia (verificado, payload real)

Payload de `LECTURA_TERMICA` (líneas 85-96 de `registrar_lectura_termica.py`): incluye `device_id`, temperaturas, `apertura_refrigerador`, `nivel_riesgo` (con `None` explícito si aplica), `confianza_ia`, `origen_clasificacion`, `modelo_version` (añadido en esta serie de sesiones). Payload de `ALERTA_TERMICA`: incluye `reading_id`, `device_id`, `nivel_riesgo`, `mensaje` — **no incluye `modelo_version` ni `confianza_ia`** (inconsistencia menor respecto al payload de lectura).

## Garantías de integridad — re-confirmadas, no solo recordadas

- Modificar `nivel_riesgo` en `thermal_readings` sin recalcular la cadena sería detectado por `VerificarIntegridadRegistroUseCase` (el payload hasheado incluye `nivel_riesgo`) — **sin cambios en esta sesión**, mecanismo intacto.
- Corrección P0 de concurrencia (candado de proceso `_CandadoDeProceso` + `pg_advisory_xact_lock`): **archivo `registrar_hash_encadenado.py` no aparece en la lista de archivos tocados por la serie de sesiones de IA** (confirmado en `01_inventory.md`) — **intacto**.
- Deduplicación P0 (`UniqueConstraint`): **re-verificada contra PostgreSQL real en esta misma sesión** (`10_persistence_migrations.md`) — presente y funcional.
