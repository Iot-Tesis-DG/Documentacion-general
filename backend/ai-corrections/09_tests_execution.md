# 09 — Ejecución de pruebas

Baseline reportado por trabajo previo: 123 pruebas con Python 3.12.13.

Tras P0/P1/P2, ejecución final local:

```text
Python 3.12.13
pytest -q: 134 passed, 2414 warnings, 27.12s
npm run typecheck: OK
npm run build: OK
alembic heads: 0004_ia_correcciones_p1 (head)
```

Cobertura ejercida: generación reproducible, checksum alterado, sensor nulo,
NaN/infinito, fallback, 0 °C, episodio, cooldown, deduplicación MQTT, hash
concurrente, API, SSE, JWT/RBAC y frontend compilado.

Advertencias no bloqueantes: deprecación `starlette.testclient`, `crypt`,
`joblib/numpy_pickle`; claves JWT cortas sólo en fixtures de prueba.
