# 03 — Verificación del dataset (oficial, v2)

## Identificación (verificada fresca, no asumida)

- Ruta: generado en runtime por `src/infrastructure/ai/train_model_v2.py::generar_dataset_v2()` — **no hay archivo de datos persistido** (CSV/parquet).
- `n_samples`: **10076** (train=7926, test=2150), confirmado tanto en `training_metrics.json` como recalculando el generador de forma independiente.
- `n_escenarios`: **400** (= número de dispositivos simulados; cada escenario usa un `escenario_id` único como `device_id`).
- `dataset_hash`: `cad04b23818d515aee08029496ad1d5ee54826c58ab78373b97cb5bb9b845cfe:c94fc260bcad61781ed210164b585bdb6bf5ef779015810dafe401bc27cd356c` (concatenación de hash de `x` + hash de `y`).
- `random_state`: 42.
- Valores nulos: **0** (`np.isnan(x).sum() == 0`, recalculado).
- Infinitos: **0**.
- Duplicados exactos de fila: **0**.
- Distribución de clases (dataset completo): `normal`=6898 (68.5%), `riesgo_preventivo`=1689 (16.8%), `excursion_critica`=1489 (14.8%).

## Origen: 100% sintético

Confirmado por lectura de `train_model_v2.py` — no hay ninguna fuente de datos reales, todo se genera con `numpy.random.default_rng(42)`. El docstring del propio archivo lo declara explícitamente.

## Procedimiento de generación (verificado)

Por cada uno de los 400 escenarios: se elige un régimen (`estable` 45%, `deriva_preventiva` 25%, `excursion_critica` 30%), se simula una trayectoria de temperatura verdadera de 15-35 ticks (caminata con deriva según régimen), y para cada tick se generan dos lecturas paralelas (`LecturaTermica` verdadera y observada). La observada añade ruido de sensor (SHT31/DS18B20, mismos valores físicos que v1: σ=0.15-0.25°C) y tiene 8% de probabilidad de "mensaje perdido" (simulando reenvío MQTT fallido), lo cual excluye ese tick del historial observado usado para calcular las features derivadas.

## Tabla de features (10, verificadas contra `features.py` y `_construir_features`)

| Feature | Tipo | Unidad | Origen | Fórmula | Disponible en producción |
|---|---|---|---|---|---|
| `temperatura_ambiental` | float | °C | Sensor SHT31 (simulado con ruido σ=0.15°C) | lectura directa | Sí |
| `humedad_ambiental` | float | %HR | Sensor SHT31 (simulado con ruido σ=1.0%HR) | lectura directa | Sí |
| `temperatura_interna` | float | °C | Sensor DS18B20 (simulado con ruido σ=0.25°C) | lectura directa | Sí |
| `diferencia_sensores` | float | °C | Calculada | `temperatura_ambiental - temperatura_interna` | Sí |
| `duracion_fuera_rango` | float | minutos | Calculada del historial **observado** (con gaps por mensajes perdidos) | escaneo hacia atrás hasta encontrar lectura dentro de rango | Sí — misma función que producción |
| `frecuencia_desviaciones` | float | conteo | Calculada del historial observado | conteo de lecturas fuera de [2,8]°C en ventana reciente | Sí |
| `tendencia_termica` | float | °C/lectura | Calculada del historial observado | pendiente `np.polyfit` grado 1 | Sí |
| `apertura_refrigerador` | bool→float | — | Simulada (MC-38), prob. mayor en régimen crítico | — | Sí |
| `hora_evento` | int | hora | Derivada del timestamp sintético | `timestamp.hour` | Sí |
| `estado_conectividad_online` | bool→float | — | Simulada (offline si mensaje perdido) | — | Sí |

**Ninguna feature requiere datos que el dispositivo/backend no puedan producir realmente** — las 10 son exactamente las que `ClasificarRiesgoTermicoUseCase._construir_features()` calcula en producción, porque el generador v2 reutiliza literalmente esa función (no una reimplementación paralela).

## Verificación de partición por grupo (re-ejecutada de forma independiente)

```
escenarios train: 320   escenarios test: 80
intersección (debe ser vacío): set()
soporte por clase en test: {'riesgo_preventivo': 308, 'excursion_critica': 389, 'normal': 1258}
```

**Confirmado**: cero escenarios compartidos entre train/test, las 3 clases tienen soporte real en el conjunto de prueba (ninguna clase con 0 muestras).
