# Hallazgos — Backend (Fase 3)

## B-01 [CRÍTICO] Condición de carrera en la cadena de hash puede producir falso positivo de "cadena rota"

- **Categoría**: Corrupción de datos / trazabilidad.
- **Archivo**: `src/infrastructure/database/repositories/trazabilidad_repository.py::obtener_ultimo_hash()` + `src/application/use_cases/registrar_hash_encadenado.py::execute()`.
- **Evidencia**: `SELECT hash_actual ORDER BY created_at DESC LIMIT 1` sin `FOR UPDATE`, seguido de un `INSERT` en una transacción separada por caso de uso, sin candado ni aislamiento serializable.
- **Descripción**: bajo dos escrituras concurrentes (dos lecturas MQTT casi simultáneas de dispositivos distintos, o una lectura y una acción correctiva del mismo instante), ambas transacciones pueden leer el mismo `previous_hash` antes de que la otra haga commit, generando una bifurcación de la cadena. Escenario reconstruido paso a paso en `backend_hash_chain_analysis.md`.
- **Impacto**: `VerificarIntegridadRegistroUseCase` (RF-15) puede reportar `integra=False` para una cadena que nunca fue alterada maliciosamente — invalida la garantía de RNF-03 (integridad 100%) bajo condiciones de operación normales y realistas (múltiples dispositivos publicando MQTT casi simultáneamente es el escenario esperado en producción, no un caso extremo).
- **Requisito afectado**: RNF-03, RF-14, RF-15.
- **Correspondencia frontend**: la UI de `/trazabilidad` mostraría "Cadena Rota" al usuario sin que haya habido ninguna manipulación real — un falso positivo alarmante para un auditor/farmacéutico.
- **Recomendación**: `SELECT ... FOR UPDATE` sobre una fila puntero, o aislamiento `SERIALIZABLE` con reintento, o una columna de secuencia autoincremental con restricción única como verdadero orden de la cadena.
- **Estado epistemológico**: Verificado en código (análisis lógico de la secuencia de operaciones SQL/transaccionales, no reproducido con una prueba de concurrencia real ejecutada — ver `backend_tests_execution.md` para la limitación de entorno que impidió una prueba dinámica).

## B-02 [ALTO] Checklist BPA (HU-37) completamente ausente en backend

- **Categoría**: Funcionalidad oficial ausente.
- **Archivo**: N/A (ausencia transversal, confirmada por `grep` exhaustivo).
- **Descripción**: ni entidad, ni tabla, ni endpoint, ni caso de uso relacionado con "checklist BPA" existe en el backend.
- **Impacto**: HU-37 no está implementada de extremo a extremo (corrige/confirma el hallazgo F-01 de Fase 2, que solo pudo constatar la ausencia del lado frontend).
- **Requisito afectado**: RF-13 (parcial), HU-37.
- **Correspondencia frontend**: `ChecklistBPAPage` simula la función completa vía `localStorage`.
- **Recomendación**: implementar entidad + tabla + endpoint + caso de uso, firmado con JWT, integrado con trazabilidad.
- **Estado epistemológico**: Verificado en código.

## B-03 [ALTO] Exportación PDF (HU-38) completamente ausente en backend

- **Categoría**: Reporte PDF ausente.
- **Archivo**: `requirements.txt` (sin librería PDF), `report_exports.archivo_url` (campo sin poblar en ningún punto de llamada).
- **Descripción**: el backend separa correctamente "obtener datos" de "generar documento", pero la segunda mitad nunca se implementó.
- **Impacto**: HU-38/RF-13(PDF) ausente de extremo a extremo.
- **Requisito afectado**: RF-13, HU-38.
- **Recomendación**: agregar generación de PDF (server-side, p. ej. con `weasyprint`/`reportlab`) y poblar `archivo_url` con la ubicación del archivo generado.
- **Estado epistemológico**: Verificado en código.

## B-04 [ALTO] Sin restricción de deduplicación en `thermal_readings` — reenvío MQTT genera lecturas duplicadas

