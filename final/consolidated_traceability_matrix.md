# Matriz consolidada de trazabilidad — Tesis × Frontend × Backend

| Elemento oficial | Evidencia documental | Evidencia frontend | Evidencia backend | Estado final | Hallazgo | Acción |
|---|---|---|---|---|---|---|
| Problema (control manual/discontinuo) | TI cap.1, evidencia cuantitativa nacional/local | N/A | N/A | Documental sólido | Ninguno | — |
| Objetivo general | TI/PT/ACP idénticos | Prototipo alineado | Prototipo alineado | Coherente | Ninguno | — |
| OE1 (análisis requisitos) | TI, entrevistas/observación declaradas | N/A | N/A | No evaluable desde código | — | — |
| OE2 (diseño arquitectura) | TI, Anexo2 benchmarking (13 componentes, 4.89/5.00 promedio) | Arquitectura DDD replicada | Arquitectura DDD real | Coherente | 2 componentes con margen estrecho no declarado en TI | Aclarar en sustentación |
| OE3 (desarrollo prototipo) | TI | Implementado ~85% de páginas/flujos | Implementado ~90% de RF | Parcialmente implementado | Checklist BPA y PDF ausentes | Priorizar antes de sustentación |
| OE4 (validación técnica) | TI, criterios MTTD/F1/disponibilidad/SUS | No medido en esta auditoría | F1 medido y real (0.9659); disponibilidad/latencia no instrumentadas | Parcial | Faltan métricas de disponibilidad/latencia reales | Instrumentar y medir antes de sustentación |
| RF-01 a RF-07 (captura/transporte/persistencia) | TI | Consume correctamente | Implementado y operativo | Implementado integralmente (salvo RF-06 firmware no evaluable) | — | — |
| RF-08 (Random Forest 3 clases) | TI | Muestra sin calcular | **Real, F1=0.9659, con salvaguarda de regla** | Implementado integralmente | Dataset sintético, comunicar con precisión | Aclarar metodología en defensa |
| RF-09/RF-10 (alertas/acciones correctivas) | TI | Completo | Completo, auditado, trazado | Implementado integralmente | Ninguno | — |
| RF-11 (SSE tiempo real) | TI | EventSource real, ticket | Ticket+broadcaster reales | Implementado integralmente | Asimetría HTTP vs MQTT en emisión SSE | Corregir (B-06) |
| RF-12 (filtros historial) | TI | 4/5 filtros usados | 5/5 filtros soportados | Implementado parcialmente en frontend | Backend más completo que UI | Exponer filtro de conectividad en UI |
| RF-13 (reportes BPA + PDF) | TI, HU-38 | CSV/JSON reales, **sin PDF** | Datos reales auditados, **sin PDF** | **Implementado parcialmente en ambos — PDF ausente end-to-end** | Hallazgo alto confirmado en 2 fases | Implementar generación de PDF |
| RF-14 (hash SHA-256 encadenado) | TI | Visualiza correctamente, solo lectura | Fórmula correcta, tests reales | Implementado, **con defecto crítico de concurrencia** | **B-01 crítico** | Corregir antes de cualquier despliegue con tráfico concurrente real |
| RF-15 (verificación integridad) | TI | Botón conectado | Endpoint real, O(n) | Implementado, mismo defecto que RF-14 | Ídem | Ídem |
| RF-16 (audit_logs) | TI | Visualiza | 8 tipos de acción auditados | Implementado integralmente | Ninguno | — |
| RF-17 (JWT+RBAC 3 roles) | TI | Real, memoria, rutas protegidas | Real, revocación, RBAC server-side verificado | Implementado integralmente | Frontend no llama a `/logout` (B-07) | Conectar logout real |
| RF-18 (estado conectividad) | TI | **No renderizado** | Disponible y filtrable | **Implementado solo en backend** | Omisión exclusiva de frontend confirmada | Agregar a UI |
| RNF-04 (F1≥0.85) | TI | Sin superficie de UI | **Cumplido con evidencia real y reproducible** | Implementado en backend, invisible en frontend | Gap de exposición, no de sustancia | Agregar vista de métricas IA |
| RNF-06 (todo endpoint requiere JWT) | TI | Envía Bearer correctamente | Verificado en cada endpoint sensible | Implementado integralmente | Ninguno | — |
| Checklist BPA (HU-37) | TI, backlog | Simulado (`localStorage`) | **Ausente** | **Ausente end-to-end** | Contradicción confirmada en 2 fases | Implementar de cero |
| Historia huérfana HU-43 (gestión dispositivos) | Backlog (defecto propio) | Ausente | Ausente (solo alta automática interna) | **Ausente end-to-end** | Backlog nunca planificó correctamente esta historia | Redefinir HU y planificar |
| Overclaim ">96% exactitud" (PPTX slide 25) | PF_ presentación | No se repite en código | **Hoy real (accuracy=96.56%), pero el entrenamiento (jul-2026) es posterior a la presentación (may-2026)** | Coincidencia parcialmente vindicada, cronológicamente cuestionable | Riesgo de comunicación en defensa | Explicar la cronología proactivamente |
| Duplicación bibliográfica MD_ (arts. Saqib&Moon / Ali et al.) | Fase 1 (Anexo MD_) | N/A | N/A | Confirmado no respaldado en ninguno de los 2 papers | Hallazgo bibliográfico | Corregir Anexo MD_ antes de sustentar |
| Trazabilidad ≠ blockchain (posicionamiento académico) | TI explícito | Terminología correcta | Terminología correcta, sin librerías blockchain | Coherente | Ninguno | — |
| RBAC 3 roles exactos | TI | `administrador/farmaceutico/tecnico` | Mismos 3, StrEnum idéntico | Coherente | Backlog usó "Auditor" narrativamente (resuelto por el código real) | Aclarar en defensa que "Auditor" = administrador |
| Benchmarking 13 componentes | Anexo2 | N/A | Stack coincide 13/13 con lo implementado | Coherente | 2 márgenes estrechos no declarados en TI | Mencionar honestamente si se pregunta |
