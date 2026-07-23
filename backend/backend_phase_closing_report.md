# FASE 3 — BACKEND: Informe de cierre

## 1. Cobertura de archivos
128 archivos detectados, ~110 propios. Prácticamente todo el código fuente propio fue leído completo o en profundidad sustancial: 9 routers, 11 casos de uso, 5 entidades, 4 value objects, 8 interfaces de repositorio + 8 implementaciones, 5 módulos de seguridad, 4 módulos de IA (+ modelo real + métricas reales), hash service, MQTT client+schema, config, deps, migración inicial. Pruebas (18 archivos, 1281 líneas) muestreadas en profundidad, no ejecutadas (ver punto 20).

## 2. Arquitectura real
DDD/hexagonal genuina, verificada por dirección de dependencias real (no solo nombres de carpeta): domain sin dependencias externas, application depende de interfaces de dominio, infrastructure implementa esos puertos, interface es la única capa que conoce FastAPI. Sin lógica de negocio filtrada a routers, sin entidades anémicas, sin importaciones circulares.

## 3. Endpoints
19 endpoints mapeados. Todos los usados por el frontend existen y son compatibles. 2 endpoints de IA y 1 de logout existen pero no son consumidos por el frontend (oportunidad perdida, no un defecto).

## 4. Contratos frontend-backend
Compatibilidad alta, sin incompatibilidades de tipo/enum/formato detectadas. El backend consistentemente ofrece más funcionalidad (paginación, filtros, endpoints de IA) de la que el frontend consume — nunca al revés.

## 5. JWT
Real y robusto: claims completos exigidos (iss/aud/exp/iat/jti/sub), revocación server-side real por jti, secretos validados obligatoriamente en producción, bcrypt para contraseñas, rate limiting de login real. Compatible con la estrategia de "JWT en memoria" del frontend.

## 6. RBAC
Real y verificado server-side en cada endpoint sensible — coincide exactamente con las restricciones que el frontend ya mostraba, confirmando que no es "RBAC solo visual" en ninguno de los dos componentes.

## 7. Base de datos
9 tablas, migración y ORM perfectamente sincronizados. Faltan algunos índices compuestos y una restricción de deduplicación de lecturas — deuda técnica menor, no crítica.

## 8. MQTT
TLS real y forzado en producción, autorización de dispositivo por aplicación, anti-suplantación device_id-vs-topic. Topic `eventos` suscrito pero sin lógica diferenciada (hallazgo B-09).

## 9. Flujo de ingesta
Pipeline completo y coherente de 13 pasos reconstruido con evidencia de código, desde recepción hasta SSE. Defectos puntuales: sensor `None`→0.0°C (B-05), sin deduplicación (B-04), sin validación de timestamp (B-10).

## 10. Random Forest
**Real, entrenado con metodología correcta, F1=0.9659 (supera RNF-04) con evidencia reproducible y gate de build que impide publicar un modelo insuficiente.** Dataset 100% sintético (no datos reales de campo), con inyección de ruido deliberada para evitar circularidad — metodológicamente sólido mas debe comunicarse como sintético en la sustentación. Salvaguarda determinista que impide a la IA subestimar un riesgo que la regla básica detecta.

## 11. Métricas
Accuracy 96.56%, F1 ponderado 0.9659, validación cruzada 5-fold estable (std=0.003), matriz de confusión sin confusión crítica↔normal. Todas reales, en `training_metrics.json`, generadas el 2026-07-11.

## 12. Reglas térmicas
Distinción clara y verificada entre regla determinista (umbral 2-8°C + duración + tendencia) y predicción del modelo — sin contradicciones de nomenclatura de clases en ningún punto del sistema (frontend, backend, dataset).

## 13. Hash encadenado
Estructura correcta (`SHA256(previous_hash+timestamp+payload_canónico)`), verificación O(n) real que detecta alteración. **Hallazgo crítico B-01**: condición de carrera confirmada que puede bifurcar la cadena y producir falsos positivos de "cadena rota" bajo escritura concurrente realista.

## 14. Auditoría
8 tipos de acción auditados, tabla real, sin endpoints de modificación expuestos (inmutable a nivel de aplicación, no de motor de BD).

## 15. SSE
Real, ticket efímero de un solo uso con audiencia JWT separada, backpressure manejado. Limitación honesta y documentada: en memoria, no multi-worker. Asimetría: solo se emite en camino MQTT, no HTTP (B-06).

