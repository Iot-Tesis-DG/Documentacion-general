# Changelog técnico — migración al backlog de 54 HU

**Fecha:** 19 de septiembre de 2026
**Repositorios afectados:** `backend`, `frontend`, `iot-firmware`
**Rama:** `feat/51hu-audit-implementation` (los tres repos)
**Commits:**

| Repo | Commit | Archivos | Líneas |
|------|--------|----------|--------|
| backend | `109fc5e` | 58 | +1436 / −456 |
| frontend | `5ab3d75` | 21 | +1363 / −122 |
| iot-firmware | `408360d` | 11 | +326 / −7 |

**Origen:** `Backlog_Historias_Usuario.md` (Anexo 05, versión revisada — 54 HU en
6 épicas, 275 puntos, 10 sprints), contrastado contra el código real, que
correspondía a una versión **anterior** del backlog (numeración y contenido
distintos, documentada en `backlog_traceability.md` y
`analisis_final_estado_actual.md`).

| Métrica | Antes | Después |
|---------|-------|---------|
| Pruebas backend | 550 | **557** |
| Pruebas frontend | ~178 | **208** |
| Clases del modelo Random Forest | 3 (incluía `excursion_critica`) | **2** (`normal`, `riesgo_preventivo`) |
| F1 ponderado del modelo | 0.9659 (3 clases) | **0.9757** (binario) |
| Cadenas de hash SHA-256 | 1 global | **1 por unidad monitoreada** (`chain_id`) |
| Migraciones Alembic | 0012 | **0014** |

---

## Índice

