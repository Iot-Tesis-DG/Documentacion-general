# 01 — Estado inicial

Rama: `feat/random-forest-pipeline`. Python `3.12.13`, venv `.venv312`.
PostgreSQL local: 18.4, `127.0.0.1:5432`. Worktree ya tenía correcciones P0,
P1, P2 y modelos históricos; no se descartó nada.

`pytest --collect-only`: 134 pruebas. `compileall`: OK. `ruff` y `mypy` no
están instalados. `alembic current` con URL ejemplo falla por rol `user`; las
validaciones PostgreSQL usan URL temporal local explícita.