## 16. Checklist BPA
**Ausente en backend y frontend — HU-37 no implementada de extremo a extremo.**

## 17. Reportes PDF
**Ausente en backend y frontend — HU-38/RF-13(PDF) no implementada de extremo a extremo.** El backend sí separa correctamente datos de formato, pero el formato PDF nunca se construyó en ningún lado.

## 18. Seguridad
Postura mayormente alineada a OWASP/ISO 30141 (no certificada, como corresponde). Validadores de producción ejemplares. Un hallazgo de seguridad no confirmado (mínimo de contraseña), y el hallazgo crítico de concurrencia de trazabilidad tiene dimensión de seguridad también.

## 19. Pruebas
18 archivos, 1281 líneas, sustantivas por inspección estática (no triviales). **No se pudieron ejecutar realmente** — bloqueo de entorno documentado (Python 3.12 requerido, solo 3.9 disponible en esta máquina, venv del proyecto con symlink roto).

## 20. Ejecución local
No se pudo levantar el backend real ni ejecutar pruebas por el bloqueo de Python 3.12 documentado en `backend_tests_execution.md`. Toda la verificación de esta fase se basó en **lectura exhaustiva y análisis estático del código fuente real**, no en ejecución dinámica. Esto es una limitación honesta a declarar: **no se afirma que el backend "funcione" en el sentido de haberlo visto correr — se afirma que su código es coherente, completo en la mayoría de aspectos, y arquitectónicamente sólido, con hallazgos concretos y verificables por lectura.**

## 21. Matriz RF/RNF/HU
Completa en `backend_rf_rnf_traceability.md`. De 18 RF: 13 implementados integralmente (backend+frontend), 2 solo evaluables en backend por naturaleza (RF-01/02/04/05, captura/transporte), 1 no evaluable (RF-06, firmware fuera de repo), 1 parcial en ambos (RF-13, sin PDF), 1 con defecto único de frontend (RF-18). De 10 RNF: 6 implementados integralmente, 3 no evaluables sin instrumentación real (RNF-01/02/10), 1 con defecto crítico confirmado (RNF-03).

## 22. Hallazgos priorizados
13 hallazgos: 1 CRÍTICO (B-01, concurrencia de hash), 4 ALTO (B-02 checklist ausente, B-03 PDF ausente, B-04 sin deduplicación, B-05 sensor null→0.0°C), 5 MEDIO (B-06 a B-11), 2 BAJO (B-12, B-13).

## 23. Respuestas a las 20 preguntas obligatorias

1. **¿El JWT del frontend es aceptado y validado realmente por el backend?** Sí — verificado en código (`get_current_user`, claims exigidos, revocación por jti).
2. **¿Los tres roles se aplican server-side?** Sí — verificado endpoint por endpoint (`backend_rbac_matrix.md`), coincide exactamente con el frontend.
3. **¿El ticket SSE existe y es seguro?** Sí — JWT de audiencia separada, un solo uso, vida de 60s.
4. **¿Los eventos SSE coinciden con lo que espera el frontend?** Sí — mismo shape `LecturaResponse`/`LecturaTermica`.
5. **¿Todos los endpoints consumidos por el frontend existen?** Sí, los 13 endpoints que el frontend llama existen y son compatibles.
6. **¿El checklist BPA puede persistirse en backend?** **No — no existe ninguna implementación relacionada.**
7. **¿Existe exportación PDF real?** **No — en ningún componente del sistema.**
8. **¿RF-18 puede obtenerse del backend?** Sí — el campo existe y es filtrable; el frontend simplemente no lo usa.
9. **¿El Random Forest existe realmente?** Sí — artefacto `.pkl` real, entrenado, cargado perezosamente, con inferencia real.
10. **¿Con qué dataset fue entrenado?** Dataset 100% sintético generado en runtime (8000 muestras, semilla fija), con ruido gaussiano inyectado calibrado a las hojas de datos de los sensores reales — no son datos de campo reales.
11. **¿El F1≥0.85 está demostrado?** Sí — F1=0.9659, con un mecanismo de build que falla si no se alcanza el umbral.
12. **¿Existe alguna métrica propia superior a 96%?** Sí — accuracy real de 96.56%, calculada de forma independiente y propia (no tomada de artículos externos). Con la salvedad cronológica de que el entrenamiento que la produce es posterior a la presentación PPTX que mencionó una cifra similar.
13. **¿La clasificación usa IA, reglas o ambas?** Ambas: predicción de Random Forest, con una regla determinista que actúa como salvaguarda de seguridad y nunca permite que la IA subestime un riesgo.
14. **¿Las predicciones se persisten?** Sí — `nivel_riesgo` en `thermal_readings`, `confianza_ia`/`origen_clasificacion` en `traceability_records`.
15. **¿El SHA-256 encadenado es correcto?** Sí en su fórmula y serialización canónica, **pero con una condición de carrera confirmada que compromete la garantía de integridad bajo concurrencia real.**
16. **¿La cadena resiste concurrencia?** **No — ver hallazgo crítico B-01.**
17. **¿Existe endpoint de verificación?** Sí, `GET /api/trazabilidad/verificar`, funcional y correctamente diseñado (aunque puede reportar falsos positivos de ruptura por el hallazgo de concurrencia).
18. **¿La auditoría es real e inmutable?** Real sí; inmutable solo a nivel de aplicación (sin endpoints de modificación), no a nivel de motor de base de datos.
19. **¿Las migraciones crean todas las tablas declaradas?** Sí — 9 tablas, migración y ORM sincronizados sin divergencia detectada.
20. **¿El backend puede ejecutarse y probarse localmente?** **No se pudo confirmar en esta auditoría** por incompatibilidad de versión de Python en el entorno disponible — el código está bien estructurado para ello (existe `.venv`, `requirements-dev.txt`, `pytest.ini`, tests reales), pero la ejecución real queda como tarea pendiente y no verificada.

