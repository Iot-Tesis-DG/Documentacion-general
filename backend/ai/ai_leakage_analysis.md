# Fuga de información — hallazgo CRÍTICO confirmado

## Resumen

**3 de las 10 features que recibe el modelo (`duracion_fuera_rango`, `frecuencia_desviaciones`, `tendencia_termica`) se pasan al vector de entrenamiento SIN RUIDO, con el valor EXACTO que también usa la regla determinista para generar la etiqueta.** Esto infla artificialmente las métricas reportadas (F1=0.9659, accuracy=96.56%) porque el modelo no necesita "aprender a recuperar el estado verdadero desde una medición imperfecta" para estas 3 variables — las recibe limpias, siendo ellas mismas 3 de los 4 criterios que la regla `clasificar_por_regla()` usa para decidir la clase.

## Evidencia exacta (archivo, línea, código literal)

`src/infrastructure/ai/train_model.py`, función `generar_dataset()`:

```python
# líneas 57-59 — variables "reales" generadas una sola vez
duracion_fuera_rango = float(np.clip(rng.exponential(8.0), 0.0, 180.0))
frecuencia_desviaciones = float(rng.poisson(1.2))
tendencia_termica = float(rng.normal(0.0, 1.2))
...
# líneas 64-75 — "reales": usadas para la ETIQUETA
reales = FeaturesRiesgoTermico(
    ...
    duracion_fuera_rango=duracion_fuera_rango,
    frecuencia_desviaciones=frecuencia_desviaciones,
    tendencia_termica=tendencia_termica,
    ...
)
etiqueta = clasificar_por_regla(reales)   # <-- LA ETIQUETA sale de aquí
...
# líneas 84-95 — "medidas": lo que el modelo VE como feature de entrenamiento
medidas = FeaturesRiesgoTermico(
    temperatura_ambiental=temp_ambiente_medida,   # con ruido
    humedad_ambiental=humedad_medida,              # con ruido
    temperatura_interna=temp_interna_medida,        # con ruido
    diferencia_sensores=temp_ambiente_medida - temp_interna_medida,
    duracion_fuera_rango=duracion_fuera_rango,      # <-- MISMA VARIABLE, SIN RUIDO
    frecuencia_desviaciones=frecuencia_desviaciones, # <-- MISMA VARIABLE, SIN RUIDO
    tendencia_termica=tendencia_termica,             # <-- MISMA VARIABLE, SIN RUIDO
    ...
)
```

`src/infrastructure/ai/reglas_riesgo.py::clasificar_por_regla()` (líneas 11-35) usa exactamente 4 señales para decidir la clase: `temperatura_interna`, `duracion_fuera_rango`, `distancia_al_limite(temp)` (derivada de `temperatura_interna`), `tendencia_termica`, `frecuencia_desviaciones`. De estas, solo `temperatura_interna` recibe una versión "medida" distinta (con ruido) de la que generó la etiqueta. Las otras 3 son **idénticas bit a bit** entre la etiqueta y el feature de entrada.

## Por qué esto es fuga y no solo "features informativas"

Fuga de datos no es lo mismo que "una feature es predictiva de la etiqueta" (eso es deseable). Fuga es cuando el modelo recibe **una copia sin ruido de una de las variables de entrada de la función que generó la etiqueta** — es decir, el modelo puede aprender casi literalmente "si `duracion_fuera_rango >= 30` entonces `excursion_critica`", que es exactamente una de las condiciones de `clasificar_por_regla()` (línea 18: `features.duracion_fuera_rango >= DURACION_CRITICA_MINUTOS`). El modelo no está "recuperando el estado verdadero desde una medición imperfecta" para esas 3 variables — está memorizando umbrales que puede leer directamente porque nunca se les aplicó ruido de estimación.

Esto contradice directamente el docstring del propio archivo (`train_model.py` líneas 5-9): *"...y las métricas reportadas no son circulares"* — esa afirmación es **cierta solo para la dimensión de temperatura**, y **falsa para las 3 variables derivadas** (`duracion_fuera_rango`, `frecuencia_desviaciones`, `tendencia_termica`).

## Por qué esto SÍ importa en producción (no es solo un tecnicismo académico)

En producción real (`ClasificarRiesgoTermicoUseCase._construir_features()`), estas 3 variables **se calculan a partir de lecturas históricas reales con sus propias imprecisiones** (temperatura ruidosa del sensor, historial incompleto si el dispositivo estuvo offline, ventanas de solo 20 lecturas recientes) — es decir, en producción SÍ llegan con incertidumbre real. El entrenamiento nunca expuso al modelo a esa incertidumbre para estas 3 variables, así que el modelo **nunca aprendió a ser robusto ante duraciones/tendencias/frecuencias mal estimadas**. El F1=0.9659 mide un escenario más fácil que el que el modelo enfrentará realmente.

## Cuantificación del impacto (no cuantificada en el entrenamiento original, se recomienda para Etapa B)

No se puede saber sin reentrenar cuánto caerá el F1 real al inyectar ruido/incertidumbre realista en estas 3 variables. Es plausible que siga por encima de 0.85 (RNF-04), dado que la variable con mayor `feature_importance` reportada es `temperatura_interna` (0.522, según `training_metrics.json`), que sí tiene ruido correctamente aplicado — pero no se puede afirmar el número exacto sin ejecutar el experimento corregido.

## Fuga por partición (evaluada, no encontrada)

No aplica `GroupKFold`/separación por dispositivo/sesión/día porque el dataset sintético no tiene estructura de grupo (cada fila es i.i.d., ver `ai_dataset_analysis.md`). El `train_test_split` aleatorio estratificado es metodológicamente correcto **para este generador de datos tal como está construido** — el problema no es la partición, es el contenido de las features en sí.

## Clasificación del hallazgo

- **ID**: AI-01
- **Severidad**: CRÍTICO
- **Archivo**: `src/infrastructure/ai/train_model.py`, líneas 84-95 (específicamente 89, 90, 91)
- **Impacto**: las métricas reportadas (F1 ponderado 0.9659, accuracy 96.56%) sobrestiman el desempeño real esperable; RNF-04 (F1≥0.85) queda formalmente "cumplido" pero con una medición metodológicamente comprometida en 3 de 10 dimensiones de entrada
- **Relación con RNF-04**: directa — el propio umbral que el script usa para decidir si guarda o descarta el modelo (`F1_MINIMO_RNF04 = 0.85`, línea 133-137) se calcula sobre datos con esta fuga
- **Recomendación**: en Etapa B, inyectar ruido/incertidumbre de estimación realista también en `duracion_fuera_rango`, `frecuencia_desviaciones` y `tendencia_termica` (por ejemplo, simulando ventanas de historial incompletas o con jitter), reentrenar, y reportar el F1 corregido como el número válido para la tesis — documentando ambos (el original y el corregido) con transparencia
- **Estado epistemológico**: Verificado en código (lectura directa y comparación línea por línea de `reales` vs `medidas` en `generar_dataset()`)
