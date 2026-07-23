# 03 — Concurrencia alertas PostgreSQL

Base temporal PostgreSQL 18: `tesis_code_alert_concurrency_20260722`, eliminada.
Dos sesiones independientes y 20 lecturas críticas: una alerta abierta,
lectura inicial distinta de última lectura, sin excepción.

Hallazgo/fix: lock proceso + `pg_advisory_xact_lock` podía invertir locks al
crear hash de lectura y alerta. PostgreSQL usa sólo advisory transaction lock;
SQLite conserva lock proceso. Sin migración nueva.
