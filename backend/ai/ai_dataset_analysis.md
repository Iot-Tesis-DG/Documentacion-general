# Auditoría del dataset

## Origen — 100% sintético, generado en runtime

No hay archivo de datos persistido. `train_model.py::generar_dataset(n_samples=8000, rng)` (líneas 48-100) genera todo en memoria con `numpy.random.default_rng(random_state=42)`. **No hay una sola fila real de una farmacia o dispositivo físico.** Esto debe declararse explícitamente en la tesis (aún no lo está, según lo auditado en Fase 3).

## Tabla de variables

| Variable | Tipo | Unidad | Origen (generación sintética) | Rango | Valores faltantes | Uso en modelo |
|---|---|---|---|---|---|---|
| `temperatura_interna` | float | °C | `rng.uniform(-5.0, 15.0)` (magnitud real) + ruido gaussiano σ=0.25°C (medida) | [-5, 15] real; medida puede exceder levemente | 0% (siempre generado) | Feature (medida, con ruido) y base de la etiqueta (real, sin ruido) |
| `temperatura_ambiental` | float | °C | `temperatura_interna + rng.normal(2.0, 1.5)` (real) + ruido σ=0.15°C (medida) | derivado | 0% | Feature |
| `humedad_ambiental` | float | %HR | `rng.normal(55.0, 15.0)` clip [0,100] (real) + ruido σ=1.0%HR (medida) | [0,100] | 0% | Feature |
| `duracion_fuera_rango` | float | minutos | `rng.exponential(8.0)` clip [0,180] | [0,180] | 0% | Feature — **sin ruido, valor idéntico usado en la etiqueta y en el vector de entrada (ver fuga)** |
| `frecuencia_desviaciones` | float | conteo | `rng.poisson(1.2)` | ≥0 | 0% | Feature — **sin ruido, idéntico a la etiqueta (ver fuga)** |
| `tendencia_termica` | float | °C/lectura | `rng.normal(0.0, 1.2)` | sin acotar | 0% | Feature — **sin ruido, idéntico a la etiqueta (ver fuga)** |
| `apertura_refrigerador` | bool | — | `rng.random() < 0.15` | {0,1} | 0% | Feature, sin ruido (booleano, no aplica ruido de sensor) |
| `hora_evento` | int | hora | `rng.integers(0, 24)` | [0,23] | 0% | Feature |
| `estado_conectividad_online` | bool | — | `rng.random() < 0.92` | {0,1} | 0% | Feature |
| `diferencia_sensores` | float | °C | Calculada: `temperatura_ambiental - temperatura_interna` (con las versiones medidas/ruidosas) | derivado | 0% | Feature |

## Tamaño y partición

- **8000 muestras** totales (`N_SAMPLES = 8000`).
- División 80/20 (`train_test_split(..., test_size=0.2, stratify=y, random_state=42)`) → 6400 train, 1600 test.
- **Distribución de clases (test, n=1600)**: `excursion_critica`=816 (51%), `riesgo_preventivo`=630 (39%), `normal`=154 (10%) — desbalanceado, mitigado con `class_weight="balanced"` en el entrenamiento.

## Dispositivos, sesiones, farmacias representadas

**Ninguno.** Cada una de las 8000 filas es una muestra i.i.d. (independiente e idénticamente distribuida) generada por separado — no hay `device_id`, no hay agrupación por sesión térmica, no hay estructura temporal secuencial entre filas. Esto tiene una implicación importante para la sección de fuga por partición (ver `ai_leakage_analysis.md`): **no aplica `GroupKFold`/separación por dispositivo/sesión porque no existe la noción de grupo en los datos sintéticos** — cada fila es completamente independiente de las demás por construcción. Esto es correcto para este generador, pero significa que el dataset **no modela la dependencia temporal real** que existiría en lecturas MQTT consecutivas del mismo dispositivo (una limitación a declarar, no un error de partición).

## Valores faltantes, duplicados, outliers

- **Valores faltantes**: 0% — el generador siempre produce las 10 variables completas; nunca simula un sensor caído (`None`) dentro del dataset de entrenamiento. Esto es una limitación real: el modelo entrenado **nunca vio un vector de entrenamiento con datos faltantes**, aunque en producción (post-corrección P0) sí puede recibir lecturas parciales — pero esas ya se interceptan ANTES de llegar al modelo (`ClasificarRiesgoTermicoUseCase.execute()` corta el camino si `temperatura_interna is None`), así que el modelo en sí nunca necesita manejar `None` en producción tampoco. Consistente, no es una brecha explotable, pero sí una limitación de cobertura del dataset a declarar.
- **Duplicados**: no verificado exhaustivamente por hash de fila, pero estadísticamente improbable con distribuciones continuas de alta precisión (`float64`) sobre 8000 muestras — riesgo de duplicados exactos es despreciable.
- **Outliers**: `np.clip` aplicado a humedad (0-100) y duración (0-180); temperatura interna limitada a [-5,15] por diseño del `rng.uniform`, sin outliers extremos posibles fuera de ese rango sintético.

## Declaración académica requerida (aún ausente en TI según Fase 1/3)

La tesis debe declarar explícitamente: *"El modelo Random Forest fue entrenado y evaluado exclusivamente sobre un dataset sintético de 8000 muestras generado por simulación estadística, no sobre datos reales de sensores de la farmacia validada. Las métricas reportadas demuestran el desempeño del clasificador sobre este dataset simulado, no una validación clínica ni una generalización a condiciones reales de campo."*
