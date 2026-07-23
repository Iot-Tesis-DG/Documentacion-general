# 12 — Cierre P0 → P1 → P2

Rama backend: `feat/random-forest-pipeline`. Sin commit, push ni deploy.

| Hallazgo | Estado |
|---|---|
| AIV-01 reproducibilidad | Resuelto |
| AIV-02 episodio de alertas | Resuelto |
| AIV-03 guard de sensores | Resuelto |
| AIV-04 API/SSE | Resuelto |
| AIV-05 frontend | Resuelto |
| AIV-06 model hash | Resuelto |
| AIV-07 confianza/estado | Resuelto |

Artefacto oficial v3: `random_forest_termico.pkl` con SHA-256
`0294ad0a07a47b8975d6f6102ee7aab85109049a23104ebb81c445bfdf8d33d4`.
Metadata externa: `model_metadata.json`. Migración nueva:
`0004_ia_correcciones_p1.py`.

Resultado local: backend 134/134, frontend typecheck/build OK. PostgreSQL 18
validó upgrade/downgrade/upgrade de 0004 en base descartable eliminada; no se
empleó producción.
