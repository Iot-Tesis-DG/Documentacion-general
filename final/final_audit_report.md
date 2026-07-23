# Informe final consolidado — Auditoría de tesis IoT+IA Cadena de Frío Farmacéutica

## 1. Resumen ejecutivo

El proyecto es un prototipo académico de monitoreo de cadena de frío farmacéutica (UPC, 2026) que integra IoT (ESP32+sensores), IA (Random Forest para clasificación de riesgo térmico), trazabilidad digital verificable (hash SHA-256 encadenado, explícitamente no blockchain) y controles de seguridad (JWT+RBAC de 3 roles), validado en un escenario representativo de una farmacia en el Cercado de Lima. Tras auditar exhaustivamente el documento oficial (55 archivos, incluidos los 40 artículos científicos del estado del arte), el frontend (55 archivos propios) y el backend (110 archivos propios), la conclusión es que **el sistema es sustancialmente real, no una fachada**: hay una arquitectura DDD genuina en ambos componentes, autenticación/autorización verificadas de extremo a extremo, un modelo de Random Forest realmente entrenado y validado con rigor estadístico, y un mecanismo de hash encadenado correctamente diseñado en su fórmula. Sin embargo, existe **un defecto crítico de concurrencia** en la cadena de trazabilidad que compromete directamente uno de los pilares centrales de la tesis (RNF-03), y **dos funcionalidades documentadas explícitamente** (checklist BPA digital, exportación PDF) **están completamente ausentes** en ambos componentes del sistema.

## 2. Cobertura completa

- **Fase 1 (Final Oficial)**: 55 archivos, 51 inspeccionados con evidencia concreta, 4 excluidos justificadamente (metadata macOS). 40/40 artículos científicos verificados contra su texto completo, no solo su portada.
- **Fase 2 (Frontend)**: 55 archivos propios, 100% leídos (3719 líneas). Build y typecheck ejecutados realmente (exit 0).
- **Fase 3 (Backend)**: ~110 archivos propios, prácticamente 100% leídos en profundidad. Pruebas (18 archivos, 1281 líneas) inspeccionadas estáticamente pero **no ejecutadas** por incompatibilidad de entorno Python (requiere 3.12, disponible solo 3.9 en la máquina de auditoría) — limitación honestamente documentada, no ocultada.

## 3. Comprensión consolidada de la tesis

Problema: control térmico manual/discontinuo en farmacias independientes de Lima Metropolitana. Objetivo: prototipo IoT+IA con trazabilidad verificable y seguridad, validado en Cercado de Lima. Metodología: PMBOK 6ta ed. + Scrum + Lean, revisión sistemática de 40 artículos (Kitchenham y Charters). Stack seleccionado por benchmarking cuantitativo de 13 componentes (score 4.89/5.00). Ver detalle completo en `audit-output/thesis_master_context.md`.

## 4. Arquitectura documentada

5 capas (Edge/Transporte/Backend/Trazabilidad/Frontend) + capa transversal de seguridad, según TI. ESP32+SHT31+DS18B20+MC-38 → MQTT/TLS → EMQX Cloud → FastAPI/DDD → PostgreSQL → Random Forest → SHA-256 encadenado → React 19/SSE.

## 5. Arquitectura implementada

**Coincide sustancialmente con lo documentado**, con DDD genuino verificado en ambos componentes (dirección de dependencias real, no cosmética). Backend: domain/application/infrastructure/interface con puertos y adaptadores correctos. Frontend: domain/application/infrastructure/presentation con la misma disciplina. El firmware ESP32 (capa Edge) no forma parte de ninguno de los dos repositorios auditados — **no evaluable**, se asume su existencia según lo declarado en TI/backlog pero no fue verificable en código.

## 6. Flujo de datos real de extremo a extremo

1. Sensor (no verificable, fuera de repo) → 2. ESP32 (no verificable) → 3. MQTT/TLS real (backend suscrito con TLS forzado en producción) → 4. EMQX (servicio externo, no verificable) → 5. Backend recibe, valida device_id anti-suplantación → 6. Autoriza dispositivo (mínimo privilegio real) → 7. Valida rango físico → 8. IA: Random Forest real + salvaguarda de regla determinista → 9. Persiste en PostgreSQL → 10. Hash SHA-256 encadenado (con defecto de concurrencia) → 11. Genera alerta si aplica → 12. Auditoría → 13. SSE (solo si la ingesta fue por MQTT, no por HTTP) → 14. Frontend recibe y muestra en tiempo real → 15. Usuario puede registrar acción correctiva (completo, auditado, trazado).

## 7. Estado del frontend

**Parcialmente implementado y técnicamente consistente** (veredicto de Fase 2, corregido tras retroalimentación). Código de alta calidad (TypeScript estricto, cero `any`, JWT en memoria real, RBAC de rutas real, SSE real), pero con 3 funcionalidades exigidas por TI/backlog ausentes o mal resueltas (checklist BPA, PDF, métricas de IA) y cero pruebas automatizadas.

## 8. Estado del backend

**Funcional con inconsistencias menores** (veredicto de Fase 3). Arquitectura y seguridad ejemplares, IA real y validada, pero con un hallazgo crítico de concurrencia en la trazabilidad y las mismas dos funcionalidades ausentes (checklist BPA, PDF) confirmadas también del lado servidor.

## 9. Estado de IoT y MQTT

Backend: TLS real y forzado en producción, autorización de dispositivo por aplicación, anti-suplantación device_id-vs-topic. Topic `eventos` suscrito sin lógica diferenciada. Firmware ESP32 real: **no evaluable**, fuera de los repositorios auditados.

