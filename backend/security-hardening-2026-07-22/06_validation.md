# Validación ejecutada

| Comando | Resultado |
|---|---|
| `backend/.venv312/bin/python -m pytest tests/unit/test_api_protection.py tests/unit/test_rate_limiter.py tests/unit/test_revocation_store.py tests/integration/test_seguridad_api.py -q` | 36 passed |
| `backend/.venv312/bin/python -m compileall src tests && .venv312/bin/python -m pytest -q -rA` | 140 passed, 2414 warnings, 26.77 s |
| `frontend/npm run typecheck` | pasa |
| `frontend/npm run build` | pasa; warning de chunk ECharts 606 kB minificado, no vulnerabilidad |
| `frontend/npm audit --omit=dev --json` | 0 info, 0 low, 0 moderate, 0 high, 0 critical |
| `backend/.venv312/bin/python -m pip list --outdated --format=json` | detectó actualización disponible de bcrypt/pip/pydantic-core; no equivale a CVE |
| `git diff --check` en backend | pasa |

Los warnings provienen principalmente de carga de arrays Joblib/NumPy y de claves JWT deliberadamente cortas en pruebas unitarias; la validación de producción exige 32+ caracteres. La validación PostgreSQL previa de las migraciones y concurrencia se conserva; este cierre no modificó la migración `0004` ni el esquema.
