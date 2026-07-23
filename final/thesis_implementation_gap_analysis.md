# Brechas entre lo documentado y lo implementado

## Documentado en TI/backlog pero NO implementado (ni frontend ni backend)

| Elemento | Fuente documental | Estado real |
|---|---|---|
| Checklist BPA digital con firma JWT y persistencia PostgreSQL (HU-37) | TI, Anexo5 | Ausente — solo simulación local en frontend |
| Exportación de reporte en PDF con hashes visibles (HU-38, TI línea 647) | TI, Anexo5 | Ausente — solo CSV/JSON |
| Gestión de dispositivos IoT (alta/baja/reemplazo, HU-43) | Anexo5 (huérfana) | Ausente — nunca se planificó correctamente ni se implementó |
| Notificación por email/Telegram (HU-23) | Anexo5 | No evaluada explícitamente en esta auditoría de backend (fuera del foco de los 18 RF/10 RNF formales de TI) |
| Backup automático PostgreSQL (HU-29) | Anexo5 | No evaluada — es infraestructura de Railway, no lógica de aplicación auditable en el código |
| Instrumentación de disponibilidad ≥95% y latencia ≤5s (RNF-01, RNF-02) | TI | No hay medición ni logging explícito confirmado en el código para estos RNF |

## Implementado pero NO documentado explícitamente en TI

| Elemento | Dónde se implementó | Observación |
|---|---|---|
| Salvaguarda determinista sobre la predicción del Random Forest (nunca rebaja severidad) | Backend, `random_forest_service.py` | Mejora de seguridad no mencionada en TI — debe explicarse en sustentación para no parecer que "todo son reglas" |
| Revocación server-side de JWT por jti (logout real) | Backend, `jwt_handler.py` + `revocation_store.py` | Mejora de seguridad no exigida explícitamente por TI, pero alineada con OWASP — sin embargo, nunca se ejercita porque el frontend no la invoca (hallazgo B-07) |
| Ticket SSE de un solo uso con audiencia separada | Backend | Solución elegante a una limitación real de `EventSource`, no detallada en TI |
| Internacionalización ES/EN completa | Frontend | Funcionalidad extra no exigida por TI, con paridad de 186/186 claves |
| Modo demo completo (adapter Axios + SSE simulado) para despliegue Vercel público | Frontend | No mencionado en TI (aunque coherente con la necesidad de una demo pública sin exponer el backend real) |
| Rate limiting de doble capa (login + global por IP) | Backend | Excede lo mínimo exigido por OWASP citado en TI |

## Implementado pero de forma DIFERENTE a lo documentado

| Elemento | TI/backlog dice | Implementación real |
|---|---|---|
| shadcn/ui | "React + Vite + TypeScript + shadcn/ui" (benchmarking, score 5.00) | Patrón manual replicado (Radix+cva+tailwind-merge), no instalado vía CLI oficial de shadcn |
| RBAC roles | Backlog usa 4 nombres narrativos en algunas HU (incluye "Auditor") | Sistema real tiene exactamente 3 roles (`administrador/farmaceutico/tecnico`) — "Auditor" nunca existió como rol de sistema, era terminología descriptiva para "quien accede a auditoría" (el administrador) |
| Dataset de entrenamiento del Random Forest | TI no aclara el origen del dataset | 100% sintético, generado en runtime con ruido inyectado — método válido pero no explicitado en el texto de la tesis |

## Análisis de la brecha "IA real" vs. percepción de "IA de fachada"

Este es el punto más delicado de la brecha documental: TI describe "un modelo Random Forest para clasificar el riesgo térmico" sin mayor detalle técnico. La implementación real:
- **Sí es un modelo entrenado genuino** (no si/else disfrazado) con métricas rigurosas.
- **Pero su dataset es sintético**, no datos reales de campo — esto no está aclarado en el texto de TI, lo que podría dar la impresión (si no se explica) de que el modelo se entrenó con datos reales de la farmacia validada.
- **Y tiene una salvaguarda determinista** que puede sobrescribir su predicción — TI no menciona esto, lo que podría dar la impresión opuesta (que "no hay salvaguarda, solo confían ciegamente en la IA") si el jurado no lo sabe y lo pregunta de forma capciosa.

### Precisión temporal y metodológica exigida sobre la cifra ">96%"

Clasificación correcta de este hallazgo: **afirmación actualmente respaldable, pero históricamente no sustentada en la versión original de la presentación**. La presentación (PF_, diapositiva 25) es de mayo de 2026; el entrenamiento/evaluación propio identificado en el repositorio (`training_metrics.json`) es de julio de 2026. Esto significa:
- El accuracy real actual (96.56%) demuestra que **hoy** existe evidencia propia cercana a la afirmación de la diapositiva.
- **No** demuestra que la afirmación estuviera sustentada cuando se incluyó originalmente en la presentación — no hay forma de que en mayo existiera el resultado de un entrenamiento ejecutado en julio.
- No debe confundirse accuracy con F1: son métricas distintas. La métrica que exige RNF-04 es F1, y el valor real es **F1 ponderado (weighted) = 0.9659**, no accuracy 96.56% (aunque ambos números sean cercanos, no son intercambiables ni equivalentes conceptualmente).
- La diapositiva debe actualizarse con: fecha del entrenamiento, descripción del dataset (sintético, 8000 muestras), partición train/test (80/20 estratificada), y la métrica exacta reportada (F1 weighted, con desagregación por clase si se desea mayor rigor).

La recomendación central de este análisis es que el equipo **explique proactivamente** ambos matices (dataset sintético + cronología de la métrica) en la sustentación, en vez de esperar a que el jurado los descubra por su cuenta — la evidencia real es sólida y defendible, pero solo si se comunica con precisión.
