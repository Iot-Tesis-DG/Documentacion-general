# Documentación IoT — Nodo Edge ESP32

> ⚠️ **Revisión 2026-07-25.** El firmware de este repositorio **nunca se había
> compilado**. `pio run` fallaba con 4 errores, así que ninguna de las
> afirmaciones de "implementado" de este documento ni del análisis final estaba
> respaldada por un binario. Se corrigió y **hoy compila** (Flash 78.3 %, RAM
> 14.3 %). Ver `verificacion_cierre_2026-07-25.md` §10 para el detalle de los
> cinco defectos encontrados: compilación, partición LittleFS inexistente,
> telemetría en vivo nunca publicada, QoS 1 inexistente y `setMinSupportedTLS`
> como API inventada.
>
> Sigue **sin validarse con hardware real**: compilar no es ejecutar.

> **Proyecto**: ThermoTrace — Monitoreo IoT + IA de Cadena de Frío Farmacéutica
> **Tesis UPC 2026**: Soto Quispe, Diego Ulises & Gamio Upiachihua, Brenda Lucía
> **Stack Edge**: ESP32 DevKitC V4 + Arduino Core (ESP-IDF) + FreeRTOS
> **Lenguaje**: C++17
> **Build System**: PlatformIO 6.x
> **Repositorio**: `https://github.com/Iot-Tesis-DG/iot-firmware.git`
> **Fecha**: Julio 2026

---

## 1. Arquitectura General

```
┌─────────────────────────────────────────────────────────────┐
│                   REFRIGERADOR FARMACÉUTICO                 │
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│  │ DS18B20  │   │  SHT31   │   │  MC-38   │               │
│  │ (junto al│   │(ambiente)│   │(puerta)  │               │
│  │medicam.) │   │  I2C     │   │  GPIO    │               │
│  │ 1-Wire   │   │          │   │          │               │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘               │
│       │              │              │                      │
│       ▼              ▼              ▼                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │            ESP32 DevKitC V4 (Dual-Core)              │   │
│  │                                                     │   │
│  │  Core 0: Captura sensores cada 30s + LittleFS       │   │
│  │  Core 1: Wi-Fi + MQTT/TLS + Sincronización          │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
└─────────────────────────┼───────────────────────────────────┘
                          │
                    Wi-Fi / TLS 1.2
                          │
                          ▼
              ┌───────────────────────┐
              │   EMQX Cloud          │
              │   (MQTT Broker)       │
              │   Puerto 8883 (TLS)   │
              └───────────┬───────────┘
                          │
                    aiomqtt (TLS)
                          │
                          ▼
              ┌───────────────────────┐
              │   Backend FastAPI     │
              │   Railway (Python)    │
              │   PostgreSQL          │
              └──────────┬────────────┘
                         │
                    SSE (Server-Sent Events)
                         │
                         ▼
              ┌───────────────────────┐
              │   Frontend React 19   │
              │   Vercel              │
              │   Dashboard tiempo    │
              │   real + ECharts      │
              └───────────────────────┘
```

### 1.1 Stack Tecnológico del Nodo Edge

| Capa | Tecnología | Justificación (Anexo2 Benchmarking) |
|------|-----------|-------------------------------------|
| Microcontrolador | ESP32 DevKitC V4 | Score 4.80/5.00. Xtensa LX6 dual-core 240 MHz. Wi-Fi + TLS nativos sin módulos externos. |
| Framework | Arduino Core (ESP-IDF) + FreeRTOS | Score 4.90/5.00. Abstracción de hardware para sensores. FreeRTOS para dual-core sin overhead de RTOS pesado. |
| Sensores | SHT31 (I2C) + DS18B20 (1-Wire) + MC-38 (GPIO) | Score 5.00/5.00. Precisión clínica (±0.2°C SHT31, ±0.5°C DS18B20). |
| Buffer offline | LittleFS (partición Flash 1.5 MB) | FIFO con integridad verificada al reinicio. Sin dependencia de SD card. |
| Serialización | ArduinoJson v7 | Menor overhead que string concatenation. Validación de tamaño (máx 512 bytes). |
| Transporte | MQTT sobre TLS 1.2 (puerto 8883) | Score 4.70/5.00. QoS 1 garantiza entrega. MQTT consume 10x menos ancho de banda que HTTP para telemetría. |
| Broker | EMQX Cloud Serverless | Score 4.80/5.00. TLS nativo, autenticación por dispositivo, LWT, sin infraestructura propia. |
| Seguridad | One-way TLS + device_id/token + SNI | Alineado a OWASP IoT STG v1.0.0. Credenciales vía build_flags (RNF-05). |

---

## 2. Hardware

### 2.1 Componentes

| Componente | Modelo | Cantidad | Propósito |
|------------|--------|----------|-----------|
| Microcontrolador | ESP32 DevKitC V4 | 1 | Nodo Edge principal |
| Sensor temp. interna | DS18B20 (sonda impermeable acero inox.) | 1 | Temperatura junto al medicamento |
| Sensor temp. + humedad | SHT31-DIS | 1 | Condiciones ambientales del refrigerador |
| Sensor magnético | MC-38 (reed switch NC) | 1 | Apertura/cierre de puerta |
| Resistencia | 4.7 kΩ, 1/4 W | 1 | Pull-up para bus 1-Wire |
| Protoboard + cables dupont | — | 1 set | Conexiones sin soldadura |
| Cable USB micro-B | — | 1 | Alimentación + flasheo |

### 2.2 Esquema de Conexiones

