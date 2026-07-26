# Verificación de Cierre — 2026-07-25

> **Alcance**: aplicar las brechas residuales de `analisis_final_estado_actual.md`
> (§4.1, §4.2, §4.3) y `IoT-documentacion_iot.md`, verificar el resultado con
> pruebas ejecutadas, y trazar qué capa se tocó en cada caso.
>
> **Resultado**: 3 brechas cerradas. **1 defecto bloqueante encontrado y
> corregido** que ningún documento previo había detectado. 1 puerta de
> despliegue documentada.

---

## 1. Resumen ejecutivo

| # | Hallazgo | Severidad | Capa | Estado |
|---|----------|-----------|------|--------|
| V-01 | Contrato IoT↔backend roto: el backend rechazaba el 100% de las lecturas del firmware real | **CRÍTICO** | IoT + Backend | ✅ Corregido |
| V-02 | RF-18 no renderizado en el Dashboard | BAJO | Web | ✅ Corregido |
| V-03 | Escaneo de dependencias sin ejecutar | BAJO | Backend + Web | ✅ Ejecutado, limpio |
| V-04 | CSP sin verificar contra el build real | BAJO | Web | ✅ Verificado |
| V-05 | `connect-src 'self'` bloqueará el backend real | **ALTO (diferido)** | Web / Despliegue | ⚠️ Documentado, no aplicable aún |
| V-06 | Deriva de documentación IoT vs código | INFO | Docs | ✅ Corregido |
| **V-07** | **El firmware no compilaba** (4 errores) | **CRÍTICO** | IoT | ✅ Corregido |
| **V-08** | **No existe la partición `littlefs`** → bucle de arranque | **CRÍTICO** | IoT | ✅ Corregido |
| **V-09** | **La telemetría en vivo nunca se publicaba** | **CRÍTICO** | IoT | ✅ Corregido |
| V-10 | QoS 1 / PUBACK inexistentes → pérdida silenciosa | ALTO | IoT | ✅ Mitigado + documentado |
| V-11 | `setMinSupportedTLS` es API inventada (no existe en ESP32) | ALTO | IoT / Seguridad | ✅ Corregido |
| V-12 | Sin indicador de cobertura (§4.6) | BAJO | Backend + Web | ✅ Implementado |
| V-13 | Evento "online" enviaba JSON de lectura al tópico de eventos | ALTO | IoT | ✅ Corregido |
| V-14 | `syncNTP()` se llamaba ANTES de levantar la Wi-Fi | **CRÍTICO** | IoT | ✅ Corregido |
| V-15 | `struct tm` sin inicializar → UB en `mktime()` | MEDIO | IoT | ✅ Corregido |
| V-16 | `MQTT_HOST` sin guarda `#ifndef` (no inyectable) | BAJO | IoT | ✅ Corregido |
| **V-17** | **El MC-38 no detectaba la mayoría de las aperturas** | **ALTO** | IoT | ✅ Corregido |
| V-18 | Un fallo del DS18B20 era permanente hasta reiniciar | ALTO | IoT | ✅ Corregido |
| V-19 | Timeout del DS18B20 con solo 25 % de margen | MEDIO | IoT | ✅ Corregido |
| V-20 | El SHT31 no se recuperaba ni avisaba en el arranque | MEDIO | IoT | ✅ Corregido |
| V-21 | Nombres de archivo truncados en LittleFS (`.jso`) | BAJO | IoT | ✅ Corregido |
| V-22 | El simulador emitía un payload que el ESP32 nunca produce | ALTO | Pruebas | ✅ Corregido |
| V-23 | El watchdog no cubría ninguna de las dos tareas | MEDIO | IoT | ⚠️ Documentado (ver 10-bis) |
| V-24 | La cadena hash ordenaba sin criterio de desempate | MEDIO | Backend | ✅ Corregido |
| V-25 | Vercel despliega siempre la versión demo | INFO | Web / Despliegue | ⚠️ Documentado (ver 10-ter) |
| V-26 | Nueve páginas sin ninguna prueba (no rotas, sin cubrir) | BAJO | Web | ✅ 34 casos añadidos |
| — | Acceso con Google con allowlist (fuera de backlog) | — | Backend + Web | ✅ Implementado (ver 10-quater) |
| V-27 | El backend no tenía ningún análisis estático | MEDIO | Backend | ✅ Ruff, 140 avisos → 0 |
| V-28 | Código muerto en firmware (API de PUBACK que mentía) | BAJO | IoT | ✅ Eliminado |
| V-29 | `MQTT_KEEPALIVE_SEC` definido pero nunca aplicado | MEDIO | IoT | ✅ Corregido |
| V-30 | Ningún repositorio tenía CI | MEDIO | Todos | ✅ 3 workflows |
| V-31 | El aislamiento de corrupción no quedaba en `audit_logs` ni constaba su autor | MEDIO | Backend | ✅ Corregido (ver 10-sexies) |

**Capas tocadas**: IoT (C++), Backend (Python), Web (React), Documentación.
**Capa IA**: no requirió cambios — el pipeline de inferencia no estaba implicado
en ningún hallazgo.

