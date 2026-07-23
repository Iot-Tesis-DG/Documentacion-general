# 02 — Verificación de arquitectura del módulo IA

## Estructura real (no forzada a coincidir con el ejemplo del prompt)

```
backend/src/
├── domain/entities/lectura_termica.py       ← campos modelo_version/confianza_ia, CERO import de sklearn
├── application/use_cases/
│   ├── clasificar_riesgo_termico.py          ← ÚNICO lugar que construye features de producción
│   └── registrar_lectura_termica.py          ← orquesta el pipeline, persiste evidencia de IA
├── infrastructure/ai/
│   ├── features.py                            ← FEATURE_NAMES, único orden de verdad
│   ├── reglas_riesgo.py                       ← regla determinista (etiquetado + salvaguarda)
│   ├── train_model.py                         ← script offline v1 (histórico, no se ejecuta en producción)
│   ├── train_model_v2.py                      ← script offline v2 (oficial), reutiliza _construir_features real
│   ├── random_forest_service.py               ← singleton lazy, validación de compatibilidad, inferencia
│   └── models/*.pkl, *.joblib, *.json         ← artefactos versionados
└── interface/api/
    ├── ia_router.py                            ← endpoints protegidos, NO entrena, NO construye features de negocio
    └── lecturas_router.py                      ← invoca el caso de uso, NO construye features, NO carga el modelo
```

No existe una carpeta `ml/` separada del backend ni un microservicio — el Random Forest permanece un módulo interno de `infrastructure/ai/`, correctamente dentro del monolito.

## Verificaciones puntuales (evidencia fresca, no asumida)

| Verificación exigida | Resultado | Evidencia |
|---|---|---|
| Los routers no entrenan | **Confirmado** | `grep` de `RandomForestClassifier\|\.fit(` en `src/interface/` → 0 resultados |
| Los routers no construyen features manualmente | **Confirmado** | `ia_router.py` solo deserializa un `ClasificacionRequest` (Pydantic) 1:1 al dataclass `FeaturesRiesgoTermico` para el endpoint de prueba manual — no aplica lógica de negocio, solo mapeo de campos |
| El dominio no depende de scikit-learn | **Confirmado** | `grep -rn "sklearn" src/domain/` → 0 resultados |
| FastAPI no carga el `.pkl` por request | **Confirmado** | `grep -rn "joblib.load" src/` → una única ocurrencia, dentro de `random_forest_service.py::_cargar_modelo()`, protegida por `Lock()` + doble check |
| Única responsabilidad para construir features | **Confirmado** | `_construir_features` definida en **un solo archivo** (`clasificar_riesgo_termico.py`); `train_model_v2.py` la **reutiliza literalmente** (`_extractor._construir_features(...)`), no la reimplementa |
| Entrenamiento e inferencia usan el mismo orden de features | **Confirmado** | Mismo dataclass `FeaturesRiesgoTermico.to_array()` en ambos caminos |
| El modelo se carga una sola vez | **Confirmado** | `get_random_forest_service()` (singleton módulo-global) es el único punto de acceso desde `main.py` (camino MQTT), `lecturas_router.py` (camino HTTP) e `ia_router.py` (endpoints de prueba/metadata) — cero instanciación directa de `RandomForestRiesgoService()` en código de producción |
| El servicio de IA puede probarse de forma aislada | **Confirmado** | `tests/unit/test_random_forest_service.py` y `test_random_forest_service_validacion.py` instancian `RandomForestRiesgoService()` directamente sin FastAPI, sin BD, sin MQTT |
| Sin lógica duplicada | **Confirmado** | Ver "única responsabilidad" arriba; `train_model.py` (v1) y `train_model_v2.py` sí duplican la función `generar_dataset` conceptualmente entre sí (son dos generadores de dataset distintos, v1 histórico y v2 oficial) — esto es intencional (v1 se preserva como evidencia, no como código activo), no una duplicación accidental de lógica de producción |

## Clasificación de la arquitectura

**Correctamente modularizada.**

No hay acoplamiento de lógica HTTP con IA, no se convirtió en microservicio (permanece módulo interno del monolito, cumpliendo la restricción explícita), y el entrenamiento está completamente desacoplado de FastAPI (ningún import cruzado, ningún trigger de entrenamiento en `lifespan` ni en ningún router).
