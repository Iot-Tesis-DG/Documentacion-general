# 10 — Matriz E2E

| Escenario | Persistencia | IA | Alerta | Hash | SSE |
|---|---:|---|---:|---:|---|
| Normal | 1 | RF | 0 | 1 | lectura |
| Crítico | 1 | salvaguarda | 1 | 2 | lectura+alerta |
| Sensor ausente | 1 | omitida | 0 | 1 | fallo_sensor |
| 0.0 C | 1 | salvaguarda | actualización | 2 | lectura+episodio |
| Duplicado | 0 | 0 | 0 | 0 | 0 |
