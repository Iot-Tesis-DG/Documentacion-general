# Inventario recursivo — backend/

128 archivos detectados (`find` excluyendo generados). Código propio: ~110 archivos (src/ + tests/ + alembic/ + scripts/ + configs).

## Exclusiones

| Carpeta/patrón | Motivo |
|---|---|
| `.venv/` | Entorno virtual Python — dependencias, no código propio |
| `.git/` | Control de versiones |
| `.pytest_cache/` | Caché generada |
| `.DS_Store` (×3) | Metadata macOS |

`dev.db` (SQLite) — **detectado, no excluido**: es una base de datos de desarrollo local con datos de prueba/seed. Se inspeccionó su esquema (no su contenido de producción, que no existe — es entorno dev).

## Inventario por módulo

| Ruta | Tipo | Propósito | Estado de revisión |
|---|---|---|---|
| `requirements.txt` / `requirements-dev.txt` | Config | Dependencias | Leído completo |
| `.env.example` | Config | Variables de entorno de ejemplo | Leído completo, sin secretos reales |
| `alembic.ini` + `alembic/env.py` + `alembic/versions/0001_initial_schema.py` | Migraciones | Esquema de BD | Leído completo |
| `pytest.ini` | Config | Pruebas | Leído completo |
| `scripts/seed_dev.py` | Script | Seed de datos de desarrollo | Existencia confirmada, no ejecutado (evita alterar `dev.db`) |
| `src/interface/main.py` | Entrypoint | FastAPI app, lifespan, middlewares, routers | Leído completo |
| `src/interface/api/*.py` (13 archivos: 8 routers + deps + mappers + schemas + api_protection + security_headers + sse_broadcaster) | Interface | Endpoints REST + SSE | Todos leídos completos |
| `src/application/use_cases/*.py` (11 archivos) | Application | Casos de uso | Todos leídos completos |
| `src/domain/entities/*.py` (5), `value_objects/*.py` (4), `repositories/*.py` (8 interfaces), `exceptions.py` | Domain | Entidades, VOs, puertos | Todos leídos completos |
| `src/infrastructure/security/*.py` (5: jwt_handler, password_hasher, rbac, rate_limiter, revocation_store) | Infrastructure | Seguridad | Todos leídos completos |
| `src/infrastructure/ai/*.py` (train_model.py, features.py, reglas_riesgo.py, random_forest_service.py) + `models/random_forest_termico.pkl` + `models/training_metrics.json` | Infrastructure | IA | Todos leídos completos (código + métricas reales) |
| `src/infrastructure/hash/sha256_service.py` | Infrastructure | Trazabilidad | Leído completo |
| `src/infrastructure/mqtt/mqtt_client.py` + `payload_schema.py` | Infrastructure | IoT | Ambos leídos completos |
| `src/infrastructure/database/models.py`, `base.py`, `session.py`, `repositories/*.py` (8) | Infrastructure | Persistencia | `models.py` y todos los repositorios leídos completos; `session.py`/`base.py` inspeccionados |
| `src/infrastructure/config.py` | Infrastructure | Configuración/Settings | Leído completo |
| `tests/unit/*.py` (8 archivos) + `tests/integration/*.py` (8 archivos) + `conftest.py` | Pruebas | 1281 líneas totales | Muestreadas en profundidad (hash, IA), resto confirmado como pruebas reales no triviales por inspección de nombres/contenido parcial |

## Artefactos de IA (verificados como reales, no vacíos)

- `random_forest_termico.pkl`: artefacto serializado real (`joblib.dump`), formato `{"modelo": RandomForestClassifier, "metadata": dict}`.
- `training_metrics.json`: métricas reales de un entrenamiento ejecutado el 2026-07-11 (accuracy 0.9656, F1 ponderado 0.9659, validación cruzada 5-fold, matriz de confusión completa). Ver `backend_ai_validation.md`.

## Dataset y notebooks

No existe ningún dataset externo ni notebook Jupyter — el dataset se **genera sintéticamente en runtime** por `train_model.py` (8000 muestras, semilla fija 42), no se persiste como archivo CSV/parquet separado. Esto es coherente con el enfoque (dataset reproducible por código, no un archivo versionado).

## Certificados / configuración MQTT

No se encontraron archivos de certificado TLS de ejemplo (`.pem`, `.crt`) en el repositorio — el cliente MQTT usa `ssl.create_default_context()` (validación estándar contra CAs del sistema), sin certificados custom embebidos. Coherente con el uso de EMQX Cloud Serverless (TLS gestionado por el proveedor, sin necesidad de certificados propios del lado cliente).
