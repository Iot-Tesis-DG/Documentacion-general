# 01 — Baseline previo a corrección P0→P1→P2

## Rama y estado

```
$ git branch --show-current
feat/random-forest-pipeline
```

`git status --short` (36 entradas): sin pérdida de trabajo respecto al inventario de `01_inventory.md` de la verificación anterior — mismo conjunto de archivos pre-existentes/P0/IA dirty, nada descartado.

## Suite baseline

```
$ python -m pytest -q
123 passed, 2414 warnings in 33.12s
```

Python: `3.12.13` (`.venv312`).

## Artefactos preservados (hash antes de tocar nada)

```
31e9224a201631cdb0072f6c7b7c7af5b45329e0a5686c350f4d2c5097dba656  random_forest_termico.pkl
31e9224a201631cdb0072f6c7b7c7af5b45329e0a5686c350f4d2c5097dba656  random_forest_v2_corrected.joblib
ca8e3efe06e7a3bc974b488bbca88e0bbd8b2b74bdfa4554991c3b651ff931ea  random_forest_v1_original.joblib
```

v1 y v2 se preservan sin sobrescribir. v3 se generará como artefactos nuevos y separados.

## Archivos que se modificarán en esta fase (previsto, sujeto a ajuste)

- `src/infrastructure/ai/train_model_v2.py` → nueva generación de `escenario_id` determinista (AIV-01)
- `src/infrastructure/ai/train_model_v3.py` (nuevo) → script v3 con reproducibilidad + metadata externa (AIV-01/AIV-06)
- `src/infrastructure/ai/models/` → nuevos artefactos v3 + `model_metadata.json` externo
- `src/infrastructure/ai/random_forest_service.py` → validación ampliada, lectura de metadata externa, semántica `confianza_ia=None`, estados de inferencia
- `src/infrastructure/ai/features.py` → validación NaN/inf en construcción de features (si aplica)
- `src/application/use_cases/clasificar_riesgo_termico.py` → guard completo de sensores (AIV-03)
- `src/domain/entities/lectura_termica.py` → `confianza_ia: float | None`, nuevos campos
- `src/domain/entities/alerta_termica.py` / `src/application/use_cases/generar_alerta.py` → control de episodio/cooldown/histéresis (AIV-02)
- `src/infrastructure/database/models.py` → nuevas columnas (estado_inferencia, motivo_no_inferencia, campos de episodio de alerta)
- `alembic/versions/0004_*.py` (nuevo) → migración aislada, no toca 0001-0003
- `src/interface/api/schemas.py`, `mappers.py` → nuevos campos en `LecturaResponse`/`AlertaResponse`
- `frontend/src/domain/entities/LecturaTermica.ts`, `AlertaTermica.ts`, componentes de dashboard/alertas → consumo de nuevos campos

No se tocan: `alembic/versions/0001_*`, `0002_*`, `0003_*`; `registrar_hash_encadenado.py`; `trazabilidad_repository.py` (salvo lectura, sin cambios de lógica de candado); `test_hash_chain_concurrencia.py`; `test_ingesta_dedup_y_sensor_nulo.py`.

Continúa con `02_reproducibility_fix.md`.
