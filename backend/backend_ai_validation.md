# Random Forest — validación exhaustiva (sección crítica)

## Clasificación del estado de la IA

**Modelo real, entrenado, evaluado con rigor estadístico e integrado — con dataset 100% sintético (no datos reales de campo) y una salvaguarda determinista que puede sobrescribir su predicción.**

Esta no es ninguna de las categorías extremas ("simulada/hardcodeada" ni "ausente"), pero tampoco es un caso puro de "modelo real evaluado con datos de producción" — es una posición intermedia legítima y bien documentada que debe explicarse con precisión en la sustentación.

## Datos

- **Fuente**: sintética, generada en runtime por `train_model.py::generar_dataset()`. No hay ningún archivo CSV/parquet con datos reales de sensores de una farmacia — el propio código lo declara explícitamente en su docstring.
- **Filas**: 8000 muestras (`N_SAMPLES = 8000`).
- **Variables**: 10 features exactas (`FEATURE_NAMES` en `features.py`): `temperatura_ambiental`, `humedad_ambiental`, `temperatura_interna`, `diferencia_sensores`, `duracion_fuera_rango`, `frecuencia_desviaciones`, `tendencia_termica`, `apertura_refrigerador`, `hora_evento`, `estado_conectividad_online`.
- **Etiquetas**: 3 clases (`excursion_critica`, `normal`, `riesgo_preventivo`), generadas por `clasificar_por_regla()` — **la etiqueta de entrenamiento proviene de una regla determinista, no de una observación clínica/real**. Esto significa que el "techo" de lo que el Random Forest puede aprender está definido por la misma regla térmica base que también existe independientemente en el sistema (`reglas_riesgo.py`).
- **Distribución de clases en el conjunto de prueba** (n=1600, 20% del dataset): `excursion_critica`=816 (51%), `riesgo_preventivo`=630 (39%), `normal`=154 (10%) — **desbalance real**, mitigado en el entrenamiento con `class_weight="balanced"`.
- **¿Datos reales, sintéticos o mixtos?**: 100% sintéticos.
- **Fuga de información (data leakage)**: el propio docstring del script aborda esto explícitamente: la etiqueta se calcula sobre las magnitudes "reales" (sin ruido) de la simulación, mientras que las FEATURES que ve el modelo llevan ruido gaussiano inyectado (`RUIDO_TEMP_INTERNA_C=0.25`, `RUIDO_TEMP_AMBIENTE_C=0.15`, `RUIDO_HUMEDAD_PCT=1.0`, calibrado según las hojas de datos de SHT31/DS18B20). Esto es una técnica correcta y deliberada para **evitar que el modelo simplemente memorice la regla exacta con precisión perfecta** — el modelo debe recuperar el estado verdadero a partir de una medición imperfecta, un problema de aprendizaje genuino (no trivial ni circular). Es una decisión metodológica sólida para un dataset sintético.
- **Variables derivadas directamente de la etiqueta**: no — las features se derivan de las magnitudes simuladas ANTES de calcular la etiqueta, no al revés. Sin fuga directa.
- **Justificación de un dataset sintético**: razonable y explícita, dado que el prototipo no cuenta (a la fecha) con meses de datos reales de producción de una farmacia. Es la práctica esperable para un prototipo académico en esta etapa, siempre que se comunique honestamente como tal — **lo cual el código sí hace, pero TI (según lo leído en Fase 1) no aclara explícitamente que el entrenamiento es 100% sintético** — riesgo de comunicación en la sustentación si el jurado asume que hay datos reales detrás.

## Entrenamiento

- `random_state = 42` fijo — reproducible.
- Hiperparámetros: `n_estimators=200`, `max_depth=12`, `min_samples_leaf=3`, `class_weight="balanced"` — no se ve una búsqueda de hiperparámetros (`GridSearchCV`/`RandomizedSearchCV`); son valores fijos razonables, no optimizados sistemáticamente. No es necesariamente un defecto para un prototipo, pero no hay evidencia de tuning exhaustivo.
- División: `train_test_split(..., test_size=0.2, stratify=y, random_state=42)` — aleatoria estratificada, no temporal (razonable dado que el dataset es sintético sin estructura temporal real que preservar).
- **Validación cruzada**: `StratifiedKFold(n_splits=5, shuffle=True, random_state=42)` sobre el conjunto de entrenamiento, con `cross_val_score(..., scoring="f1_weighted")` — genuina, no cosmética.
- Manejo de valores faltantes: el pipeline de entrenamiento no genera `NaN` (todas las variables sintéticas se generan con distribuciones completas) — no aplica manejo de missing values en el propio entrenamiento (el manejo de sensor caído ocurre en el código de inferencia de producción, ver `backend_ingestion_flow.md`, hallazgo del `or 0.0`).
- Reproducibilidad: alta — semillas fijas en generación de datos, split y modelo.

## Métricas — reales, no inventadas (`training_metrics.json`, entrenamiento real ejecutado 2026-07-11)

| Métrica | Valor |
|---|---|
| Accuracy | **0.965625** (96.56%) |
| F1 ponderado (test) | **0.9659** |
| F1 por clase | excursion_critica=0.9815, riesgo_preventivo=0.9563, normal=0.9226 |
| Precisión/Recall por clase | todas ≥0.88 (ver tabla completa en `training_metrics.json`) |
| Validación cruzada 5-fold (f1_weighted) | media=0.9615, std=0.0031 (muy estable) |
| Matriz de confusión | sin confusiones entre `excursion_critica` y `normal` en ningún sentido (0 y 0) — solo confusión con la clase intermedia `riesgo_preventivo`, comportamiento clínicamente razonable |
| **Umbral RNF-04 (F1≥0.85)** | **Cumplido: 0.9659 ≥ 0.85** — y el propio script de entrenamiento (`train_model.py`) **falla intencionalmente con `SystemExit`** si no se cumple, impidiendo guardar un modelo por debajo del umbral. Esto es una garantía real de proceso, no solo una afirmación posterior. |

