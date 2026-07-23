# Alcance e inventario

Fecha: 2026-07-22. Alcance: todo el código, configuración y dependencias ejecutables de `backend/` y `frontend/`; se inventariaron además los archivos de la raíz. Se excluyeron entornos virtuales, `node_modules`, `dist`, binarios de modelo y documentos académicos PDF/DOCX/XLSX/MPP porque no son código ejecutable revisable estáticamente.

- Backend: FastAPI, SQLAlchemy/Alembic, MQTT, IA, seguridad, tests y scripts.
- Frontend: React, rutas, API/SSE, exportación, configuración Vite/Vercel y dependencias.
- Raíz: no hay firmware ESP32, Docker/Compose, CI ni secretos reales versionados.

No se hicieron commits, push, despliegues ni conexiones a infraestructura de producción. Los cambios previos del árbol se preservaron.
