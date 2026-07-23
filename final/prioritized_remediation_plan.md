# Plan de corrección priorizado

Veredicto que motiva este orden: **Parcialmente implementado** — se prioriza primero lo que compromete integridad de datos, luego lo ausente del alcance oficial, luego la validación reproducible, y por último la preparación de sustentación.

## P0 — Integridad y datos

1. Hacer **atómica** la obtención del último hash y la inserción del siguiente registro (eliminar la ventana de carrera entre `obtener_ultimo_hash()` y el `INSERT` posterior).
2. Definir explícitamente la estrategia de cadena: **global, por farmacia o por dispositivo** — la implementación actual asume una cadena global única sin declararlo como decisión de diseño consciente.
3. Usar **transacción y bloqueo apropiado en PostgreSQL** (`SELECT ... FOR UPDATE` sobre la fila puntero, o aislamiento `SERIALIZABLE` con reintento ante `SerializationFailure`).
4. Añadir **pruebas concurrentes** que demuestren de forma reproducible que no se generan bifurcaciones (dos o más inserciones simultáneas deben serializarse correctamente).
5. Implementar **deduplicación e idempotencia de lecturas MQTT** (restricción única `(device_id, timestamp)` o equivalente a nivel de aplicación) — corrige B-04.
6. Corregir el tratamiento de **sensores sin lectura**: `None` no puede convertirse en `0.0°C` para fines de clasificación de riesgo; debe marcarse explícitamente como "no clasificable" — corrige B-05.

## P1 — Requisitos oficiales ausentes

7. Implementar **persistencia backend del checklist BPA** vinculada al usuario JWT (entidad, tabla, endpoint, caso de uso) — HU-37.
8. Implementar **historial y auditoría del checklist** (quién completó qué ítem, cuándo, con qué identidad).
9. Implementar **generación real de PDF** para el reporte BPA (backend, no solo CSV/JSON) — HU-38/RF-13.
10. Incluir **hashes verificables y datos de trazabilidad** dentro del PDF generado.
11. **Integrar ambos requisitos con el frontend** (botones, vistas, flujo de usuario completo hasta la descarga/consulta real).

## P2 — Validación reproducible

12. **Ejecutar todas las pruebas con Python 3.12** real (reconstruir `.venv` correctamente) — ninguna de las 18 pruebas del backend fue ejecutada en esta auditoría; no se puede afirmar que pasan sin correrlas.
13. **Registrar la versión exacta de dependencias** usada en la ejecución real (congelar `requirements.txt` con versiones exactas o `pip freeze`).
14. **Reproducir el entrenamiento y evaluación del Random Forest** de forma verificable por terceros (documentar el comando exacto, semilla, y confirmar que el resultado es reproducible).
15. **Documentar el dataset**: origen sintético, 8000 muestras, distribución de clases, separación train/test (80/20 estratificada), y la inyección de ruido calibrada a los sensores.
16. **Reportar accuracy y F1 como métricas diferentes**, sin usar "exactitud" como sinónimo genérico de desempeño del modelo — especificar si el F1 reportado es macro, weighted, micro o por clase (el valor real disponible es F1 weighted = 0.9659).
17. **Añadir pruebas de JWT, RBAC, SSE, MQTT, hash y migraciones** donde no existan ya (confirmar cobertura real tras ejecutar el punto 12; el flujo MQTT y las migraciones no tienen archivo de test dedicado confirmado).

## P3 — Sustentación

18. **Actualizar la diapositiva 25** con la evidencia real de julio de 2026 (fecha, dataset, partición, número de muestras, métrica exacta F1=0.9659).
19. **Eliminar cualquier indicio de que la métrica estaba validada en mayo de 2026** — la afirmación original no tenía sustento propio en ese momento.
20. **Preparar explicación técnica de la corrección de concurrencia** (qué se cambió, por qué, y evidencia de que ya no se bifurca la cadena bajo carga concurrente).
21. **Preparar demostración de alteración y verificación de cadena** en vivo (mostrar que una alteración manual de un registro es detectada por `GET /api/trazabilidad/verificar`).
22. **Preparar evidencia de que el sistema no utiliza blockchain** (mostrar el código de `HashEncadenado`, la ausencia de librerías blockchain en `requirements.txt`, y la fórmula SHA-256 explícita).

## P4 — Integración y calidad adicional

23. Agregar visualización de `estado_conectividad` (RF-18) en el Dashboard o una nueva vista de dispositivos.
24. Agregar índice compuesto `(device_id, timestamp)` en `thermal_readings` e índices en `created_at` de `traceability_records`/`audit_logs` — corrige B-11.
25. Conectar `authStore.logout()` del frontend con `POST /api/auth/logout` del backend para que la revocación real se ejerza en la práctica — corrige B-07.
26. Agregar una vista de métricas de IA en el frontend que consuma `GET /api/ia/modelo` — corrige B-08.
27. Emitir el evento SSE también desde el camino de ingesta HTTP — corrige B-06.
28. Definir manejador dedicado para el topic MQTT `farmacias/+/eventos` — corrige B-09.
29. Agregar validación de timestamp futuro/antiguo en la ingesta de lecturas — corrige B-10.
30. Confirmar/aplicar realmente `password_min_length` en la validación de creación de usuario — corrige B-12.
31. Instrumentar medición real de disponibilidad (RNF-02) y latencia (RNF-01), y medir tiempo de carga real del dashboard (RNF-10) — actualmente no evaluables por falta de instrumentación.
32. Agregar pruebas automatizadas al frontend (Vitest + Testing Library) y configurar ESLint — corrige F-06/F-07 de Fase 2.

## P5 — Documentación y presentación

33. Corregir el Anexo MD_ (Fase 1): localizar el origen real de las cifras duplicadas entre Saqib&Moon/Ali et al., o retirarlas si no son verificables.
34. Corregir la historia HU-43 corrupta en el Product Backlog (Anexo5); definir e implementar o formalmente descartar la gestión de dispositivos IoT.
35. Ajustar la redacción de TI sobre "ningún margen de benchmarking fue estrecho" para reflejar los 2 casos reales de margen de 2%.
36. Aclarar la terminología "Auditor" del backlog frente a los 3 roles reales del sistema.
37. Corregir la inconsistencia de identidad en la diapositiva de cierre del PPTX (código de estudiante, nombre).
38. Agregar `ErrorBoundary` global al frontend; aplicar code-splitting por ruta (ECharts); documentar la decisión de shadcn/ui manual.