**El F1≥0.85 de RNF-04 está demostrado con evidencia real, reproducible y con un mecanismo de build que impide activamente publicar un modelo que no lo cumpla.**

## El overclaim "Exactitud >96%" de la diapositiva 25 — investigación específica solicitada

**Hallazgo clave**: el `training_metrics.json` real reporta `accuracy: 0.965625` (96.56%), un número que **efectivamente respalda** una cifra ">96%" — pero con una precisión cronológica importante:

- `trained_at` del artefacto real: **2026-07-11T11:39:03 UTC**.
- Fecha de la presentación PF_ (diapositiva 25, según Fase 1): **16 de mayo de 2026**.
- **La presentación es casi dos meses ANTERIOR al entrenamiento real que hoy sustenta esa cifra.** No es posible que la cifra de la diapositiva se haya derivado de esta ejecución de `train_model.py`, porque esa ejecución no existía todavía.

Dos explicaciones posibles, ninguna verificable con certeza desde este repositorio:
1. El equipo entrenó una versión anterior del modelo (no versionada / sobrescrita) que ya arrojaba un resultado similar, y luego reentrenó (versión 2.0.0, según `metadata.model_version` en el artefacto actual) obteniendo un número parecido por coincidencia del mismo enfoque metodológico.
2. La cifra de la diapositiva fue una proyección/expectativa (quizás inspirada en el artículo de Quispe-Astorga et al., que reporta 96% en un dominio distinto, según lo detectado en Fase 1) que **coincidió después** con el resultado real, sin que en mayo hubiera evidencia que la sustentara.

**Conclusión de este hallazgo**: hoy, con este repositorio, el número ">96%" **sí tiene sustento real y propio** (no de un artículo externo). Pero el equipo debe estar preparado para explicar en sustentación que la cifra fue afirmada meses antes de tener el entrenamiento real que hoy la respalda — el jurado puede preguntar "¿cómo sabían esto en mayo?" y la respuesta honesta es "no lo sabían con esta evidencia; el número resultó coincidir tras el entrenamiento real posterior". Esto no invalida el resultado actual, pero sí es un riesgo de comunicación en la defensa si no se aclara proactivamente.

## Integración

- **Dónde se carga**: `RandomForestRiesgoService._cargar_modelo()`, carga perezosa (lazy, thread-safe con `Lock`) desde `models/random_forest_termico.pkl` — no se carga en el arranque del backend (`lifespan`), solo al primer uso real.
- **Cuándo se ejecuta**: en cada lectura ingresada (MQTT o HTTP) vía `ClasificarRiesgoTermicoUseCase`, y bajo demanda vía `POST /api/ia/clasificar`.
- **Orden de features**: `FeaturesRiesgoTermico.to_array()` serializa en el orden exacto de `FEATURE_NAMES` — coincide con el orden usado en `train_model.py::generar_dataset()`. Verificado consistente.
- **Validación de esquema de entrada**: `ClasificacionRequest` (Pydantic, en `ia_router.py`) valida rangos físicos para el endpoint de prueba manual; el pipeline de producción construye el vector internamente sin validación Pydantic adicional (ya validado en el paso de ingesta).
- **Versión del modelo**: embebida en el artefacto (`metadata.model_version = "2.0.0"`), expuesta vía `GET /api/ia/modelo`.
- **Fallback**: si el `.pkl` no existe, degrada a la regla determinista (`origen="regla_fallback"`, confianza=1.0) — el backend **no falla al arrancar** por falta de modelo, decisión de resiliencia documentada.
- **Confianza/probabilidad**: se calcula y se persiste en el payload de trazabilidad (`confianza_ia`) — es real (`predict_proba()[argmax]`), no un valor fijo, excepto cuando el origen es una regla (donde se fija a 1.0 por diseño, ya que una regla determinista no tiene "probabilidad").
- **Generación de alertas**: depende del resultado de `inferir()` después de aplicar la salvaguarda — es decir, las alertas pueden originarse tanto de una predicción genuina del Random Forest como de la regla de seguridad que lo sobrescribe. **Ambos casos quedan documentados con el campo `origen_clasificacion` en la trazabilidad**, lo cual permite, en teoría, auditar cuántas decisiones fueron IA pura vs regla — dato valioso no explotado en ninguna pantalla de frontend.

## Distinción exigida: regla determinista vs. predicción del modelo

Confirmado con evidencia de código que el sistema **si distingue internamente** estos dos orígenes (`ORIGEN_MODELO = "modelo"`, `ORIGEN_REGLA_FALLBACK`, `ORIGEN_REGLA_SALVAGUARDA"`), y que la salvaguarda determinista existe precisamente para que la IA nunca pueda ocultar un riesgo que la regla básica sí detecta. Esto es una arquitectura de "IA supervisada por reglas de seguridad" — un patrón defendible y explicable ante un jurado, siempre que se comunique con precisión (no es "solo Random Forest", ni "solo reglas" — es ambos, con jerarquía explícita a favor de la seguridad).

## Persistencia de las predicciones

Sí: `nivel_riesgo` se persiste en `thermal_readings`, y `confianza_ia`/`origen_clasificacion` se persisten en el payload JSON de `traceability_records` — auditable, no efímero.
