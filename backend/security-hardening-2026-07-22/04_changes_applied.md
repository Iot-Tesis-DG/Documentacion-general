# Cambios aplicados

| Área | Cambio | Evidencia |
|---|---|---|
| Configuración | `ENVIRONMENT` solo acepta development/test/production; producción exige secretos, DB/MQTT no-placeholder, HTTPS CORS y hosts explícitos. | `backend/src/infrastructure/config.py` |
| HTTP | `Content-Length` inválido devuelve 400 y middleware ASGI cuenta bytes reales, incluidos cuerpos chunked. | `backend/src/interface/api/api_protection.py` |
| MQTT | Se exige `farmacias/{device_id}/lecturas`; el `device_id` debe coincidir; payload cerrado y sin NaN/infinito. | `backend/src/interface/main.py`, `payload_schema.py` |
| Memoria | Limitador IP y JTI tienen capacidad máxima con expulsión controlada. | `rate_limiter.py`, `revocation_store.py` |
| Frontend | CSV neutraliza `=`, `+`, `-`, `@`; CSP elimina wildcard Railway y build ya no es demo. | `useReportesBPA.ts`, `vercel.json` |
| Dependencias | ECharts 6.1.0 y React/React DOM 19.2.8. | `package.json`, `package-lock.json` |
| Higiene | `.venv312` queda cubierto por `.venv*/`; ejemplos no mencionan Railway. | `.gitignore`, `.env.example` |

No se modificó la migración `0004`.