- **Categoría**: Validación incompleta / integridad de datos.
- **Archivo**: `src/infrastructure/database/models.py::ThermalReadingModel` (sin constraint única sobre `device_id`+`timestamp`).
- **Descripción**: un reenvío de QoS1 (PUBACK perdido, escenario documentado en HU-11 del backlog) produciría un registro duplicado real, con su propio eslabón de hash independiente.
- **Impacto**: infla el historial, duplica alertas potencialmente, y agrega ruido a las métricas de disponibilidad/lecturas del reporte BPA.
- **Requisito afectado**: RF-07, calidad de datos general.
- **Recomendación**: restricción única `(device_id, timestamp)` o deduplicación explícita a nivel de aplicación antes de insertar.
- **Estado epistemológico**: Verificado en código (esquema + ausencia de lógica de deduplicación en el use case de ingesta).

## B-05 [ALTO] Sensor caído (`None`) tratado como 0.0°C para fines de clasificación de riesgo

- **Categoría**: Lógica de negocio incorrecta.
- **Archivo**: `src/application/use_cases/clasificar_riesgo_termico.py`, línea `temperatura_interna = lectura.temperatura_interna or 0.0`.
- **Descripción**: si `temperatura_interna` llega como `None` al backend, se sustituye por 0.0°C (fuera del rango 2-8°C) para el cálculo de riesgo, en vez de tratarse como "dato ausente".
- **Impacto**: puede generar una alerta de excursión crítica **falsa**, basada en la ausencia de lectura, no en una condición térmica real — un riesgo directo si el firmware alguna vez enviara `null` explícitamente (aunque el firmware, según el backlog, debería filtrar estos casos antes de construir el payload, el backend no tiene una segunda línea de defensa).
- **Requisito afectado**: RF-08, calidad de la clasificación de riesgo.
- **Recomendación**: distinguir explícitamente "sensor sin dato" de "0.0°C real", marcando la lectura como no clasificable en ese caso (patrón que HU-18 del backlog sí contempla: "no clasificable" cuando faltan valores).
- **Estado epistemológico**: Verificado en código.

## B-06 [MEDIO] Ingesta HTTP no emite evento SSE (asimetría con ingesta MQTT)

- **Categoría**: Inconsistencia funcional.
- **Archivo**: `src/interface/api/lecturas_router.py::ingestar_lectura` — no llama a `broadcaster.publicar(...)` tras persistir, a diferencia de `_procesar_mensaje_mqtt` en `main.py`.
- **Impacto**: una lectura ingresada por HTTP no actualiza el dashboard en tiempo real; el usuario solo la vería al recargar/re-consultar.
- **Requisito afectado**: RF-11 (parcial, solo para el camino HTTP).
- **Recomendación**: extraer la difusión SSE al propio `RegistrarLecturaTermicaUseCase` o llamarla también desde el router HTTP.
- **Estado epistemológico**: Verificado en código.

## B-07 [MEDIO] Frontend nunca invoca `POST /api/auth/logout` — la revocación server-side nunca se ejercita en la práctica

- **Categoría**: Contrato subutilizado / seguridad.
- **Archivo**: `frontend/src/application/stores/authStore.ts::logout()` (solo limpia estado en memoria), vs. `backend/src/interface/api/auth_router.py::logout` (existe y revoca por jti).
- **Descripción**: el backend implementó una revocación real de tokens al hacer logout, pero el frontend nunca llama a ese endpoint — el "logout" del usuario, desde la perspectiva del servidor, no invalida el JWT, que sigue siendo válido hasta su expiración natural (60 min) si fuera interceptado antes de esa acción.
- **Impacto**: la mejora de seguridad más sofisticada del backend (revocación server-side) queda sin efecto práctico en el flujo real de la aplicación.
- **Requisito afectado**: RNF-06 (seguridad de sesión).
- **Recomendación**: hacer que `authStore.logout()` llame primero a `apiClient.post('/api/auth/logout')` antes de limpiar el estado local.
- **Estado epistemológico**: Verificado en código (ambos lados).

## B-08 [MEDIO] Endpoints de IA (`/api/ia/modelo`, `/api/ia/clasificar`) implementados pero nunca consumidos por el frontend