```
ESP32 DevKitC V4          Sensores
══════════════════        ═══════════════════════

GPIO4  ───────────────────────────── DS18B20 — DATA  (amarillo)
                                   ├─ VCC → 3.3V   (rojo)
                                   └─ GND → GND    (negro)
                                   [4.7kΩ pull-up entre DATA y 3.3V]

GPIO21 (SDA) ────────────────────── SHT31 — SDA
GPIO22 (SCL) ────────────────────── SHT31 — SCL
3.3V  ───────────────────────────── SHT31 — VIN
GND   ───────────────────────────── SHT31 — GND

GPIO15 ──────────────────────────── MC-38 — (un terminal)
GND   ───────────────────────────── MC-38 — (otro terminal)
                                   [Pull-up interno del ESP32 en GPIO15]

Alimentación:
USB ── ESP32 (5V → regulador interno 3.3V)
```

### 2.3 Especificaciones del ESP32 DevKitC V4

| Parámetro | Valor |
|-----------|-------|
| Procesador | Xtensa LX6 dual-core 240 MHz |
| SRAM | 520 KB (160 KB para datos dinámicos) |
| Flash | 4 MB (particionado: 1.2 MB app, 1.5 MB LittleFS, 64 KB NVS) |
| Wi-Fi | 802.11 b/g/n (2.4 GHz), hasta 150 Mbps |
| Bluetooth | BLE 4.2 (no usado en este proyecto) |
| GPIO | 34 pines (ADC 12-bit, I2C x2, SPI x3, UART x3) |
| Voltaje operación | 3.3V (regulado desde USB 5V) |
| Consumo activo | ~160-260 mA (Wi-Fi activo) |
| Consumo deep sleep | ~10 µA |

### 2.4 Especificaciones de Sensores

| Sensor | Protocolo | Rango | Precisión | Resolución | Tiempo lectura |
|--------|-----------|-------|-----------|------------|----------------|
| DS18B20 | 1-Wire (GPIO4) | -55°C a +125°C | ±0.5°C (-10°C a +85°C) | 12 bits (0.0625°C) | ~750 ms |
| SHT31-DIS | I2C (0x44) | -40°C a +125°C / 0-100% HR | ±0.2°C / ±2% HR | 16 bits | ~15 ms |
| MC-38 | GPIO digital (GPIO15) | Abierto/Cerrado | — | — | instantáneo (50 ms debounce) |

### 2.5 Referencia de Pines del ESP32 DevKitC V4

```
         ┌─────────────────────────┐
         │   ESP32 DevKitC V4      │
         │                         │
    3.3V │● ●│ GND                 │
    EN   │● ●│ GPIO23              │
  GPIO36 │● ●│ GPIO22 ← SHT31 SCL  │
  GPIO39 │● ●│ GPIO1  (TX0)        │
  GPIO34 │● ●│ GPIO3  (RX0)        │
  GPIO35 │● ●│ GPIO21 ← SHT31 SDA  │
  GPIO32 │● ●│ GND                 │
  GPIO33 │● ●│ GPIO19              │
  GPIO25 │● ●│ GPIO18              │
  GPIO26 │● ●│ GPIO5               │
  GPIO27 │● ●│ GPIO17              │
  GPIO14 │● ●│ GPIO16              │
  GPIO12 │● ●│ GPIO4  ← DS18B20    │
    GND  │● ●│ GPIO0               │
    VIN  │● ●│ GPIO2               │
    EN   │● ●│ GPIO15 ← MC-38      │
         │   │                     │
         │   │         USB micro-B │
         └───┴─────────────────────┘
```

---

## 3. Firmware

### 3.1 Estructura de Archivos

```
iot-firmware/
├── platformio.ini                # Configuración PlatformIO: ESP32, librerías, particiones
├── .gitignore
├── data/
│   └── certs/
│       ├── README.md             # Instrucciones para obtener root_ca.pem
│       └── root_ca.pem           # Certificado CA raíz (NO se sube al repo — .gitignore)
└── src/
    ├── main.cpp                  # Entry point + setup() + tareas FreeRTOS dual-core
    ├── config.h                  # Pines, credenciales, timeouts, constantes, macros
    ├── sensors/
    │   ├── DS18B20Sensor.h       # Driver DS18B20 — interfaz
    │   ├── DS18B20Sensor.cpp     # Driver DS18B20 — implementación (1-Wire, 12 bits)
    │   ├── SHT31Sensor.h         # Driver SHT31-DIS — interfaz
    │   ├── SHT31Sensor.cpp       # Driver SHT31-DIS — implementación (I2C)
    │   ├── MC38Sensor.h          # Driver MC-38 — interfaz
    │   └── MC38Sensor.cpp        # Driver MC-38 — implementación (GPIO + debounce)
    ├── connectivity/
    │   ├── WiFiManager.h         # Gestión Wi-Fi con backoff exponencial — interfaz
    │   ├── WiFiManager.cpp       # Gestión Wi-Fi — implementación
    │   ├── MQTTManager.h         # Cliente MQTT/TLS 1.2 con LWT + QoS 1 — interfaz
    │   └── MQTTManager.cpp       # Cliente MQTT/TLS — implementación
    ├── storage/
    │   ├── LittleFSBuffer.h      # Buffer offline en Flash — interfaz
    │   └── LittleFSBuffer.cpp    # Buffer offline — implementación (FIFO, 200 archivos)
    └── payload/
        ├── PayloadBuilder.h      # Constructor JSON con ArduinoJson v7 — interfaz
        └── PayloadBuilder.cpp    # Constructor JSON — implementación (NTP, ISO 8601)

18 archivos · 1394 líneas de código C++ (`src/`, verificado 2026-07-25)
```

### 3.2 Arquitectura Dual-Core (FreeRTOS)

El ESP32 tiene dos núcleos físicos. El firmware los particiona para que una falla de red nunca bloquee la captura de sensores:

```
Core 0 (Protocol Core) — xTaskCreatePinnedToCore(..., 0)
├── Prioridad: 2 (más alta)
├── Stack: 8 KB
├── Tarea: taskSensores()
│
│  ┌─ vTaskDelayUntil(&lastWakeTime, 30s) ← CADENCIA EXACTA
│  │
│  ├─ 1. ds18b20.readTemperatureC()     → float | NAN
│  ├─ 2. sht31.readTemperatureC()       → float | NAN
│  ├─ 3. sht31.readHumidity()           → float | NAN
│  ├─ 4. mc38.isOpen()                  → bool (con debounce 50 ms)
│  ├─ 5. mc38.openDurationSec()         → unsigned long
│  ├─ 6. PayloadBuilder::build(512)     → String JSON (~250 bytes)
│  └─ 7. buffer.saveReading(json)       → LittleFS (/littlefs/pending/00001.json)
│
│  Si JSON vacío → saltar ciclo, no guardar.
│  Si LittleFS lleno → FIFO: eliminar archivo más antiguo.
│
│  Repetir cada 30 segundos exactos (vTaskDelayUntil garantiza
│  que el jitter del ciclo no se acumula).


Core 1 (Application Core) — xTaskCreatePinnedToCore(..., 1)
├── Prioridad: 1 (menor que sensores)
├── Stack: 12 KB (TLS necesita más stack)
├── Tarea: taskRed()
│
│  Loop infinito (delay 100 ms):
│
│  ├─ wifi.maintain()
│  │   Si no conectado → backoff exponencial (1s→2s→4s...→60s).
│  │   Si conectado → continuar.
│  │
│  ├─ mqtt.connect()
│  │   Si no conectado:
│  │     a. TLS handshake con CA desde LittleFS.
│  │     b. MQTT CONNECT con LWT (offline).
│  │     c. Publicar LWT "online" → backend actualiza estado_conectividad.
│  │     d. SINCRONIZAR LittleFS → MQTT:
│  │        - Leer archivos pendientes en orden (00001.json, 00002.json...)
│  │        - Publicar cada uno en farmacias/{device_id}/lecturas (QoS 1)
│  │        - Eliminar de LittleFS solo tras publish() exitoso
│  │        - Si falla un publish → detener sincronización, reintentar en
│  │          próximo ciclo (los duplicados son inocuos por UNIQUE en BD)
│  │
│  ├─ mqtt.loop()
│  │   Procesa callbacks MQTT, keep-alive PINGREQ/PINGRESP, PUBACK.
│  │
│  └─ delay(100) → ceder CPU al scheduler de FreeRTOS
```

### 3.3 Máquina de Estados de Conectividad

```
                    ┌──────────┐
            ┌──────►│  WI-FI   │◄──────────────┐
            │       │ CONECTADO│                │
            │       └────┬─────┘                │
            │            │                      │
            │    ┌───────▼────────┐             │
            │    │  MQTT          │  falla      │
            │    │  CONECTADO     │─────────────┤
            │    │  (publicando)  │             │
            │    └───────┬────────┘             │
            │            │                      │
            │  publica   │                      │
            │  LWT       │                      │
            │  "online"  │                      │
            │            │                      │
            │    ┌───────▼────────┐    ┌────────┴──────────┐
            │    │  SINCRONIZAR   │    │  BACKOFF           │
            │    │  LittleFS→MQTT │    │  EXPONENCIAL       │
            │    │  (FIFO order)  │    │  1s→2s→4s...→60s  │
            │    └───────┬────────┘    └────────┬──────────┘
            │            │                      │
            │            │  cola vacía          │ timer vence
            │            │  o error             │
            │            │                      │
            └────────────┘              ┌───────▼────────┐
                                        │  INTENTAR       │
                                        │  RECONEXIÓN     │
                                        │  WI-FI          │
                                        └───────┬────────┘
                                                │
                                        ¿conectado?
                                        │        │
                                        Sí       No
                                        │        └──► volver a BACKOFF
                                        ▼
                                   ┌──────────┐
                                   │ MQTT     │
                                   │ CONNECT  │
                                   │ + LWT    │
                                   └──────────┘
```

### 3.4 Flujo de Datos Completo (End-to-End)

