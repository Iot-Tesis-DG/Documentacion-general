# 13 — Ejecución de pruebas (qué cubren realmente, no solo "verde")

## Ejecución fresca, esta sesión

```
$ python -m pytest -q
123 passed, 2414 warnings in 25.90s
```

Entorno: `.venv312`, Python 3.12.13, mismo entorno usado en el resto de esta verificación. `pip freeze` no vuelto a listar (ya documentado en `backend_tests_execution.md` de la sesión P0, sin cambios de dependencias en esta sesión — no se tocó `requirements.txt`/`pyproject.toml`).

## Inventario de archivos de prueba (21 archivos, colección fresca confirmada = 123)

| Archivo | Qué cubre | Qué NO cubre |
|---|---|---|
| `test_hash_encadenado.py` | Cálculo de hash SHA-256, encadenamiento básico | Concurrencia (cubierta aparte) |
| `test_hash_chain_concurrencia.py` | 2 escrituras concurrentes vía `asyncio.gather`, verifica no bifurcación | Solo 2 escritores concurrentes, no prueba carga alta (10+, 100+) ni contención real bajo PostgreSQL con múltiples procesos/workers reales (usa sesiones async dentro del mismo proceso de test) |
| `test_sha256_service.py` | Verificación de cadena completa, detección de alteración/registro faltante | — |
| `test_ingesta_dedup_y_sensor_nulo.py` | Dedup exacto, dedup con datos distintos mismo timestamp, sensor `temperatura_interna=None`, caso de control con dato real | **No cubre `temperatura_ambiental=None`** (el gap documentado en `08_fastapi_integration.md`/`09_feature_parity.md` no tiene una prueba que lo ejercite — consistente con que tampoco está corregido) |
| `test_random_forest_service.py` | Carga del modelo oficial, metadata (`dataset_hash`, `model_hash`, `particion`, `rnf04.cumplido`), inferencia básica | No prueba explícitamente el camino de "modelo no disponible" en producción (fallback) de forma aislada del resto del pipeline |
| `test_random_forest_service_validacion.py` | 4 casos de incompatibilidad de artefacto (features, clases, orden) + caso oficial válido | No prueba corrupción real de archivo (`.pkl` truncado/corrupto) — solo mocks de objetos con atributos incorrectos |
| `test_ai_persistencia_version.py` | `confianza_ia`/`modelo_version` en la respuesta API; sensor `None` → `confianza_ia=0.0`, `nivel_riesgo=None` | **No verifica que `origen_clasificacion` aparezca o no en la respuesta** (relevante dado el hallazgo de `12_sse_contract.md` de que no se expone) |
| `test_ia_api.py` | 5 pruebas de `/api/ia/modelo` y `/api/ia/clasificar` (RBAC, forma de respuesta) | No verifica contenido exacto de `dataset_hash`/`model_hash` contra los valores reales del artefacto en disco (solo que existan como claves) |
| `test_reglas_riesgo.py` | Regla determinista de clasificación por umbrales | — |
| `test_rango_termico.py` | Validación de rangos físicos de sensor | — |
| `test_lecturas_api.py`, `test_alertas_api.py`, `test_trazabilidad_api.py`, `test_auditoria_api.py`, `test_auth_api.py`, `test_usuarios_api.py`, `test_reportes_api.py`, `test_seguridad_api.py` | Endpoints REST, RBAC, JWT, seguridad — **no relacionados con IA**, pre-existentes | Confirman que P0/IA no rompió funcionalidad previa (ninguna regresión detectada) |
| `test_jwt_handler.py`, `test_password_hasher.py`, `test_rbac.py`, `test_revocation_store.py` | Seguridad de autenticación — pre-existentes, sin relación con IA | — |

## Lo que NINGUNA prueba automatizada cubre (gaps reales, no cosméticos)

1. **Ninguna prueba SSE real de extremo a extremo** (conexión `EventSource`, recepción de evento, contenido exacto del payload emitido) — la verificación del contrato SSE (`12_sse_contract.md`) se hizo por lectura de código, no por prueba automatizada que conecte un cliente SSE real y aserte sobre el JSON recibido.
2. **Ninguna prueba de "tormenta de alertas"** — no existe una prueba que envíe N lecturas críticas consecutivas y assert sobre cuántas alertas se generan (el hallazgo de `11_alerts_traceability.md` se confirmó por lectura de `generar_alerta.py`, no por prueba).
3. **Ninguna prueba de reproducibilidad del entrenamiento** (`train_model_v2.py` no tiene un test automatizado en `tests/` que corra el pipeline dos veces y compare — la verificación de `05_training_pipeline_verification.md`/`06_reproducibility.md` se hizo manualmente en esta sesión, ejecutando el script directamente, no vía pytest).
4. **Ninguna prueba de migraciones contra PostgreSQL real dentro del suite de pytest** — el suite usa SQLite in-memory exclusivamente (`conftest.py`); la verificación contra PostgreSQL real (`10_persistence_migrations.md`) se hizo manualmente en esta sesión, fuera de pytest, precisamente porque SQLite ya demostró (en el hallazgo de la migración 0003 original) que puede ocultar errores reales de PostgreSQL.
5. **Ninguna prueba del camino HTTP `POST /api/lecturas` verificando que NO emite SSE** (asimetría documentada en `08_fastapi_integration.md`, hallazgo B-06 pre-existente) — se confirmó por lectura de código, no por prueba negativa automatizada.

## Conclusión de esta fase

**123/123 en verde es evidencia real de que no hay regresiones conocidas y de que los casos ya cubiertos (dedup, concurrencia básica, sensor `None`, validación de compatibilidad del modelo, persistencia de campos IA) funcionan como se documentó.** No es evidencia de que el sistema esté libre de los hallazgos abiertos (tormenta de alertas, reproducibilidad de partición, gap de `temperatura_ambiental`, ausencia de `origen_clasificacion` en la API, asimetría SSE) — **ninguno de esos casos tiene una prueba automatizada, verde o roja, que los ejercite.** Confirmar esto explícitamente responde al mandato de la Fase 16 del prompt: "No declares 'todo correcto' solo porque pytest termina en verde."