## 24. Riesgos para sustentación

- El hallazgo crítico de concurrencia de hash (B-01) es el más serio: si el jurado pide una demostración con múltiples dispositivos simulados publicando simultáneamente, existe una probabilidad real de presenciar un falso "cadena rota" en vivo.
- La cronología del overclaim ">96%" (presentación de mayo, entrenamiento real de julio) puede generar una pregunta incómoda si el jurado indaga en fechas — la respuesta honesta disponible es que la cifra actual SÍ tiene sustento real hoy, aunque no lo tenía cuando se presentó.
- Checklist BPA y exportación PDF, dos funcionalidades nombradas explícitamente en TI/backlog, están completamente ausentes — riesgo directo si el jurado pide verlas en funcionamiento.
- No haber podido ejecutar las pruebas reales en esta auditoría significa que el equipo debe hacerlo ellos mismos antes de sustentar, para confirmar que efectivamente pasan (no se puede asumir).
- El dataset sintético del Random Forest debe comunicarse con precisión — es una práctica válida pero debe explicarse, no ocultarse, para evitar que parezca una omisión si el jurado pregunta por el origen de los datos.

## 25. Veredicto del backend

**Parcialmente implementado.** (Corregido — ver nota de actualización abajo.)

> El sistema presenta una base técnica sólida y componentes reales en autenticación JWT, RBAC server-side, SSE y clasificación mediante Random Forest. Sin embargo, permanece parcialmente implementado respecto del alcance oficial debido a la ausencia de funcionalidades exigidas, la falta de verificación reproducible de pruebas y un defecto crítico de concurrencia que compromete la trazabilidad digital verificable.

Justificación: la arquitectura es genuinamente sólida (DDD real, seguridad ejemplar, IA real y validada con rigor estadístico, hash encadenado correctamente diseñado en su fórmula), y la gran mayoría de RF/RNF centrales de la tesis están implementados de forma completa y verificable por código — no hay fachada. Pero el veredicto no puede calificarse por encima de "Parcialmente implementado" porque: existe **un hallazgo crítico sin corregir** (concurrencia de la cadena hash, incumplimiento directo de RNF-03), **HU-37 y HU-38 están ausentes de extremo a extremo**, falta deduplicación MQTT, el tratamiento de sensores `None`→0.0°C puede generar alertas falsas, y **ninguna de las 18 pruebas del backend fue ejecutada realmente** — no hay verificación reproducible de funcionamiento integral.

**Nota de actualización**: el veredicto de esta sección se corrigió de "Funcional con inconsistencias menores" a "Parcialmente implementado" tras revisión — el primero subestimaba el peso de un hallazgo crítico sin corregir, dos requisitos oficiales completamente ausentes, y la ausencia de pruebas ejecutadas reproduciblemente. Ver `audit-output/final/final_audit_report.md` para el veredicto consolidado autoritativo.
