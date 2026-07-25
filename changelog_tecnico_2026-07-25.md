# Changelog técnico — cierre de alcance del sistema

**Fecha:** 25 de julio de 2026
**Repositorios afectados:** `backend`, `frontend`
**Commits:** `523e770` (backend, 47 archivos, +4 025 / −68) y `30e3c52` (frontend, 30 archivos, +4 948 / −302)
**Origen:** ejecución completa de `plan_implementacion_100porciento.md`

| Métrica | Antes | Después |
|---------|-------|---------|
| Pruebas backend | 178 | **340** |
| Pruebas frontend | 0 | **60** |
| Tareas del plan pendientes | 27 | **0** |
| Vulnerabilidades en dependencias de producción | 0 | 0 |

---

## Índice

1. [Qué se implementó y por qué](#1-qué-se-implementó-y-por-qué)
2. [Backend — detalle por módulo](#2-backend--detalle-por-módulo)
3. [Frontend — detalle por módulo](#3-frontend--detalle-por-módulo)
4. [Seguridad](#4-seguridad)
5. [Defectos encontrados y corregidos](#5-defectos-encontrados-y-corregidos)
6. [Decisiones que se apartan del plan](#6-decisiones-que-se-apartan-del-plan)
7. [Estrategia y cobertura de pruebas](#7-estrategia-y-cobertura-de-pruebas)
8. [Validación en sistema real](#8-validación-en-sistema-real)
9. [Trazabilidad con el backlog](#9-trazabilidad-con-el-backlog)
10. [Limitaciones conocidas](#10-limitaciones-conocidas)
11. [Cómo reproducir la verificación](#11-cómo-reproducir-la-verificación)

---

## 1. Qué se implementó y por qué

El plan identificaba cuatro tareas de prioridad **ALTA** como determinantes para
cerrar el alcance documentado de la tesis. Las cuatro estaban ausentes del
código:

| Tarea | HU/RF | Por qué bloqueaba el alcance |
|-------|-------|------------------------------|
| Checklist BPA persistente | HU-37 | Vivía en `localStorage`: se perdía al cambiar de navegador, no era auditable y no dejaba rastro en la cadena de hashes — es decir, **no servía como evidencia de cumplimiento** |
| Exportación PDF de reportes | HU-38, RF-13 | RF-13 exige reportes exportables; solo existía la consulta en pantalla |
| Notificación ante excursión crítica | HU-23 | Una excursión térmica se registraba pero no avisaba a nadie |
| Registro de calibración de sensores | HU-30 | Sin calibración vigente, la lectura no es evidencia metrológicamente válida |

Se implementaron esas cuatro, más las trece de prioridad media/baja y las dos
marcadas como OPCIONAL. Varias tareas del plan ya estaban resueltas en el
repositorio (el plan se redactó contra un estado anterior del frontend); se
verificaron una a una antes de tocar nada, y se documentan como tales.

---

## 2. Backend — detalle por módulo

### 2.1 Checklist BPA (HU-37)

Verificación diaria de Buenas Prácticas de Almacenamiento, anclada al Manual de
BPA (**RM N.º 132-2015/MINSA**).

**Archivos nuevos**

| Capa | Archivo |
|------|---------|
| Dominio | `src/domain/entities/checklist_bpa.py` |
| Dominio | `src/domain/repositories/i_checklist_repository.py` |
| Aplicación | `src/application/use_cases/gestionar_checklist_bpa.py` |
| Infraestructura | `src/infrastructure/database/repositories/checklist_repository.py` |
| Interfaz | `src/interface/api/checklist_router.py` |

**Modelo de datos.** Diez ítems booleanos como columnas con nombre propio, no
como JSON: permite consultar «cuántos días falló el cierre hermético de la
puerta» con SQL, que es justo el tipo de pregunta que hace una inspección.
Restricción `UniqueConstraint("usuario_id", "fecha")` — una verificación por
persona y día.

**Encadenamiento en trazabilidad.** Cada guardado emite un eslabón de la cadena
de hashes, **incluidas las correcciones**, con el campo
`correccion_de_registro_previo: bool` en la carga útil. Sobrescribir en silencio
destruiría la evidencia de qué se declaró primero.

**Endpoints**

```
POST /api/checklist-bpa           → registra o corrige la verificación del día
GET  /api/checklist-bpa           → la del día (null si aún no se registró)
GET  /api/checklist-bpa/historial → rango de fechas
```

El `GET` devuelve `null` y **no 404** cuando aún no se ha registrado: «todavía
sin verificar» es un estado normal del flujo diario, no un error.

**Upsert por SELECT + INSERT/UPDATE** en vez de `ON CONFLICT`: el despliegue es
PostgreSQL y las pruebas corren en SQLite, y una sentencia específica de dialecto
haría que el código probado no fuera el desplegado.

---

### 2.2 Exportación PDF (HU-38, RF-13)

**Archivos nuevos:** `src/infrastructure/pdf/generador_pdf.py` (573 líneas),
`src/application/use_cases/exportar_reporte_bpa_pdf.py`

**Endpoint:** `GET /api/reportes/bpa/pdf` → `StreamingResponse` con
`Content-Disposition` y `Access-Control-Expose-Headers` (sin esta cabecera el
navegador no puede leer el nombre del archivo en una petición de origen cruzado).

**Contenido del documento**

1. Resumen (periodo, dispositivo, % de tiempo en rango)
2. Verificación de integridad de la cadena
3. Verificaciones BPA declaradas
4. Historial de lecturas (máx. 250 filas)
5. Alertas (máx. 100)
6. Cadena de trazabilidad (máx. 60)

Los topes evitan que un rango amplio genere un PDF de miles de páginas que nadie
va a leer y que puede agotar la memoria del proceso.

**Cálculo del % en rango.** Se computa **solo sobre las lecturas que tienen
temperatura** (rango 2–8 °C). Si se contaran las lecturas sin dato, una avería
del sensor se reportaría como incumplimiento térmico, que es una acusación
distinta y falsa.

**Paleta** alineada con el dashboard (pino `#1F4D3D`, crema `#FAF7F0`) para que
el documento se reconozca como salida del sistema.

---

### 2.3 Notificaciones (HU-23)

**Archivo nuevo:** `src/infrastructure/notifications/notificacion_service.py`

Correo (SMTP) y Telegram. Tres decisiones de diseño:

- **Solo al abrirse un episodio crítico.** `_avisar_si_es_apertura_critica()`
  dispara cuando un episodio se abre o reabre, nunca en su continuación. Una
  excursión de tres horas con lectura por minuto generaría 180 avisos; tras el
  segundo, el farmacéutico silencia el canal y deja de ver los importantes.
- **Cooldown por dispositivo** (`notificacion_cooldown_minutos`, 15 por defecto).
- **Nunca interrumpe la ingesta.** El SMTP va en `asyncio.to_thread` y toda
  excepción se registra y se traga: un servidor de correo caído no puede impedir
  que se persista una lectura térmica.

---

### 2.4 Calibración de sensores (HU-30)

Cuatro columnas nuevas en `devices` y dos casos de uso:
`RegistrarCalibracionUseCase` y `ConsultarEstadoCalibracionUseCase`.

```
PATCH /api/dispositivos/{device_id}/calibracion
GET   /api/dispositivos/calibracion/estado     → vigente | por_vencer | vencida
```

La ruta `/calibracion/estado` se declara **antes** que `/{device_id}`; en el orden
inverso, FastAPI interpretaría `calibracion` como un identificador de dispositivo.

---

### 2.5 Ingesta y MQTT

**Validación de timestamp (`lectura_termica.py`).**

```python
DERIVA_FUTURO_MAXIMA = timedelta(minutes=10)
ANTIGUEDAD_MAXIMA    = timedelta(hours=48)
```

Asimetría deliberada. Hacia atrás, 48 h para no descartar el volcado del buffer
del ESP32 tras una caída nocturna — precisamente cuando la evidencia térmica más
importa. Hacia el futuro, 10 min (solo deriva de reloj): un dispositivo no puede
reportar el futuro, y aceptarlo permitiría **desplazar el orden temporal de la
cadena de hashes**. Todo rechazo queda en `audit_logs` como
`LECTURA_RECHAZADA_TIMESTAMP`.

**SSE desde la vía HTTP.** Antes solo el camino MQTT emitía eventos: una lectura
enviada por REST se persistía sin que ninguna pantalla abierta se enterara.
`_emitir_eventos_sse()` replica la semántica exacta del camino MQTT
(`lectura` / `fallo_sensor` / `alerta` / `episodio_actualizado` / `recuperacion`).

**Topic MQTT de eventos** con despacho por tipo y **comprobación antisuplantación**:
si el `device_id` del cuerpo no coincide con el del topic, el mensaje se audita y
se descarta. Los dispositivos desconocidos se auditan, no se registran.

---

### 2.6 Base de datos

`alembic/versions/0006_checklist_calibracion_indices.py` — tabla `checklist_bpa`,
cuatro columnas de calibración y **cuatro índices** sobre las columnas que filtran
las vistas de historial y alertas.

Las migraciones **0002, 0004 y 0005** se convirtieron a `op.batch_alter_table`
(ver defecto 5.1).

---

## 3. Frontend — detalle por módulo

### 3.1 Checklist BPA contra el backend real

`ChecklistBPAPage.tsx` reescrita; `useChecklistBPA.ts` y `ChecklistBPA.ts` nuevos.

El formulario **se rehidrata** con lo ya declarado, de modo que corregir un solo
ítem no obliga a volver a marcar los diez.

Las claves del backend son `snake_case` y dos traducciones existentes usan
`camelCase`. Se mantiene un mapeo explícito (`CLAVE_I18N`) en vez de renombrar:
tocar las claves de i18n rompería traducciones ya revisadas, y tocar las del
backend rompería los registros ya persistidos.

### 3.2 Página de métricas del modelo (RNF-04)

`MetricasIAPage.tsx` — F1 ponderado, validación cruzada, desempeño por clase,
peso de cada variable y **hashes de procedencia** del dataset y del modelo.
Contrasta contra `UMBRAL_F1 = 0.85` y emite veredicto explícito.

### 3.3 Exportación PDF

`useReportesBPA.descargarPdf` con `responseType: 'blob'`; botón como acción
primaria en Reportes.

### 3.4 Calidad y robustez

| Cambio | Efecto medido |
|--------|---------------|
| `ErrorBoundary` global | Una excepción en render ya no deja pantalla en blanco sin indicio de si el monitoreo sigue activo |
| Code-splitting por ruta | ECharts (**606 KB**) sale del paquete inicial; antes retrasaba incluso el login |
| ESLint | `no-explicit-any` en error: `any` desactiva el tipado justo en el límite con la API |
| `react-router` 7.18.1 → 8.3.0 | Cierra **GHSA-qwww-vcr4-c8h2** |

---

## 4. Seguridad

### 4.1 Cierre de sesión con revocación real (B-07)

El logout ahora llama a `POST /api/auth/logout`, que revoca el `jti`. Sin esa
llamada, un JWT copiado del tráfico seguiría siendo válido hasta expirar aunque
el usuario hubiera cerrado sesión.

No se espera la respuesta a propósito: el estado local debe limpiarse aunque la
red falle, o un backend caído dejaría al usuario dentro.

> El primer intento de este arreglo **no funcionaba**. Ver defecto 5.2.

### 4.2 Aviso de expiración de sesión (FS5)

`useExpiracionSesion.ts` + `AvisoExpiracionSesion.tsx`.

`decodificarSesion` normaliza el `exp` del JWT (segundos, RFC 7519) a
milisegundos. Aviso a los 5 minutos, cierre forzado al vencer.

Se usa un **intervalo de un segundo** y no un `setTimeout` calculado de una vez:
con temporizador único, suspender el equipo retrasaría el disparo y la sesión
seguiría viva en pantalla mucho después de haber caducado el token.

Sin este aviso, el usuario descubría la caducidad al pulsar «guardar»: 401 y el
trabajo a medio escribir —una verificación BPA, una acción correctiva— perdido
sin explicación.

### 4.3 Cuotas por endpoint (B13)

El límite global (240 req/min por IP) trata igual a un `GET` barato que a una
verificación de cadena O(n).

| Endpoint | Cuota | Clave | Motivo |
|----------|-------|-------|--------|
| `GET /api/trazabilidad/verificar` | 5 / min | **usuario** | Rehashea la cadena entera sobre una tabla que solo crece |
| `POST /api/lecturas` | 120 / min (configurable) | IP | Vía REST secundaria |

**Por usuario y no por IP** en la verificación: en una farmacia todos los puestos
salen por la misma dirección, y con clave por IP el primero en verificar dejaría
sin cuota a los demás.

**No compromete el RNF-07** (sync del buffer ≤30 s tras reconectar): ese reenvío
viaja por MQTT (RF-05/RF-06) y no atraviesa la pila HTTP.

Los limitadores viven en `app.state`, no a nivel de módulo, para que cada
instancia arranque con la cuota limpia. Global, las pruebas heredarían cuota
agotada y fallarían de forma intermitente según el orden.

### 4.4 Verificación de integridad de solo lectura en el PDF

`GET /api/trazabilidad/verificar` tiene efectos colaterales **por diseño**: evento
de emergencia encadenado, snapshot forense y flag global. Reutilizarlo tal cual en
la exportación habría hecho que **descargar un reporte disparara eventos
forenses** y marcara la cadena como comprometida.

El caso de uso del PDF instancia el verificador **sin repositorio de corrupción**.
Cubierto por `test_descargar_pdf_no_dispara_evento_de_corrupcion`.

### 4.5 Escaneo de dependencias

- `pip-audit`: sin vulnerabilidades.
- `npm audit --omit=dev`: **0 vulnerabilidades**.
- `npm audit` completo reporta 5 altas (`minimatch`) del árbol transitivo de
  ESLint. Son **dependencias de desarrollo**: nunca se empaquetan ni llegan al
  navegador. Conviene tener presente la distinción durante la sustentación.

---

## 5. Defectos encontrados y corregidos

Ninguno figuraba en el plan. Los cinco se detectaron al escribir pruebas o al
validar en navegador.

### 5.1 Las migraciones solo funcionaban en PostgreSQL

`alembic upgrade head` fallaba desde cero:

```
NotImplementedError: No support for ALTER of constraints in SQLite dialect
```

Las migraciones 0002, 0004 y 0005 usaban `create_unique_constraint` y
`create_foreign_key` directos.

**Por qué la suite no lo veía:** `conftest` construye el esquema con
`Base.metadata.create_all`, que **no ejecuta ni una migración**. Un modelo podía
divergir de su migración sin que nadie se enterara hasta desplegar.

**Arreglo:** conversión a `op.batch_alter_table` — en PostgreSQL emite exactamente
los mismos `ALTER` — y `tests/integration/test_migraciones.py`, que corre las
migraciones de verdad.

### 5.2 Condición de carrera en el cierre de sesión

La petición de revocación salía **sin cabecera de autorización**: el interceptor
de Axios lee el token en un microtask, y `setAccessToken(null)` se ejecuta antes.
El servidor habría respondido 401 y el `jti` habría quedado sin revocar — es
decir, **el arreglo de seguridad habría sido puramente cosmético**.

```typescript
// La cabecera se fija a mano en vez de dejarla al interceptor.
void apiClient.post('/api/auth/logout', undefined, {
  headers: { Authorization: `Bearer ${token}` },
})
```

### 5.3 El checklist no permitía registrar un hallazgo

El botón de guardar quedaba deshabilitado mientras no estuvieran los diez ítems
marcados. Un farmacéutico que encontrara una no conformidad —**justo el caso que
hay que documentar**— no podía guardarla. El backend sí lo permitía.

Un ítem sin marcar es ahora una **declaración de no conformidad**, y la
verificación se registra como «con observaciones».

*Detectado en navegador, no por pruebas unitarias.*

### 5.4 Pruebas con fechas de calendario fijas

Cinco pruebas usaban timestamps absolutos (`datetime(2026, 7, 22, …)`). Eran
bombas de tiempo latentes que la nueva validación de timestamp hizo estallar.
Ancladas al reloj actual conservando lo que verifican.

### 5.5 `agregar()` de usuarios descartaba la mitad de la entidad

`SQLAlchemyUsuarioRepository.agregar` construía el `UserModel` solo con nombre,
email, hash y rol. Los campos de privacidad (HU-44) y de ciclo de vida (HU-45)
los rellenaban los `default` del modelo:

| Se pedía | Se guardaba |
|----------|-------------|
| `is_active=False` | **activo** — podía iniciar sesión |
| `privacy_accepted=False` | **aceptado** — se daba por consentido a quien no consintió |
| `privacy_accepted_at`, `privacy_version_accepted` | `NULL` — sin evidencia de cuándo ni a qué política |

Fallaba **en silencio**: `agregar` devolvía la entidad releída del propio modelo
y por tanto siempre coherente consigo misma.

**Por qué la suite no lo veía:** creaba usuarios siempre con los valores por
defecto, que coinciden con los del modelo.

**Cómo se detectó:** apareció el modal de privacidad durante la validación de FS5
con un usuario creado como ya consentido.

**Arreglo:** persistir los trece campos. `_to_entity` leía trece y `agregar`
escribía cuatro — esa asimetría *era* el defecto. La prueba
`test_agregar_no_pierde_ningun_campo_de_la_entidad` recorre los campos del
dataclass de forma genérica, así que un campo nuevo olvidado en `agregar` la
rompe sin escribir nada más.

---

## 6. Decisiones que se apartan del plan

| Plan proponía | Se hizo | Razón |
|---------------|---------|-------|
| WeasyPrint | **ReportLab** | WeasyPrint exige `libpango` y `libcairo` del sistema; `ctypes.util.find_library` devuelve `None` para ambas en el equipo de desarrollo — el backend no habría arrancado. ReportLab es un *wheel* puro |
| Checklist de 6 ítems | **10 ítems** | El frontend ya mostraba diez, traducidos y anclados a la RM N.º 132-2015/MINSA. Superconjunto del plan; evita regresión de interfaz |
| Rechazar lecturas > 2 h | **48 h atrás / 10 min adelante** | 2 h descartaría en silencio el volcado del buffer tras una caída nocturna |
| Ingesta 120/min *y* verificación 5/min por IP | Verificación **por usuario** | Con clave por IP, un usuario agotaría la cuota de toda la farmacia |

---

## 7. Estrategia y cobertura de pruebas

### 7.1 Backend — 340 pruebas

| Archivo (nuevo en este cambio) | N.º | Qué fija |
|--------------------------------|-----|----------|
| `test_rbac_cobertura.py` | 79 | Barrido RBAC de los 36 endpoints |
| `test_ingesta_timestamp_y_sse.py` | 12 | Ventana de timestamp y emisión SSE por HTTP |
| `test_rate_limit_endpoints.py` | 11 | Cuotas por endpoint |
| `test_checklist_bpa_api.py` | 11 | HU-37 completa |
| `test_notificacion_service.py` | 10 | SMTP, Telegram, cooldown, fallos tragados |
| `test_mqtt_eventos.py` | 8 | Topic de eventos y antisuplantación |
| `test_usuario_repository_persistencia.py` | 7 | Fidelidad de `agregar` (defecto 5.5) |
| `test_reporte_pdf_api.py` | 7 | Generación de PDF y ausencia de efectos colaterales |
| `test_calibracion_api.py` | 7 | HU-30 |
| `test_notificacion_episodio.py` | 5 | Aviso solo al abrir episodio |
| `test_migraciones.py` | 5 | `alembic upgrade head` desde cero |

**El barrido RBAC se genera desde el esquema OpenAPI.** La enumeración de rutas
sale de `app.openapi()["paths"]`, no de una lista escrita a mano: un endpoint
nuevo entra automáticamente en el barrido y, si no declara roles, la prueba lo
delata.

### 7.2 Frontend — 60 pruebas (Vitest + Testing Library)

| Archivo | N.º | Qué fija |
|---------|-----|----------|
| `application/stores/authStore.test.ts` | 14 | Login, logout, revocación del `jti`, consentimiento |
| `application/hooks/useExpiracionSesion.test.ts` | 12 | Cuenta atrás, expiración, suspensión del equipo |
| `presentation/pages/ChecklistBPAPage.test.tsx` | 9 | Diez ítems, rehidratación, no conformidades |
| `presentation/pages/LoginPage.test.tsx` | 7 | Formulario, credenciales, 429, aviso de sesión |
| `presentation/guards/RouteGuards.test.tsx` | 7 | RBAC, jerarquía de administrador, `roles={[]}` |
| `application/hooks/useChecklistBPA.test.ts` | 6 | Carga, `null` como «sin verificar», guardado y fallos |
| `infrastructure/apiClient.test.ts` | 5 | Cabecera de autorización, 401 frente a 403 |

**Se sustituye el adaptador de Axios, no el módulo `apiClient`.** Así la petición
atraviesa los interceptores reales — que es donde vivía el defecto 5.2 — y se
puede afirmar sobre lo que *realmente* saldría por la red.

### 7.3 Verificación por mutación

Una prueba verde que no puede romperse no demuestra nada. Las pruebas de los tres
defectos con regresión se validaron **reintroduciendo el defecto a propósito**:

| Defecto reintroducido | Resultado |
|-----------------------|-----------|
| 5.2 — cabecera de logout | 1 prueba `REGRESIÓN` en rojo |
| 5.3 — botón de checklist bloqueado | 1 prueba `REGRESIÓN` + 2 colaterales en rojo |
| 5.5 — `agregar()` incompleto | **6 de 7** en rojo |

En los tres casos el código se restauró y la suite volvió a verde.

---

## 8. Validación en sistema real

Sistema completo levantado (backend + frontend + base de datos migrada desde
cero) y recorrido con navegador real controlado por Playwright.

**Recorrido funcional**

- Inicio de sesión y consentimiento de privacidad (Ley N.° 29733)
- Ingesta de **90 lecturas** con perfil térmico realista: normal, riesgo
  preventivo y una excursión crítica sostenida
- Dashboard: KPIs, curva térmica, clasificación de IA con confianza y versión
- Checklist BPA: se comprobó que la página **carga desde el backend** el estado
  exacto guardado (9/10, «con observaciones», texto de la observación y sello de
  última modificación)
- Métricas IA: F1 ponderado **0.9671**, validación cruzada 0.9727 ± 0.0023
- Reportes: 90 lecturas, 3 alertas, 123 registros de trazabilidad
- **PDF descargado desde la interfaz**, verificado byte a byte contra el
  descargado por API: **23 117 bytes**, idéntico. Confirma que ambas vías
  producen el mismo documento
- Registro del servidor durante toda la sesión: **solo 200 y 201**, ninguna
  excepción

**Validación específica de FS5** (sesión con token de 4 minutos, capturas en
`evidencia/`):

1. `fs5-aviso-expiracion.png` — banda con cuenta atrás
2. `fs5-banner-limpio.png` — integración con el dashboard, contador decreciendo
3. `fs5-logout-forzado.png` — redirección automática a `/login` con el aviso

Consola del navegador: **cero errores**.

---

## 9. Trazabilidad con el backlog

| HU / RF / RNF | Estado tras este cambio |
|---------------|-------------------------|
| HU-23 — Notificación de excursión | ✅ correo + Telegram, solo al abrir episodio |
| HU-30 — Calibración de sensores | ✅ registro y estado (vigente/por vencer/vencida) |
| HU-37 — Checklist BPA | ✅ persistido y encadenado en trazabilidad |
| HU-38 — Reportes exportables | ✅ PDF |
| HU-43, HU-44, HU-45, HU-47 | ✅ ya existían; HU-44/HU-45 reforzadas por el defecto 5.5 |
| RF-13 — Reportes BPA exportables | ✅ |
| RF-14 / RF-15 — Cadena SHA-256 y verificación | ✅ reforzados: checklist encadenado y verificación con cuota |
| RF-17 — JWT + RBAC | ✅ barrido de 79 pruebas sobre 36 endpoints |
| RF-18 — Estado de conectividad | ✅ ya existía |
| RNF-04 — F1 ≥ 0.85 | ✅ 0.9671, visible en la interfaz |
| RNF-06 — Ningún endpoint sin JWT | ✅ verificado por barrido OpenAPI |

---

## 10. Limitaciones conocidas

Se declaran de forma explícita para que no se confundan con alcance cubierto.

**Firmware del ESP32.** No hay código de dispositivo en el repositorio: cero
`.ino`, `.cpp` o `platformio.ini`. Existe `backend/scripts/simulador_esp32.py`,
que es un **simulador en Python**, no firmware. RF-01 a RF-06 (SHT31, DS18B20,
reed switch, MQTT/TLS, buffer LittleFS) tienen su lado servidor implementado y
probado, pero no su lado dispositivo.

**MQTT sin broker real.** `MQTT_ENABLED=false` en toda la suite y en la
validación E2E. Las 8 pruebas del manejador inyectan mensajes directamente, sin
pasar por EMQX ni TLS. En consecuencia, **RNF-05 (TLS obligatorio) y RNF-07
(sync ≤30 s) no están demostrados empíricamente**.

**Estado de seguridad en memoria y de un solo proceso.** La revocación de `jti`
y los limitadores de tasa viven en memoria del proceso. En un despliegue con más
de una réplica, el logout dejaría de revocar de forma efectiva y el límite se
multiplicaría por réplica. Sustituible por Redis sin tocar la capa de dominio.

**Datos sintéticos.** Las 90 lecturas de la validación y el F1 de 0.9671 provienen
de datos generados, no de un refrigerador real.

**Cobertura de frontend parcial.** 7 archivos con pruebas. Dashboard, Reportes,
Trazabilidad, Dispositivos, Usuarios, Auditoría, Firmware y Métricas IA no tienen
pruebas automatizadas; se validaron manualmente en navegador.

---

## 11. Cómo reproducir la verificación

```bash
# ── Backend ──────────────────────────────────────────────
cd backend
.venv312/bin/python -m pytest -q          # 340 en verde
.venv312/bin/pip-audit                    # sin vulnerabilidades

# Migraciones desde cero (comprueba portabilidad SQLite/PostgreSQL)
.venv312/bin/alembic upgrade head

# ── Frontend ─────────────────────────────────────────────
cd frontend
npm run test:run                          # 60 en verde
npm run typecheck                         # sin errores
npm run lint                              # sin incidencias
npm run build                             # compila
npm audit --omit=dev                      # 0 vulnerabilidades
```

---

**Documentos relacionados**

- `plan_implementacion_100porciento.md` — plan de origen
- `implementacion_aplicada_2026-07-25.md` — informe de ejecución del plan
- `evidencia/` — capturas de la validación en navegador