```
┌─────────────────────────────────────────────────────────────────────────┐
│ CAPA EDGE — ESP32                                                        │
│                                                                          │
│  Core 0: taskSensores()                    Core 1: taskRed()             │
│  ════════════════════                      ═════════════════             │
│                                                                          │
│  DS18B20 ──┐                              ┌── WiFiManager::maintain()   │
│  SHT31   ──┼─► lectura cada 30s ──► JSON  │                              │
│  MC-38   ──┘         │                    ├── MQTTManager::connect()    │
│                       ▼                    │   ├─ TLS handshake          │
│              PayloadBuilder::build()       │   ├─ LWT offline            │
│              {                             │   ├─ CONNECT                │
│                "device_id": "FARM-01-CDL", │   └─ publicar online        │
│                "timestamp": "2026-07-25T   │                              │
│                  12:34:56Z",              ├── MQTTManager::publish()    │
│                "estado_conectividad":      │   topic: farmacias/         │
│                  "online",                 │     FARM-01-CDL/lecturas    │
│                "firmware_version":         │   qos: 1                    │
│                  "1.0.0",                 │   payload: JSON (≤512 B)    │
│                "temperatura_interna":      │                              │
│                  4.5,                     ├── LittleFSBuffer::           │
│                "temperatura_ambiental":    │     listPendingFiles()       │
│                  5.2,                     │   → sincronizar cola FIFO    │
│                "humedad_ambiental":        │   → eliminar tras publish()  │
│                  62.0,                    │                              │
│                "apertura_refrigerador":     └── mqtt.loop()              │
│                  false,                        (PUBACK, keep-alive)      │
│                "duracion_apertura_        │                              │
│                  segundos": 0             │                              │
│              }                             │                              │
│                       │                    │                              │
│                       ▼                    │                              │
│              LittleFSBuffer::saveReading() │                              │
│              /littlefs/pending/00001.json  │                              │
└──────────────────────┬─────────────────────┴──────────────────────────────┘
                       │
                 TLS 1.2 (puerto 8883)
                 QoS 1 (at least once)
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ CAPA TRANSPORTE — EMQX Cloud Serverless                                   │
│                                                                           │
│  - Recibe en farmacias/FARM-01-CDL/lecturas (QoS 1)                       │
│  - Verifica autenticación (device_id + token)                             │
│  - Reenvía a suscriptores (backend aiomqtt)                               │
│  - Si ESP32 se desconecta abruptamente → publica LWT en                    │
│    farmacias/FARM-01-CDL/eventos:                                         │
│    {"device_id":"FARM-01-CDL","tipo_evento":"lwt_offline",...}            │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                      aiomqtt (Python async)
                      suscripción: farmacias/+/lecturas
                      suscripción: farmacias/+/eventos
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ CAPA BACKEND — FastAPI (Railway)                                           │
│                                                                           │
│  1. LecturaPayload.model_validate_json()  ← Pydantic v2                   │
│     ├─ device_id: str                                                     │
│     ├─ timestamp: datetime (ISO 8601)                                     │
│     ├─ temperatura_interna: float | None                                  │
│     ├─ temperatura_ambiental: float | None                                │
│     ├─ humedad_ambiental: float | None                                    │
│     ├─ apertura_refrigerador: bool                                        │
│     └─ estado_conectividad: "online" | "offline"                          │
│                                                                           │
│  2. Anti-spoofing: device_id en payload == segmento del topic             │
│                                                                           │
│  3. Validar timestamp: ±10 min futuro, ±2h pasado (B-10)                  │
│                                                                           │
│  4. Deduplicación: SELECT (device_id, timestamp) → si existe, ignorar     │
│     (idempotencia para QoS 1 reenvíos)                                    │
│                                                                           │
│  5. Clasificar riesgo:                                                    │
│     a. Extraer 10 features de la lectura + historial                      │
│     b. Guard de sensores: None/NaN/inf → sin inferencia                   │
│     c. Random Forest (200 árboles, max_depth=12)                          │
│     d. Salvaguarda: si regla > modelo, regla gana                         │
│     e. Resultado: normal | riesgo_preventivo | excursion_critica          │
│                                                                           │
│  6. Persistir en thermal_readings (PostgreSQL JSONB)                      │
│                                                                           │
│  7. Hash SHA-256 encadenado (pg_advisory_xact_lock)                       │
│                                                                           │
│  8. Generar alerta si aplica (anti-tormenta: cooldown 15 min)              │
│                                                                           │
│  9. Notificar email/Telegram si excursión crítica (HU-23)                  │
│                                                                           │
│ 10. Auditoría (audit_logs)                                                │
│                                                                           │
│ 11. Emitir SSE → dashboard tiempo real                                    │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.5 Formato del Payload JSON

```json
{
  "device_id": "FARM-01-CDL",
  "timestamp": "2026-07-25T12:34:56Z",
  "estado_conectividad": "online",
  "firmware_version": "1.0.0",
  "temperatura_interna": 4.5,
  "temperatura_ambiental": 5.2,
  "humedad_ambiental": 62.0,
  "apertura_refrigerador": false,
  "duracion_apertura_segundos": 0
}
```

**Reglas de serialización (HU-05)**:
- Campos con sensor fallido → se envían como `null` explícito, NUNCA se omiten
- Timestamp en ISO 8601 UTC con precisión de segundos (`YYYY-MM-DDTHH:MM:SSZ`)
- `estado_conectividad`: `"online"` si Wi-Fi conectado, `"offline"` si no
- Tamaño máximo: 512 bytes (si se excede, no se publica y se registra error en log)
- `duracion_apertura_segundos`: 0 si puerta cerrada, contador incremental si abierta

**Validación Pydantic v2 correspondiente en el backend** (`payload_schema.py`):

```python
class LecturaPayload(BaseModel):
    # Contrato cerrado: un campo no declarado es un error, no algo que se
    # silencie. NaN/infinito nunca llegan a persistencia, IA ni trazabilidad.
    model_config = ConfigDict(extra="forbid", allow_inf_nan=False)

    device_id: str = Field(min_length=1, max_length=50)
    timestamp: datetime
    message_id: str | None = Field(default=None, max_length=100)
    temperatura_ambiental: float | None = Field(default=None, ge=-40.0, le=125.0)
    humedad_ambiental: float | None = Field(default=None, ge=0.0, le=100.0)
    temperatura_interna: float | None = Field(default=None, ge=-55.0, le=125.0)
    apertura_refrigerador: bool = False
    estado_conectividad: str = "online"
    firmware_version: str | None = None
    duracion_apertura_segundos: int = Field(default=0, ge=0)