- **Categoría**: Funcionalidad no explotada.
- **Descripción**: el backend expone evidencia real de RNF-04 (métricas de entrenamiento) vía API, resolviendo por completo el gap detectado en Fase 2 (F-03) — pero como el frontend nunca llama a este endpoint, la evidencia sigue siendo invisible para el usuario/jurado en la aplicación en ejecución.
- **Impacto**: oportunidad perdida de mostrar evidencia concreta de RNF-04 en vivo durante la sustentación.
- **Requisito afectado**: RNF-04.
- **Recomendación**: agregar una vista (aunque sea solo para rol farmacéutico/administrador) que consuma `GET /api/ia/modelo` y muestre F1/accuracy/fecha de entrenamiento.
- **Estado epistemológico**: Verificado en código.

## B-09 [MEDIO] Topic MQTT `farmacias/+/eventos` suscrito sin lógica de procesamiento diferenciada

- **Archivo**: `src/infrastructure/mqtt/mqtt_client.py::TOPIC_EVENTOS`, `src/interface/main.py::_procesar_mensaje_mqtt`.
- **Descripción**: cualquier mensaje que no sea una `LecturaPayload` válida en este topic fallaría la validación Pydantic sin ser capturada explícitamente como un tipo de evento distinto (LWT, conectividad, etc.) — solo hay un manejador genérico de excepciones que descarta silenciosamente.
- **Impacto**: eventos de conectividad/LWT del firmware (si se publican en este topic) no se procesarían ni auditarían.
- **Requisito afectado**: RF-18 (estado de conectividad, en su componente de detección de eventos LWT).
- **Recomendación**: definir un esquema Pydantic para eventos de conectividad y un manejador dedicado para `TOPIC_EVENTOS`.
- **Estado epistemológico**: Verificado en código.

## B-10 [MEDIO] Sin validación de timestamp futuro/antiguo en la ingesta

- **Archivo**: `src/domain/entities/lectura_termica.py::es_lectura_valida()` — solo valida rangos físicos de temperatura/humedad, no el timestamp.
- **Impacto**: una lectura con reloj de dispositivo desincronizado se aceptaría sin objeción, pudiendo distorsionar el orden cronológico visible y el cálculo de features basadas en tiempo (duración fuera de rango, tendencia).
- **Requisito afectado**: RF-07, calidad de datos.
- **Recomendación**: rechazar o marcar como sospechosas las lecturas con timestamp fuera de una ventana razonable (ej. ±10 minutos respecto al reloj del servidor, considerando el buffer offline).
- **Estado epistemológico**: Verificado en código.

## B-11 [MEDIO] Índices y restricciones de esquema mejorables

Ver detalle completo en `backend_database_schema.md`: falta índice compuesto `(device_id, timestamp)` en `thermal_readings`, falta índice en `created_at` para `traceability_records`/`audit_logs` (usado para ordenar/paginar).

## B-12 [BAJO] `password_min_length` declarado pero no confirmado aplicado en validación

Ver `backend_security_review.md`. No se verificó con evidencia directa que el schema de creación de usuario aplique el mínimo configurado.

## B-13 [BAJO] Sin ejecución real de pruebas por bloqueo de entorno (Python 3.12 no disponible)

Documentado exhaustivamente en `backend_tests_execution.md`. No es un defecto del código del proyecto, es una limitación del entorno de esta auditoría — pero significa que **ninguna afirmación sobre "las pruebas pasan" puede hacerse con evidencia de ejecución real** en este informe.

## Hallazgos positivos (contexto, no defectos)

- RBAC server-side genuinamente real y verificado, coincide exactamente con las restricciones del frontend.
- JWT con claims completos, revocación real, tickets SSE de un solo uso con audiencia separada — ingeniería de seguridad notablemente cuidadosa para un proyecto de tesis.
- Random Forest real, entrenado con metodología correcta (ruido inyectado para evitar circularidad, validación cruzada, umbral RNF-04 aplicado como gate de build), con salvaguarda determinista de seguridad clínica.
- Hash SHA-256 encadenado con serialización canónica correcta, timestamp canonicalizado, tests unitarios reales.
- Arquitectura DDD genuina (no cosmética), dirección de dependencias verificada correcta en todos los módulos leídos.
- Migraciones y ORM perfectamente sincronizados.
- Postura de seguridad de configuración (validadores que impiden arrancar en producción con secretos por defecto) ejemplar.
- Contratos frontend-backend prácticamente sin discrepancias — desarrollo evidentemente coordinado entre ambos lados.