1. [Por qué — hallazgo inicial](#1-por-qué--hallazgo-inicial)
2. [Backend — modelo de datos base](#2-backend--modelo-de-datos-base)
3. [Backend — inteligencia artificial](#3-backend--inteligencia-artificial)
4. [Backend — notificaciones (HU-53/HU-54)](#4-backend--notificaciones-hu-53hu-54)
5. [Backend — continuidad eléctrica (HU-52)](#5-backend--continuidad-eléctrica-hu-52)
6. [Backend — auditoría de páginas no revisadas y correcciones](#6-backend--auditoría-de-páginas-no-revisadas-y-correcciones)
7. [Firmware ESP32](#7-firmware-esp32)
8. [Frontend](#8-frontend)
9. [Limitaciones conocidas / trabajo no realizado](#9-limitaciones-conocidas--trabajo-no-realizado)
10. [Cómo reproducir la verificación](#10-cómo-reproducir-la-verificación)

---

## 1. Por qué — hallazgo inicial

El usuario entregó `Backlog_Historias_Usuario.md`: 54 historias de usuario en
6 épicas, con contenido revisado respecto a lo que el código implementaba.
Antes de tocar nada se auditó el estado real contra el backlog nuevo (tres
auditorías en paralelo: backend, firmware, frontend). Hallazgo principal: el
código ya estaba sustancialmente alineado con una versión previa del backlog
(comentarios en el propio código citan "backlog de 51 HU finales"), pero
tres piezas centrales habían cambiado de fondo:

1. **Trazabilidad**: el backlog nuevo exige una cadena SHA-256 independiente
   por unidad monitoreada (`chain_id`, HU-24/25); el código tenía una única
   cadena global, decisión de diseño documentada explícitamente como tal.
2. **IA**: el backlog nuevo exige un clasificador **binario**
   (`normal`/`riesgo_preventivo`); el modelo en producción predecía 3 clases,
   incluyendo `excursion_critica` como salida del propio Random Forest.
3. **Idempotencia de telemetría**: el backlog nuevo exige
   `device_id + boot_id + seq_no`; el contrato usaba `device_id + timestamp`.

Estrategia acordada con el usuario: **migración por deltas**, empezando por
el modelo de datos base (del que dependen IA y trazabilidad), sin reescribir
lo que ya cumplía.

---

## 2. Backend — modelo de datos base

### 2.1 Cadena de hash por unidad monitoreada (HU-24/HU-25)

**Antes.** `RegistrarHashEncadenadoUseCase` mantenía una única cadena global
(`previous_hash` compartido por lecturas, alertas, auditoría, checklist,
etc.), documentado como decisión de diseño explícita. El hash se calculaba
como `SHA256(previous_hash + timestamp + json(payload))` — sin `chain_id`.

**Después.**

- `src/domain/value_objects/hash_encadenado.py`: fórmula nueva
  `SHA256(canonical({chain_id, previous_hash, timestamp, payload}))`
  (`calcular_hash`), conservando la fórmula anterior como
  `calcular_hash_v1` **solo para verificar registros ya persistidos** —
  ningún hash histórico se recalcula.
- `TraceabilityRecordModel` (`src/infrastructure/database/models.py`): nuevas
  columnas `chain_id`, `chain_seq`, `hash_version` (1 = fórmula legada,
  2 = con `chain_id`).
- `registrar_hash_encadenado.py`: `chain_id` se deriva automáticamente
  (`chain_id_explícito or device_id or "SISTEMA"`) — los ~10 casos de uso que
  ya llamaban a este punto único no necesitaron cambios de firma.
- Candado de escritura (PostgreSQL `pg_advisory_xact_lock`) ahora se deriva
  de `hashtext(chain_id)`: cadenas de dispositivos distintos ya no se
  bloquean entre sí (antes, una clave fija serializaba TODAS las escrituras).
- `VerificarIntegridadRegistroUseCase` (HU-26): sin `chain_id` recorre todas
  las cadenas a la vez (vista de administrador); con `chain_id`, verifica
  solo esa cadena — la forma que exige el criterio literal ("verificación
  completa de una cadena... identificada por chain_id").
- `VerificarIntegridadPorDispositivoYPeriodoUseCase` (HU-37): al ser
  `chain_id = device_id`, dejó de necesitar la lógica de "puede incluir
  bloques intermedios de otro dispositivo" — mejora real de precisión, no
  solo refactor.
- `AislarCorrupcionUseCase` (HU-47): el aislamiento y el reinicio en génesis
  ahora afectan **solo** la cadena del registro corrupto, no todo el sistema.
- Migración `0013_chain_id_boot_seq_time_quality.py`: backfill de
  `chain_id = device_id or "SISTEMA"` y `chain_seq` secuencial por cadena en
  orden de inserción, para las filas existentes (`hash_version=1`).

**Hallazgo posterior (sección 6.1)**: `AuditarAccionCriticaUseCase` —el
punto genérico de auditoría usado por prácticamente todos los routers— no
pasaba `device_id`, así que TODOS los eventos de auditoría (reconocer
alerta, registrar acción correctiva, editar instalación, etc.) caían en la
cadena `SISTEMA` en vez de la cadena del dispositivo. Corregido; ver §6.1.

### 2.2 Identidad real de la lectura (HU-01/HU-05/HU-11)

- `LecturaPayload` (`src/infrastructure/mqtt/payload_schema.py`) y
  `LecturaIngestRequest` (REST): campos nuevos `boot_id`, `seq_no`,
  `time_quality` (`"synced"|"unsynced"`), todos opcionales (compatibilidad
  con firmware que aún no los declara).
- `ThermalReadingModel`: columnas `boot_id`, `seq_no`, `time_quality`,
  `received_at` (instante en que el backend recibió el mensaje, distinto de
  `timestamp`=`captured_at`).
- Idempotencia: `obtener_por_device_boot_seq()` es la clave preferida
  cuando el payload trae `boot_id`+`seq_no`; `obtener_por_device_y_timestamp()`
  se conserva como respaldo para firmware anterior. Índice único parcial
  `(device_id, boot_id, seq_no)` donde ambos no son nulos.
- `received_at` se estampa en `RegistrarLecturaTermicaUseCase.execute()` —
  único punto de entrada compartido por MQTT y REST.

---

## 3. Backend — inteligencia artificial (HU-16/HU-17/HU-18)

**Antes.** `RandomForestClassifier` de 3 clases (`normal`,
`riesgo_preventivo`, `excursion_critica`). La "salvaguarda determinista"
(`reglas_riesgo.clasificar_por_regla`, usada también para etiquetar el
dataset sintético de entrenamiento) podía escalar a `excursion_critica` por
duración o distancia al límite térmico.

**Después.**

- `reglas_riesgo.clasificar_por_regla()`: capado a `normal`/`riesgo_preventivo`
  — la rama que producía `excursion_critica` se eliminó (ya no hacía falta:
  colapsaba al mismo resultado que la rama general de "fuera de rango").
- `random_forest_service._CLASES_ESPERADAS`: restringido a las 2 clases —
  un artefacto que declare `excursion_critica` entre sus clases se **rechaza
  al cargar**, sin importar su exactitud.
- `train_model_v3.py`: `MODEL_VERSION = "4.0.0-binario"`. Reentrenado con la
  regla ya binaria (el dataset sintético usa la misma función para
  etiquetar): **F1 ponderado = 0.9757** sobre 10 076 muestras (8060 train /
  2016 test), muy por encima del umbral RNF-04 (≥0.85) y de los baselines
  (mayoritario 0.5087, árbol simple 0.9683).
- `excursion_critica` sigue existiendo como valor de `riesgo_efectivo`
  (`LecturaTermica.calcular_riesgo_efectivo()`), producido **exclusivamente**
  por la regla determinista de rango 2–8 °C sobre la temperatura actual —
  nunca por el modelo. Esta separación ya existía a nivel de entidad de
  dominio; lo que cambió fue que el pipeline de IA dejara de "fabricarla".

---

## 4. Backend — notificaciones (HU-53/HU-54)

**Antes.** Un único destinatario de correo global (`SMTP_TO`) para todos los
dispositivos. Sin canal SMS.

**Después.**

- `DeviceModel`: columnas nuevas `responsable_nombre`, `responsable_email`,
  `responsable_telefono` (migración `0014_hu53_hu54_responsable_dispositivo.py`).
- `PATCH /api/dispositivos/{id}/responsable` (`ActualizarResponsableDispositivoUseCase`):
  actualiza el contacto y deja un evento en el historial de configuración
  (HU-49) cuando cambia.
- `GenerarAlertaUseCase` recibe `device_repository` opcional y resuelve
  `email_destino`/`telefono_destino` del dispositivo antes de notificar;
  `NotificacionService.notificar_excursion_critica()` usa el email del
  dispositivo si existe, o cae a `SMTP_TO` si no (nunca deja una excursión
  sin avisar a nadie).
- Canal SMS nuevo (`_enviar_sms`): pasarela compatible con la API REST de
  Twilio (Account SID + Auth Token, HTTP Basic). Sin equivalente global — un
  dispositivo sin teléfono registrado simplemente no recibe SMS (a
  diferencia del correo). Settings nuevos: `sms_enabled`, `sms_api_url`,
  `sms_account_sid`, `sms_auth_token`, `sms_from`, con validador de
  producción exigiendo credenciales si está habilitado.

---

## 5. Backend — continuidad eléctrica (HU-52)

Sin hardware real (el circuito de respaldo de 5V no existe todavía), se
implementó el lado backend de la telemetría:

- `TipoEventoDispositivo`: dos valores nuevos, `conmutacion_respaldo` y
  `respaldo_bajo`.
- `_procesar_evento_mqtt` (`src/interface/main.py`): dos `case` nuevos en el
  `match` — se auditan igual que `BUFFER_SATURADO`/`WIFI_RECONEXION_PROLONGADA`
  (sin lógica de negocio especial: el nodo sigue midiendo y persistiendo en
  LittleFS con o sin red, HU-06).
- Aprovechado para corregir el mismo hallazgo de la §2.1: el `auditoria.execute()`
  de este manejador tampoco pasaba `device_id` — corregido.

---

## 6. Backend — auditoría de páginas no revisadas y correcciones

El usuario pidió auditar específicamente `AlertasPage`, `HistorialPage` y
`ReportesPage` (no cubiertas en la primera pasada). Se encontraron 6 gaps
reales, todos corregidos:

### 6.1 Chain_id no se propagaba en eventos de auditoría (HU-23/HU-25/HU-28)

`AuditarAccionCriticaUseCase.execute()` no aceptaba `device_id`, así que
`REVISAR_ALERTA`, `REGISTRAR_ACCION_CORRECTIVA` (el evento de auditoría, no
el de dominio — ese sí encadenaba bien), `RECTIFICAR_ACCION_CORRECTIVA`,
`ACTUALIZAR_INSTALACION_DISPOSITIVO`, `ACTUALIZAR_RESPONSABLE_DISPOSITIVO`,
`EXPORTAR_REPORTE_BPA(_PDF)` y los eventos MQTT de dispositivo caían todos
en la cadena `SISTEMA`. Esto contradecía HU-23 ("trazado en la cadena por
chain_id") y HU-28 ("reconstruir el ciclo de una alerta... mediante la
cadena por chain_id").

**Corrección**: `device_id: str | None = None` agregado a
`AuditarAccionCriticaUseCase.execute()`, propagado a
`RegistrarHashEncadenadoUseCase`. Se actualizaron todos los call sites
device-scoped: `alertas_router.py` (revisar/registrar/rectificar, con
`CorregirAccionCorrectivaUseCase` recibiendo ahora `alerta_repository` para
resolver el `device_id` de la alerta original), `dispositivos_router.py`
(instalación/responsable), `reportes_router.py` (BPA/PDF), `main.py`
(eventos MQTT de dispositivo).

Test nuevo: `test_revisar_alerta_encadena_en_la_cadena_del_dispositivo`
(`test_alertas_api.py`) y `test_conmutacion_respaldo_se_audita_y_encadena_en_el_dispositivo`
(`test_mqtt_eventos.py`) verifican `chain_id == device_id` explícitamente.

### 6.2 HU-23 criterio 4 — sin cronología de atención de la alerta

No existía forma de ver, para una alerta, el reconocimiento + las acciones
correctivas + sus rectificaciones en orden. Backend nuevo:

- `AccionCorrectivaRepository.listar_por_alerta()`: ahora ordena por
  `created_at ASC` (antes sin orden explícito).
- `ConsultarAccionesCorrectivasUseCase` (`consultar_alertas.py`).
- `GET /api/alertas/{id}/acciones-correctivas`.

### 6.3 HistorialPage sin gráfica (HU-36 criterio 1)

El criterio exige que el filtro alimente "tanto la gráfica de Apache ECharts
como la tabla" — solo existía la tabla. Backend no requirió cambios (el
endpoint ya soportaba `desde`/`hasta`/`device_id`/`nivel_riesgo`); ver §8.2.

### 6.4 HistorialPage sin validación de rango en frontend (HU-36 criterio 3)

El backend ya rechazaba `desde > hasta` con 422; el criterio exige que el
frontend TAMBIÉN lo bloquee antes de consultar. Sin cambios de backend; ver
§8.2.

### 6.5 PDF sin fecha de verificación ni digest (HU-38 criterio 2)

`ResultadoVerificacion` no exponía qué hash quedó como último eslabón
verificado. Se agregó `hash_final: str | None` (hash del último registro de
la cadena verificada), y `ExportarReporteBPAPDFUseCase.execute()` ahora:

- Verifica la integridad **acotada al `chain_id` del dispositivo del
  reporte** (antes verificaba el sistema completo, independientemente del
  filtro del reporte).
- Agrega `verificado_en` (timestamp de la generación del PDF) y
  `digest_cadena` (los primeros 16 caracteres del `hash_final`) a la sección
  de integridad del documento (`generador_pdf.py`).

### 6.6 PDF con lenguaje de cumplimiento sanitario (HU-38 criterio 4)

El texto de cierre decía "elaborado... conforme al Manual de BPA", que se
puede leer como una afirmación de cumplimiento. El criterio exige
explícitamente lo contrario. Reescrito: "constituye evidencia del monitoreo
y la trazabilidad térmica... no declara por sí mismo cumplimiento sanitario
integral. La certificación sanitaria corresponde exclusivamente a la
autoridad competente (DIGEMID)."

---

## 7. Firmware ESP32

**No se pudo compilar ni ejecutar `pio test -e native` en este entorno** (sin
PlatformIO CLI ni compilador C++ disponibles) — los cambios se verificaron
por revisión manual cuidadosa, no por ejecución. Se recomienda correr la
suite del host antes de flashear.

### 7.1 Identidad real de la lectura (HU-01/HU-11)

- `core::Lectura` (`PayloadCore.h`): campos nuevos `bootId`, `seqNo`,
  `tiempoSincronizado`; `serializarLectura()` los emite como `boot_id`,
  `seq_no`, `time_quality` en el JSON.
- `system/BootId.{h,cpp}` (archivos nuevos): `cargarEIncrementarBootId()` lee
  y escribe un contador en NVS (mismo namespace que las credenciales, pero
  lectura-escritura a diferencia de `cargarCredenciales()`, que es
  solo-lectura). Primer arranque tras flashear empieza en 1 (0 reservado
  para "no asignado").
- `PayloadBuilder::setBootId()` (estático, se llama una vez en `setup()`,
  antes de crear las tareas — mismo patrón single-writer que `credenciales`);
  `seqNo` es un contador de proceso que solo `taskSensores` incrementa, vía
  `build()`.
- Confirmado por inspección: el buffer offline LittleFS almacena el JSON ya
  serializado (con `boot_id`/`seq_no` ya fijados en el momento de captura),
  así que una lectura re-sincronizada tras una desconexión conserva su
  identidad original — no hizo falta ningún cambio en `LittleFSBuffer`.
- 3 pruebas nuevas en `test/test_core/test_payload.cpp`; golden test
  (`test_payload_coincide_con_el_documentado`) actualizado con los 3 campos.

### 7.2 Continuidad eléctrica — scaffolding (HU-52)

- `config.h`: `RESPALDO_INSTALADO` (0 por defecto, mismo patrón que
  `MC38_INSTALADO`), `PIN_RESPALDO_SENSE` (GPIO27, digital), 
  `PIN_RESPALDO_BATERIA_ADC` (GPIO34), `UMBRAL_RESPALDO_BATERIA_ADC`,
  `INTERVALO_CHEQUEO_RESPALDO_MS` (10 s).
- `system/PowerBackup.{h,cpp}` (archivos nuevos): `revisarEstadoRespaldo()`
  detecta transición mains→respaldo, respaldo→mains, y nivel bajo (una sola
  vez por episodio de corte, vía flag `_avisoNivelBajoEnviado`). Sin
  `RESPALDO_INSTALADO`, no toca ningún pin y siempre reporta "sin cambios".
- `taskRed` (`main.cpp`): nuevo paso 8, revisa el estado cada
  `INTERVALO_CHEQUEO_RESPALDO_MS` y publica `conmutacion_respaldo` /
  `respaldo_bajo` en `/eventos` (best-effort, igual que
  `reportarSaturacionSiHubo`).

**Limitación documentada en el propio código** (no oculta): el chequeo vive
después del `continue` de Wi-Fi/MQTT desconectado (pasos 1-2 de `taskRed`),
así que un corte que se autorresuelve ANTES de que la red se reconecte puede
no dejar rastro del evento de conmutación — la lectura térmica de ese
periodo sí queda íntegra en LittleFS (HU-06), solo el evento de diagnóstico
puede perderse. Es la misma clase de limitación que ya acepta
`WIFI_RECONEXION_PROLONGADA` (solo se puede avisar DESPUÉS de reconectar).
Corregirlo de raíz requiere detección en Core 0 (que corre sin depender de
la red) y una cola cruzada entre núcleos — no se justificó construirlo sin
el circuito real para validar contra qué duración de corte importa.

---

## 8. Frontend

### 8.1 Dashboard (HU-31/HU-32/HU-33)

- **Badge "En vivo" junto al valor** (HU-32 criterio 4): antes el indicador
  SSE vivía solo en la cabecera de la página, lejos de la temperatura
  mostrada. Ahora la tarjeta de temperatura interna muestra
  "En vivo · HH:MM:SS" (o "Desconectado · última lectura HH:MM:SS") debajo
  del valor, con un punto animado igual al de la cabecera.
- **Frescura de KPI** (HU-33 criterio 2): nuevo umbral
  `UMBRAL_FRESCURA_SEGUNDOS = 90` — pasado ese tiempo sin lectura nueva, las
  tres tarjetas de sensor se atenúan y muestran "Sin lectura nueva hace Ns",
  distinto del caso "falla de sensor" (dato ausente) ya existente.
- **Ventana de 24h por defecto + selector** (HU-31 criterio 4):
  `useMonitoreoTermico(horasVentana)` pasó de pedir "las últimas 60
  lecturas" a pedir "las lecturas desde hace `horasVentana` horas" (default
  24). Botones 1h/24h/7d sobre la gráfica para "ampliar o modificar el
  rango" sin salir del dashboard. Techo defensivo de memoria subido de 60 a
  5000 lecturas (≈24h a 30s/lectura cabe cómodo).
- **Downsampling** (HU-31 criterio 3, no estaba pedido explícitamente en la
  auditoría pero es parte del mismo criterio): `sampling: 'lttb'` +
  `large: true` en ambas series de la curva térmica.

### 8.2 HistorialPage (HU-36)

- **Gráfica nueva**: `construirOpcionHistorial()` (calcada de la del
  dashboard, con downsampling incluido) — la misma consulta filtrada
  alimenta ahora la gráfica Y la tabla, ordenada cronológicamente ascendente
  (el backend devuelve descendente).
- **Validación de rango en frontend** (criterio 3): si `desde > hasta`, se
  bloquea el envío con un mensaje inline (`role="alert"`) antes de llamar al
  backend — el 422 del servidor sigue existiendo para una llamada directa a
  la API.

### 8.3 TrazabilidadPage (HU-37)

Formulario nuevo "Verificar por dispositivo y periodo": device_id + rango
`datetime-local`, resultado con el listado de bloques del segmento marcados
íntegro/corrupto. Usa el endpoint `/api/trazabilidad/verificar-dispositivo`
que ya existía en el backend (gap era solo de UI).

### 8.4 AlertasPage (HU-23/HU-28)

Botón "Ver cronología de atención" (visible para alertas reconocidas o
atendidas) abre un diálogo de solo lectura con la línea de tiempo: alerta
generada → reconocida (con responsable) → cada acción correctiva o
rectificación (con responsable, descripción y timestamp).

### 8.5 DispositivosPage (HU-30/HU-51/HU-53/HU-54)

Reescrita para agregar, por fila: botón de calibración (badge de estado
vigente/por vencer/vencida/sin registrar + diálogo de registro), botón de
instalación/ubicación (diálogo + el cambio queda en el historial), botón de
responsable (nombre/email/teléfono, con aviso de qué canales lo usan), y
botón de historial de configuración (diálogo de solo lectura). La columna de
firmware, presente antes, se restauró tras haberse perdido accidentalmente
en la primera versión del rediseño (detectado por el test existente).

### 8.6 Modo demo

`datosDemo.ts` y `demoAdapter.ts` actualizados para no romper con el campo
`corrige_accion_id` nuevo, y para simular el endpoint de listado/rectificación
de acciones correctivas — el modo demo es una build pública y debía seguir
funcionando con las páginas nuevas.

---

## 9. Limitaciones conocidas / trabajo no realizado

| Ítem | Estado | Motivo |
|------|--------|--------|
| HU-48 (indicador de sincronización/brecha LittleFS) | **No implementado** | Requiere que el firmware reporte en tiempo real cuántas lecturas tiene pendientes en el buffer — esa telemetría no existe todavía; sin ella la UI no tiene qué mostrar. |
| HU-52 — apagado ordenado ante nivel de respaldo crítico | **Solo diagnóstico, sin apagado** | Fabricar una secuencia de apagado/deep-sleep sin hardware real para validarla es más riesgoso que útil; se emite el evento `respaldo_bajo` y se documenta como pendiente de validación en piloto. |
| HU-52 — corte breve durante ventana sin red | **Puede no reportarse** | Ver limitación documentada en §7.2. |
| Nombres de roles (`FARMACEUTICO`/`TECNICO` vs `QUIMICO_FARMACEUTICO`/`TECNICO_FARMACIA`) | **Sin cambiar** | Los 4 roles y sus permisos ya son correctos; es una diferencia cosmética de naming, no funcional. |
| Firmware — verificación por ejecución | **No realizada aquí** | Sin PlatformIO CLI ni compilador en este entorno. Ejecutar `pio test -e native` antes de flashear. |
| AlertasPage/HistorialPage/ReportesPage — solo se auditaron estas tres | Resto de páginas (Usuarios, MetricasIA, Checklist, Firmware, Auditoría) no se re-auditaron línea por línea contra el texto exacto del backlog nuevo en esta sesión, aunque el backend que las sostiene ya estaba confirmado alineado en auditorías previas. |

---

## 10. Cómo reproducir la verificación

```bash
# Backend
cd backend
.venv/Scripts/python.exe -m pytest tests/ -q
# 557 passed, 5 errors (flakes de limpieza de archivos temporales en Windows,
# no relacionados con el código — el cuerpo de esos 5 tests pasa).

# Frontend
cd frontend/frontend
npm run typecheck && npm run lint
npx vitest run --pool=threads   # el pool "forks" por defecto puede colgarse
                                 # en sandboxes con recursos limitados; usar
                                 # --pool=threads o correr en lotes si la
                                 # suite completa da timeout por contención.

# Firmware (no ejecutado en esta sesión)
cd iot-firmware
pio test -e native
```

Reentrenamiento del modelo IA (si se vuelve a tocar `reglas_riesgo.py` o el
dataset sintético):

```bash
cd backend
.venv/Scripts/python.exe -m src.infrastructure.ai.train_model_v3
```
