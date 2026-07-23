# 04 — AIV-03 resuelto: guard completo de sensores

## Cambios

`src/application/use_cases/clasificar_riesgo_termico.py::execute()` reescrito con un guard completo, en orden:

1. `temperatura_interna is None` → `fallo_sensor`, `omitida`, motivo `sensor_interno_ausente`.
2. `temperatura_interna` no finito (NaN/inf) → `fallo_sensor`, `omitida`, motivo `sensor_interno_valor_no_finito`.
3. `temperatura_ambiental` no finito → `fallo_sensor`, `omitida`, motivo `sensor_ambiental_valor_no_finito`.
4. `temperatura_ambiental is None` (sensor crítico sí válido) → busca el último valor válido en el historial (`_ultimo_valor_valido`); si existe, se usa como fallback documentado (nunca `0.0`) y la inferencia continúa; si no existe, `dato_insuficiente`, `omitida`.
5. Con ambos sensores válidos (directos o con fallback), se construyen features y se ejecuta la inferencia normalmente.

`_construir_features` ya no hace `lectura.temperatura_ambiental or 0.0` — asume que el guard ya garantizó un valor no-None y finito. `humedad_ambiental` usa un fallback neutro documentado (`50.0`, punto medio) solo cuando es `None`, nunca un valor inventado que sesgue hacia riesgo.

## Verificación — defensa en capas (no una sola)

`temperatura_interna`/`temperatura_ambiental` con NaN o infinito ya eran rechazados en una capa ANTERIOR: `LecturaTermica.es_lectura_valida()` (invocada por `RegistrarLecturaTermicaUseCase` antes de clasificar) evalúa `-55.0 <= valor <= 125.0`, que en Python es `False` para NaN/inf, por lo que la lectura completa se rechaza con `LecturaInvalidaError` antes de llegar a clasificación. Confirmado por prueba real (`test_temperatura_interna_nan_es_rechazada_como_lectura_invalida`, `test_temperatura_interna_infinito_es_rechazada_como_lectura_invalida`).

El guard de `ClasificarRiesgoTermicoUseCase` es una SEGUNDA capa de defensa en profundidad, verificada de forma aislada (sin pasar por `es_lectura_valida`) en `test_guard_de_clasificacion_bloquea_nan_e_infinito_como_defensa_en_profundidad` — confirma que aunque un valor no finito llegara por otra vía (llamada directa al caso de uso), el guard lo bloquea igual.

## Casos probados (10 pruebas nuevas, todas verdes)

| Caso | Resultado esperado | Prueba |
|---|---|---|
| `temperatura_interna=None` | `fallo_sensor` / `omitida` / `confianza_ia=None` | `test_ambos_sensores_ausentes_no_ejecuta_inferencia` (variante con ambos None) |
| `temperatura_interna=NaN` | Rechazo en `es_lectura_valida` (capa 1) | `test_temperatura_interna_nan_es_rechazada_como_lectura_invalida` |
| `temperatura_interna=inf` | Rechazo en `es_lectura_valida` (capa 1) | `test_temperatura_interna_infinito_es_rechazada_como_lectura_invalida` |
| NaN/inf directo al guard (capa 2, sin pasar por capa 1) | `fallo_sensor` / `omitida` | `test_guard_de_clasificacion_bloquea_nan_e_infinito_como_defensa_en_profundidad` |
| Ambos sensores ausentes | Sin inferencia, `confianza_ia=None` | `test_ambos_sensores_ausentes_no_ejecuta_inferencia` |
| Solo ambiental ausente, CON historial de respaldo | Inferencia completa (fallback aplicado) | `test_sensor_ambiental_ausente_con_historial_aplica_fallback` |
| `0.0 °C` real (dentro del guard, fuera de rango BPA) | Clasifica normalmente, NO se confunde con "sin dato" | `test_temperatura_real_0_grados_es_valida_y_critica` |

## Persistencia de la decisión

`registrar_lectura_termica.py` ahora persiste `lectura.confianza_ia = None` (nunca `0.0`) cuando `clasificacion.confianza is None`, más `estado_inferencia` y `motivo_no_inferencia` en la propia entidad/lectura, y en el payload de trazabilidad `LECTURA_TERMICA`.

## Estado AIV-03: **RESUELTO**