> **Corrección respecto a la primera versión de este informe.** Inicialmente se
> dio el firmware por correcto. Al instalar PlatformIO y ejecutar `pio run` se
> comprobó que **no compilaba**, y con ello aparecieron cuatro defectos más. El
> análisis previo ("~99 % completo", "HU-01 a HU-14 implementadas en el firmware
> ESP32") describía código que nunca se había construido. Lección aplicable a la
> sustentación: *revisar* código no es *ejecutarlo*.

---

## 2. V-01 — Contrato IoT↔backend roto (CRÍTICO)

### El defecto

`src/infrastructure/mqtt/payload_schema.py::LecturaPayload` declara
`model_config = ConfigDict(extra="forbid", ...)`, pero **no declaraba
`duracion_apertura_segundos`**.

`PayloadBuilder::build()` (`iot-firmware/src/payload/PayloadBuilder.cpp:140-146`)
escribe ese campo en **todas** las lecturas — `0` con la puerta cerrada, el
contador cuando está abierta:

```cpp
doc["apertura_refrigerador"] = _doorOpen;

if (_doorOpen && _doorDurationSec > 0) {
    doc["duracion_apertura_segundos"] = _doorDurationSec;
} else {
    doc["duracion_apertura_segundos"] = 0;
}
```

Consecuencia: **cada mensaje del ESP32 real fallaba con `ValidationError` y se
descartaba**. El sistema completo (persistencia, IA, cadena hash, alertas, SSE,
dashboard) nunca habría recibido un solo dato del hardware.

```
pydantic_core._pydantic_core.ValidationError: 1 validation error for LecturaPayload
duracion_apertura_segundos
  Extra inputs are not permitted [type=extra_forbidden, input_value=240, input_type=int]
```

### Por qué ninguna prueba lo detectó

Las 340 pruebas del backend construían el payload MQTT **a mano**, con
exactamente los campos que el backend ya esperaba. Ninguna se validaba contra el
payload literal que emite el firmware. La suite verificaba que el backend era
consistente consigo mismo, no que fuera compatible con el nodo edge.

Es también la razón por la que `analisis_final_estado_actual.md` daba el sistema
al ~99%: la sección 3.5 de la documentación IoT y el esquema Pydantic real
nunca se habían contrastado entre sí.

### La corrección

| Archivo | Cambio |
|---------|--------|
| `src/infrastructure/mqtt/payload_schema.py` | `+ duracion_apertura_segundos: int = Field(default=0, ge=0)` |
| `src/interface/api/schemas.py` | Mismo campo en `LecturaIngestRequest` (espejo REST) + `firmware_version` |
| `src/interface/api/mappers.py` | `+ evidencia_edge()` — construye la evidencia del nodo edge |
| `src/interface/main.py` | Camino MQTT: persiste la evidencia en la columna JSONB `payload` |
| `src/interface/api/lecturas_router.py` | Camino REST: idem, misma semántica |

Se conservó `extra="forbid"`: el contrato sigue cerrado, solo se amplió con el
campo que faltaba. `firmware_version` y `duracion_apertura_segundos` no tienen
columna propia en `thermal_readings`, así que se guardan en la columna JSONB
`payload` (HU-22) — antes se validaban y se descartaban sin dejar rastro.

**Sin migración de base de datos**: la columna `payload` ya existía.

### Verificación

`tests/integration/test_contrato_firmware_esp32.py` — 6 pruebas que validan
contra el payload **literal** de `IoT-documentacion_iot.md` §3.5:

1. El payload literal del firmware valida
2. Una apertura con duración conserva el valor
3. Una duración negativa se rechaza
4. Un campo desconocido **sigue** rechazándose (el contrato no se volvió permisivo)
5. La lectura se persiste end-to-end y emite SSE
6. La evidencia del nodo queda en el JSONB

**Prueba de que la prueba sirve**: con la corrección revertida (`git stash`),
4 de las 6 fallan con el `extra_forbidden` exacto de arriba. Con la corrección
aplicada, 6/6 pasan.

---

## 3. V-02 — RF-18 en el Dashboard

TI exige: *"El dashboard muestra el estado de conectividad de cada dispositivo
registrado"*. El campo ya viajaba en `LecturaTermica.estado_conectividad` y en
el payload SSE, pero solo se renderizaba en `DispositivosPage`.

`DashboardPage.tsx` ahora muestra una `Badge` junto al `device_id` de la curva
térmica. Claves i18n **propias** (`dashboard.dispositivoOnline` /
`dispositivoOffline`) en vez de reutilizar `dashboard.conectado`: ese par
describe la salud del stream SSE del navegador, que es un estado distinto. Un
dashboard con SSE sano puede estar mirando un nodo caído — justo el caso que la
farmacia necesita ver.

Paridad i18n mantenida: **310 claves en ES, 310 en EN, 0 divergencias**.

Verificado por `src/tests/presentation/pages/DashboardPage.test.tsx` (3 pruebas,
incluida una que afirma que la insignia del dispositivo y el indicador SSE son
independientes). Revertido el componente, las 3 fallan.

---

## 4. V-03 — Escaneo de dependencias

| Componente | Herramienta | Resultado |
|-----------|-------------|-----------|
| backend | `pip-audit` | **0 vulnerabilidades** |
| landing-page | `npm audit` | **0 vulnerabilidades** |
| frontend | `npm audit` | 5 altas, todas de desarrollo |
| frontend | `npm audit --omit=dev` | **0 vulnerabilidades** en producción |

Las 5 altas son una sola cadena transitiva:
`eslint@9 → @eslint/config-array → minimatch@3 → brace-expansion@1.1.16`
(GHSA-mh99-v99m-4gvg, DoS por expansión no acotada).

**No entran al bundle**: `eslint` es `devDependency` y el árbol de producción
sale limpio. No hay parche en la línea 1.x (1.1.16 es la última publicada), así
que el único remedio sería `eslint@10`, un cambio mayor con ruptura.

**Decisión**: documentar y no aplicar antes de la sustentación. Es un DoS en un
linter que solo procesa el código del propio repositorio; el costo/riesgo de un
salto de versión mayor en vísperas de defensa no se justifica.

---

## 5. V-04 / V-05 — CSP

La preocupación original («¿el CSP del backend rompe SSE / Tailwind / ECharts?»)
partía de una premisa equivocada: el CSP del **backend**
(`default-src 'none'; frame-ancestors 'none'`) solo gobierna las respuestas JSON
de la API. No es el CSP del documento del dashboard, así que no puede romper el
render del frontend. El CSP que sí lo gobierna es el de `frontend/vercel.json`.

Contrastado directiva por directiva contra el build de producción real:

| Directiva | Necesidad real | Veredicto |
|-----------|---------------|-----------|
| `style-src 'self' 'unsafe-inline' https://fonts.googleapis.com` | Tailwind inyecta inline; `index.css` importa CSS de Google Fonts | ✅ |
| `font-src https://fonts.gstatic.com` | No hay fuentes auto-hospedadas | ✅ |
| `script-src 'self'` | ECharts va empaquetado, sin CDN | ✅ |
| `img-src 'self' data:` | Sin imágenes remotas | ✅ |
| `connect-src 'self'` | axios + `EventSource` al backend | ⚠️ **V-05** |

### V-05 — Puerta de despliegue

`connect-src 'self'` **hoy no rompe nada**: Vercel construye con `build:demo`, y
en modo demo `apiClient` usa `demoAdapter` y `sseClient` usa `simularStream`
— no sale ninguna petición de red.

En cuanto se despliegue el frontend real contra el backend de Railway (Vercel →
Railway, la arquitectura de la tesis), `connect-src 'self'` **bloqueará todas
las llamadas de la API y el `EventSource`**, porque el backend es otro origen.

**Fix cuando exista el hostname definitivo**:

```jsonc
// frontend/vercel.json
"connect-src 'self' https://<backend>.up.railway.app"
```

No se aplicó porque requiere el hostname real: un origen inventado fallaría en
silencio y sería peor que la ausencia del cambio. El lado del backend ya está
resuelto (`cors_origins` configurable por entorno, con validadores que rechazan
`*` y orígenes sin `https://`).

---

## 6. V-06 — Deriva de documentación

| Corrección en `IoT-documentacion_iot.md` | Antes | Ahora |
|------------------------------------------|-------|-------|
| Conteo de archivos y líneas | 19 archivos · 1510 líneas | 18 archivos · 1394 líneas (verificado) |
| Snippet Pydantic §3.5 | `modelo_config` (nombre inválido), sin rangos, `Literal` inexistente | Copia fiel del esquema real |

El snippet anterior no era código válido de Pydantic v2 (`modelo_config` no
activa nada; el nombre correcto es `model_config`), lo que probablemente
contribuyó a que la desalineación de V-01 pasara desapercibida.

---

## 7. Estado de las pruebas

| Suite | Antes | Después |
|-------|-------|---------|
| Backend `pytest` | 340 ✅ | **346 ✅** (+6) |
| Frontend `vitest` | 60 ✅ | **63 ✅** (+3) |
| Frontend `tsc -b` | ✅ | ✅ |
| Frontend `eslint` | ✅ | ✅ |
| Frontend `npm run build` | — | ✅ |
| Frontend `npm run build:demo` | — | ✅ |
| Landing `tsc` + `build` | — | ✅ |
| **Firmware `pio run`** | ❌ **4 errores** | ✅ **SUCCESS** (Flash 78.3 %, RAM 14.3 %) |
| Backend cobertura | — | 77 % |
| Frontend cobertura | — | 25.29 % |

**Cero regresiones.** Los 9 casos nuevos cubren código nuevo, y se comprobó en
ambos sentidos: cada prueba falla al revertir su corrección.

---

## 8. Trazabilidad

| Requisito | Efecto de este cierre |
|-----------|----------------------|
| **RF-03 / HU-04** | La duración de apertura del MC-38 ya llega al backend y se persiste. Antes se perdía el mensaje completo. |
| **RF-04 / HU-05** | Payload canónico del firmware validado end-to-end contra el backend. |
| **RF-05 / HU-11** | El camino MQTT QoS 1 ahora entrega de verdad: antes toda lectura moría en la validación. |
| **RF-07 / HU-22** | Evidencia del nodo edge (`firmware_version`, duración) persistida en JSONB. |
| **RF-18** | Cerrado: conectividad del dispositivo visible en el Dashboard. |
| **RNF-05** | Escaneo de dependencias ejecutado; producción sin vulnerabilidades conocidas. |
| **RNF-07** | El volcado del buffer offline ya puede completarse: antes el backend rechazaba cada mensaje reenviado. |
| **OE4** | La validación técnica con hardware real (§4.4) ahora es viable. Antes habría fallado en el paso 7 sin causa evidente en el nodo. |

---

## 9. Qué falta (sin cambios respecto al análisis previo)

1. **V-05**: añadir el origen del backend a `connect-src` antes del primer
   despliegue no-demo.
2. **Integración física IoT ↔ Backend** (§4.4 del análisis, 10 pasos con
   hardware real, EMQX Cloud y backend desplegado).
3. **Métricas de infraestructura** (RNF-01, RNF-02, RNF-10): requieren medición
   sobre el despliegue real.

Nada de esto es resoluble desde el código; los tres dependen de infraestructura
o hardware físico.

---

## 10. Capa IoT — cinco defectos que solo aparecen al compilar

Se instaló PlatformIO 6.1.19 y se ejecutó `pio run`. El firmware **no
compilaba**. Todo lo que sigue estaba invisible para una revisión por lectura.

### V-07 — El firmware no compila (CRÍTICO)

```
src/connectivity/MQTTManager.cpp:41: error: 'class WiFiClientSecure' has no member named 'setMinSupportedTLS'
src/connectivity/MQTTManager.cpp:41: error: 'TLSv1_2' was not declared in this scope
src/connectivity/MQTTManager.cpp:114: error: passing 'const PubSubClient' as 'this' argument discards qualifiers
src/storage/LittleFSBuffer.cpp:5:  error: definition of implicitly-declared 'constexpr LittleFSBuffer::LittleFSBuffer()'
```

Implicación de fondo: **ninguna** de las afirmaciones "HU-01 … HU-14
implementadas" estaba respaldada por un binario. No existía firmware, existía
código fuente que no llegaba a serlo.

**Estado**: compila. `RAM 14.3 % · Flash 78.3 % (975 441 de 1 245 184 B)`.

### V-11 — `setMinSupportedTLS` no existe (ALTO, seguridad)

`WiFiClientSecure` del core Arduino-ESP32 no tiene ese método: es API de
ESP8266/BearSSL. El doc §3.7 lo citaba como el control OWASP **ISVS-CRYPT-01**
("TLS 1.2 mínimo"). Era un control **inexistente en un archivo que no
compilaba**.

Se eliminó la llamada. El suelo TLS 1.2 lo garantiza mbedTLS de ESP-IDF, que se
compila con TLS 1.2 como único protocolo. La garantía que sí depende del código
—autenticar al servidor con `setCACert()`— estaba presente y se conserva.

### V-08 — No existe la partición `littlefs` (CRÍTICO)

`platformio.ini` usaba `board_build.partitions = default_ffat.csv`, cuya única
partición de datos es `ffat` (tipo FAT). El firmware monta con:

```cpp
LittleFS.begin(true, LITTLEFS_MOUNT_POINT, 10, "littlefs")
```

que busca una partición **etiquetada `littlefs`**. No existía → el montaje
fallaba → `setup()` llamaba a `ESP.restart()` → **bucle de arranque infinito**.
El nodo no habría leído un solo sensor. Y como el certificado CA se carga desde
LittleFS, TLS tampoco habría funcionado nunca.

**Corregido** con `partitions_thermotrace.csv` (app0/app1 1216 KB + `littlefs`
1600 KB, subtipo `spiffs`, 4 MB exactos).

### V-09 — La telemetría en vivo nunca se publicaba (CRÍTICO)

En `main.cpp`, el paso 6 del bucle de red era **solo un comentario**:

```cpp
// ── 6. Publicar lectura en tiempo real si hay red ────────────
// Las lecturas se guardan en LittleFS por el Core 0. El Core 1
// las publica inmediatamente si hay conexión. [...]

delay(100);
```

No había código. Y el drenaje del buffer vivía **dentro** de
`if (!mqtt.isConnected()) { … }`, es decir, solo se ejecutaba en la transición
de reconexión.

Comportamiento real: el Core 0 escribía en LittleFS cada 30 s; el Core 1, ya
conectado, no publicaba nada. La cola crecía hasta 200 archivos y el FIFO
empezaba a descartar las lecturas más antiguas. **El dashboard solo habría visto
datos justo después de una reconexión.** Rompía RF-05, RF-11, RNF-01 y RNF-07.

**Corregido**: el drenaje se extrajo a `drenarBuffer()` y se ejecuta en cada
pasada con conexión. Camino único para telemetría en vivo y backlog offline,
manteniendo offline-first (se persiste antes de publicar, sobrevive a un corte
de corriente) y orden FIFO. Acotado a 20 publicaciones por ciclo para no
disparar el watchdog, que el propio manual advierte en §7.2.

### V-10 — QoS 1 y PUBACK no existen (ALTO)

`PubSubClient::publish()` publica **siempre en QoS 0**. No hay PUBACK, ni
packetId, ni callback de confirmación — el `onPublishAcknowledged` que la clase
declaraba jamás llegaba a invocarse.

El código borraba el archivo de LittleFS en cuanto `publish()` devolvía cierto,
que solo significa "escrito en el socket". Una lectura perdida en vuelo se
borraba igual: **pérdida silenciosa**, justo lo que RF-06 y RNF-07 prometen
evitar.

**Mitigado**: antes de borrar se revalida que la sesión MQTT siga viva; si se
cayó, el archivo se conserva y se reintenta. Acota la ventana, no la cierra.
Se corrigieron además todos los comentarios y logs que afirmaban QoS 1 (10
puntos en 5 archivos), y las tablas §3.6, §3.7 y §3.8 del manual IoT.

**HU-11 queda marcada como NO CUMPLIDA.** Cerrarla de verdad exige cambiar de
cliente MQTT (AsyncMqttClient), que es la mejora §8.2 nº 4 del manual. No se
acometió: es una reescritura de la capa de transporte que no se puede validar
sin hardware, y con `UNIQUE(device_id, timestamp)` en el backend el reintento ya
es idempotente. Es una decisión consciente, no un olvido.

### Lo que sigue sin poder verificarse

Compilar **no** es ejecutar. Sigue sin comprobarse en hardware real: lectura de
los tres sensores, handshake TLS contra EMQX, montaje efectivo de LittleFS,
cadencia real de 30 s y comportamiento del watchdog. Son los 10 pasos de §4.4
del análisis.

---

## 10-bis. Revisión de los drivers de sensores (ronda 2)

Los tres drivers nunca se habían ejecutado ni revisado línea a línea. Al hacerlo
aparecieron cinco defectos más, todos con efecto directo sobre la toma de datos
de la tesis.

### V-17 — El MC-38 no detectaba la mayoría de las aperturas (ALTO)

Dos problemas encadenados:

1. **El antirrebote no rebotaba.** `isOpen()` reiniciaba `_lastChangeTime` en
   cada lectura que coincidía con el estado actual. Como se llamaba una vez cada
   30 s, el temporizador se ponía a cero en cada muestra y el umbral de 50 ms no
   llegaba a aplicarse jamás.
2. **Muestreo puntual, no continuo.** `apertura_refrigerador` se tomaba del
   estado instantáneo en el momento de construir el payload. Una puerta de
   farmacia se abre y se cierra en 10-20 s; con muestreo cada 30 s, **la mayoría
   de las aperturas reales eran invisibles** y `duracion_apertura_segundos` solo
   contaba si la puerta seguía abierta justo en ese instante.

RF-03, HU-04 y HU-35 dependen enteramente de esto.

**Corregido**: `MC38Sensor` separa muestreo de reporte. `poll()` se llama cada
50 ms desde el Core 0 mientras espera su ventana (la cadencia de 30 s sigue
anclada a `vTaskDelayUntil`, así que el jitter no se acumula), con antirrebote
real sobre la muestra cruda. El payload informa ahora `huboApertura()` —¿se abrió
en algún momento de la ventana?— y `duracionAperturaSegundos()` acumulada.
`limpiarReporte()` cierra la ventana tras publicar.

### V-18 — Un fallo del DS18B20 era permanente (ALTO)

`_connected = false` no se revertía nunca: `begin()` solo corre en `setup()`. Un
único timeout del bus 1-Wire dejaba el sensor apagado hasta reiniciar el ESP32, y
el nodo publicaba `temperatura_interna: null` para siempre. Es el sensor que va
junto al medicamento, y el guard del backend interpreta el null como "sin dato",
no como avería: **degradación silenciosa e indefinida de la variable principal
del experimento**.

**Corregido**: se reintenta la detección en cada ciclo (30 s).

### V-19 — Timeout del DS18B20 demasiado justo (MEDIO)

`SENSOR_READ_TIMEOUT_MS = 1000` frente a los 750 ms de conversión a 12 bits: 25 %
de margen, con sondeo en pasos de `delay(10)` desde una tarea que comparte
núcleo. Un pico de planificación bastaba para marcar el sensor como averiado.
Subido a **2000 ms**.

### V-20 — El SHT31 no se recuperaba ni avisaba (MEDIO)

Si no respondía en el arranque —alimentación estabilizándose, un dupont flojo—
quedaba descartado toda la sesión. Además `main.cpp` ignoraba el valor de retorno
de `sht31.begin()`, así que no había ni un aviso en el monitor serie: el único
síntoma era un `null` en el dashboard, indistinguible de "lectura no disponible".

**Corregido**: reintento de detección en cada lectura y error explícito en el
arranque indicando los pines I2C.

### V-21 — Nombres de archivo truncados en LittleFS (BAJO)

`char buf[10]` con `snprintf(buf, sizeof(buf), "%05d.json", i)`: `"00001.json"`
son 10 caracteres **más** el terminador, así que todos los archivos se creaban
como `"00001.jso"`. Funcionaba por coherencia interna (se listaban y borraban con
el mismo nombre truncado), pero no coincidía con lo documentado y desorientaba
cualquier inspección del sistema de archivos. Buffer subido a 16 bytes.

### Efecto de V-17 sobre la capa de IA (revisado)

`apertura_refrigerador` es una de las **10 features** del Random Forest
(`training_metrics.json`), así que cambiar su semántica toca la entrada del
modelo. El cambio va en la dirección correcta:

- **Antes**: estado instantáneo en el momento del muestreo. En operación real
  habría sido `False` casi siempre, porque las aperturas cortas caían entre
  muestras. El modelo habría recibido una feature prácticamente constante.
- **Ahora**: "hubo apertura durante la ventana". Es lo que el dataset sintético
  representa —una apertura correlacionada con la subida térmica— así que la
  distribución real se acerca a la de entrenamiento en vez de alejarse.

No hace falta reentrenar: el tipo (booleano) y el significado pretendido no
cambian; lo que cambia es que ahora se mide de verdad.

**RNF-04 verificado**: `f1_weighted = 0.9671` ≥ 0.85, con gate en
`tests/unit/test_random_forest_service.py:158` y
`tests/integration/test_ia_api.py:32`.

### V-23 — El watchdog no cubría ninguna de las dos tareas (MEDIO)

`main.cpp` incluía `esp_task_wdt.h` y afirmaba en un comentario que "el watchdog
se alimenta en cada tarea", pero `esp_task_wdt_add()` no se llama en ninguna
parte: sólo `loopTask` queda cubierta por el TWDT que inicializa Arduino. Si
`taskRed` se colgara dentro del handshake TLS, nada reiniciaría el nodo.

**No se suscribieron las tareas**, y es una decisión deliberada: el TWDT de
Arduino viene con 5 s de plazo y `taskRed` tiene dos bloqueos legítimos más
largos (15 s de conexión Wi-Fi, 10 s de NTP). Suscribirlas sin subir el plazo y
sin sembrar `esp_task_wdt_reset()` en esos bucles produciría un ciclo de
reinicios — peor que no tener watchdog, y no se puede validar sin hardware. Se
corrigió el comentario para que no afirme una protección inexistente y queda
como mejora a validar sobre el nodo físico.

### Revisado y correcto

- `File::name()` devuelve el nombre base en arduino-esp32 3.x
  (`pathToFileName(path())`), así que el orden FIFO por `std::sort` lexicográfico
  sobre `%05d` sí es cronológico.
- El backoff exponencial del `WiFiManager` (1 s → 60 s) es correcto; su espera
  bloqueante de 15 s usa `delay()`, que en Arduino-ESP32 cede al planificador y
  alimenta al watchdog.

---

## 10-ter. Auditoría de backend, IA y web (ronda 3)

Barrido de las tres capas que quedaban por revisar a fondo.

### V-24 — La cadena hash ordenaba sin desempate (MEDIO)

`obtener_ultimo_hash()` ordenaba por `created_at DESC` y
`listar_todos_ordenados()` por `created_at ASC`, sin criterio de desempate.

`created_at` se genera en Python con resolución de microsegundos —el autor ya
había evitado a propósito el `CURRENT_TIMESTAMP` de SQLite, que tiene resolución
de 1 s— así que un empate es improbable. Pero si ocurriera, lo grave no sería el
empate: sería que el **camino de escritura** y el **camino de verificación**
podrían resolverlo en sentidos opuestos. La cadena se habría escrito en un orden
y se verificaría en otro, denunciando corrupción sobre evidencia intacta. En una
tesis cuya aportación es la trazabilidad verificable, un falso positivo de
corrupción es tan dañino como un falso negativo.

**Corregido**: ambos caminos desempatan ahora por `id`. Es un UUID4, no aporta
orden cronológico; aporta lo único necesario, que es que los dos recorridos
coincidan. Verificado con las 36 pruebas de hash/trazabilidad/corrupción.

### V-25 — Vercel despliega SIEMPRE la versión demo (informativo, pero importante)

`frontend/vercel.json` fija `"buildCommand": "npm run build:demo"`. La URL
pública **sirve siempre datos fabricados**: `apiClient` usa `demoAdapter` y
`sseClient` usa `simularStream`, sin una sola petición de red.

No es un defecto —es lo que se quiso para la demo— pero conviene tenerlo
presente: si en la sustentación se enseña la URL de Vercel, lo que se ve son
datos simulados, no el sistema conectado al backend. Para enseñar el sistema
real hay que cambiar ese `buildCommand` **y** añadir el origen del backend a
`connect-src` (V-05). Los dos cambios van juntos o ninguno funciona.

### Revisado y correcto (sin cambios necesarios)

| Comprobación | Resultado |
|---|---|
| **Deriva migraciones ↔ modelos** | **0 tablas y 0 columnas divergentes** (`alembic upgrade head` contra `Base.metadata`, comparadas una a una) |
| **Paridad de features IA** | Las 10 features coinciden **en orden** entre `features.py` y `training_metrics.json`, y `_validar_compatibilidad()` lo comprueba al cargar el modelo lanzando `RuntimeError` si difieren |
| Validación del artefacto IA | Checksum externo verificado antes de deserializar; clases predichas contrastadas contra las esperadas; `n_features_in_` contrastado |
| RNF-04 | `f1_weighted = 0.9671` ≥ 0.85, con gate en dos pruebas |
| **Aislamiento del modo demo** | El build de producción **no contiene** `VITE_MODO_DEMO` ni `demoAdapter`: Vite pliega la constante y elimina la rama. El camino demo es inalcanzable salvo compilando con `build:demo` |
| Backoff Wi-Fi | Correcto (1 s → 60 s); su espera bloqueante usa `delay()`, que cede al planificador |
| Orden FIFO del buffer | `File::name()` devuelve el nombre base en arduino-esp32 3.x, así que el `std::sort` lexicográfico sobre `%05d` sí es cronológico |

---

## 10-quater. Acceso con Google (ampliación fuera de backlog)

Solicitado durante el cierre. **No figura en el Product Backlog (HU-01..HU-47)
ni en los RF/RNF**, así que si se menciona en la sustentación conviene
presentarlo como ampliación posterior, no como requisito cumplido.

Se implementó **solo Google**. Facebook se descartó: exige App Review de Meta y
es difícil de justificar como proveedor de identidad para un sistema que
custodia evidencia farmacéutica.

### La decisión que lo hace defendible: Google autentica, la BD autoriza

El inicio de sesión con Google **no da de alta usuarios**. Si lo hiciera,
cualquier persona con cuenta de Google entraría, y además llegaría sin rol: el
RBAC de tres roles (RF-17, HU-41) dejaría de gobernar quién revisa alertas o
exporta reportes.

La lista de acceso es la propia tabla `users`. Un administrador da de alta el
correo y solo entonces ese correo puede entrar por Google. El rol, el estado
activo (HU-45) y el consentimiento de la Ley 29733 (HU-44) siguen viniendo de la
base de datos, y el token emitido es **el mismo JWT interno** que el del login
con contraseña — la cadena de auditoría no distingue el método salvo en el
registro de la acción.

### Verificación del ID token

Un ID token es un JWT firmado por Google. Saltarse cualquiera de estos cuatro
controles convertiría el acceso en un formulario donde el atacante escribe quién
dice ser:

| Control | Sin él |
|---|---|
| Firma contra el JWKS de Google (RS256) | Un token fabricado a mano se aceptaría |
| `aud` == nuestro `client_id` | Serviría un ID token emitido para **cualquier otra** aplicación de Google |
| `iss` de Google | — |
| `exp` / `iat` | Un token caducado seguiría valiendo |

Además se exige `email_verified`: un correo sin verificar puede pertenecer a
otra persona, y aceptarlo permitiría suplantar a un usuario dado de alta con
solo registrar ese correo en una cuenta de Google sin confirmar.

### Postura de seguridad del endpoint

- `POST /api/auth/google` comparte **la misma cuota** que `/login`; sin ella
  sería el camino sin límite para sondear qué correos están dados de alta.
- **Anti-enumeración**: "correo no provisionado", "usuario desactivado" y "token
  inválido" devuelven idéntico 401 con idéntico mensaje.
- **Deshabilitado por defecto** (`GOOGLE_OAUTH_ENABLED=false` → 404). Un
  validador de producción impide habilitarlo sin `GOOGLE_CLIENT_ID`, porque sin
  él no se puede validar `aud`.
- Éxito y fallo quedan auditados (`LOGIN_EXITOSO` con `metodo: google`,
  `LOGIN_GOOGLE_FALLIDO`).
- El barrido RBAC de todos los endpoints exigió declararlo público
  explícitamente, con su motivo, junto a `/login`.

### Frontend

`VITE_GOOGLE_CLIENT_ID` vacío ⇒ se conserva el botón placeholder y **no se carga
nada de Google**. Verificado sobre el bundle real: sin la variable, la URL de
Google Identity Services **no aparece** en el build (Vite pliega la constante y
elimina la rama); con ella, sí. Una instalación que no use el método no carga
código de terceros ni filtra visitas a Google.

La CSP de `vercel.json` se amplió con `script-src`, `connect-src`, `frame-src` e
`img-src` hacia los dominios de Google. **Sin esas directivas el botón se monta
pero el navegador bloquea el flujo sin mensaje útil** — es el fallo más probable
al desplegarlo.

### Cobertura

14 pruebas nuevas (9 backend + 5 frontend), sin salir a la red: el verificador
criptográfico se sustituye para ejercitar el flujo completo —bandera, cuota,
allowlist, auditoría, anti-enumeración— sin credenciales reales.

Incluye la comprobación de que **un usuario desactivado tampoco entra por
Google**: si fuera un camino paralelo, dar de baja a alguien (HU-45) no lo
dejaría fuera del sistema.

### Lo que falta para usarlo

Credenciales que solo el equipo puede obtener: crear el proyecto en Google Cloud
Console, obtener el `client_id` de tipo *Web application* y registrar los
*Authorized JavaScript origins* del dominio de despliegue. Sin eso el flujo real
no se ha ejercitado contra Google.

---

## 10-quinquies. Calidad de código y automatización

### V-27 — El backend no tenía ningún análisis estático (MEDIO)

Asimetría llamativa: el frontend tenía ESLint + `tsc`, y el backend —el
componente más grande, 3718 sentencias— no tenía linter, formateador ni
verificador de tipos. Nada revisaba imports muertos, nombres ambiguos ni
construcciones frágiles.

Se añadió **Ruff** (`pyproject.toml`), configurado para no pelearse con FastAPI:
`B008` se ignora porque `Depends(...)` como valor por defecto es su API, no un
descuido; `E501` porque el proyecto usa comentarios explicativos largos a
propósito.

**140 avisos → 0.** De ellos, los que eran defectos reales y no estilo:

| Corrección | Por qué importaba |
|---|---|
| `assert False` → `pytest.raises` (2 casos) | Bajo `python -O` los `assert` desaparecen: la prueba habría pasado aunque una lectura con NaN se aceptara sin error |
| 4 imports muertos | — |
| `zip(xs, xs[1:])` → `itertools.pairwise` | En la prueba de concurrencia del hash chain (RNF-03) |
| 5 variables `l` renombradas | `l` es indistinguible de `1` y de `I` al leer |
| 15 bloques de imports ordenados, 6 `dict()` → `{}`, 18 uniones a sintaxis PEP 604 | Consistencia |

Se descartaron por ser estilo sin valor: `UP017` (`timezone.utc` → `UTC`, 71
reescrituras sin cambio de comportamiento) y `RUF001` (los guiones largos de
«2–8 °C» son tipografía correcta, no caracteres confundibles).

Las 355 pruebas siguen pasando tras las 68 correcciones.

### V-28 — Código muerto en el firmware (BAJO, uno de ellos engañoso)

| Eliminado | Motivo |
|---|---|
| `OnPublishAck`, `_onPublishAck`, `onPublishAcknowledged()` | **API que mentía**: prometía confirmación por PUBACK, pero PubSubClient publica en QoS 0 y el callback no llegaba a invocarse nunca. Dejarla invitaba a construir encima de una garantía inexistente |
| `lastSensorRead`, `lastBufferFlush`, `RAM_BUFFER_SIZE` | Declaradas y nunca usadas |

### V-29 — `MQTT_KEEPALIVE_SEC` estaba definido pero nunca se aplicaba (MEDIO)

`config.h` declaraba 60 s y la documentación (§3.6) lo daba por vigente, pero
nunca se llamaba a `setKeepAlive()`: regía el valor por defecto de PubSubClient,
15 s. No es inocuo — **ese plazo es el que decide cuánto tarda EMQX en dar el
nodo por muerto y publicar su LWT**, es decir, la latencia de detección de una
caída. Ahora se aplica el valor documentado.

### V-30 — Ningún repositorio tenía CI (MEDIO)

Cuatro repositorios, cero automatización. Es la causa de raíz de por qué el
firmware pudo estar meses sin compilar: comprobarlo exigía tener PlatformIO
instalado en local, así que nadie lo comprobaba.

Se añadieron tres workflows de GitHub Actions:

| Repositorio | Qué verifica en cada push y PR |
|---|---|
| `backend` | `ruff check` · `pytest --cov` · `pip-audit` |
| `frontend` | `typecheck` · `lint` · `test:run` · **build de producción Y build demo** |
| `iot-firmware` | `pio run` · `pio run --target buildfs` (imagen LittleFS) |

El frontend construye **las dos variantes** a propósito: Vercel despliega
`build:demo`, así que sin comprobar también el build real éste podría estar roto
mucho tiempo sin que nadie lo notara. Ambos comandos se verificaron localmente
antes de escribir los workflows.

---

## 10-sexies. Verificación empírica del sistema en ejecución

Hasta aquí todo se había verificado leyendo código y ejecutando pruebas. Esta
ronda **levantó el sistema real** —backend con Uvicorn sobre SQLite, migraciones
aplicadas desde cero, usuarios sembrados— y lo ejercitó por HTTP.

### La prueba que sostiene la tesis: detección de alteración (RF-14/RF-15)

Con 17 registros reales en la cadena, la verificación devolvió `integra: true`.

Después se **alteró un registro directamente en la base de datos**, modificando
el payload de una alerta térmica sin tocar su hash — exactamente lo que haría
quien edita la base para ocultar una excursión:

```json
{
  "integra": false,
  "total_registros": 17,
  "primer_registro_inconsistente": 8,
  "detalle_inconsistencia": {
    "tipo_evento": "ALERTA_TERMICA",
    "hash_esperado":   "f95906cf01125497bc9b7af819060e43…",
    "hash_almacenado": "e1410c5d10748c5bfa0d2d317bc0cb27…",
    "mensaje": "Alteración detectada: el payload fue modificado post-registro"
  },
  "registros_posteriores_afectados": 8
}
```

Señala **el registro exacto**, con hash esperado frente a almacenado, y cuántos
registros posteriores quedan comprometidos. La aportación central del trabajo,
demostrada sobre un sistema en marcha y no solo en pruebas unitarias.

**HU-47 completo**: `/estado` pasó a `cadena_comprometida: true`, el aislamiento
devolvió 204, y tras él el estado volvió a `false` con 1 registro marcado
`is_corrupted` y 9 `is_after_corruption`.

### V-31 — El aislamiento de corrupción no quedaba en la bitácora (MEDIO)

Encontrado precisamente porque se miró `audit_logs` **después** de aislar: solo
había `LOGIN_EXITOSO` y las exportaciones. La cuarentena no aparecía.

`AislarCorrupcionUseCase` escribía un registro `REGISTRO_AISLADO_CORRUPCION` en
la propia cadena, pero **no en `audit_logs` y sin identificar al administrador**.
Es la intervención manual más sensible del sistema —un admin declara rota la
evidencia y arranca una cadena nueva— y la bitácora no podía responder *quién*
la ordenó, que es justo lo que preguntaría una auditoría.

**Corregido**: se audita como `CADENA_CORRUPCION_AISLADA` con `usuario_id` e IP,
y el bloque génesis de la cadena nueva incluye `aislado_por`. Cubierto por
`test_aislamiento_queda_en_la_bitacora_con_su_autor`.

### Lo demás, comprobado en vivo

| Requisito | Resultado |
|---|---|
| **RF-08 / RNF-04** | 19.89 °C y 15.49 °C → `excursion_critica`; 7.8 °C y 2.21 °C → `riesgo_preventivo`; 4.74 °C → `normal`. Clasificación clínicamente correcta |
| **RF-07 (dedup)** | 20 lecturas enviadas → 10 persistidas, todas con timestamp distinto |
| **RF-11 (SSE)** | Sobre HTTP real: ticket, `event: lectura` con evidencia de IA completa, `event: episodio_actualizado` |
| **RF-13** | JSON 24 KB y **PDF 11.8 KB con cabecera `%PDF-` válida**. El CSV se genera en el cliente (con defensa contra inyección de fórmulas y BOM para Excel) |
| **RF-09** | 2 alertas con mensajes clínicos correctos |
| **RF-16** | 15 entradas, incluidas las exportaciones de reportes |
| **RF-17 / HU-41** | Técnico → 403 en `/api/usuarios`; administrador → 200 |
| **Migraciones** | `alembic upgrade head` desde cero, limpio |

### Un falso positivo que conviene registrar

En la primera pasada, lecturas de 14 °C aparecieron clasificadas como `normal`
con confianza idéntica (0.9989) en todas — parecía un fallo grave del modelo.

No lo era. El simulador se lanzó con `--intervalo 0`, y como el timestamp del
firmware tiene **precisión de segundo**, las 12 lecturas colisionaron en el mismo
instante: la deduplicación (RF-07/B-04) las rechazó y la API devolvió la
clasificación de la lectura ya existente. Con intervalo real, el modelo acertó
en todos los casos.

Es decir: el comportamiento «anómalo» era el control de idempotencia
funcionando. Merece quedar escrito porque el mismo síntoma podría desconcertar
durante la demostración con hardware si se fuerzan envíos muy seguidos.

---

## 11. §4.6 — Indicador de cobertura (V-12)

| Componente | Herramienta | Cobertura |
|-----------|-------------|-----------|
| backend | `pytest-cov` | **77 %** (3718 sentencias, 848 sin cubrir) |
| frontend | `@vitest/coverage-v8` | **38.74 %** sentencias · **49.54 %** ramas |

```bash
cd backend  && .venv/bin/python -m pytest --cov=src --cov-report=term
cd frontend && npm run test:coverage
```

Se excluyen del cómputo los tipos, `main.tsx` y la capa demo. Artefactos de
cobertura añadidos a `.gitignore` y a los ignores de ESLint.

### Las nueve páginas sin pruebas, cubiertas

Las nueve páginas al 0 % **no estaban rotas**: compilaban, pasaban `tsc` y
ESLint y se servían en el build. Simplemente nadie había escrito pruebas para
ellas. Se añadieron 9 suites (34 casos), una por página, aislando cada una
mediante el mock de su hook:

| Página | Casos | Qué se afirma |
|--------|-------|---------------|
| `AlertasPage` | 5 | Alerta renderizada, tres filtros, **RBAC en la vista**: el botón de revisión aparece para farmacéutico y NO para técnico |
| `AuditoriaPage` | 3 | Acción, recurso e **IP de origen** (sin ella la entrada no vale como evidencia), estado vacío explícito |
| `DispositivosPage` | 4 | `device_id`, firmware, online vs offline, y un nodo dado de baja claramente marcado (HU-43) |
| `FirmwarePage` | 3 | Versión, descripción y **hash SHA-256 visible** — el control anti-manipulación de HU-46 |
| `HistorialPage` | 4 | Lecturas, los filtros de RF-12, y que aplicar un filtro **vuelve a consultar el backend** (si filtrara en memoria, el historial se limitaría a la página descargada) |
| `MetricasIAPage` | 4 | F1 ponderado, versión del modelo, y que la pantalla **sabe declarar el incumplimiento** cuando F1 < 0.85, no sólo felicitar |
| `ReportesPage` | 4 | Periodo, los **tres formatos** de RF-13 (CSV/JSON/PDF), sin descargas antes de generar, y el error |
| `TrazabilidadPage` | 4 | Verificación, cadena íntegra, **alteración señalando cuál registro** (RF-15), y el aviso de cadena comprometida (HU-47) |
| `UsuariosPage` | 3 | Usuarios con su rol, desactivado distinguible del activo (HU-45), alta de usuario |

Ninguna prueba encontró un defecto en las páginas: las cuatro que fallaron al
principio lo hicieron por fixtures mal construidos por quien las escribió
(forma de `RespuestaModelo`, envío del formulario de historial, y el uso de
`login()` —que es asíncrono y llama al backend real— en vez de
`useAuthStore.setState()`). Corregidos los fixtures, las nueve pasan.

**Frontend: de 63 a 97 pruebas** en 17 archivos.