```

> **Nota de integración (2026-07-25).** `duracion_apertura_segundos` faltaba en
> `LecturaPayload`. Como el contrato es `extra="forbid"` y `PayloadBuilder::build()`
> emite ese campo en **todas** las lecturas (0 con la puerta cerrada), el backend
> rechazaba el 100% de los mensajes del firmware real con `ValidationError`.
> Ninguna prueba lo detectaba porque todas construían el payload a mano. Corregido
> y cubierto por `tests/integration/test_contrato_firmware_esp32.py`, que valida
> contra el payload literal de esta sección.

### 3.6 Estrategia de Tolerancia a Fallos

| Escenario | Comportamiento del ESP32 | Recuperación |
|-----------|--------------------------|--------------|
| **Sin Wi-Fi** | Core 0 sigue capturando. JSON se guarda en LittleFS. `estado_conectividad = "offline"`. | Al reconectar, Core 1 sincroniza toda la cola pendiente en orden FIFO. |
| **Sin MQTT (EMQX caído)** | Ídem. LittleFS retiene. Backoff exponencial de reconexión MQTT. | QoS 1 garantiza "al menos una vez". PUBACK del broker confirma entrega. |
| **Buffer LittleFS lleno (200 archivos)** | Se elimina el archivo más antiguo (FIFO). Se registra evento de saturación en log serie. | El backend ya recibió ~100 minutos de datos. Las lecturas más antiguas se sacrifican. |
| **Corte de energía durante escritura a Flash** | Al reiniciar, LittleFS detecta archivo corrupto (CRC falla). Lo descarta. | Se retoma captura normal desde la última lectura íntegra. |
| **Sensor DS18B20 desconectado** | `readTemperatureC()` retorna `NAN`. JSON envía `"temperatura_interna": null`. | Backend: guard de sensores → sin inferencia. No se genera alerta falsa. |
| **Sensor SHT31 no responde** | Reintenta 1 vez. Si falla, retorna `NAN`. JSON: ambos campos como `null`. | Backend trata ambos como ausentes. La clasificación sigue con sensor interno. |
| **Timestamp no sincronizado (NTP falló)** | Usa `__DATE__` + `__TIME__` de compilación + `millis()` como fallback. | Backend valida ventana ±2h. Si timestamp muy desviado, rechaza lectura. |
| **Paquete perdido en vuelo** | ⚠️ La publicación es QoS 0: no hay PUBACK. Se revalida la sesión antes de borrar el archivo, lo que acota la ventana pero no la cierra. | Backend: UNIQUE(device_id, timestamp) → todo reenvío es idempotente. Cerrar del todo exige un cliente con QoS 1 (§8.2). |
| **Reconexión Wi-Fi durante publicación** | PubSubClient reintenta internamente. Si falla, `publish()` retorna 0. | Archivo permanece en LittleFS. Se reintenta en el próximo ciclo de taskRed(). |
| **ESP32 se apaga abruptamente** | EMQX detecta timeout de keep-alive (60s). Publica LWT `offline` automáticamente. | Backend recibe `lwt_offline` → actualiza `estado_conectividad = false`. Dashboard muestra badge rojo. |
| **Múltiples ESP32 publicando simultáneamente** | Cada uno en su propio topic `farmacias/{device_id}/lecturas`. | Backend suscrito a `farmacias/+/lecturas` (comodín). Sin contención entre dispositivos. |

### 3.7 Seguridad en el Nodo Edge

| Control | Código | Alineamiento |
|---------|--------|-------------|
| **TLS 1.2** | Lo impone mbedTLS de ESP-IDF (TLS 1.2 es el único protocolo habilitado). `setMinSupportedTLS` **no existe** en `WiFiClientSecure` de ESP32 — era API de ESP8266 e impedía compilar; se eliminó. | OWASP IoT STG v1.0.0 — ISVS-CRYPT-01 |
| **Validación CA raíz** | `_client.setCACert(ca.c_str())` cargado desde LittleFS — `MQTTManager.cpp:26` | OWASP IoT STG — ISVS-CRYPT-02 |
| **SNI automático** | `WiFiClientSecure` envía el hostname del broker durante handshake TLS (nativo del SDK) | OWASP IoT STG — ISVS-NET-01 |
| **Autenticación por dispositivo** | `device_id` como username + token como password en `_mqtt.connect()` — `MQTTManager.cpp:90-98` | OWASP IoT STG — ISVS-AUTH-01 |
| **Credenciales fuera del código** | `WIFI_PASSWORD` y `MQTT_TOKEN` definibles vía `build_flags` en `platformio.ini`, no en texto plano en `config.h` | RNF-05 |
| **LWT (Last Will)** | Registrado en el propio CONNECT (QoS 1) — `MQTTManager.cpp:90-98`. Payloads en `config.h:41-42` | OWASP IoT STG — ISVS-COM-02 |
| **Puerto seguro** | `MQTT_PORT 8883` en `config.h:28`. Puerto 1883 bloqueado en EMQX Cloud (HU-14) | OWASP IoT STG — ISVS-NET-02 |
| **Anti-spoofing** | Backend verifica `device_id` en payload == segmento del topic (`main.py::_device_id_del_topic`) | OWASP IoT STG — ISVS-AUTH-03 |
| **Deduplicación** | `UNIQUE(device_id, timestamp)` en PostgreSQL (`models.py:109`). Previene replay attacks vía QoS 1 reenvíos. | OWASP IoT STG — ISVS-DATA-01 |
| **Particiones flash** | `partitions_thermotrace.csv`: app0/app1 1216 KB + littlefs 1600 KB. Antes se usaba `default_ffat.csv`, **sin partición `littlefs`**, y el nodo no arrancaba. | OWASP IoT STG — ISVS-STOR-01 |

### 3.8 Trazabilidad de HUs del Backlog

| HU | Título | Archivo(s) | Función clave |
|----|--------|-----------|---------------|
| HU-01 | Captura DS18B20 cada 30s | `DS18B20Sensor.cpp` | `readTemperatureC()` → 12 bits, detección -127°C |
| HU-02 | Captura SHT31 temp | `SHT31Sensor.cpp` | `readTemperatureC()` → validación rango -40..125°C |
| HU-03 | Humedad relativa SHT31 | `SHT31Sensor.cpp` | `readHumidity()` → validación rango 0-100% HR |
| HU-04 | MC-38 con debounce | `MC38Sensor.cpp` | `isOpen()` → debounce 50 ms, `openDurationSec()` |
| HU-05 | Payload JSON canónico | `PayloadBuilder.cpp` | `build()` → ArduinoJson v7, null explícito |
| HU-06 | Buffer LittleFS offline | `LittleFSBuffer.cpp` | `saveReading()` → FIFO, `pendingCount()` |
| HU-07 | Sync buffer | `main.cpp::drenarBuffer()` | FIFO acotado a 20/ciclo; elimina tras publicar y revalidar sesión. **QoS 0**, no QoS 1 |
| HU-08 | Backoff exponencial Wi-Fi | `WiFiManager.cpp` | `_increaseBackoff()` → 1s→2s→4s...→60s |
| HU-09 | TLS 1.2 con validación CA | `MQTTManager.cpp:26` | `setCACert()` desde LittleFS (TLS 1.2 lo fija mbedTLS) |
| HU-10 | device_id + SNI + token | `MQTTManager.cpp:90-98` | `_mqtt.connect()` con username=device_id |
| HU-11 | ~~QoS 1 publish~~ | `MQTTManager.cpp:130-164` | ⚠️ **NO CUMPLIDA**. `PubSubClient::publish()` es QoS 0: no hay PUBACK ni packetId. Requiere cambiar de cliente MQTT (§8.2) |
| HU-13 | LWT online/offline | `MQTTManager.cpp:90-98`, `:116` | LWT en el CONNECT (QoS 1) + publicación de `online` al conectar |

---

## 4. Dependencias (Librerías)

| Librería | Versión | Uso |
|----------|---------|-----|
| `bblanchon/ArduinoJson` | ^7.0.0 | Serialización JSON del payload |
| `adafruit/Adafruit SHT31 Library` | ^2.2.0 | Driver I2C para SHT31-DIS |
| `adafruit/Adafruit BusIO` | ^1.14.0 | Abstracción I2C/SPI (dependencia de SHT31) |
| `milesburton/DallasTemperature` | ^3.11.0 | Driver 1-Wire para DS18B20 |
| `paulstoffregen/OneWire` | ^2.3.7 | Protocolo 1-Wire (dependencia de DallasTemperature) |
| `knolleary/PubSubClient` | ^2.8 | Cliente MQTT para Arduino/ESP32 |

**Nota sobre PubSubClient**: Esta librería NO es nativamente asíncrona. El firmware compensa esto ejecutándola en el Core 1 (separado del Core 0 de sensores). El `mqtt.loop()` se llama cada ~100 ms para procesar keep-alive, PUBACK y callbacks. Para una versión futura se podría migrar a `AsyncMqttClient` (basado en eventos, más eficiente) pero PubSubClient es suficiente para la cadencia de 30s de este prototipo.

---

## 5. Instalación y Despliegue

### 5.1 Requisitos

- [PlatformIO IDE](https://platformio.org/) (extensión de VS Code) o PlatformIO Core (CLI)
- Python 3.9+ (para scripts internos de PlatformIO)
- Driver USB-UART: CP210x (la mayoría de ESP32 DevKitC) o CH340 (clones)
- Cable USB micro-B (datos + alimentación)
- Git (para clonar el repositorio)

### 5.2 Clonar y Configurar

```bash
git clone https://github.com/Iot-Tesis-DG/iot-firmware.git
cd iot-firmware
```

### 5.3 Obtener Certificado CA Raíz

```bash
# Let's Encrypt ISRG Root X1 (el que usa EMQX Cloud Serverless)
curl -o data/certs/root_ca.pem https://letsencrypt.org/certs/isrgrootx1.pem

