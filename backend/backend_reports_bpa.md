# Checklist BPA y Reportes/Exportación PDF — verificación cruzada final

## Checklist BPA — confirmado ausente en AMBOS lados

Búsqueda exhaustiva (`grep -irn "checklist"`) en todo `backend/src/`: **cero resultados**. No existe entidad, tabla, esquema Pydantic, endpoint, caso de uso, ni ninguna referencia a "checklist" o "BPA check" en el backend.

**Clasificación del hallazgo de Fase 2 corregida**: no es "backend existe, frontend no lo consume" — es **"ausente en ambos componentes"**. HU-37 (formulario checklist BPA digital con firma JWT y persistencia PostgreSQL) **no está implementada en ningún punto del sistema real**, pese a que el frontend simula su existencia con una interfaz completa (barra de progreso, ítems, persistencia en `localStorage`).

## Acciones correctivas — sí completamente implementadas (contraste positivo)

A diferencia del checklist, las acciones correctivas SÍ tienen la cadena completa: entidad de dominio (`AccionCorrectiva`), tabla (`corrective_actions`), endpoint (`POST /api/alertas/{id}/acciones-correctivas`), caso de uso (`RegistrarAccionCorrectivaUseCase`), auditoría (`REGISTRAR_ACCION_CORRECTIVA` en `audit_logs`), vinculación con la alerta (`alert_id` FK), trazabilidad (hash encadenado vía `RegistrarHashEncadenadoUseCase` — confirmado en el import de `alertas_router.py`), timestamp (`created_at`), y consumo real desde el frontend (`AlertasPage`). **Esta funcionalidad está genuinamente completa de extremo a extremo.**

## Exportación PDF — confirmado ausente en AMBOS lados

- `requirements.txt`/`requirements-dev.txt`: sin ninguna librería de generación de PDF (`reportlab`, `weasyprint`, `fpdf`, `pdfkit`, `wkhtmltopdf` bindings) — confirmado por `grep`.
- `ReportExportModel.archivo_url`: campo presente en el esquema (sugiere que en algún momento se planificó generar y almacenar un archivo, posiblemente subido a un storage externo tipo S3/Railway volumes), pero **`SQLAlchemyReporteRepository.registrar_exportacion()` nunca recibe un valor para `archivo_url`** en ningún punto de llamada revisado (`ExportarReporteBPAUseCase.execute()` no lo pasa) — el campo queda sistemáticamente `NULL`.
- `ExportarReporteBPAUseCase` retorna únicamente los datos crudos (lecturas, alertas, trazabilidad) en un objeto `ReporteBPA` — la responsabilidad de "formatear" (CSV/JSON/PDF) recae enteramente en el consumidor, y el único consumidor (el frontend) solo implementa CSV/JSON.

**Conclusión**: RF-13 (parcial, exportación) y HU-38 (PDF específicamente) están **implementados solo parcialmente de extremo a extremo** — el backend correctamente separa "obtener los datos del período" (sí implementado, con auditoría real) de "darles formato de documento" (nunca implementado en ningún formato más allá de la serialización JSON que el frontend transforma a CSV). El campo `archivo_url` sugiere que el equipo planificó una fase de generación de archivo real que no se completó.

## Evidencia de auditoría de exportación — sí real

Cada llamada a `GET /api/reportes/bpa` registra un `EXPORTAR_REPORTE_BPA` en `audit_logs` con el rango de fechas y `device_id` solicitado — trazabilidad de "quién exportó qué y cuándo" **sí está completamente implementada**, independientemente de que el archivo final no sea PDF.
