# 14 — Hallazgos (formato mandatado)

---
**ID**: AIV-01
**Severidad**: Media
**Archivo**: `src/infrastructure/ai/train_model_v2.py`
**Línea**: función `generar_dataset_v2`, asignación `escenario_id = uuid4().hex[:12]`
**Evidencia**: Re-ejecución independiente del generador dos veces con `random_state=42`: `x`/`y` byte-idénticos (`np.array_equal` → `True`), `grupos` (escenario_id) distintos cada corrida (`np.array_equal` → `False`). F1 reportado varía: 0.9748 (oficial guardado) / 0.9744 (rerun1) / 0.9712 (rerun2) — todos >0.85 pero no bit-reproducibles en su decimal exacto.
**Impacto**: La partición train/test (`GroupShuffleSplit`/`StratifiedGroupKFold`) depende de `escenario_id`, que usa `uuid4()` (entropía del SO), no del `rng` sembrado que se pasa explícitamente al resto del generador. Esto rompe la reproducibilidad total del pipeline de entrenamiento pese a que los datos subyacentes sí son reproducibles.
**Relación con RNF-04**: No compromete el cumplimiento del umbral F1≥0.85 (las 3 corridas lo superan cómodamente), pero sí compromete la trazabilidad científica exigida: un tercero que reproduzca el entrenamiento con la misma semilla no obtendrá exactamente el mismo modelo/partición, solo un modelo estadísticamente equivalente.
**Recomendación**: Derivar `escenario_id` determinísticamente del `rng` sembrado (p.ej. `rng.integers(...)` o un contador combinado con el índice de escenario), eliminando la dependencia de `uuid4()`.
**Estado**: Abierto, no corregido (fase de solo lectura).

---
**ID**: AIV-02
**Severidad**: Media
**Archivo**: `src/application/use_cases/generar_alerta.py`
**Línea**: `execute()`, único guard `if nivel_riesgo == NivelRiesgo.NORMAL: return None`
**Evidencia**: Lectura directa del código confirma ausencia de verificación de alerta previa abierta/no revisada para el mismo `device_id` antes de insertar una nueva.
**Impacto**: Un dispositivo que permanece en `excursion_critica` durante múltiples lecturas consecutivas genera una alerta nueva por cada lectura, sin cooldown/histéresis — riesgo real de "tormenta de alertas" que puede saturar la bandeja de revisión del farmacéutico y degradar la utilidad operativa del sistema de alertas.
**Relación con RNF-04**: No afecta directamente la exactitud del modelo, pero sí la usabilidad y confiabilidad operativa del sistema de trazabilidad/alertas que depende de esa clasificación.
**Recomendación**: Añadir una verificación de alerta abierta no revisada para el mismo `device_id` antes de crear una nueva, o un cooldown temporal configurable.
**Estado**: Abierto, no corregido (fuera del alcance declarado de las sesiones P0 e IA; confirmado nuevamente en esta verificación).

---
**ID**: AIV-03
**Severidad**: Media
**Archivo**: `src/application/use_cases/clasificar_riesgo_termico.py` (guard) + `_construir_features` (feature ambiental)
**Línea**: guard solo verifica `lectura.temperatura_interna is None`; construcción de features usa `temperatura_ambiental=lectura.temperatura_ambiental or 0.0`
**Evidencia**: Confirmado por lectura directa en `08_fastapi_integration.md` y `09_feature_parity.md`; ninguna prueba en `tests/` ejercita el caso `temperatura_ambiental=None` con `temperatura_interna` presente.
**Impacto**: Si el sensor ambiental falla pero el interno funciona, se sustituye silenciosamente `0.0°C` como si fuera un dato real, pudiendo sesgar la clasificación sin que el sistema registre que fue una sustitución artificial (`origen_clasificacion` seguiría siendo `"modelo"` o `"regla_salvaguarda"`, no `"sin_dato_sensor"`).
**Relación con RNF-04**: Reduce la confiabilidad de la trazabilidad verificable — una clasificación basada parcialmente en un dato inventado (0.0°C) se presenta con la misma etiqueta de origen que una basada en datos reales.
**Recomendación**: Extender el guard de `ClasificarRiesgoTermicoUseCase.execute()` para also cubrir `temperatura_ambiental is None`, devolviendo `ORIGEN_SIN_DATO` en ese caso también.
**Estado**: Abierto, no corregido (mismo punto ciego presente en el dataset de entrenamiento v2, documentado como consistente pero no resuelto).