# Verificar que se descargó correctamente
head -1 data/certs/root_ca.pem
# Debe mostrar: -----BEGIN CERTIFICATE-----
```

### 5.4 Configurar Credenciales (2 opciones)

**Opción A — `platformio.ini` (recomendado para producción)**:

```ini
build_flags =
    -DCORE_DEBUG_LEVEL=0
    -DWIFI_SSID='"MiRedWiFi"'
    -DWIFI_PASSWORD='"contraseña_segura"'
    -DMQTT_TOKEN='"token_generado_en_emqx"'
```

**Opción B — `config.h` (solo desarrollo local)**:

```cpp
#define WIFI_SSID    "MiRedWiFi"
#define WIFI_PASSWORD "contraseña_segura"
#define MQTT_TOKEN   "token_generado_en_emqx"
```

### 5.5 Compilar, Flashear, Ejecutar

```bash
# Compilar
pio run

# Flashear firmware a ESP32
pio run --target upload

# Flashear LittleFS (certificados TLS)
pio run --target uploadfs

# Monitorear salida serie (115200 baud)
pio device monitor
```

### 5.6 Configurar EMQX Cloud

1. Crear instancia Serverless gratuita en https://www.emqx.com/en/cloud
2. Copiar el **hostname** (ej. `abc123.emqx.cloud`) → `MQTT_HOST` en `config.h`
3. Ir a **Authentication & ACL** → Database → Built-in Database
4. Agregar usuario:
   - Username: `FARM-01-CDL`
   - Password: token seguro (generar con `openssl rand -hex 16`)
5. En **TLS/SSL** → verificar puerto **8883** habilitado
6. En **ACL** → regla: `pub farmacias/FARM-01-CDL/#` para el usuario `FARM-01-CDL`
7. Copiar el token generado → `MQTT_TOKEN` en `config.h`

### 5.7 Configurar Backend

En `backend/.env`:

```bash
MQTT_ENABLED=true
MQTT_HOST=abc123.emqx.cloud
MQTT_PORT=8883
MQTT_USERNAME=backend_service
MQTT_PASSWORD=token_del_backend_en_emqx
MQTT_TLS_ENABLED=true
```

El backend se suscribe a `farmacias/+/lecturas` y `farmacias/+/eventos`.  
Asegurar que el usuario `backend_service` tenga permisos de suscripción en EMQX ACL.

### 5.8 Probar End-to-End

```bash
# Terminal 1: Monitor serie del ESP32
pio device monitor
# Debe mostrar:
# ╔══════════════════════════════════════════════════════╗
# ║   ThermoTrace — Firmware IoT Cadena de Frío        ║
# ║   Device ID : FARM-01-CDL                          ║
# ║   Firmware  : 1.0.0                                ║
# ╚══════════════════════════════════════════════════════╝
# [LittleFS] Montado. 0 lecturas pendientes. 1400 KB libres.
# [NTP] Hora sincronizada: 2026-07-25T12:30:00Z
# [WiFi] Conectado! IP: 192.168.1.100, RSSI: -45 dBm.
# [MQTT] Certificado CA cargado (1934 bytes).
# [MQTT] Conectado a EMQX. Publicando LWT 'online'...
# [MQTT] Listo. Publicando en 'farmacias/FARM-01-CDL/lecturas'.
# [Core0] Ciclo completado. Pendientes: 1 en RAM, 1 en Flash.

# Terminal 2: Logs del backend (Railway)
railway logs
# Debe mostrar:
# INFO:interface.main:Lectura recibida de FARM-01-CDL
# INFO:application.use_cases.clasificar_riesgo_termico:Clasificación: normal (confianza: 0.98)

# Terminal 3: Dashboard
# Abrir http://localhost:5173/dashboard
# Debe mostrar temperatura en tiempo real actualizándose cada 30s
```

---

## 6. Personalización para Múltiples Dispositivos

Para desplegar varios ESP32 (uno por refrigerador):

```cpp
// config.h — ÚNICAS 3 LÍNEAS A CAMBIAR POR DISPOSITIVO:

#define DEVICE_ID    "FARM-02-CDL"        // <-- diferente por cada ESP32
// WIFI_SSID y WIFI_PASSWORD pueden ser iguales (misma red)
// MQTT_TOKEN debe ser diferente → crear credenciales separadas en EMQX

// Alternativa vía build_flags (sin tocar config.h):
//   -DDEVICE_ID='"FARM-02-CDL"'
//   -DMQTT_TOKEN='"token_farm02"'
```

Cada dispositivo publica en `farmacias/{DEVICE_ID}/lecturas`.  
El backend recibe de todos vía `farmacias/+/lecturas` (comodín MQTT).  
En EMQX, crear reglas ACL separadas por cada `DEVICE_ID` para mínimo privilegio.

---

## 7. Solución de Problemas

### 7.1 Códigos de Error MQTT (PubSubClient)

| Código | Significado | Causa típica |
|--------|-------------|-------------|
| -4 | MQTT_CONNECTION_TIMEOUT | EMQX no responde. ¿Host correcto? ¿Puerto 8883 abierto? |
| -3 | MQTT_CONNECTION_LOST | Conexión TCP caída durante transmisión |
| -2 | MQTT_CONNECT_FAILED | Error de red (Wi-Fi caído, DNS no resuelve) |
| -1 | MQTT_DISCONNECTED | Desconectado limpiamente por el cliente |
| 1 | MQTT_CONNECT_BAD_PROTOCOL | Versión de protocolo MQTT no soportada |
| 2 | MQTT_CONNECT_BAD_CLIENT_ID | Client ID rechazado por el broker |
| 3 | MQTT_CONNECT_UNAVAILABLE | Broker no disponible |
| 4 | MQTT_CONNECT_BAD_CREDENTIALS | Usuario o contraseña incorrectos |
| 5 | MQTT_CONNECT_UNAUTHORIZED | Credenciales válidas pero sin permisos ACL |

### 7.2 Problemas Comunes