## 10. Estado de la IA

**Modelo real, entrenado, evaluado con rigor estadístico e integrado**, con **F1 ponderado (weighted) = 0.9659** (supera RNF-04 ≥0.85) y un mecanismo de build que impide publicar un modelo insuficiente. Dataset 100% sintético (no datos de campo reales), metodológicamente defendible pero no aclarado en TI. Salvaguarda determinista que impide a la IA subestimar riesgos que la regla básica detecta — mejora no documentada en TI.

**Sobre el overclaim ">96%" de la diapositiva 25**: clasificado como **afirmación actualmente respaldable, pero históricamente no sustentada en la versión original de la presentación**. La presentación es de mayo de 2026; el entrenamiento/evaluación propio identificado (`training_metrics.json`) es de julio de 2026. La métrica actual de accuracy 96.56% demuestra que **hoy** existe evidencia propia cercana a la afirmación — no demuestra que la afirmación estuviera sustentada cuando se incluyó originalmente en la presentación. Además, accuracy y F1 son métricas distintas y no deben confundirse: la métrica que exige RNF-04 es F1 (weighted) = 0.9659, no accuracy. La diapositiva debe actualizarse con fecha, dataset, partición train/test, número de muestras y métrica exacta (especificando si es macro, weighted, micro o por clase), y debe eliminarse cualquier indicio de que el número estaba validado en mayo de 2026.

## 11. Estado de la trazabilidad

Fórmula y serialización canónica correctas, verificación O(n) real que detecta alteración, terminología correctamente no-blockchain. **Defecto crítico confirmado**: condición de carrera en la lectura del último hash antes de insertar, sin bloqueo ni aislamiento serializable, que puede bifurcar la cadena y producir falsos positivos de "cadena rota" bajo concurrencia real y esperable en producción.

## 12. Estado de seguridad

Postura mayormente alineada a OWASP/ISO 30141 (no certificada). JWT con claims completos, revocación real (aunque no ejercida por el frontend), RBAC server-side verificado en cada endpoint sensible, rate limiting de doble capa, validadores de configuración de producción ejemplares. Ningún secreto expuesto en ningún repositorio auditado.

## 13. Estado de validación técnica

F1≥0.85 (RNF-04): demostrado con evidencia real. Disponibilidad ≥95%, latencia ≤5s, carga dashboard ≤3s (RNF-01/02/10): **no evaluables** en esta auditoría por falta de instrumentación/medición real ejecutada. MAE≤0.5°C: no evaluable sin hardware real. Integridad 100% (RNF-03): **comprometida por el hallazgo crítico de concurrencia**.

## 14. Matriz tesis–frontend–backend

Ver `consolidated_traceability_matrix.md` — 27 filas cubriendo problema, objetivos, 18 RF, RNF clave, historias de backlog, y hallazgos cruzados de las 3 fases.

## 15. Hallazgos críticos, altos, medios y bajos

Ver `consolidated_findings.md` — 1 crítico, 7 altos, 12 medios, 5 bajos (25 hallazgos totales consolidados de las 3 fases).

## 16. Riesgos para la sustentación

Ver `defense_risks.md` — 10 riesgos concretos, priorizados, con mitigación recomendada para cada uno.

## 17. Plan de corrección priorizado

Ver `prioritized_remediation_plan.md` — 29 acciones organizadas en 6 niveles de prioridad (0 a 5).

## 18. Veredicto final

**Parcialmente implementado.**

> El sistema presenta una base técnica sólida y componentes reales en autenticación JWT, RBAC server-side, SSE y clasificación mediante Random Forest. Sin embargo, permanece parcialmente implementado respecto del alcance oficial debido a la ausencia de funcionalidades exigidas, la falta de verificación reproducible de pruebas y un defecto crítico de concurrencia que compromete la trazabilidad digital verificable.

Justificación: la evidencia recolectada a lo largo de las 3 fases confirma componentes genuinos y no de fachada — arquitectura DDD real en ambos componentes, JWT/RBAC verificados de extremo a extremo con evidencia de código, un modelo de Random Forest realmente entrenado con metodología estadística correcta, y una fórmula de hash encadenado correctamente diseñada. Esto descarta "Mayormente simulado", "Incompleto" o "No ejecutable" como veredictos.

Pero el conjunto de hallazgos impide calificarlo por encima de "Parcialmente implementado":

- **Un hallazgo CRÍTICO sin corregir**: condición de carrera en la generación de la cadena SHA-256, con incumplimiento directo de RNF-03 (integridad 100%).
- **HU-37 (checklist BPA) ausente de extremo a extremo.**
- **HU-38 (exportación PDF) ausente de extremo a extremo.**
- **Sin deduplicación/idempotencia de lecturas MQTT.**
- **Conversión de sensor `None` a `0.0°C`**, capaz de generar alertas críticas falsas.
- **Ninguna prueba (18 archivos, 1281 líneas) fue ejecutada realmente** — bloqueo de entorno Python 3.12 documentado, no sustituido por una ejecución bajo un intérprete incompatible. No hay evidencia reproducible de que el sistema funcione integralmente end-to-end.
- Requisitos oficiales (RF-13 completo, HU-37, HU-38) permanecen sin implementación pese a estar explícitamente exigidos en TI y el Product Backlog.

El veredicto **"Parcialmente implementado"** es el que se deriva de esta evidencia: reconoce los componentes genuinamente construidos y validables por código, sin inflar la calificación por encima de lo que la falta de verificación reproducible y las ausencias funcionales permiten sostener.
