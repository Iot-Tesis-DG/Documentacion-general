# Reglas térmicas — distinción regla vs. modelo

## Regla determinista base (`reglas_riesgo.py::clasificar_por_regla`)

Constantes: `DURACION_CRITICA_MINUTOS=30.0`, `DURACION_PREVENTIVA_MINUTOS=10.0`, `TENDENCIA_CRITICA=1.5`, `MARGEN_PREVENTIVO_C=1.0`. Rango base: `RANGO_TERMICO_BPA` = 2.0-8.0°C (`domain/value_objects/rango_termico.py`).

Lógica (determinista, sin IA):
1. Si la temperatura está fuera de [2,8]°C **y** (lleva ≥30min fuera **o** está a ≥2°C del límite): → `excursion_critica`.
2. Si está fuera de [2,8]°C (sin cumplir lo anterior): → `riesgo_preventivo`.
3. Si está dentro del rango pero cerca del límite (≤1°C), o lleva ≥10min cerca, o la tendencia es fuerte (≥1.5°C/lectura), o hay ≥3 desviaciones recientes: → `riesgo_preventivo`.
4. En cualquier otro caso: → `normal`.

## Distinción exacta de orígenes (evidencia de código, no inferencia)

| Concepto | Dónde vive | Es determinista o IA |
|---|---|---|
| Umbral 2-8°C | `RangoTermico` (VO de dominio) | Determinista |
| `clasificar_por_regla()` | `reglas_riesgo.py` | 100% determinista (if/else), **usada como generador de etiquetas de entrenamiento Y como salvaguarda de producción** |
| Features derivadas (duración fuera de rango, tendencia, frecuencia) | `ClasificarRiesgoTermicoUseCase._construir_features()` | Cálculo determinista (regresión lineal simple con `numpy.polyfit` para la tendencia) que ALIMENTA al modelo — no son predicciones, son insumos |
| Predicción de clase + confianza | `RandomForestRiesgoService.inferir()`, rama `modelo.predict_proba()` | Genuina inferencia de Random Forest entrenado |
| Severidad final devuelta | `RandomForestRiesgoService.inferir()` | **Híbrida**: predicción del modelo, salvo que la regla determinista indique mayor severidad, en cuyo caso prevalece la regla |
| Generación de alerta | `GenerarAlertaUseCase` | Determinista: cualquier nivel != `normal` genera alerta, sin importar si ese nivel vino del modelo o de la regla |

## Contradicciones de nomenclatura entre clases

**Ninguna encontrada.** Verificado en los tres puntos del sistema:
- Backend: `NivelRiesgo(StrEnum)` = `"normal"`, `"riesgo_preventivo"`, `"excursion_critica"`.
- Frontend: `NivelRiesgo` (TypeScript) = idéntico.
- Dataset de entrenamiento: `clasificar_por_regla()` retorna instancias del mismo enum — `modelo.classes_` en `training_metrics.json` confirma exactamente `["excursion_critica", "normal", "riesgo_preventivo"]` (orden alfabético de sklearn, mismos 3 literales).

No hay variantes con tildes, mayúsculas, guiones, ni nombres alternativos en ningún punto del sistema. **Consistencia terminológica perfecta de extremo a extremo.**

## Histéresis / ventanas temporales

Hay histéresis implícita en la lógica de "duración fuera de rango": una lectura puntual fuera de rango no escala automáticamente a crítica — necesita persistir ≥30min o alejarse ≥2°C del límite. Esto evita que un pico de ruido instantáneo dispare una alerta crítica innecesaria, sin necesidad de un mecanismo de histéresis explícito nombrado como tal — es una consecuencia del diseño de features basado en duración e historial.

## ¿Combinación de reglas e IA es coherente con lo que TI declara?

TI describe "un modelo Random Forest para clasificar el riesgo térmico" sin mencionar explícitamente una salvaguarda de reglas superpuesta. La implementación real es más sofisticada y más segura de lo mínimo declarado (una capa de seguridad adicional que TI no detalla) — esto es una mejora no documentada en el texto de la tesis, no una contradicción negativa, pero **debe explicarse con precisión en la sustentación** para que no parezca que "en realidad todo son reglas y la IA no hace nada" (falso: el modelo sí decide en la mayoría de los casos, la regla solo actúa como red de seguridad cuando discrepa hacia una severidad mayor).