| Síntoma | Causa probable | Solución |
|---------|---------------|----------|
| `[WiFi] ERROR: Timeout de conexión` | SSID/password incorrectos o AP fuera de alcance | Verificar credenciales. ESP32 solo soporta Wi-Fi 2.4 GHz, no 5 GHz. Verificar que el AP no tenga filtro MAC. |
| `[MQTT] ERROR: -1` | Handshake TLS falló | ¿Certificado CA flasheado? Ejecutar `pio run --target uploadfs`. Verificar que `root_ca.pem` sea ISRG Root X1 (no el intermedio). |
| `[MQTT] ERROR: 4` | Credenciales inválidas | Verificar en EMQX Cloud Console → Authentication que el usuario y contraseña coincidan. |
| `[MQTT] ERROR: 5` | Usuario sin permisos de publicación | En EMQX ACL, verificar regla: `pub farmacias/FARM-01-CDL/#` para el usuario. |
| `[DS18B20] ERROR: Sensor no responde` | Pull-up de 4.7kΩ faltante | Conectar resistencia entre DATA (GPIO4) y 3.3V. Medir con multímetro: debe haber ~3.3V en DATA. |
| `[DS18B20] ERROR: Lectura fuera de rango físico: 85.00 °C` | Sensor mal conectado o bus 1-Wire con ruido | 85°C es el valor power-on reset del DS18B20. Verificar que GND esté bien conectado. Reducir longitud del cable (<3m). |
| `[SHT31] ERROR: No se detectó el sensor en 0x44` | Cableado I2C incorrecto | SDA=GPIO21 (cable amarillo), SCL=GPIO22 (cable verde). Usar escáner I2C para verificar dirección. |
| `[SHT31] ERROR: Fallo en lectura de temperatura/humedad` | Sensor en modo sleep o necesitando calentamiento | El SHT31-DIS se autocalienta para evitar condensación. Esperar 1-2 ciclos. Verificar alimentación 3.3V estable. |
| `[LittleFS] ERROR: Fallo al montar` | Partición no existe o está corrupta | Ejecutar `pio run --target uploadfs`. Verificar en `platformio.ini`: `board_build.filesystem = littlefs`. |
| `[Payload] JSON demasiado grande: 600 bytes (máx 512)` | Payload excede límite | Verificar longitud de `device_id`, `firmware_version`. Reducir bajo 50 y 20 caracteres respectivamente. |
| Dashboard no muestra datos | Backend no conectado a EMQX o SSE caído | Verificar logs del backend. Verificar `MQTT_ENABLED=true` en `.env`. Verificar CORS para SSE. |
| Timestamps con fechas de 1970 | NTP no sincronizó | El ESP32 necesita acceso a Internet (puerto 123/UDP). Si está en red aislada, configurar `configTime()` con servidor NTP local. |
| `Guru Meditation Error: Core 0 panic'ed (LoadProhibited)` | Stack overflow en taskSensores | Aumentar stack: cambiar `8192` a `10240` en `xTaskCreatePinnedToCore` para taskSensores en `main.cpp:235`. |
| `Guru Meditation Error: Core 1 panic'ed (LoadProhibited)` | Stack overflow en taskRed | Aumentar stack: cambiar `12288` a `16384` en `xTaskCreatePinnedToCore` para taskRed en `main.cpp:245`. TLS necesita más stack. |
| ESP32 se reinicia cada pocos minutos | Watchdog timer (WDT) | Verificar que `mqtt.loop()` y `delay(100)` se llamen frecuentemente en taskRed. Si hay operaciones bloqueantes >5s, el WDT reinicia. |

---

## 8. Limitaciones Conocidas y Mejoras Futuras

### 8.1 Limitaciones

| Limitación | Impacto | Plan |
|-----------|---------|------|
| PubSubClient es síncrono (no async) | `mqtt.loop()` debe llamarse frecuentemente. Si `publish()` toma >5s, el WDT reinicia el ESP32. | Bajo. Con 30s de cadencia y Wi-Fi estable, `publish()` completa en <500ms. |
| Sin OTA (Over-The-Air updates) | Firmware solo se actualiza por USB. | Las particiones de flash ya reservan `app1` (1.2 MB) para OTA futura. Implementar con `ArduinoOTA` o `AsyncElegantOTA`. |
| Timestamp depende de NTP (sin RTC) | Si NTP falla, se usa hora de compilación + millis(). Puede desfasarse minutos tras horas de uptime. | El backend valida ±2h. Para precisión sub-second, agregar RTC externo (DS3231). |
| Sin deep sleep entre lecturas | Consumo constante ~200mA (Wi-Fi activo). | Para batería: usar `esp_deep_sleep` entre lecturas. Despertar cada 30s, leer sensores, publicar, dormir. Consumo baja a ~10µA. |
| Sin pantalla/LED de estado avanzado | Solo monitor serie. Sin indicador visual in situ. | Agregar LED RGB: verde (todo OK), amarillo (sin MQTT), rojo (sin Wi-Fi), azul (publicando). |
| Certificado CA hardcodeado a una ruta | Si el CA cambia (ej. Let's Encrypt revoca ISRG Root X1 en 2035), hay que reflashear. | Usar `Preferences` (NVS) para almacenar múltiples CA. Intentar cada uno hasta que uno funcione. |

### 8.2 Mejoras Futuras (no requeridas para el prototipo)

1. **OTA firmware updates** → `app1` partición ya reservada. Usar endpoint `GET /api/firmware/releases` del backend.
2. **Deep sleep entre lecturas** → consumo <1 mAh/día con batería LiPo 2000mAh (meses de autonomía).
3. **Indicador LED RGB** → diagnóstico visual in situ sin necesidad de monitor serie.
4. **Migrar a AsyncMqttClient** → cliente MQTT verdaderamente asíncrono basado en eventos (sin necesidad de `loop()`).
5. **RTC externo DS3231** → timestamp preciso incluso sin NTP.
6. **BLE provisioning** → configurar Wi-Fi desde app móvil sin recompilar.
7. **Watchdog externo (TPS3823)** → reinicio automático si el ESP32 se cuelga.
8. **Encriptación de LittleFS** → `mbedtls` AES-256 para datos en reposo.

---

## 9. Referencias

| Documento | Ubicación |
|-----------|-----------|
| Trabajo de Investigación (TI) | `documentacion/Documentacion-general/Final - Oficial/TI_Soto_Diego_Gamio_Brenda.docx` |
| Product Backlog (Anexo5) | `C:\Users\gamio\Downloads\Anexo5_PB_Soto_Diego_Gamio_Brenda.md` |
| Auditoría Fase 3 (Backend) | `documentacion/Documentacion-general/backend/ANALISIS_COMPLETO_BACKEND.md` |
| Análisis Final Consolidado | `documentacion/Documentacion-general/analisis_final_estado_actual.md` |
| Plan de Implementación 100% | `documentacion/Documentacion-general/plan_implementacion_100porciento.md` |
| Código fuente IoT | `https://github.com/Iot-Tesis-DG/iot-firmware.git` |
| EMQX Cloud Docs | https://docs.emqx.com/en/cloud/latest/ |
| ESP32 Technical Reference | https://www.espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf |
| ArduinoJson v7 Docs | https://arduinojson.org/v7/ |
| OWASP IoT Security Testing Guide | https://owasp.org/www-project-internet-of-things-security-testing-guide/ |