---
**ID**: AIV-04
**Severidad**: Baja
**Archivo**: `src/interface/api/schemas.py`, `src/interface/api/mappers.py`
**Línea**: `LecturaResponse` (líneas 73-74), `lectura_to_response()`
**Evidencia**: Grep directo confirma `origen_clasificacion` no aparece en `schemas.py` ni en `mappers.py`, solo `confianza_ia`/`modelo_version`.
**Impacto**: Un consumidor de la API/SSE no puede distinguir "confianza 0.0 porque el modelo decidió así" de "confianza 0.0 porque no hubo inferencia (sensor caído)" sin acceso directo a la base de datos.
**Relación con RNF-04**: Reduce transparencia/auditabilidad expuesta externamente (el dato existe en BD, pero no es parte del contrato público).
**Recomendación**: Añadir `origen_clasificacion: str | None` a `LecturaResponse` y al mapper.
**Estado**: Abierto, no corregido.

---
**ID**: AIV-05
**Severidad**: Baja
**Archivo**: `frontend/src/domain/entities/LecturaTermica.ts`, `frontend/src/infrastructure/sse/sseClient.ts`
**Línea**: interfaz completa; `onmessage` línea 66 (`as LecturaTermica`)
**Evidencia**: Grep confirma 0 referencias a `confianza_ia`/`modelo_version` en componentes de UI reales (fuera de la capa demo simulada).
**Impacto**: Los campos de evidencia IA llegan por red pero no se muestran a ningún rol de usuario en tiempo real — brecha frontend/backend ya reflejada en el veredicto consolidado previo ("Parcialmente implementado").
**Relación con RNF-04**: La trazabilidad es auditable a nivel de API/BD, no a nivel de experiencia de usuario.
**Recomendación**: Extender la interfaz TS y un componente de UI (p.ej. tooltip en el dashboard) para mostrar `modelo_version`/`confianza_ia`/`origen_clasificacion`.
**Estado**: Abierto (fuera del alcance de esta sesión backend-only; ya conocido de auditorías de frontend previas).

---
**ID**: AIV-06
**Severidad**: Informativa
**Archivo**: `src/infrastructure/ai/train_model_v2.py`, metadata `model_hash`
**Línea**: cálculo de hash antes del segundo `joblib.dump`
**Evidencia**: Confirmado en `07_official_artifact.md`: `model_hash` embebido (`e86b4cb0...`) no coincide con el sha256 real del archivo `.pkl` final en disco (`31e9224a...`) por circularidad de auto-referencia inevitable.
**Impacto**: Ninguno crítico — es una limitación matemática conocida (un archivo no puede contener el hash de sí mismo incluyéndose). No compromete la distinción real v1/v2 (verificada por el hash del archivo completo, no por el campo interno).
**Relación con RNF-04**: Ninguna — el checksum de integridad real del archivo en disco (calculado externamente vía `shasum`) sigue siendo válido y fue el usado para todas las comparaciones de esta verificación.
**Recomendación**: Documentar explícitamente en la tesis que `model_hash` en metadata representa el hash del estimador antes de la re-serialización final, no un checksum de archivo completo.
**Estado**: No requiere corrección de código, solo aclaración documental.

---
**ID**: AIV-07
**Severidad**: Informativa
**Archivo**: `src/infrastructure/database/repositories/lectura_repository.py` (persistencia `confianza_ia=0.0` cuando `nivel_riesgo=None`)
**Evidencia**: Confirmado en `10_persistence_migrations.md`.
**Impacto**: Ambigüedad de interpretación entre "confianza matemática 0" y "no hubo inferencia", mitigable solo revisando `origen_clasificacion` (que a su vez no está expuesto en la API — ver AIV-04).
**Relación con RNF-04**: Menor, mitigado parcialmente por el campo `origen_clasificacion` a nivel de BD.
**Recomendación**: Persistir `confianza_ia=None` (no `0.0`) cuando `origen_clasificacion="sin_dato_sensor"`, requeriría cambiar el tipo del value object `ResultadoInferencia.confianza` a `float | None`.
**Estado**: Abierto, no corregido.
