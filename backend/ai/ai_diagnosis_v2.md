# Diagnóstico metodológico exacto — corrección del hallazgo AI-01

## Los 4 chequeos exigidos

**1. ¿Existe fuga clásica train/test (duplicados exactos entre partición)?**
**No.** El dataset v1 es i.i.d. (cada fila se genera por un sorteo independiente de `numpy.random.default_rng`), sin estructura de sesión ni dispositivo. La probabilidad de duplicados exactos con variables continuas `float64` sobre 8000 filas es despreciable. `train_test_split(..., stratify=y, random_state=42)` es una partición aleatoria correcta **para este generador tal como estaba construido**.

**2. ¿Existen registros duplicados o secuencias del mismo escenario repartidas entre train y test?**
**No aplica en v1** — no existe la noción de "escenario" ni de secuencia temporal en el generador v1; cada fila es un evento aislado sin relación con las demás. **Este es precisamente el problema de fondo del diseño v1**: al no modelar secuencias reales, el generador tampoco modela la incertidumbre de estimación que esas secuencias producirían en producción.

**3. ¿Las features utilizan información futura?**
**No.** `duracion_fuera_rango`, `frecuencia_desviaciones` y `tendencia_termica` en v1 no se derivan de ninguna secuencia temporal (ni pasada ni futura) — son escalares aleatorios independientes generados directamente. No hay violación de "no mirar el futuro" porque no hay una línea de tiempo en absoluto en v1.

**4. ¿Es circularidad entre las reglas de etiquetado y las variables predictoras?**
**Sí — este es el diagnóstico correcto, confirmado.** `clasificar_por_regla()` decide la clase usando 4 señales: `temperatura_interna`, `duracion_fuera_rango`, `tendencia_termica`, `frecuencia_desviaciones`. De estas, 3 llegan al modelo **exactamente iguales** a lo que vio la regla (sin ninguna transformación ni fuente de incertidumbre), y solo 1 (`temperatura_interna`) recibe ruido de sensor antes de llegar al modelo.

## Clasificación formal del problema

> **Circularidad entre las reglas de etiquetado y las variables predictoras, que infla la evaluación al medir principalmente la capacidad del modelo para reproducir las reglas deterministas.**

No es fuga de datos en el sentido clásico (train/test, información futura, duplicados). Es un problema de **diseño del generador sintético**: 3 de 10 variables de entrada no atraviesan ningún proceso de medición/estimación con incertidumbre antes de llegar al modelo, a diferencia de la temperatura.

## Escenario aplicable

**Escenario B — Clasificación basada en reglas expertas.** No existen etiquetas independientes (no hay datos reales etiquetados por expertos, ni resultado observado posterior de una excursión real, ni severidad clínica registrada). Construir Escenario A requeriría datos de campo que el proyecto no tiene en esta etapa. Escenario C (predicción a futuro, ej. "riesgo en los próximos 10 minutos") **cambiaría el objetivo oficial de la tesis** (TI define clasificación del estado térmico actual, RF-08, no predicción prospectiva) — no se aplica sin aprobación explícita, que no se ha dado.

**Declaración académica correcta para la tesis (Escenario B)**:
> "El modelo Random Forest aprende a aproximar los criterios térmicos deterministas (BPA 2-8°C, duración, tendencia, frecuencia de desviaciones) a partir de mediciones con incertidumbre de sensor. No descubre una verdad clínica independiente; las métricas reportadas miden la concordancia del modelo con la política de clasificación térmica definida, evaluada bajo incertidumbre de estimación de sus variables de entrada. No constituyen validación clínica ni generalización a condiciones reales de campo no simuladas."

## Estrategia de partición para v2

El generador v2 (ver `ai_training_pipeline_v2.md`) introduce **escenarios temporales** (episodios de ~20-40 lecturas consecutivas simuladas, análogas a la ventana de 20 lecturas que usa `ConsultarHistorialTermicoUseCase` en producción). Esto crea dependencia real entre filas de un mismo escenario (temperaturas consecutivas correlacionadas) — exactamente el caso que el prompt advierte: *"ninguna secuencia o escenario puede quedar parcialmente en entrenamiento y parcialmente en prueba"*.

Se usa:
- **`GroupShuffleSplit`** (agrupado por `escenario_id`) para la partición train/test 80/20.
- **`StratifiedGroupKFold`** (agrupado por `escenario_id`, 5 folds) para la validación cruzada sobre el conjunto de entrenamiento.

Ambas disponibles en scikit-learn 1.9.0 (confirmado). Ningún escenario aparece simultáneamente en train y test.
