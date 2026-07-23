# 09 — Paridad de features entrenamiento vs inferencia

## Verificación central: ¿usan la MISMA función?

**Sí, literalmente la misma, no una réplica.** `train_model_v2.py` importa y ejecuta directamente `ClasificarRiesgoTermicoUseCase._construir_features()` (instanciando el caso de uso con `ai_service=None`, ya que ese método no toca `self._ai_service`). Esto es la garantía más fuerte posible de paridad: no hay dos implementaciones que puedan divergir con el tiempo, hay una sola función invocada desde ambos contextos.

## Tabla de paridad (10 features)

| Feature entrenamiento | Feature inferencia | Misma fórmula | Mismo tipo | Misma unidad | Riesgo |
|---|---|---|---|---|---|
| `temperatura_ambiental` | `temperatura_ambiental` | Sí (mismo código) | float | °C | Ninguno |
| `humedad_ambiental` | `humedad_ambiental` | Sí | float | %HR | Ninguno |
| `temperatura_interna` | `temperatura_interna` | Sí | float | °C | Ninguno |
| `diferencia_sensores` | `diferencia_sensores` | Sí (`LecturaTermica.diferencia_sensores()`) | float | °C | Ninguno |
| `duracion_fuera_rango` | `duracion_fuera_rango` | Sí (misma función, sobre historial observado en train / historial real en producción) | float | minutos | Ninguno — ambos casos usan datos con incertidumbre real |
| `frecuencia_desviaciones` | `frecuencia_desviaciones` | Sí | float | conteo | Ninguno |
| `tendencia_termica` | `tendencia_termica` | Sí (`np.polyfit` grado 1) | float | °C/lectura | Ninguno |
| `apertura_refrigerador` | `apertura_refrigerador` | Sí | bool→float | — | Ninguno |
| `hora_evento` | `hora_evento` | Sí (`timestamp.hour`) | int | hora [0-23] | Ninguno |
| `estado_conectividad_online` | `estado_conectividad_online` | Sí (`== "online"`) | bool→float | — | Ninguno |

**Orden**: idéntico en ambos caminos, definido una sola vez en `FEATURE_NAMES` (`features.py`) y serializado por `FeaturesRiesgoTermico.to_array()` — el mismo método, sin reordenamiento manual en ningún punto.

## Ventanas temporales

Producción: `listar_recientes_por_device(device_id, limite=20)` — hasta 20 lecturas recientes reales de PostgreSQL. Entrenamiento v2: episodios de 15-35 ticks simulados, tamaño de ventana comparable (no idéntico pero del mismo orden de magnitud).

## Tratamiento de `None`

Verificado en `08_fastapi_integration.md`: **`temperatura_interna=None` se intercepta antes de construir features (P0)**. **`temperatura_ambiental=None` NO se intercepta** — se sustituye silenciosamente por `0.0` dentro de `_construir_features` (línea `temperatura_ambiental=lectura.temperatura_ambiental or 0.0`). El dataset de entrenamiento v2 **nunca genera `None` para `temperatura_ambiental`** (siempre se simula con ruido, nunca ausente) — es decir, el modelo nunca vio este caso en entrenamiento y en producción se le alimentaría un `0.0` artificial sin que ningún test lo cubra. Ver hallazgo en `14_findings.md`.

## UTC y timestamps

`_a_utc()` normaliza cualquier datetime naive a UTC antes de operar — usado consistentemente en el cálculo de `duracion_fuera_rango`/`tendencia_termica` tanto en el caso de uso real como en el generador v2 (que también construye objetos `LecturaTermica` con `timestamp` timezone-aware, `datetime.now(tz=timezone.utc)` como base).

## Conclusión

**Un modelo bien entrenado sería inválido si producción construyera features de forma diferente — verificado que NO es el caso aquí.** La única discrepancia real de tratamiento de datos faltantes es la ya señalada (`temperatura_ambiental=None` no interceptado), que es un caso no cubierto por el dataset de entrenamiento tampoco (paridad "consistente en su punto ciego", no una divergencia activa entre train e inferencia).
