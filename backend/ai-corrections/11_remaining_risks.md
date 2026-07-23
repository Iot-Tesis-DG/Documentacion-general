# 11 — Riesgos residuales

- Migración PostgreSQL 18 validada en base descartable durante cierre. Para
  producción aún corresponde ejecutar pipeline CI contra versión objetivo.
- Pickle/joblib no promete identidad binaria entre versiones de Python,
  scikit-learn, plataforma o paralelismo. Se garantiza reproducibilidad
  funcional para misma configuración; el checksum valida cada artefacto final.
- Bundle ECharts supera 500 kB; rendimiento frontend mejorable con code split.
- Alertas concurrentes quedan protegidas por constraint de BD; validar carga
  real con PostgreSQL sigue pendiente.

No se usaron Railway, EMQX productivo, credenciales reales, commit, push ni deploy.
