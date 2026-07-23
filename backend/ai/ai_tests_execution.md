# Ejecución de pruebas — módulo de IA (Etapa A)

Entorno: `backend/.venv312` (Python 3.12.13, ya construido en la validación P0 previa).

| Comando | Resultado |
|---|---|
| `.venv312/bin/python -m pytest tests/unit/test_random_forest_service.py tests/integration/test_ia_api.py -v` | **13/13 pasan** (8 unitarias + 5 de integración) |
| `.venv312/bin/python -m pytest -q` (suite completa) | **117/117 pasan** (incluye las 13 de IA + 104 del resto del sistema, incluyendo las 6 de P0) |
| Reproducción de entrenamiento (`python -m src.infrastructure.ai.train_model`) | Éxito, métricas byte-idénticas al original (ver `ai_evaluation_report.md`); artefactos restaurados al finalizar |

## Cobertura real de las 13 pruebas de IA

- Disponibilidad del modelo entrenado.
- Clasificación estable → `normal`.
- Excursión prolongada → `excursion_critica`.
- Fallback a regla cuando no hay modelo (`.pkl` ausente).
- Confianza y origen reportados correctamente (`origen="modelo"`).
- Origen `regla_fallback` cuando no hay modelo.
- **Salvaguarda determinista**: un modelo doble de prueba que "miente" (siempre predice `normal`) es sobrescrito correctamente por la regla cuando hay una excursión crítica real.
- Metadata del artefacto v2 disponible (`model_version == "2.0.0"`, `f1_weighted >= 0.85`).
- `GET /api/ia/modelo` requiere autenticación (401 sin token).
- `GET /api/ia/modelo` reporta métricas RNF-04 correctamente para un farmacéutico autenticado.
- `POST /api/ia/clasificar` clasifica lectura normal y excursión crítica correctamente vía HTTP.
- `POST /api/ia/clasificar` deniega a un técnico (403) — confirma RBAC real en el endpoint de IA.

## Lo que NO cubren las pruebas existentes (gaps confirmados, no hallazgos de "prueba rota" sino de "prueba ausente")

- Ninguna prueba de **fuga de datos** (no hay un test que compare `reales` vs `medidas` en el generador sintético).
- Ninguna prueba de **reproducibilidad del entrenamiento** como parte de la suite automatizada (se verificó manualmente en esta sesión, no está codificado como test).
- Ninguna prueba de **valores `NaN`/infinitos** llegando al vector de features (Pydantic los rechaza en la capa de ingesta, pero no hay un test unitario específico para `FeaturesRiesgoTermico`/`RandomForestRiesgoService` con esos valores).
- Ninguna prueba de **comparación con baseline**.
- Ninguna prueba de **checksum del modelo**.
- Ninguna prueba de **feature en orden incorrecto** (si alguien invirtiera accidentalmente dos campos de `to_array()`, ninguna prueba lo detectaría explícitamente).

Estos gaps se documentan como hallazgos en `ai_findings.md`, no se corrigen en Etapa A.
