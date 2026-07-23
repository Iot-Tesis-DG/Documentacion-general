# Hallazgos consolidados — las 3 fases

Veredicto final derivado de este conjunto: **Parcialmente implementado** (ver `final_audit_report.md`, sección 18).

## CRÍTICO

1. **[Backend B-01]** Condición de carrera en generación de hash encadenado puede bifurcar la cadena y producir falsos positivos de "cadena rota" bajo escritura concurrente real — compromete RNF-03 directamente. Ver `backend/backend_hash_chain_analysis.md`.

## ALTO

2. **[Fase 1]** MD_ atribuye cifras cuantitativas (41.6bps, 0.00146J, 2.3ms) a dos artículos distintos (Saqib&Moon / Ali et al.) que, verificados contra el texto completo de ambos PDFs, **no las contienen** — cifra no verificable/posiblemente fabricada en la bibliografía propia de la tesis.
3. **[Fase 1/2] Afirmación actualmente respaldable, pero históricamente no sustentada en la versión original de la presentación.** Presentación PPTX (slide 25) afirma "Exactitud >96%" en mayo de 2026; el entrenamiento/evaluación propio que hoy existe (`training_metrics.json`) data de julio de 2026 — no pudo sustentar la afirmación al momento de incluirse. El accuracy real actual (96.56%) no debe confundirse con F1: la métrica que exige RNF-04 es F1 ponderado (weighted) = 0.9659, no accuracy. La diapositiva debe actualizarse con fecha, dataset, partición, número de muestras y métrica exacta.
4. **[Fase 2/3]** Checklist BPA (HU-37) **ausente de extremo a extremo** — ni frontend ni backend lo implementan; el frontend simula persistencia con `localStorage` puro.
5. **[Fase 2/3]** Exportación PDF (HU-38, RF-13) **ausente de extremo a extremo** — ni frontend ni backend generan PDF; el campo `archivo_url` del backend nunca se puebla.
6. **[Fase 2/3]** RNF-04 (F1≥0.85) sin ninguna superficie de presentación pese a estar genuinamente demostrado en backend (F1=0.9659) — invisible para el usuario/jurado en la aplicación real.
7. **[Backend B-04]** Sin deduplicación de lecturas — reenvío MQTT por PUBACK perdido duplica registros reales en `thermal_readings` y en la cadena de trazabilidad.
8. **[Backend B-05]** Sensor caído (`None`) tratado como 0.0°C para clasificación de riesgo — puede generar alertas críticas falsas basadas en ausencia de dato, no en condición térmica real.

## MEDIO

9. **[Fase 1]** Benchmarking (Anexo2): 2 de 13 componentes tienen margen ganador de solo 2% (0.10/5.00), pero TI afirma categóricamente que "ninguna selección quedó definida por margen estrecho" — solo reconoce uno de los dos casos.
10. **[Fase 1]** Backlog (Anexo5): historia HU-43 huérfana/corrupta — título sobre gestión de dispositivos IoT con criterios de aceptación copiados literalmente de HU-41 (RBAC), sin épica ni sprint asignado.
11. **[Fase 1]** RBAC: TI define 3 roles fijos; backlog usa de facto 4 nombres narrativos ("Auditor" adicional) — resuelto por el código real (solo 3 roles existen), pero debe aclararse en sustentación.
12. **[Fase 2]** RF-18 (estado de conectividad) no se renderiza en ninguna pantalla del frontend, pese a que el backend lo expone y permite filtrar por él.
13. **[Fase 2]** Cero pruebas automatizadas en el frontend, sin ESLint configurado.
14. **[Backend B-06]** Ingesta HTTP de lecturas no emite evento SSE (asimetría con el camino MQTT, que sí lo hace).
15. **[Backend B-07]** Frontend nunca invoca `POST /api/auth/logout` — la revocación server-side de tokens (funcionalidad real del backend) nunca se ejercita en la práctica.
16. **[Backend B-08]** Endpoints `/api/ia/modelo` y `/api/ia/clasificar` (evidencia real de RNF-04) implementados pero nunca consumidos por el frontend.
17. **[Backend B-09]** Topic MQTT `farmacias/+/eventos` suscrito sin lógica de procesamiento diferenciada de la de lecturas.
18. **[Backend B-10]** Sin validación de timestamps futuros/antiguos en la ingesta de lecturas.
19. **[Backend B-11]** Índices de base de datos mejorables (`(device_id, timestamp)` compuesto ausente en `thermal_readings`; `created_at` sin índice en `traceability_records`/`audit_logs`).
20. **[Backend B-13]** No se pudieron ejecutar las 18 pruebas reales del backend por incompatibilidad de Python en el entorno de auditoría (requiere 3.12, disponible solo 3.9) — pendiente de confirmación real por el equipo antes de sustentar.

## BAJO

21. **[Fase 1]** OWASP WSTG v4.2 declarado como "versión estable" — verificar vigencia a la fecha real de sustentación.
22. **[Fase 2]** shadcn/ui usado como patrón manual (sin `components.json`/CLI oficial) — diferencia de proceso, no de resultado.
23. **[Fase 2]** Sin `ErrorBoundary` global en el frontend.
24. **[Fase 2]** Bundle de ECharts pesado (568KB) sin code-splitting por ruta.
25. **[Backend B-12]** `password_min_length` declarado en configuración, no confirmado como aplicado en la validación real del schema de creación de usuario.

## Hallazgos positivos transversales (para contexto del veredicto)

- Arquitectura DDD genuina en ambos componentes (frontend y backend), con dirección de dependencias verificada, no cosmética.
- JWT + RBAC + revocación + tickets SSE: ingeniería de seguridad notablemente cuidadosa para un proyecto de tesis de bachillerato.
- Random Forest real, con metodología de entrenamiento sólida (ruido inyectado, validación cruzada, gate de build sobre F1) y salvaguarda determinista de seguridad clínica.
- Hash SHA-256 encadenado con fórmula, serialización y timestamp correctamente canonicalizados, y tests unitarios reales que lo confirman.
- Contratos frontend-backend prácticamente sin discrepancias en 13 endpoints verificados.
- Los 40 artículos del estado del arte verificados con evidencia real (DOI, título, autor, cifras cruzadas contra el texto completo del PDF) — 34/40 con respaldo directo o parcial confiable.
