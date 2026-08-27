# Historias de Usuario — Product Backlog

**Informe de tareas pendientes de productos (Product Backlog)**

Sistema de monitoreo de cadena de frío para refrigeradores farmacéuticos (ESP32 + EMQX Cloud + FastAPI + React + PostgreSQL), con trazabilidad criptográfica SHA-256 y cumplimiento normativo DIGEMID / BPA / Ley N.° 29733 / OWASP.

---

## 1. Resumen general

| Concepto | Valor |
|---|---|
| Historias de usuario | 51 |
| Épicas | 6 (EP01 – EP06) |
| Sprints | 10 (SP-01 – SP-10), de 4 semanas cada uno |
| Duración total | 40 semanas · 200 días · 1600 horas |
| **Total Story Points** | **271** |
| Estado general | No se ha iniciado |

**Distribución por prioridad:**

| Prioridad | Cantidad | Historias |
|---|---|---|
| Alta | 32 | HU-01, 02, 05, 06, 07, 09, 10, 11, 12, 15, 16, 17, 18, 21, 22, 23, 24, 25, 26, 31, 32, 34, 39, 40, 41, 43, 44, 45, 46, 47, 48, 49 |
| Media | 17 | HU-03, 04, 08, 13, 14, 19, 20, 27, 28, 30, 33, 35, 36, 38, 42, 50, 51 |
| Baja | 2 | HU-29, 37 |

---

## 2. Épicas

| ID | Épica | Descripción | HU incluidas |
|---|---|---|---|
| **EP01** | Adquisición de Datos y Tolerancia a Fallos (Edge/ESP32) | Captura de variables térmicas/ambientales/apertura, estructuración en JSON interoperable, y continuidad del registro ante cortes de energía o conectividad mediante persistencia local y reconexión automática. Base de toda la cadena de datos. | HU-01, 02, 03, 04, 05, 06, 07, 08, 46, 51 |
| **EP02** | Comunicación y Seguridad MQTT (EMQX Cloud) | Transporte cifrado y autenticado de telemetría: handshake TLS, autenticación por dispositivo, QoS garantizado y bloqueo de canales no cifrados. Primera línea de defensa contra manipulación de datos en tránsito. | HU-09, 10, 11, 12, 13, 14 |
| **EP03** | Procesamiento e Inteligencia Artificial (Backend/FastAPI) | Validación de payloads, inferencia de riesgo con Random Forest, generación de alertas preventivas/críticas, persistencia asíncrona, notificación SSE al dashboard, y notificación asíncrona para alertar al responsable cuando el dashboard no esté abierto. Componente central para el procesamiento y clasificación del riesgo térmico. | HU-15, 16, 17, 18, 19, 20, 21, 22, 23 |
| **EP04** | Trazabilidad Digital e Integridad (PostgreSQL) | Encadenamiento criptográfico SHA-256, verificación de integridad de la cadena, registro y auditoría de acciones correctivas, backup automático, y seguimiento del vencimiento de calibración de sensores conforme al marco normativo aplicable. Componente orientado a reemplazar registros manuales por evidencia digital verificable ante auditorías internas o inspecciones sanitarias. | HU-24, 25, 26, 27, 28, 29, 30, 43, 47, 48, 49 |
| **EP05** | Monitoreo Web y Visualización (Frontend React) | Interfaz para ver el estado en tiempo real, filtrar historial, completar el checklist BPA y exportar evidencia en PDF. Capa de interacción operativa para el personal responsable del monitoreo y la evidencia sanitaria. | HU-31, 32, 33, 34, 35, 36, 37, 38 |
| **EP06** | Autenticación, Roles y Auditoría (Seguridad transversal) | Login seguro, protección anti-XSS del JWT, control de acceso por roles (RBAC) y bitácora inmutable de auditoría. Componente transversal que protege las operaciones críticas de las demás épicas. | HU-39, 40, 41, 42, 44, 45, 50 |

---

## 3. Historias de usuario por épica

> Formato de cada historia: **Como** [rol], **quiero** [capacidad], **para** [beneficio].

---

# Épica EP01 · Adquisición de Datos y Tolerancia a Fallos (Edge/ESP32)

---

### HU-01 · Captura de temperatura interna (DS18B20)

- **Como** farmacéutico responsable
- **Quiero** que el sistema capture la temperatura del sensor DS18B20 cada 30 segundos
- **Para** vigilar con precisión la condición térmica cercana al medicamento
- **Prioridad:** Alta · **Sprint:** 1 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (lectura periódica correcta):** Dado que el ESP32 está energizado y el sensor está conectado correctamente, cuando transcurre el intervalo de 30s, entonces se captura la lectura con precisión de ±0.5 °C y se marca con timestamp.
2. **Escenario 2 (sensor desconectado o en falla):** Dado que el bus 1-Wire no detecta el sensor o este devuelve un valor de error característico (p. ej. -127 °C), cuando se ejecuta el ciclo de lectura, entonces el valor se descarta, se registra un evento de error de sensor y no se incluye como lectura válida en el payload.
3. **Escenario 3 (valor fuera de rango físico):** Dado que la lectura excede el rango de operación del DS18B20 (-55 °C a 125 °C), cuando se valida el dato, entonces se marca como anómalo sin detener el ciclo de muestreo.

---

### HU-02 · Captura de temperatura ambiental (SHT31)

- **Como** farmacéutico responsable
- **Quiero** que el sistema capture la temperatura ambiental mediante el sensor SHT31
- **Para** complementar el control de la temperatura interna del refrigerador
- **Prioridad:** Alta · **Sprint:** 1 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (lectura correcta):** Dado que el bus I2C está activo y el SHT31 responde con ACK, cuando se solicita lectura, entonces se registra la temperatura ambiental con precisión de ±0.2 °C.
2. **Escenario 2 (fallo de comunicación I2C):** Dado que el sensor no responde dentro del timeout configurado, cuando se intenta la lectura, entonces el sistema reintenta hasta un máximo de 3 veces antes de marcar la lectura como no disponible.
3. **Escenario 3 (valor fuera de rango de fábrica):** Dado que el valor excede el rango de operación del SHT31 (-40 °C a 125 °C), cuando se valida, entonces se descarta y se registra como anomalía de sensor.

---

### HU-03 · Captura de humedad relativa (SHT31)

- **Como** farmacéutico responsable
- **Quiero** que el sistema registre la humedad relativa del sensor SHT31
- **Para** detectar condiciones ambientales de riesgo adicionales a la temperatura
- **Prioridad:** Media · **Sprint:** 1 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (lectura correcta):** Dado que se captura exitosamente la temperatura por I2C, cuando se procesa la respuesta del SHT31, entonces se extrae el % de humedad relativa con precisión de ±2% RH.
2. **Escenario 2 (riesgo de condensación):** Dado que la humedad relativa registrada supera el 80% RH, cuando se evalúa la lectura, entonces se marca con un indicador de riesgo de condensación dentro del payload.
3. **Escenario 3 (lectura fuera de rango):** Dado que el valor está fuera de 0–100% RH, cuando se valida, entonces se descarta como lectura inválida.

---

### HU-04 · Detección de apertura de puerta (MC-38)

- **Como** técnico de farmacia
- **Quiero** que el sistema registre los eventos del sensor magnético MC-38
- **Para** relacionar aperturas reales de la puerta con fluctuaciones térmicas
- **Prioridad:** Media · **Sprint:** 1 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (apertura/cierre real):** Dado que la puerta cambia de estado físicamente, cuando el GPIO se mantiene en el nuevo estado más allá del tiempo mínimo de antirrebote (debounce), entonces se actualiza apertura_refrigerador y se registra el timestamp del cambio.
2. **Escenario 2 (rebote de contacto):** Dado que el GPIO cambia de estado por menos del tiempo de antirrebote, cuando se evalúa la señal, entonces el cambio se ignora y no se reporta.
3. **Escenario 3 (apertura prolongada):** Dado que la puerta permanece abierta más allá de un umbral configurado, cuando se detecta esta condición, entonces se añade al payload un contador de duración de apertura.

---

### HU-05 · Construcción de Payload JSON

- **Como** administrador del sistema
- **Quiero** que el sistema estructure las métricas en un payload JSON estándar
- **Para** asegurar interoperabilidad con el backend aunque alguna variable no esté disponible
- **Prioridad:** Alta · **Sprint:** 1 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (payload completo y ajustado a la arquitectura):** Dado que las variables de los sensores se capturaron correctamente, cuando se preparan para envío, entonces se genera un JSON con deviceid, timestamp (ISO 8601 UTC), estadoconectividad y firmware_version.
2. **Escenario 2 (variable faltante):** Dado que algún sensor reportó error en el ciclo actual, cuando se construye el payload, entonces el campo correspondiente se serializa explícitamente como null con bandera de error, en vez de omitirse.
3. **Escenario 3 (validación de tamaño):** Dado que el payload excede el tamaño máximo definido (aprox. 250 bytes), cuando se intenta serializar, entonces se registra un error en el log local y no se intenta publicar.

---

### HU-06 · Persistencia en Buffer Offline (LittleFS)

- **Como** farmacéutico responsable
- **Quiero** que el sistema conserve las lecturas en memoria local cuando se pierda conectividad
- **Para** no perder trazabilidad térmica durante cortes prolongados
- **Prioridad:** Alta · **Sprint:** 2 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (guardado por bloques para cuidar la memoria flash):** Dado que no hay red Wi-Fi o MQTT disponible, cuando se intenta publicar una lectura, entonces la lectura se acumula temporalmente en RAM y se vuelca periódicamente en bloques a LittleFS con un nombre secuencial, para minimizar los ciclos de escritura en la memoria no volátil preservando el orden cronológico.
2. **Escenario 2 (buffer lleno):** Dado que el espacio en LittleFS alcanza el umbral máximo configurado, cuando se intenta guardar una nueva lectura, entonces se aplica política FIFO descartando el archivo más antiguo y se registra un evento de saturación.
3. **Escenario 3 (corte de energía durante escritura):** Dado que ocurre una pérdida de energía mientras se escribe un archivo, cuando el ESP32 reinicia, entonces se descarta cualquier archivo incompleto o corrupto detectado por verificación de integridad antes de reanudar la operación.

---

### HU-07 · Sincronización de Buffer Offline

- **Como** farmacéutico responsable
- **Quiero** que el sistema reenvíe automáticamente los registros acumulados en LittleFS
- **Para** completar el histórico térmico cuando la conexión se recupere, sin duplicidades ni pérdidas
- **Prioridad:** Alta · **Sprint:** 2 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (sincronización exitosa bajo QoS 1):** Dado que el ESP32 se reconecta a EMQX y existen archivos de telemetría pendientes, cuando se inicia el proceso de sincronización, entonces se publican en estricto orden FIFO utilizando QoS 1, y cada bloque se elimina de LittleFS únicamente tras recibir el paquete PUBACK de confirmación del broker.
2. **Escenario 2 (fallo a mitad de sincronización):** Dado que la conexión se interrumpe abruptamente durante la transmisión de los registros, cuando se recupera la red nuevamente, entonces el sistema reanuda la cola de envíos partiendo estrictamente desde el siguiente archivo al último confirmado por PUBACK, sin reenviar duplicados.
3. **Escenario 3 (orden cronológico e integridad):** Dado que existen múltiples archivos pendientes de distintas horas o días, cuando se sincronizan hacia el backend, entonces se publican respetando su marca de tiempo (timestamp) original de captura, permitiendo a la base de datos rearmar la serie temporal correctamente.

---

### HU-08 · Gestión de reconexión Wi-Fi

- **Como** administrador del sistema
- **Quiero** que el nodo se reconecte usando una estrategia de backoff exponencial
- **Para** restablecer el servicio sin saturar la red local ni el microcontrolador
- **Prioridad:** Media · **Sprint:** 2 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (backoff progresivo):** Dado que el nodo detecta la caída de la red, cuando el ESP32 intenta reconectar, entonces el tiempo de espera entre intentos aumenta de forma exponencial (1s, 2s, 4s, 8s, 16s…) hasta alcanzar un tope de espera máximo configurado, momento en el cual el intervalo se estabiliza.
2. **Escenario 2 (reconexión exitosa):** Dado que un intento de reconexión logra enlazarse al punto de acceso (AP), cuando se restablece la comunicación, entonces el contador del backoff se reinicia a su valor base (1s) para futuros eventos de desconexión.
3. **Escenario 3 (credenciales inválidas o fallo físico):** Dado que las credenciales Wi-Fi almacenadas ya no son válidas o el AP ha sido apagado, cuando se agotan los reintentos iniciales críticos, entonces el nodo suspende la búsqueda agresiva, registra el fallo en el log local de LittleFS y activa un indicador visual (ej. LED de estado en rojo/parpadeo) para diagnóstico in situ por parte del personal de la farmacia.

---

### HU-46 · Actualización Segura de Firmware OTA (Over-The-Air)

- **Como** Administrador del Sistema / Administrador IoT
- **Quiero** desplegar actualizaciones de firmware cifradas al ESP32 de forma remota (OTA), verificando integridad y permitiendo rollback en caso de fallo
- **Para** parchear vulnerabilidades de seguridad detectadas (OWASP IoT v1.0.0) sin requerir intervención física en cada refrigerador farmacéutico
- **Prioridad:** Alta · **Sprint:** 9 · **Story Points:** 13 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (Preparación y validación de firmware):** Dado que se descubre una vulnerabilidad de seguridad en la librería de comunicación MQTT del ESP32, cuando el equipo de desarrollo compila una nueva versión del firmware con el parche, entonces el flujo incluye:
   - Generar hash SHA-256 del binario .bin compilado.
   - Cifrar el binario con AES-256 en el backend (la clave NO queda hardcodeada en el código fuente).
   - Subir el binario cifrado al almacenamiento interno de la infraestructura de bajo coste (opción del proyecto: Railway/Postgres como BLOB/bytea), evitando dependencia de servicios externos costosos.
   - Registrar en BD: versión, hash, fecha de compilación, descripción del parche, metadatos del release.
   - Crear evento en traceability_records: tipo_evento: "FIRMWARE_PREPARACION", payload: {versión_firmware, hash_sha256, fecha_compilacion, descripción_parche}, con su hash SHA-256 encadenado.
2. **Escenario 2 (Despliegue selectivo del firmware):** Dado que el firmware ha sido validado y aprobado, cuando el Administrador accede al módulo "OTA - Actualización de Dispositivos", entonces ve:
   - Un listado de todos los dispositivos ESP32 activos con versión de firmware actual.
   - La opción "Desplegar nueva versión" con controles para: seleccionar dispositivos específicos (por farmacia, por tipo de sensor, o todos); ventana de tiempo de despliegue (ej. "20:00 a 06:00" para horario no operativo); número máximo de dispositivos simultáneos (ej. 5 en paralelo para no sobrecargar red).
   - Un botón "Programar Despliegue" que requiere confirmación adicional.
   - Antes de aceptar la programación, FastAPI valida que la versión indicada sea superior a la current_version de cada dispositivo objetivo y rechaza despliegues que supongan downgrade (esto evita descargas innecesarias y ataques de bandwidth).
3. **Escenario 3 (Envío cifrado y verificación en ESP32):** Dado que el Administrador confirma el despliegue, cuando llega la ventana de tiempo programada, entonces el backend (FastAPI):
   - Crea un canal MQTT seguro específico para OTA: /farmacias/{device_id}/ota/update.
   - Envía el firmware cifrado en fragmentos (chunks) con: número de secuencia, hash SHA-256 del fragmento, checksum del total.
   - El ESP32 recibe cada fragmento y verifica: hash SHA-256 de cada chunk, checksums progresivos; si algún fragmento es corrupto o ataca man-in-the-middle, rechaza y solicita reenvío.
   - Una vez recibido completo, el ESP32: procesa la desencriptación on-the-fly usando la clave AES-256 aprovisionada en NVS; verifica el hash SHA-256 del binario completo contra lo esperado; si falla, rechaza la actualización y notifica al backend.
4. **Escenario 4 (Instalación y validación del firmware):** Dado que el ESP32 ha verificado el firmware completo correctamente, cuando confirma la integridad, entonces procede a:
   - Escribir el nuevo firmware directamente en la partición OTA inactiva (ej. OTA_1) — el ESP32 no mantiene una copia en RAM del binario completo.
   - Cambiar el puntero de arranque al nuevo slot y reiniciar.
   - Durante boot, ejecutar autotests de: conectividad TLS (intenta conectar a broker), lectura de sensores (valida que DS18B20 y SHT31 responden), autenticación MQTT (prueba credenciales).
   - Si autotests pasan, marca versión nueva como estable.
   - Si falla, el bootloader revierte el puntero de arranque a la partición previa (rollback) sin copiar firmware en memoria intermedia.
5. **Escenario 5 (Notificación y auditoría del despliegue):** Dado que se completa la actualización de firmware, cuando el ESP32 confirma éxito o fallo, entonces registra en traceability_records: tipo_evento: "FIRMWARE_ACTUALIZADO" o "FIRMWARE_ROLLBACK", device_id, versión_anterior, versión_nueva, timestamp del cambio, resultado (éxito/fallo), hash SHA-256 encadenado. Y el Administrador recibe un reporte: dispositivos actualizados exitosamente [lista], dispositivos que requieren atención (rollback) [lista], timestamp de cada operación.
6. **Escenario 6 (Protección contra downgrade attacks):** Dado que un atacante intenta forzar una versión anterior de firmware (con una vulnerabilidad conocida), cuando envía OTA con versión 1.0.0 (antigua) a un dispositivo con 1.0.2 (actual), entonces el ESP32: valida que el número de versión sea mayor o igual al actual; si es anterior, rechaza el despliegue; registra un intento de downgrade malicioso en el log local (LittleFS); notifica al backend del ataque detectado.

---

### HU-51 · Registro de Metadata del Mapeo Térmico y Coordenadas del Sensor

- **Como** Químico Farmacéutico Responsable
- **Quiero** configurar la ubicación física exacta del sensor de temperatura interno (DS18B20) basado en el informe del estudio de Mapeo Térmico del refrigerador
- **Para** garantizar a los inspectores de DIGEMID que el sensor está midiendo el punto de mayor oscilación de temperatura de la unidad de frío (BPA Componente 3)
- **Prioridad:** Media · **Sprint:** 10 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (Metadata de posicionamiento por mapeo):** Dado que el Administrador registra o edita un dispositivo de refrigeración en el sistema, cuando ingresa el "Código de Informe del Mapeo Térmico" (ej: "INF-MAP-2026-004"), la "Ubicación del sensor" (ej: "Estante 2, fondo derecho - Punto Caliente") y la "Fecha de vencimiento de la calibración", entonces el sistema persiste estos datos en el modelo de dispositivo y los añade como metadata fija y auditable en el encabezado de todos los reportes históricos y gráficos emitidos.

---

# Épica EP02 · Comunicación y Seguridad MQTT (EMQX Cloud)

---

### HU-09 · Handshake TLS 1.2 en el Borde

- **Como** administrador del sistema
- **Quiero** que el nodo valide el certificado de la Autoridad Certificadora (CA) del servidor mediante One-way TLS 1.2/1.3
- **Para** cifrar los datos en tránsito y evitar ataques Man-in-the-Middle
- **Prioridad:** Alta · **Sprint:** 2 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (handshake exitoso):** Dado que el ESP32 inicia conexión hacia el broker EMQX en el puerto seguro 8883, cuando el certificado del servidor es validado como confiable contra la CA raíz almacenada, entonces se establece un canal cifrado y la sesión MQTT puede continuar de forma segura.
2. **Escenario 2 (certificado inválido o expirado):** Dado que el certificado presentado por el servidor no es válido o no coincide con el dominio esperado, cuando se evalúa durante el handshake criptográfico, entonces el ESP32 rechaza la conexión inmediatamente y no publica ningún dato térmico en texto plano.
3. **Escenario 3 (timeout del handshake):** Dado que la negociación asimétrica TLS no se completa dentro del tiempo límite configurado por latencia de red, cuando se agota dicho tiempo, entonces el intento se cancela de forma segura, se registra como fallo de conexión en el log local y se activa el flujo de reconexión con backoff exponencial.

---

### HU-10 · Autenticación de dispositivo (SNI)

- **Como** administrador del sistema
- **Quiero** que cada nodo ESP32 use credenciales únicas y validación Server Name Indication (SNI)
- **Para** impedir el acceso de hardware no autorizado conforme a OWASP IoT
- **Prioridad:** Alta · **Sprint:** 2 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (credenciales válidas):** Dado que un cliente IoT se conecta utilizando un device_id único como usuario, su token/contraseña correspondiente y un SNI correcto, cuando el broker valida las credenciales, entonces acepta la conexión y restringe sus permisos de publicación únicamente a sus tópicos designados.
2. **Escenario 2 (credenciales inválidas):** Dado que un cliente intenta establecer conexión con un token incorrecto o revocado, cuando el broker evalúa la petición, entonces se rechaza la conexión enviando el código de respuesta MQTT 0x05 (Not Authorized) y se deniega cualquier intento de publicación.
3. **Escenario 3 (SNI ausente o incorrecto):** Dado que un cliente omite el encabezado SNI o lo envía con un dominio no reconocido durante el inicio de la capa de transporte, cuando el servidor procesa la solicitud, entonces deniega la conexión cerrando el socket antes de completar el handshake TLS, ahorrando recursos del servidor.

---

### HU-11 · Publicación de Telemetría (QoS 1)

- **Como** administrador del sistema
- **Quiero** que las lecturas térmicas se publiquen con QoS 1 (al menos una vez)
- **Para** confirmar la recepción en el broker y evitar vacíos en la serie temporal
- **Prioridad:** Alta · **Sprint:** 3 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (publicación confirmada):** Dado que el ESP32 publica un payload JSON con QoS 1 hacia el tópico correspondiente, cuando el broker EMQX lo recibe e ingesta con éxito, entonces responde con el paquete PUBACK y el ESP32 libera de forma segura la memoria RAM asociada a esa lectura.
2. **Escenario 2 (PUBACK no recibido):** Dado que el paquete PUBACK no llega dentro del tiempo de espera configurado debido a intermitencias en la red, cuando se agota el timeout, entonces el ESP32 reintenta la publicación de la misma lectura con el flag DUP (Duplicate) activado, alertando al broker de un posible reenvío.
3. **Escenario 3 (reintentos agotados y contingencia):** Dado que se agota el número máximo de reintentos de publicación QoS 1 sin respuesta del servidor, cuando esto ocurre, entonces se declara la caída de la conexión y la lectura térmica se conserva intacta en el buffer persistente (LittleFS) para su sincronización posterior, en lugar de descartarse.

---

### HU-12 · Suscripción centralizada del Backend

- **Como** administrador del sistema
- **Quiero** que el backend se suscriba al tópico farmacias/+/lecturas mediante aiomqtt
- **Para** consumir telemetría de cualquier refrigerador sin bloquear el servidor
- **Prioridad:** Alta · **Sprint:** 3 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (suscripción al arranque sin bloqueo):** Dado que la aplicación FastAPI arranca, cuando se ejecuta el evento de inicio en el ciclo de vida (lifespan), entonces el cliente aiomqtt se suscribe al tópico mediante comodín (+) integrándose nativamente al event loop de Uvicorn, quedando listo para recibir mensajes de cualquier dispositivo.
2. **Escenario 2 (pérdida de conexión con el broker):** Dado que la conexión de red del backend hacia EMQX Cloud se interrumpe, cuando la librería detecta la desconexión, entonces el cliente asíncrono se re-suscribe automáticamente al recuperar la conexión, sin requerir reinicio manual del contenedor en Railway.
3. **Escenario 3 (tópico no reconocido):** Dado que llega un mensaje bajo un patrón de tópico distinto al esperado (ej. inyección de datos basura), cuando el backend lo recibe, entonces el sistema lo descarta mediante validación estricta y registra un log de advertencia, sin afectar el procesamiento del resto de mensajes.

---

### HU-13 · Emisión de Estado de Conectividad (LWT)

- **Como** administrador del sistema
- **Quiero** que el nodo configure un mensaje Last Will and Testament (LWT) en el broker
- **Para** notificar caídas abruptas de energía o red al backend y al dashboard
- **Prioridad:** Media · **Sprint:** 3 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (desconexión abrupta):** Dado que el nodo pierde energía o conectividad sin enviar un cierre ordenado, cuando el broker detecta el timeout del keep-alive, entonces EMQX publica el LWT preconfigurado enviando un payload JSON con el campo estado_conectividad: offline al tópico de alertas del dispositivo.
2. **Escenario 2 (desconexión ordenada):** Dado que el dispositivo se desconecta limpiamente (ej. actualización de firmware OTA o mantenimiento), cuando se envía el paquete de red DISCONNECT, entonces el broker descarta el mensaje LWT, ya que la desconexión fue intencional.
3. **Escenario 3 (reconexión tras LWT):** Dado que el dispositivo se reconecta y reinicia su ciclo después de haber disparado su LWT, cuando se restablece la sesión segura, entonces el nodo publica un nuevo payload explícito con el estado online para normalizar los indicadores en el dashboard web.

---

### HU-14 · Descarte de conexiones no seguras (Texto Plano)

- **Como** administrador del sistema
- **Quiero** que el broker MQTT gestionado bloquee conexiones por el puerto 1883 en texto plano
- **Para** forzar comunicaciones seguras y evitar sniffers de red
- **Prioridad:** Media · **Sprint:** 3 · **Story Points:** 2 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (intento por puerto inseguro):** Dado que un cliente IoT mal configurado o no autorizado intenta conectarse al puerto tradicional 1883, cuando llega la solicitud al servidor, entonces el firewall del broker la rechaza inmediatamente cerrando el socket sin completar ningún intercambio de datos.
2. **Escenario 2 (puerto cifrado disponible):** Dado que el puerto 8883 (MQTTS) está habilitado, cuando un cliente legítimo (ESP32) inicia el handshake TLS por ese puerto, entonces la conexión procede normalmente a la validación de credenciales.
3. **Escenario 3 (intento reiterado o ataque):** Dado que una misma dirección IP intenta repetidamente conectarse por el puerto inseguro, cuando se supera un número de intentos anómalos en una ventana de tiempo corta, entonces el evento queda registrado en los logs de observabilidad de EMQX Cloud para su posible análisis de intrusión.

---

# Épica EP03 · Procesamiento e Inteligencia Artificial (Backend/FastAPI)

---

### HU-15 · Validación de esquema entrante (Pydantic)

- **Como** administrador del sistema
- **Quiero** que el backend valide los payloads MQTT entrantes con Pydantic v2
- **Para** rechazar estructuras corruptas o maliciosas antes de persistir los datos
- **Prioridad:** Alta · **Sprint:** 3 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (payload válido):** Dado que llega un mensaje MQTT con todos los campos obligatorios y tipos de datos correctos desde la capa Edge, cuando se valida contra el esquema de Pydantic, entonces el modelo de datos se acepta y el flujo asíncrono continúa hacia la capa de Dominio.
2. **Escenario 2 (campo faltante):** Dado que el payload carece de una variable crítica (ej. temperatura_interna), cuando se ejecuta la validación estructural, entonces se rechaza el mensaje, se genera un log de error detallando el campo faltante y no se persiste en la base de datos.
3. **Escenario 3 (tipo de dato incorrecto o JSON malformado):** Dado que un campo tiene un tipo no esperado (ej. string en lugar de float) o el JSON está malformado, cuando se intenta deserializar, entonces Pydantic captura la excepción de validación, descarta el mensaje y registra la advertencia, garantizando que el event loop de aiomqtt no se bloquee ni se detenga el consumo de telemetría de otros dispositivos.

---

### HU-16 · Carga en memoria del Modelo IA (Lifespan)

- **Como** administrador del sistema
- **Quiero** que el backend cargue el modelo Random Forest al iniciar el servicio
- **Para** procesar inferencias de riesgo sin latencia de lectura de disco por mensaje
- **Prioridad:** Alta · **Sprint:** 4 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (carga exitosa en RAM):** Dado que el servidor Uvicorn inicia su ejecución, cuando se dispara el contexto asíncrono de lifespan, entonces el archivo .pkl del modelo se deserializa mediante scikit-learn y queda inyectado globalmente en la memoria de la aplicación.
2. **Escenario 2 (archivo de modelo no encontrado o corrupto):** Dado que el archivo del modelo predictivo no existe en la ruta especificada o está corrupto, cuando se intenta cargar en el arranque, entonces el servicio registra un error crítico y lanza un RuntimeError interrumpiendo el inicio de FastAPI, evitando así operar en un estado inconsistente sin IA.
3. **Escenario 3 (recarga en caliente - hot reload):** Dado que se publica una nueva versión entrenada del modelo Random Forest, cuando se solicita una recarga a través de un endpoint administrativo seguro, entonces la instancia del modelo en memoria se actualiza atómicamente sin necesidad de reiniciar el proceso Uvicorn completo, garantizando alta disponibilidad.

---

### HU-17 · Extracción de Características (Feature Engineering)

- **Como** farmacéutico responsable
- **Quiero** que el sistema calcule métricas derivadas como gradiente térmico y derivadas de tiempo
- **Para** alimentar correctamente el modelo Random Forest que clasifica el riesgo
- **Prioridad:** Alta · **Sprint:** 4 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (cálculo completo):** Dado que se recibe un modelo de datos validado por Pydantic, cuando se invoca el servicio de extracción de dominio, entonces se calcula la matriz completa de variables operativas (ej. delta térmico) necesarias para alimentar al clasificador.
2. **Escenario 2 (historial insuficiente):** Dado que el nodo acaba de conectarse y no hay suficiente historial térmico reciente en caché para calcular una variable derivada (ej. tendencia térmica móvil), cuando se intenta construir la matriz, entonces el módulo inyecta un valor neutral por defecto documentado, permitiendo que el proceso de inferencia continúe sin fallar.
3. **Escenario 3 (valores atípicos / outliers):** Dado que una variable de entrada (ej. 80 °C en el refrigerador) está fuera de los rangos físicamente plausibles para una operación normal, cuando se construye la matriz de características, entonces el registro se marca preventivamente como anomalía de hardware y se excluye de la inferencia estándar para no sesgar las predicciones del modelo.

---

### HU-18 · Inferencia de Riesgo Térmico (Random Forest)

- **Como** farmacéutico responsable
- **Quiero** que el sistema evalúe la matriz de características operativas
- **Para** clasificar el estado térmico en Normal, Riesgo Preventivo o Excursión Crítica
- **Prioridad:** Alta · **Sprint:** 4 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (clasificación exitosa en RAM):** Dado que se ingresan las features correctamente desde el módulo de dominio, cuando el modelo Random Forest (cargado previamente vía joblib) procesa la matriz, entonces retorna una de las 3 clases junto con la probabilidad (confianza) de la predicción en milisegundos.
2. **Escenario 2 (entrada incompleta):** Dado que la matriz contiene valores faltantes no imputables (ej. sensor desconectado), cuando se intenta inferir, entonces el sistema evita generar una clasificación no confiable, registra el estado como "no clasificable" y delega la regla a la validación estricta de temperatura.
3. **Escenario 3 (baja confianza en la predicción):** Dado que la probabilidad de la clase predicha está por debajo del umbral mínimo configurado, cuando se evalúa el resultado, entonces la lectura se marca adicionalmente con un flag de revisión manual, sin impedir el registro de la clase de mayor probabilidad.

---

### HU-19 · Emisión de Eventos SSE (Flujo Unidireccional)

- **Como** técnico de farmacia
- **Quiero** ver en tiempo real la lectura procesada y la clasificación de riesgo en el dashboard
- **Para** monitorear incidentes sin recargar la interfaz
- **Prioridad:** Media · **Sprint:** 4 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (cliente conectado):** Dado que se persiste la lectura y su clasificación, cuando existen clientes web activos (técnicos/farmacéuticos), entonces el servidor despacha el evento JSON vía SSE a todos los suscriptores al instante.
2. **Escenario 2 (sin clientes conectados):** Dado que no hay clientes suscritos al canal SSE al momento de procesar la telemetría, cuando se intenta emitir el evento, entonces el despacho se omite para ahorrar recursos, pero el dato no se pierde ya que queda disponible en PostgreSQL para consultas históricas.
3. **Escenario 3 (desconexión abrupta del cliente):** Dado que un cliente cierra el navegador o pierde red abruptamente, cuando el servidor FastAPI detecta la ruptura del canal SSE, entonces libera los recursos de memoria y cierra el generador asíncrono sin afectar la transmisión a los demás clientes conectados.

---

### HU-20 · Generación de Alerta Preventiva

- **Como** farmacéutico responsable
- **Quiero** que el sistema genere una alerta cuando el modelo detecte Riesgo Preventivo
- **Para** actuar antes de que se rompa la cadena de frío
- **Prioridad:** Media · **Sprint:** 4 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (alerta generada):** Dado que la IA retorna la clase riesgo_preventivo, cuando el motor de reglas procesa la salida, entonces se inserta un nuevo registro en la tabla thermal_alerts de PostgreSQL con nivel, timestamp y dispositivo, y se emite un aviso urgente por el canal SSE.
2. **Escenario 2 (riesgo preventivo sostenido):** Dado que ya existe una alerta preventiva activa y sin atender para el mismo dispositivo, cuando llega otra clasificación de riesgo_preventivo, entonces se actualiza la fecha de la alerta existente en lugar de duplicar registros, evitando la saturación (spam) en el dashboard.
3. **Escenario 3 (resolución automática):** Dado que el dispositivo vuelve a clasificarse como normal de forma sostenida tras una alerta preventiva, cuando se confirma esta estabilización, entonces la alerta activa se marca como "resuelta automáticamente" en la base de datos.

---

### HU-21 · Generación de Alerta Crítica (Normativa DIGEMID)

- **Como** farmacéutico responsable
- **Quiero** que el sistema registre una excursión térmica crítica si la temperatura sale del rango de 2 °C a 8 °C
- **Para** ejecutar acciones correctivas auditables conforme a las Buenas Prácticas de Almacenamiento
- **Prioridad:** Alta · **Sprint:** 4 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (excursión detectada por hardware):** Dado que la lectura térmica es < 2 °C o > 8 °C, cuando el motor de reglas la evalúa, entonces se marca y persiste la alerta como excursion_critica de inmediato, independientemente del resultado predictivo del modelo de IA, despachando la alarma visual al frontend.
2. **Escenario 2 (retorno al rango seguro):** Dado que existe una excursión crítica activa y la siguiente lectura vuelve al rango de 2-8 °C, cuando se procesa, entonces se registra el evento de retorno al rango seguro, pero no se elimina ni se cierra la excursión original hasta que reciba una justificación manual del farmacéutico.
3. **Escenario 3 (excursión sin acción correctiva):** Dado que una excursión crítica permanece sin una acción correctiva asociada (ej. traslado de productos) tras un tiempo prudencial, cuando se evalúa su estado, entonces queda anclada como "Pendiente de Acción" forzando al dashboard a destacarla en rojo hasta su resolución.

---

### HU-22 · Consolidación de Historial en Capa de Infraestructura

- **Como** farmacéutico responsable
- **Quiero** que la telemetría se almacene en PostgreSQL usando JSONB
- **Para** conservar un histórico térmico flexible junto con su clasificación asociada
- **Prioridad:** Alta · **Sprint:** 3 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (persistencia asíncrona exitosa):** Dado que el payload es válido y el riesgo térmico ya fue clasificado por la IA, cuando se mapea el objeto al ORM, entonces se persiste de forma asíncrona en la tabla thermal_readings, incluyendo el campo JSONB con la telemetría completa de los sensores.
2. **Escenario 2 (fallo temporal de conexión a BD en Railway):** Dado que la instancia de PostgreSQL en la nube (Railway) no está disponible temporalmente al momento de persistir, cuando ocurre el intento de guardado, entonces el repositorio reintenta la conexión un número limitado de veces (política de reintentos) antes de registrar el fallo en el log, sin perder el mensaje validado en memoria.
3. **Escenario 3 (concurrencia de escrituras IoT):** Dado que llegan múltiples lecturas casi simultáneas desde distintos dispositivos (refrigeradores/farmacias), cuando el backend invoca la capa de persistencia, entonces cada registro se guarda en la base de datos de forma asíncrona e independiente, sin bloquear el event loop del servidor ni retrasar la ingesta de los demás dispositivos.

---

### HU-23 · Notificaciones Asíncronas de Excursión Crítica

- **Como** farmacéutico responsable
- **Quiero** recibir una notificación automática por correo electrónico o Telegram cuando se genere una excursión crítica
- **Para** enterarme del incidente de forma inmediata aunque no tenga el dashboard web abierto
- **Prioridad:** Alta · **Sprint:** 7 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (excursión fuera de horario):** Dado que se genera una excursión crítica térmica, cuando el backend (FastAPI) registra la alerta, entonces despacha una tarea en segundo plano (Background Task) que envía automáticamente una notificación vía SMTP o API de Telegram al responsable designado, sin bloquear el hilo principal.
2. **Escenario 2 (fallo en el servicio de terceros):** Dado que el servicio de envío externo (SMTP/Telegram) no responde o presenta latencia, cuando se intenta notificar, entonces el sistema reintenta un número limitado de veces y registra el fallo en el log, asegurando que la persistencia de la alerta en PostgreSQL no se vea afectada.
3. **Escenario 3 (Throttling / prevención de spam):** Dado que se generan múltiples lecturas de excursión crítica seguidas para el mismo dispositivo, cuando el sistema evalúa el envío de mensajes, entonces aplica una política de limitación de frecuencia (throttling) para enviar un solo resumen, evitando saturar al responsable con notificaciones duplicadas del mismo incidente.

---

# Épica EP04 · Trazabilidad Digital e Integridad (PostgreSQL)

---

### HU-24 · Cálculo de Hash Criptográfico (SHA-256)

- **Como** responsable de auditoría
- **Quiero** que el sistema calcule un hash SHA-256 por cada evento térmico u operativo
- **Para** asegurar su inmutabilidad algorítmica sin depender de infraestructuras blockchain de terceros
- **Prioridad:** Alta · **Sprint:** 5 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (cálculo exitoso con insumos exactos):** Dado que ocurre un evento relevante (lectura, alerta o acción correctiva), cuando se genera su registro en el backend, entonces la librería hashlib de Python calcula una firma SHA-256 de 256 bits concatenando estrictamente los campos (previous_hash + payload + timestamp).
2. **Escenario 2 (campo vacío o nulo):** Dado que algún campo del evento llega vacío o nulo de forma inesperada, cuando se calcula el hash criptográfico, entonces el campo se normaliza a un valor representable estándar (ej. string vacío o "null") antes del cálculo, evitando que un campo ausente rompa la reproducibilidad de la firma.
3. **Escenario 3 (determinismo matemático):** Dado que se recalcula el hash de un evento ya almacenado ingresando exactamente los mismos datos originales, cuando se compara con la firma original, entonces ambos valores alfanuméricos coinciden con exactitud matemática, demostrando el determinismo del algoritmo.

---

### HU-25 · Encadenamiento con Previous Hash (Ledger Centralizado)

- **Como** responsable de auditoría
- **Quiero** que cada nuevo registro incluya el hash del evento anterior en traceability_records
- **Para** formar una cadena criptográfica inviolable
- **Prioridad:** Alta · **Sprint:** 5 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (encadenamiento normal secuencial):** Dado que se guarda un nuevo evento validado, cuando se calcula su firma hash, entonces el algoritmo concatena el valor de la columna previous_hash del registro N-1 con los datos actuales, garantizando la dependencia temporal.
2. **Escenario 2 (primer evento de la cadena / Génesis):** Dado que el sistema se inicializa y no existe un registro anterior en la tabla, cuando se calcula su hash, entonces se utiliza un valor semilla inicial predefinido de 256 bits (bloque génesis) en lugar de un previous_hash real.
3. **Escenario 3 (resolución de inserciones concurrentes):** Dado que dos o más eventos (de distintos refrigeradores) intentan registrarse casi simultáneamente en PostgreSQL, cuando se determina el orden de la cadena por transacciones atómicas, entonces el sistema garantiza que cada nuevo hash referencie de forma única y consistente al hash inmediatamente anterior consolidado, evitando bifurcaciones de la cadena.

---

### HU-26 · Verificación O(n) de Cadena de Integridad

- **Como** Responsable de Auditoría o Fiscalización interna
- **Quiero** solicitar una verificación algorítmica en bloque de toda la tabla de trazabilidad
- **Para** detectar manipulaciones en los registros y presentar evidencias confiables ante inspecciones de DIGEMID
- **Prioridad:** Alta · **Sprint:** 5 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (cadena íntegra confirmada):** Dado que el auditor solicita la verificación en el dashboard, cuando el backend recalcula los hashes secuencialmente (complejidad O(n)), entonces el sistema confirma que cada firma coincide con la almacenada y retorna un estado global "Íntegro / Válido".
2. **Escenario 2 (detección de registro alterado):** Dado que un registro malicioso fue modificado a nivel de base de datos tras su creación, cuando se recalcula su hash, entonces el valor resultante no coincide con el almacenado y la cadena se rompe, señalando el ID del bloque específico como "Corrupto", sin detener la evaluación de los registros restantes.
3. **Escenario 3 (interrupción del proceso pesado):** Dado que la verificación O(n) se interrumpe (por timeout o cierre de conexión) antes de recorrer toda la extensa tabla, cuando el auditor reintenta la acción, entonces el proceso puede reanudarse o reiniciarse de forma controlada sin bloquear otras operaciones concurrentes de lectura en la base de datos.

---

### HU-27 · Persistencia de Acción Correctiva

- **Como** Químico Farmacéutico (Regente)
- **Quiero** registrar la acción correctiva aplicada tras una alerta
- **Para** cumplir con el soporte documental exigido por la normativa de Buenas Prácticas de Almacenamiento de DIGEMID
- **Prioridad:** Media · **Sprint:** 5 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (registro exitoso):** Dado que hay una alerta activa en el dashboard, cuando el usuario ingresa la justificación y la acción tomada, entonces el sistema extrae su identidad desde el token JWT, asocia el evento al ID de la alerta y la marca como atendida.
2. **Escenario 2 (alerta ya resuelta):** Dado que el usuario intenta registrar una acción sobre una alerta que acaba de ser cerrada por otro compañero, cuando envía la solicitud, entonces el sistema la rechaza mediante concurrencia e informa visualmente que la alerta ya no está vigente.
3. **Escenario 3 (campos incompletos):** Dado que no se completa la descripción mínima requerida de la acción, cuando intenta enviar el formulario, entonces el sistema bloquea el envío en el frontend y solicita completar la información obligatoria.

---

### HU-28 · Auditoría Automática de Acciones Correctivas

- **Como** responsable de auditoría
- **Quiero** que el sistema calcule hash SHA-256 también sobre las acciones manuales de los usuarios
- **Para** impedir cambios posteriores en las justificaciones registradas
- **Prioridad:** Media · **Sprint:** 5 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (acción trazada criptográficamente):** Dado que se envía una acción correctiva desde el dashboard, cuando se persiste en la tabla traceability_records, entonces se genera un evento encadenado tipo ACCION_CORRECTIVA con su propio hash SHA-256, utilizando el previous_hash del último movimiento del sistema.
2. **Escenario 2 (intento de edición posterior / Append-Only):** Dado que un usuario intenta modificar una justificación ya registrada en la base de datos, cuando se procesa la solicitud, entonces la base de datos rechaza la alteración del registro original; exigiendo que se cree un nuevo evento que referencie la corrección.
3. **Escenario 3 (verificación cruzada):** Dado que el auditor verifica la cadena completa, cuando se incluyen eventos de tipo ACCION_CORRECTIVA, entonces el sistema los valida secuencialmente con el mismo mecanismo algorítmico que las lecturas y alertas, garantizando la integridad de todo el ciclo de vida del dato.

---

### HU-29 · Backup Automático Diario (Disaster Recovery)

- **Como** administrador de infraestructura
- **Quiero** que la base de datos PostgreSQL se respalde de forma automatizada
- **Para** asegurar la continuidad operativa y proteger la evidencia histórica ante caídas o desastres
- **Prioridad:** Baja · **Sprint:** 7 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (backup exitoso en la nube):** Dado que la base de datos PostgreSQL se aloja en el proveedor cloud Railway, cuando transcurren 24 horas del ciclo operativo, entonces el servicio genera un snapshot o volcado automático de los datos sin afectar el rendimiento de escritura IoT.
2. **Escenario 2 (fallo del backup):** Dado que el proceso automático de respaldo falla (ej. por límites de almacenamiento), cuando el servidor detecta el error de la tarea programada, entonces se registra una alerta operativa crítica en el log del backend para intervención manual del administrador.
3. **Escenario 3 (retención y purga de backups):** Dado que se acumulan múltiples snapshots a lo largo de las semanas, cuando se supera la política de retención definida (ej. últimos 7 días), entonces los respaldos más antiguos se eliminan automáticamente para liberar espacio en Railway, sin afectar a los archivos recientes.

---

### HU-30 · Alerta de Vencimiento de Calibración de Sensores

- **Como** Químico Farmacéutico
- **Quiero** registrar la fecha de última calibración y el certificado del sensor de temperatura
- **Para** recibir alertas antes del vencimiento y mantener el cumplimiento regulatorio
- **Prioridad:** Media · **Sprint:** 5 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (registro trazable y auditable):** Dado que el Farmacéutico ingresa la fecha de última calibración y el número de certificado de un nodo, cuando guarda el registro, entonces el sistema lo almacena asociado al device_id y genera un evento en la tabla de trazabilidad con su respectiva firma SHA-256 encadenada, para evitar alteraciones futuras.
2. **Escenario 2 (vencimiento próximo):** Dado que faltan 30 días o menos para el vencimiento anual del certificado, cuando el backend evalúa diariamente las fechas registradas, entonces genera un "Riesgo Operativo" visible en el dashboard, con un color e iconografía distintivos para no confundirlo con una alerta de excursión térmica.
3. **Escenario 3 (certificado vencido):** Dado que la fecha de vencimiento superó el límite sin que se haya registrado una recalibración, cuando se evalúa el estado del sensor, entonces el riesgo operativo escala a nivel crítico y las lecturas de ese dispositivo se marcan visualmente con una advertencia de "calibración vencida" hasta que se actualice la documentación.

---

### HU-43 · Gestión del Ciclo de Vida de Hardware (Alta/Baja de Dispositivos)

- **Como** Administrador del Sistema / Técnico de Mantenimiento
- **Quiero** registrar la baja de un dispositivo IoT (ESP32/sensores) cuando se malogre o sea reemplazado, sin corromper la cadena de trazabilidad histórica
- **Para** mantener la consistencia administrativa, auditar el reemplazo de hardware y cumplir con DIGEMID en caso de cambios en la configuración del monitoreo
- **Prioridad:** Alta · **Sprint:** 8 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (Baja ordenada de dispositivo funcional):** Dado que un técnico decide retirar un ESP32 activo para mantenimiento preventivo o reemplazo, cuando accede al módulo de "Gestión de Dispositivos" en el dashboard administrativo, entonces encuentra la opción "Dar de baja" con un formulario que solicita: motivo de baja (falla de hardware, mantenimiento, reemplazo, retiro de servicio), descripción detallada del incidente, nuevo device_id del reemplazo (si aplica). Y al confirmar, el sistema registra un evento especial en la tabla traceability_records con: tipo_evento: "BAJA_HARDWARE", previous_hash del último evento del dispositivo original, payload: {device_id_anterior, motivo_baja, device_id_reemplazo_si_existe, timestamp_baja}, su respectivo hash SHA-256 encadenado. Y el device_id anterior se marca como "inactivo" en la tabla devices sin eliminar registros históricos, y se genera una notificación para el Administrador informando el evento.
2. **Escenario 2 (Reemplazo de dispositivo con vinculación de historial):** Dado que un técnico reporta que el sensor DS18B20 del ESP32 "pharmacy_fridge_01" falló, cuando registra la baja y especifica que será reemplazado por "pharmacy_fridge_01_v2", entonces el sistema crea una "cadena de custodia" virtual donde: el nuevo device_id hereda acceso a los datos históricos del anterior (lectura únicamente), se registra una vinculación bidireccional en la tabla devices para auditoría, los futuros registros del nuevo dispositivo comienzan su propia cadena hash independiente, y en reportes y dashboards aparece una nota: "Dispositivo reemplazado de [device_id_anterior] el [fecha]".
3. **Escenario 3 (Baja de dispositivo sin reemplazo - Fin de servicio):** Dado que un refrigerador farmacéutico es retirado de operación permanentemente, cuando se registra la baja con motivo "Fin de servicio / Decommissioning", entonces el dispositivo queda marcado como inactivo pero su histórico completo permanece íntegro en la base de datos. Y si alguien intenta consultarlo desde el dashboard, ve un estado "Dispositivo retirado" con la fecha y razón, y su cadena hash se marca como "cerrada" pero verificable para auditorías futuras.
4. **Escenario 4 (Validación de permisos e integridad):** Dado que un Técnico (rol = TECNICO) intenta dar de baja un dispositivo, cuando hace clic en "Dar de baja", entonces el sistema verifica que solo ADMINISTRADOR o TECNICO con permiso especial pueden ejecutar esta acción. Y si falta permiso, rechaza la operación con HTTP 403 y registra el intento fallido en audit_logs.

---

### HU-47 · Manejo de Corrupción y Recuperación de Cadena Hash

- **Como** Responsable de Base de Datos / Administrador del Sistema
- **Quiero** que cuando se detecte corrupción en la cadena hash (un registro alterado), el sistema me notifique, identifique el punto exacto de ruptura y me guíe en la recuperación sin perder integridad futura
- **Para** mantener la confiabilidad del sistema ante intentos de manipulación y poder restaurar la continuidad de la auditoría tras incidentes de seguridad
- **Prioridad:** Alta · **Sprint:** 9 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (Detección de corrupción durante verificación):** Dado que se ejecuta la verificación O(n) de integridad de la cadena hash (HU-26), cuando el sistema recalcula los hashes y encuentra un registro alterado, entonces la verificación se detiene y retorna:
   - `integra: false`
   - `total_registros_verificados: 8542`
   - `primer_registro_inconsistente`: id "550e8400-e29b-41d4-a716-446655440050", tipo_evento "LECTURA_TERMICA", timestamp "2026-07-15T14:32:00Z", hash_esperado "a3f9e2c1b8d9e7f4c3a2b1d9e7f4c3a2b1d9e7f4c3a2b1d9e7f4c3a2b1d9e7", hash_almacenado "x9f9e2c1b8d9e7f4c3a2b1d9e7f4c3a2b1d9e7f4c3a2b1d9e7f4c3a2b1d9e8", mensaje "Alteración detectada: el payload fue modificado post-registro"
   - `registros_posteriores_afectados: 542`

   Y se registra un evento de emergencia tipo_evento: "CORRUPCION_CADENA_DETECTADA". El registro de emergencia se inserta utilizando como previous_hash el hash del último bloque íntegro conocido (no el hash del bloque corrupto), para dejar constancia inmutable del punto exacto del hallazgo.
2. **Escenario 2 (Notificación y escalada):** Dado que se detecta corrupción, cuando el backend genera la notificación, entonces ejecuta:
   - Envía email crítico al Administrador: "⚠️ ALERTA DE SEGURIDAD: Corrupción detectada en cadena hash".
   - Registra en los logs de auditoría de urgencia.
   - Publica un evento SSE a todos los dashboards administrativos.
   - Guarda un snapshot forense de la situación en la base de datos: inserta un registro en audit_logs o en forensic_snapshots (JSONB) con la metadata exacta del punto de ruptura (id_registro, hash_esperado, hash_almacenado, timestamp, contexto), garantizando persistencia frente a reinicios de contenedor en Railway.
   - NO pausa la ingesta de telemetría IoT. En su lugar, activa un flag global cadena_comprometida = true. Todos los nuevos registros continúan insertándose en traceability_records, pero se marcan con is_after_corruption = true y el dashboard muestra una advertencia "Historial posterior al punto de ruptura: revisar" hasta que se restaure la integridad.
3. **Escenario 3 (Análisis del punto de ruptura):** Dado que se detectó corrupción en el registro ID 550e8400..., cuando el Administrador accede al módulo de "Análisis de Integridad", entonces ve: gráfico de la cadena mostrando dónde se rompió; información detallada del registro corrupto (quién lo creó — usuario_id, cuándo se creó — timestamp, qué datos contiene — payload visible, el hash anterior que esperaba y el hash que tiene); lista de 542 registros posteriores afectados (ya no son verificables); posibles escenarios de recuperación (ver Escenario 4).
4. **Escenario 4 (Opciones de Recuperación):**
   - **Opción 1 - Restauración desde Backup:** Dado que la corrupción se detectó, cuando hay un backup disponible (HU-29), entonces el Administrador puede seleccionar "Restaurar desde Backup del 2026-07-14". El sistema restaura los registros de traceability_records hasta el último bloque íntegro disponible (no borra audit_logs ni los registros forenses); registra qué registros se pierden (los creados después del backup hasta la corrupción) y preserva la tabla audit_logs para mantener la evidencia del incidente; inserta un "Bloque de Restauración" en traceability_records que actúe como puente criptográfico entre la copia restaurada y el estado actual, y solicita a los dispositivos ESP32 que sincronicen lecturas retenidas en LittleFS para re-ingestarlas en cuarentena; notifica al usuario: "Sistema restaurado hasta último bloque íntegro. Se perdieron XXX registros en el periodo afectado. Audit_logs preservados para investigación."
   - **Opción 2 - Aislamiento de Corrupción (Quarantine):** Dado que se prefiere no perder datos recientes, cuando el Administrador elige "Aislar corrupción", entonces el sistema: marca el registro corrupto con flag is_corrupted = true; registra un evento especial tipo_evento: "REGISTRO_AISLADO_CORRUPCION"; crea un nuevo "bloque génesis" (reset de cadena) que servirá como inicio de una nueva cadena para los registros posteriores; los futuros registros comienzan una nueva cadena hash independiente desde ese punto; el historial anterior hasta el punto de ruptura exacto se mantiene marcado como "Íntegro y Verificable", sólo el bloque corrupto y los registros posteriores que dependían de él se marcan como "Cadena Rota / Aislada".
   - **Opción 3 - Análisis Forense (No-action - Preserve Evidence):** Dado que se sospecha ataque malicioso, cuando el Administrador elige "Modo Forense", entonces el sistema: congela las operaciones administrativas y de configuración en el frontend (modo lectura para usuarios administrativos) pero mantiene habilitada la ingesta de telemetría IoT en una tabla o partición de cuarentena para evitar pérdida de datos del edge; inserta un snapshot forense en BD (JSONB) y exporta, además, un dump cifrado si es requerido por el equipo forense; los dashboards muestran estado "Mantenimiento Forense" para usuarios humanos, la ingestión y sincronización de dispositivos continúa en cuarentena; se notifica a las autoridades de la farmacia y, si corresponde, a DIGEMID/DIRIS.
5. **Escenario 5 (Prevención de re-ocurrencia):** Dado que se recuperó de la corrupción, cuando el Administrador revisa el incidente, entonces el sistema propone medidas: ¿El registro corrupto fue modificado directamente en la BD? Recomendación: revisar permisos de BD, implementar triggers de auditoría a nivel DB. ¿Fue alterado por la API? Recomendación: revisar logs de la aplicación, verificar si endpoint de actualización tiene lógica append-only. Implementar monitoreo continuo: ejecutar verificación O(n) cada 24 horas (en vez de bajo demanda).
6. **Escenario 6 (Continuidad operativa post-recuperación):** Dado que se resolvió la corrupción, cuando el Administrador autoriza la reanudación, entonces el sistema: reactiva la inserción de nuevos registros; ejecuta una verificación completa post-recuperación para confirmar integridad; notifica a todos los usuarios: "Sistema restaurado. Operación normal reanudada."; genera un reporte ejecutivo para DIGEMID/auditoría interna con: fecha/hora de detección, punto de ruptura identificado, acción de recuperación tomada, registros afectados, medidas preventivas implementadas.

---

### HU-48 · Cuarentena y Bloqueo Virtual de Lotes de Medicamentos

- **Como** Químico Farmacéutico Responsable
- **Quiero** asociar un lote específico de medicamentos a un dispositivo de refrigeración y activar el bloqueo de dispensación virtual ante una excursión térmica crítica sostenida
- **Para** evitar que se despachen medicamentos termosensibles que hayan perdido su estabilidad biológica, cumpliendo con el Componente 4 de las BPA
- **Prioridad:** Alta · **Sprint:** 10 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (Inmovilización por Excursión Crítica):** Dado que el lote "LOTE-INSULINA-101" está asociado al dispositivo "ESP32-REF-01" en estado activo, cuando el backend procesa lecturas térmicas en estado excursion_critica por un periodo ininterrumpido superior a 15 minutos, entonces el estado del lote cambia automáticamente a "Inmovilizado por Alerta Térmica", se bloquea su opción de dispensación en la interfaz y se registra el evento LOTE_CUARENTENA en la cadena de hash SHA-256 de trazabilidad.
2. **Escenario 2 (Dispensación Bloqueada):** Dado que un lote de medicamento se encuentra en estado "Inmovilizado por Alerta Térmica", cuando un técnico de farmacia intenta registrar una dispensación de dicho lote, entonces el backend rechaza la transacción con un código HTTP 403 (Operación Prohibida - Lote en Cuarentena) y el frontend muestra un aviso en rojo indicando que el lote está inmovilizado por pérdida de cadena de frío.

---

### HU-49 · Registro y Control de Retiro de Mercado de Lotes Comprometidos

- **Como** Químico Farmacéutico (Regente)
- **Quiero** registrar y gestionar órdenes de Retiro de Mercado (simuladas o reales) asociadas a lotes que presentaron fallas de conservación
- **Para** documentar la eficacia del procedimiento de retiro ante auditorías de DIGEMID bajo el Componente 7 de las BPA
- **Prioridad:** Alta · **Sprint:** 10 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (Registro de Retiro de Mercado):** Dado que el Químico Farmacéutico inicia el flujo de retiro de mercado de un lote comprometido, cuando ingresa el número de documento de alerta (ej: Alerta DIGEMID o Acta Interna de Excursión), el motivo del retiro y la cantidad de unidades inmovilizadas, entonces el sistema cambia el estado del lote a "Retirado del Mercado", inhabilita permanentemente su uso en el sistema, calcula el hash SHA-256 de la operación y lo ancla a la tabla traceability_records.

---

# Épica EP05 · Monitoreo Web y Visualización (Frontend React)

---

### HU-31 · Visualización de Tendencia Térmica

- **Como** técnico de farmacia
- **Quiero** visualizar una gráfica de temperatura con umbrales fijos de 2 °C a 8 °C
- **Para** evaluar rápidamente la tendencia térmica del refrigerador
- **Prioridad:** Alta · **Sprint:** 6 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (carga con datos):** Dado que abro el dashboard de un refrigerador y existen lecturas históricas en la base de datos, cuando el componente de React 19 carga la instancia de ECharts, entonces se renderiza la línea histórica superpuesta sobre zonas de colores (rojo/verde) que delimitan visualmente la normativa de 2-8 °C.
2. **Escenario 2 (sin datos históricos):** Dado que un dispositivo IoT recién registrado no ha transmitido lecturas aún, cuando se abre su vista de gráfica, entonces el sistema muestra un "estado vacío" (empty state) informativo indicando que está a la espera de datos, en lugar de un lienzo en blanco confuso.
3. **Escenario 3 (gran volumen de puntos):** Dado que el rango de fechas seleccionado contiene miles de lecturas continuas, cuando ECharts renderiza el gráfico, entonces se aplica muestreo o agregación visual (data downsampling) de forma automática para mantener la fluidez de la interfaz sin perder la forma real de la tendencia térmica.

---

### HU-32 · Actualización en Tiempo Real del Dashboard

- **Como** técnico de farmacia
- **Quiero** que el dashboard reciba actualizaciones del backend mediante Server-Sent Events (SSE)
- **Para** ver el último punto de la gráfica y los KPI en tiempo real
- **Prioridad:** Alta · **Sprint:** 6 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (actualización asíncrona en tiempo real):** Dado que la conexión SSE hacia la API FastAPI está activa, cuando llega un nuevo evento JSON con telemetría fresca, entonces el gestor de estado global del frontend actualiza automáticamente los componentes dependientes sin necesidad de recargar la página.
2. **Escenario 2 (pérdida de conexión SSE):** Dado que la red local de la farmacia o el enlace con el servidor falla, cuando el cliente React detecta la ruptura del flujo continuo, entonces muestra un indicador visual en amarillo de "reconectando..." e intenta restablecer el canal SSE automáticamente mediante un backoff de reintentos.
3. **Escenario 3 (múltiples pestañas):** Dado que el usuario tiene el dashboard abierto en más de una pestaña o dispositivo, cuando el servidor despacha un evento SSE, entonces todas las instancias del cliente reflejan la actualización instantáneamente de forma independiente y concurrente.

---

### HU-33 · Tarjetas de KPI (Estado Actual y Tolerancia a Fallo)

- **Como** técnico de farmacia
- **Quiero** visualizar el último valor de temperatura (DS18B20) y humedad ambiental (SHT31) en tarjetas KPI
- **Para** obtener una lectura rápida del estado del equipo a un solo vistazo
- **Prioridad:** Media · **Sprint:** 6 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (datos disponibles y vigentes):** Dado que el dashboard web se renderiza y la caché cuenta con datos válidos, cuando se cargan los componentes de las tarjetas KPI, entonces se muestran los valores numéricos actuales exactos en °C y % HR.
2. **Escenario 2 (última lectura desactualizada por red):** Dado que el último timestamp recibido supera el umbral máximo esperado de inactividad (ej. más de 5 minutos sin datos del edge), cuando se muestran las tarjetas, entonces se oscurecen ligeramente o incluyen un ícono de advertencia indicando la antigüedad del dato.
3. **Escenario 3 (lectura en error de sensor):** Dado que el flujo de datos indica que un sensor en particular está fallando (valor null), cuando React intenta renderizar su KPI correspondiente, entonces muestra el texto "Sin lectura disponible" o "Falla de sensor" en lugar de mostrar información potencialmente engañosa que confunda al personal.

---

### HU-34 · Semáforo de Riesgo Térmico IA (Random Forest)

- **Como** farmacéutico
- **Quiero** ver un semáforo visual (Verde, Amarillo, Rojo) en la tarjeta del equipo según la inferencia del Random Forest
- **Para** identificar de inmediato el nivel de atención clínica o técnica requerido
- **Prioridad:** Alta · **Sprint:** 6 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (estado normal y riesgo preventivo):** Dado que el dashboard recibe una actualización vía SSE, cuando el campo de clase IA indica normal, entonces el semáforo se muestra en Verde; y si cambia a riesgo_preventivo, el componente muta dinámicamente a color Amarillo indicando precaución.
2. **Escenario 2 (excursión crítica):** Dado que la clasificación algorítmica del backend detecta la clase excursion_critica, cuando la interfaz (UI) se actualiza, entonces el semáforo cambia a Rojo con alerta visual y permanece en este estado hasta que el sistema reciba el estado normal de forma sostenida.
3. **Escenario 3 (clasificación no disponible por fallo):** Dado que la última lectura fue rechazada o quedó catalogada como "no clasificable" (ej. datos insuficientes), cuando React intenta actualizar el semáforo, entonces se muestra un estado neutro (ej. color gris) distinto de los tres niveles operativos, evitando proporcionar una falsa sensación de seguridad.

---

### HU-35 · Notificación UI de Puerta Abierta (MC-38)

- **Como** personal técnico de farmacia
- **Quiero** visualizar una alerta intermitente en la interfaz si el refrigerador permanece abierto
- **Para** cerrarlo rápidamente y evitar fuga térmica y condensación
- **Prioridad:** Media · **Sprint:** 6 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (apertura en tiempo real):** Dado que el sensor magnético (MC-38) registra apertura y envía el estado true empaquetado en el flujo SSE, cuando la aplicación web procesa el evento, entonces renderiza un ícono de puerta abierta en rojo con una animación intermitente (pulsing) en la tarjeta del dispositivo.
2. **Escenario 2 (cierre confirmado):** Dado que la puerta física se cierra y el estado del sensor vuelve a false, cuando el frontend recibe la actualización asíncrona, entonces la notificación de alerta y el ícono desaparecen inmediatamente sin necesidad de recargar la página.
3. **Escenario 3 (apertura prolongada crítica):** Dado que la puerta permanece abierta más allá del tiempo máximo configurado de seguridad (ej. 2 minutos), cuando el estado global detecta esta prolongación, entonces la alerta visual roja se acompaña de un contador que muestra el tiempo exacto transcurrido desde la apertura.

---

### HU-36 · Panel de Filtros de Historial y Auditoría

- **Como** Químico Farmacéutico (Regente)
- **Quiero** consultar y filtrar los datos históricos por fecha y hora
- **Para** analizar eventos retrospectivos, excursiones pasadas y generar evidencia para auditorías
- **Prioridad:** Media · **Sprint:** 7 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (filtro aplicado exitosamente):** Dado que entro al módulo de historial, cuando selecciono un rango temporal en el DatePicker (basado en shadcn/ui) y ejecuto la búsqueda, entonces se lanza una petición REST a la base de datos (PostgreSQL) y tanto la gráfica de Apache ECharts como la tabla de datos recargan la información exacta de ese periodo.
2. **Escenario 2 (rango temporal sin datos):** Dado que el rango de fechas seleccionado no contiene telemetría registrada (ej. equipo apagado), cuando se responde la consulta, entonces el sistema captura la matriz vacía y muestra un componente ilustrativo con el mensaje "Sin datos registrados en el rango seleccionado".
3. **Escenario 3 (validación de rango inválido):** Dado que selecciono un rango incongruente donde la fecha de fin es cronológicamente anterior a la fecha de inicio, cuando intento aplicar el filtro, entonces la validación del formulario (Frontend) bloquea la consulta y muestra un mensaje de error indicando que se debe ajustar la selección.

---

### HU-37 · Formulario Check-list BPA Digital

- **Como** Químico Farmacéutico (Regente)
- **Quiero** completar un check-list digital basado en el Manual de Buenas Prácticas de Almacenamiento (BPA)
- **Para** digitalizar el formato en papel, centralizar la evidencia operativa y facilitar autoinspecciones periódicas
- **Prioridad:** Baja · **Sprint:** 7 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (envío completo y auditado):** Dado que abro el módulo BPA en el dashboard y completo todos los ítems obligatorios normativos, cuando confirmo el envío del formulario, entonces el documento se envía al backend (FastAPI) firmando la transacción con la identidad extraída de mi token JWT y la fecha exacta del registro en PostgreSQL.
2. **Escenario 2 (validación de campos incompletos):** Dado que algún ítem obligatorio del check-list queda sin marcar por descuido, cuando intento hacer clic en enviar, entonces la interfaz de React impide el envío, detiene la petición al backend y resalta visualmente en rojo los campos pendientes para su corrección.
3. **Escenario 3 (guardado de borrador local):** Dado que no logro terminar la inspección del checklist en una sola sesión continua, cuando decido salir de la vista o recargar la página, entonces el estado global del frontend guarda mi avance temporalmente como borrador para permitirme continuar desde el mismo punto posteriormente.

---

### HU-38 · Exportación de Reporte PDF (Evidencia Sanitaria)

- **Como** responsable de auditoría o inspector sanitario (ej. DIRIS/DIGEMID)
- **Quiero** generar y exportar un reporte en formato PDF del historial térmico y las alertas
- **Para** adjuntarlo como evidencia digital verificable del cumplimiento de la cadena de frío
- **Prioridad:** Media · **Sprint:** 7 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (exportación exitosa con trazabilidad Hash):** Dado que visualizo un rango de fechas filtrado en el dashboard, cuando presiono el botón "Exportar PDF", entonces el navegador descarga un documento formal que incluye el resumen gráfico de ECharts, la tabla de telemetría y las firmas criptográficas SHA-256 asociadas a cada registro para respaldar su inmutabilidad algorítmica.
2. **Escenario 2 (rango temporal muy extenso):** Dado que el rango de fechas solicitado abarca un gran volumen de datos (ej. todo un mes de registros continuos), cuando el frontend procesa la generación del reporte, entonces el sistema pagina adecuadamente las tablas dentro del PDF y resume los KPIs sin omitir el periodo crítico solicitado.
3. **Escenario 3 (error en la generación del documento):** Dado que ocurre un error inesperado al intentar compilar o descargar el archivo PDF por falta de memoria o red, cuando el sistema detecta la falla, entonces muestra un mensaje de error claro al usuario (ej. Toast notification) y permite reintentar la descarga sin perder los filtros de fecha previamente aplicados.

---

# Épica EP06 · Autenticación, Roles y Auditoría (Seguridad transversal)

---

### HU-39 · Autenticación de Usuarios (Login seguro)

- **Como** usuario del sistema (Técnico/Farmacéutico)
- **Quiero** ingresar mi correo y contraseña de forma segura
- **Para** obtener acceso al dashboard cerrado conforme a OWASP Web Security Testing Guide v4.2
- **Prioridad:** Alta · **Sprint:** 1 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (credenciales correctas):** Dado que envío credenciales válidas al endpoint /api/login, cuando el backend (FastAPI) las verifica contra el hash bcrypt almacenado en PostgreSQL, entonces recibo un token JWT firmado criptográficamente que incluye mi rol operativo autorizado.
2. **Escenario 2 (mitigación de enumeración de usuarios):** Dado que envío una contraseña incorrecta o un correo inexistente, cuando el sistema procesa la validación, entonces se rechaza el acceso con un mensaje genérico (ej. "Credenciales inválidas") sin revelar si el correo existe en la base de datos, y se registra el evento en la bitácora.
3. **Escenario 3 (protección contra fuerza bruta):** Dado que se superan varios intentos fallidos consecutivos para la misma cuenta en una ventana de tiempo corta, cuando ocurre el siguiente intento de login, entonces se aplica una restricción temporal (rate limiting / bloqueo de cuenta) antes de permitir un nuevo intento, mitigando ataques automatizados.

---

### HU-40 · Gestión Segura de JWT en Memoria

- **Como** usuario autenticado
- **Quiero** que mi token JWT se almacene solo en la memoria volátil del navegador y nunca en localStorage
- **Para** reducir el riesgo de robo de sesión por ataques XSS
- **Prioridad:** Alta · **Sprint:** 1 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (sesión activa y segura):** Dado que me autentico correctamente en la plataforma, cuando navego por la SPA (Single Page Application), entonces el token JWT vive únicamente en el gestor de estado de la aplicación (ej. Zustand o React Context), sin persistir en localStorage ni sessionStorage.
2. **Escenario 2 (cierre de pestaña o navegador):** Dado que cierro la pestaña actual o el navegador por completo, cuando intento volver a acceder a la URL protegida del dashboard, entonces el token ya no está disponible en memoria y el sistema me redirige obligatoriamente a la vista de login.
3. **Escenario 3 (recarga manual defensiva):** Dado que recargo manualmente la página (F5) estando autenticado, cuando la aplicación React se reinicializa, entonces se me solicita iniciar sesión nuevamente, ya que el estado en memoria se purga de forma esperada garantizando que no queden rastros del token en el cliente.

---

### HU-41 · Autorización por Roles (RBAC y Principio de Mínimo Privilegio)

- **Como** Administrador del sistema
- **Quiero** restringir el acceso a rutas visuales y funciones de la API según el rol del usuario autenticado (Técnico, Farmacéutico, Auditor)
- **Para** aplicar el principio de mínimo privilegio y cumplir con las normativas de seguridad OWASP
- **Prioridad:** Alta · **Sprint:** 7 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (acceso autorizado):** Dado que un usuario con rol de Administrador navega a la ruta de gestión de usuarios, cuando React Router en el frontend y FastAPI en el backend verifican el claim de rol dentro del JWT, entonces el sistema permite renderizar la vista y retornar los datos solicitados.
2. **Escenario 2 (acceso no autorizado y protección de API):** Dado que un Técnico intenta acceder a la ruta /usuarios o invoca directamente el endpoint protegido, cuando se verifica el rol del JWT, entonces React Router bloquea la vista con "Acceso Denegado" y FastAPI rechaza la petición con un código HTTP 403 (Forbidden), sin exponer información sensible.
3. **Escenario 3 (cambio de rol en sesión activa):** Dado que los privilegios de un usuario son modificados por un Administrador mientras el usuario tiene una sesión activa, cuando el usuario realiza su siguiente acción y el token se refresca o revalida, entonces el sistema aplica inmediatamente los permisos actualizados, invalidando los privilegios originales del inicio de sesión.

---

### HU-42 · Bitácora Inmutable de Auditoría (Audit Logs con SHA-256)

- **Como** Responsable de Auditoría
- **Quiero** tener un registro automatizado e inmutable de cada inicio de sesión, cambio de estado y exportación de datos
- **Para** cumplir con el seguimiento estricto de identidades ante inspecciones sanitarias
- **Prioridad:** Media · **Sprint:** 5 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (evento registrado criptográficamente):** Dado que un usuario inicia sesión, falla un intento de login o exporta evidencia, cuando el backend (FastAPI) detecta la acción, entonces guarda un registro asíncrono en la base de datos (PostgreSQL) con el ID de usuario, IP, timestamp y su firma criptográfica SHA-256.
2. **Escenario 2 (intento de modificación / Append-only):** Dado que un actor malicioso o administrador intenta alterar o borrar un registro de auditoría ya almacenado, cuando se procesa la solicitud a nivel de base de datos, entonces se rechaza la operación, ya que la tabla de trazabilidad está configurada estrictamente como solo escritura (append-only) y rompería la cadena del previous_hash.
3. **Escenario 3 (consulta filtrada y segura):** Dado que el Auditor necesita revisar los eventos de seguridad de un periodo o usuario específico, cuando aplica los filtros en el módulo de auditoría de React, entonces obtiene únicamente los registros coincidentes en modo de solo lectura, sin ninguna posibilidad técnica de editarlos desde la interfaz.

---

### HU-44 · Aceptación de Términos, Consentimiento y Política de Privacidad (Ley 29733)

- **Como** Usuario del sistema (Técnico/Farmacéutico/Administrador)
- **Quiero** aceptar explícitamente la política de privacidad y los términos de uso al iniciar sesión por primera vez
- **Para** cumplir con la Ley N.° 29733 de Protección de Datos Personales en Perú, obteniendo consentimiento informado para el tratamiento de datos sensibles (credenciales, IP, historial de acciones) en la bitácora de auditoría
- **Prioridad:** Alta · **Sprint:** 8 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (Flujo de consentimiento en primer login):** Dado que un usuario se autentica por primera vez en la plataforma con credenciales válidas, cuando el backend verifica que es la primera sesión del usuario (flag privacy_accepted = false en tabla users), entonces FastAPI retorna un código HTTP 200 con el token JWT pero adjunta un flag especial require_privacy_consent: true. Y el frontend React detecta este flag y renderiza un modal no dismissible (no se puede cerrar con X) titulado "Política de Privacidad y Términos de Uso". Y el contenido incluye dos secciones desplegables: Ley N.° 29733 - Protección de Datos Personales (explica qué datos se recopilan, cómo se procesan, duración de retención y derechos del usuario) y Términos de Uso de la Plataforma (responsabilidades operativas, cumplimiento normativo farmacéutico). Y al pie hay dos botones: "Rechazar" (logout) y "Aceptar y Continuar" (guardar consentimiento).
2. **Escenario 2 (Aceptación y registro criptográfico):** Dado que el usuario lee las políticas y hace clic en "Aceptar y Continuar", cuando envía la confirmación, entonces FastAPI registra en la tabla users el campo privacy_accepted = true y privacy_accepted_at = <timestamp>. Y genera un evento en traceability_records con: tipo_evento: "ACEPTACION_PRIVACIDAD", usuario_id del usuario que acepta, payload: {usuario_id, email, ip_origen, versión_política_aceptada}, su propio hash SHA-256 encadenado. Y el frontend autoriza la navegación normal al dashboard. Y en auditoría queda registro de: "Usuario [email] aceptó política de privacidad el [fecha] desde IP [ip]".
3. **Escenario 3 (Rechazo de consentimiento - Prohibición de acceso):** Dado que un usuario hace clic en "Rechazar", cuando confirma el rechazo, entonces FastAPI: invalida el JWT token retornado en el login anterior; registra el rechazo en audit_logs con tipo_evento: "RECHAZO_PRIVACIDAD"; retorna HTTP 401 Unauthorized con mensaje: "No se puede continuar sin aceptar la política de privacidad". Y el frontend redirige al usuario a la página de login. Y en el próximo intento de login, se vuelve a mostrar la política (sin cachear el rechazo).
4. **Escenario 4 (Re-aceptación ante actualizaciones de política):** Dado que la política de privacidad se actualiza (versión 2.0), cuando se incrementa la versión en el backend, entonces todos los usuarios con versión anterior ven el modal nuevamente al siguiente login, y deben aceptar la nueva versión para continuar, y el historial de versiones aceptadas queda registrado criptográficamente.

---

### HU-45 · Anonimización y Desactivación de Usuarios (Derecho al Olvido - Ley 29733)

- **Como** Administrador del Sistema
- **Quiero** desactivar o anonimizar un usuario (técnico farmacéutico cesado) sin eliminar su histórico de acciones auditables
- **Para** cumplir con el derecho al olvido de la Ley 29733 mientras mantengo la cadena de trazabilidad íntegra para auditorías (21 CFR Part 11)
- **Prioridad:** Alta · **Sprint:** 8 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (Desactivación de usuario sin eliminar audit trail):** Dado que un Técnico de Farmacia renuncia o es despedido, cuando el Administrador accede a "Gestión de Usuarios" y selecciona la opción "Desactivar" en el usuario, entonces el sistema abre un formulario que requiere: motivo de desactivación (renuncia, despido, jubilación, otros), fecha efectiva de desactivación, confirmación del Administrador. Y al confirmar: el campo users.is_active = false; el usuario NO puede iniciar sesión nuevamente (login rechazado); los registros de audit_logs donde aparece su ID se preservan intactos (lectura únicamente); se registra un evento en traceability_records: tipo_evento: "DESACTIVACION_USUARIO", usuario_id del usuario desactivado, usuario_id del administrador que desactivó, payload: {usuario_desactivado_id, motivo, ip_origen_admin}, con su respectivo hash SHA-256 encadenado.
2. **Escenario 2 (Anonimización progresiva en reportes futuros):** Dado que un usuario fue desactivado hace 3 meses, cuando se genera un reporte PDF o se consulta el módulo de auditoría, entonces en el frontend: los registros históricos del usuario desactivado muestran "[Usuario eliminado]" en lugar de su nombre/email; el campo revisada_por en alertas muestra "Usuario anónimo" si fue él quien las revisó; el usuario_id persiste en la BD (para integridad criptográfica) pero se oculta en la interfaz. Y en la BD, los datos personales (nombre, email) NO se eliminan pero se marcan con un flag anonymized_for_gdpr = true.
3. **Escenario 3 (Imposibilidad de modificar audit_logs):** Dado que se desactiva un usuario cuyas acciones están registradas en audit_logs, cuando el Administrador intenta modificar cualquier registro histórico donde aparece el usuario desactivado, entonces la BD rechaza la operación con error "Append-only table: modificaciones no permitidas". Y se registra un intento fallido de modificación en los logs de seguridad.
4. **Escenario 4 (Validación de cumplimiento Ley 29733):** Dado que se ejecuta una auditoría de cumplimiento normativo, cuando se solicita un reporte de "Usuarios desactivados" desde el módulo de Administración, entonces aparece una tabla con: nombre del usuario desactivado, fecha de desactivación, motivo de desactivación, número de acciones registradas mientras estuvo activo (sin revelar detalles si está anonimizado), confirmación de que sus datos se han anonimizado según GDPR/Ley 29733.

---

### HU-50 · Registro de Firmas, Siglas y Rúbricas Digitales del Personal

- **Como** Responsable de Auditoría
- **Quiero** registrar la firma digitalizada, iniciales y siglas autorizadas de cada usuario en su perfil del sistema
- **Para** que se estampen de forma visible en los reportes de control térmico bidiarios e historiales de acciones correctivas, cumpliendo con la validez probatoria exigida por el Componente 5 (Documentación) del manual de BPA
- **Prioridad:** Media · **Sprint:** 10 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (Configuración de firma autorizada):** Dado que el Químico Farmacéutico edita su perfil de usuario, cuando sube un archivo de imagen válido con su firma/rúbrica y escribe sus siglas de Colegiatura Farmacéutica (ej: "Q.F. D.S. Q. - CQFP 12345"), entonces el backend valida el formato, asocia los metadatos al modelo de usuario de forma segura y registra un evento de actualización criptográfico en el ledger de trazabilidad.
2. **Escenario 2 (Incrustación en Reportes BPA):** Dado que el usuario exporta el Reporte BPA en PDF (HU-38) o el Checklist digital (HU-37), cuando se renderizan las acciones correctivas aplicadas a las alertas térmicas, entonces el PDF incluye la imagen de la firma y las siglas del profesional que aprobó la acción, firmando digitalmente el documento completo con la firma hash calculada en la base de datos.

---

# 4. Plan de Sprints

> Cada sprint dura 4 semanas (20 días · 160 horas). Total: 40 semanas · 200 días · 1600 horas · 51 historias de usuario distribuidas en 10 sprints.

| Sprint | Story Points | Historias de usuario |
|---|---|---|
| **SP-01** | 29 | HU-01 · HU-02 · HU-05 · HU-39 · HU-40 · HU-03 · HU-04 |
| **SP-02** | 31 | HU-06 · HU-07 · HU-09 · HU-10 · HU-08 |
| **SP-03** | 23 | HU-11 · HU-12 · HU-15 · HU-22 · HU-13 · HU-14 |
| **SP-04** | 34 | HU-16 · HU-17 · HU-18 · HU-21 · HU-19 · HU-20 |
| **SP-05** | 39 | HU-24 · HU-25 · HU-26 · HU-27 · HU-28 · HU-30 · HU-42 |
| **SP-06** | 24 | HU-31 · HU-32 · HU-34 · HU-33 · HU-35 |
| **SP-07** | 28 | HU-23 · HU-41 · HU-36 · HU-38 · HU-29 · HU-37 |
| **SP-08** | 21 | HU-43 · HU-44 · HU-45 |
| **SP-09** | 21 | HU-46 · HU-47 |
| **SP-10** | 21 | HU-48 · HU-49 · HU-50 · HU-51 |
| **TOTAL** | **271** | **51 historias de usuario** |

## Detalle por sprint

### SP-01 · Sprint 1 (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario |
|---|---|
| HU-01 | Como farmacéutico responsable, quiero que el sistema capture la temperatura del sensor DS18B20 cada 30 segundos, para vigilar con precisión la condición térmica cercana al medicamento. |
| HU-02 | Como farmacéutico responsable, quiero que el sistema capture la temperatura ambiental mediante el sensor SHT31, para complementar el control de la temperatura interna del refrigerador. |
| HU-05 | Como administrador del sistema, quiero que el sistema estructure las métricas en un payload JSON estándar, para asegurar interoperabilidad con el backend aunque alguna variable no esté disponible. |
| HU-39 | Como usuario del sistema (Técnico/Farmacéutico), quiero ingresar mi correo y contraseña de forma segura, para obtener acceso al dashboard cerrado conforme a OWASP Web Security Testing Guide v4.2. |
| HU-40 | Como usuario autenticado, quiero que mi token JWT se almacene solo en la memoria volátil del navegador y nunca en localStorage, para reducir el riesgo de robo de sesión por ataques XSS. |
| HU-03 | Como farmacéutico responsable, quiero que el sistema registre la humedad relativa del sensor SHT31, para detectar condiciones ambientales de riesgo adicionales a la temperatura. |
| HU-04 | Como técnico de farmacia, quiero que el sistema registre los eventos del sensor magnético MC-38, para relacionar aperturas reales de la puerta con fluctuaciones térmicas. |

### SP-02 · Sprint 2 (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario |
|---|---|
| HU-06 | Como farmacéutico responsable, quiero que el sistema conserve las lecturas en memoria local cuando se pierda conectividad, para no perder trazabilidad térmica durante cortes prolongados. |
| HU-07 | Como farmacéutico responsable, quiero que el sistema reenvíe automáticamente los registros acumulados en LittleFS, para completar el histórico térmico cuando la conexión se recupere, sin duplicidades ni pérdidas. |
| HU-09 | Como administrador del sistema, quiero que el nodo valide el certificado de la Autoridad Certificadora (CA) del servidor mediante One-way TLS 1.2/1.3, para cifrar los datos en tránsito y evitar ataques Man-in-the-Middle. |
| HU-10 | Como administrador del sistema, quiero que cada nodo ESP32 use credenciales únicas y validación Server Name Indication (SNI), para impedir el acceso de hardware no autorizado conforme a OWASP IoT. |
| HU-08 | Como administrador del sistema, quiero que el nodo se reconecte usando una estrategia de backoff exponencial, para restablecer el servicio sin saturar la red local ni el microcontrolador. |

### SP-03 · Sprint 3 (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario |
|---|---|
| HU-11 | Como administrador del sistema, quiero que las lecturas térmicas se publiquen con QoS 1 (al menos una vez), para confirmar la recepción en el broker y evitar vacíos en la serie temporal. |
| HU-12 | Como administrador del sistema, quiero que el backend se suscriba al tópico farmacias/+/lecturas mediante aiomqtt, para consumir telemetría de cualquier refrigerador sin bloquear el servidor. |
| HU-15 | Como administrador del sistema, quiero que el backend valide los payloads MQTT entrantes con Pydantic v2, para rechazar estructuras corruptas o maliciosas antes de persistir los datos. |
| HU-22 | Como farmacéutico responsable, quiero que la telemetría se almacene en PostgreSQL usando JSONB, para conservar un histórico térmico flexible junto con su clasificación asociada. |
| HU-13 | Como administrador del sistema, quiero que el nodo configure un mensaje Last Will and Testament (LWT) en el broker, para notificar caídas abruptas de energía o red al backend y al dashboard. |
| HU-14 | Como administrador del sistema, quiero que el broker MQTT gestionado bloquee conexiones por el puerto 1883 en texto plano, para forzar comunicaciones seguras y evitar sniffers de red. |

### SP-04 · Sprint 4 (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario |
|---|---|
| HU-16 | Como administrador del sistema, quiero que el backend cargue el modelo Random Forest al iniciar el servicio, para procesar inferencias de riesgo sin latencia de lectura de disco por mensaje. |
| HU-17 | Como farmacéutico responsable, quiero que el sistema calcule métricas derivadas como gradiente térmico y derivadas de tiempo, para alimentar correctamente el modelo Random Forest que clasifica el riesgo. |
| HU-18 | Como farmacéutico responsable, quiero que el sistema evalúe la matriz de características operativas, para clasificar el estado térmico en Normal, Riesgo Preventivo o Excursión Crítica. |
| HU-21 | Como farmacéutico responsable, quiero que el sistema registre una excursión térmica crítica si la temperatura sale del rango de 2 °C a 8 °C, para ejecutar acciones correctivas auditables conforme a las Buenas Prácticas de Almacenamiento. |
| HU-19 | Como técnico de farmacia, quiero ver en tiempo real la lectura procesada y la clasificación de riesgo en el dashboard, para monitorear incidentes sin recargar la interfaz. |
| HU-20 | Como farmacéutico responsable, quiero que el sistema genere una alerta cuando el modelo detecte Riesgo Preventivo, para actuar antes de que se rompa la cadena de frío. |

### SP-05 · Sprint 5 (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario |
|---|---|
| HU-24 | Como responsable de auditoría, quiero que el sistema calcule un hash SHA-256 por cada evento térmico u operativo, para asegurar su inmutabilidad algorítmica sin depender de infraestructuras blockchain de terceros. |
| HU-25 | Como responsable de auditoría, quiero que cada nuevo registro incluya el hash del evento anterior en traceability_records, para formar una cadena criptográfica inviolable. |
| HU-26 | Como Responsable de Auditoría o Fiscalización interna, quiero solicitar una verificación algorítmica en bloque de toda la tabla de trazabilidad, para detectar manipulaciones en los registros y presentar evidencias confiables ante inspecciones de DIGEMID. |
| HU-27 | Como Químico Farmacéutico (Regente), quiero registrar la acción correctiva aplicada tras una alerta, para cumplir con el soporte documental exigido por la normativa de Buenas Prácticas de Almacenamiento de DIGEMID. |
| HU-28 | Como responsable de auditoría, quiero que el sistema calcule hash SHA-256 también sobre las acciones manuales de los usuarios, para impedir cambios posteriores en las justificaciones registradas. |
| HU-30 | Como Químico Farmacéutico, quiero registrar la fecha de última calibración y el certificado del sensor de temperatura, para recibir alertas antes del vencimiento y mantener el cumplimiento regulatorio. |
| HU-42 | Como Responsable de Auditoría, quiero tener un registro automatizado e inmutable de cada inicio de sesión, cambio de estado y exportación de datos, para cumplir con el seguimiento estricto de identidades ante inspecciones sanitarias. |

### SP-06 · Sprint 6 (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario |
|---|---|
| HU-31 | Como técnico de farmacia, quiero visualizar una gráfica de temperatura con umbrales fijos de 2 °C a 8 °C, para evaluar rápidamente la tendencia térmica del refrigerador. |
| HU-32 | Como técnico de farmacia, quiero que el dashboard reciba actualizaciones del backend mediante Server-Sent Events (SSE), para ver el último punto de la gráfica y los KPI en tiempo real. |
| HU-34 | Como farmacéutico, quiero ver un semáforo visual (Verde, Amarillo, Rojo) en la tarjeta del equipo según la inferencia del Random Forest, para identificar de inmediato el nivel de atención clínica o técnica requerido. |
| HU-33 | Como técnico de farmacia, quiero visualizar el último valor de temperatura (DS18B20) y humedad ambiental (SHT31) en tarjetas KPI, para obtener una lectura rápida del estado del equipo a un solo vistazo. |
| HU-35 | Como personal técnico de farmacia, quiero visualizar una alerta intermitente en la interfaz si el refrigerador permanece abierto, para cerrarlo rápidamente y evitar fuga térmica y condensación. |

### SP-07 · Sprint 7 (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario |
|---|---|
| HU-23 | Como farmacéutico responsable, quiero recibir una notificación automática por correo electrónico o Telegram cuando se genere una excursión crítica, para enterarme del incidente de forma inmediata aunque no tenga el dashboard web abierto. |
| HU-41 | Como Administrador del sistema, quiero restringir el acceso a rutas visuales y funciones de la API según el rol del usuario autenticado (Técnico, Farmacéutico, Auditor), para aplicar el principio de mínimo privilegio y cumplir con las normativas de seguridad OWASP. |
| HU-36 | Como Químico Farmacéutico (Regente), quiero consultar y filtrar los datos históricos por fecha y hora, para analizar eventos retrospectivos, excursiones pasadas y generar evidencia para auditorías. |
| HU-38 | Como responsable de auditoría o inspector sanitario (ej. DIRIS/DIGEMID), quiero generar y exportar un reporte en formato PDF del historial térmico y las alertas, para adjuntarlo como evidencia digital verificable del cumplimiento de la cadena de frío. |
| HU-29 | Como administrador de infraestructura, quiero que la base de datos PostgreSQL se respalde de forma automatizada, para asegurar la continuidad operativa y proteger la evidencia histórica ante caídas o desastres. |
| HU-37 | Como Químico Farmacéutico (Regente), quiero completar un check-list digital basado en el Manual de Buenas Prácticas de Almacenamiento (BPA), para digitalizar el formato en papel, centralizar la evidencia operativa y facilitar autoinspecciones periódicas. |

### SP-08 · Sprint 8 (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario |
|---|---|
| HU-43 | Como Administrador del Sistema / Técnico de Mantenimiento, quiero registrar la baja de un dispositivo IoT (ESP32/sensores) cuando se malogre o sea reemplazado, sin corromper la cadena de trazabilidad histórica, para mantener la consistencia administrativa, auditar el reemplazo de hardware y cumplir con DIGEMID en caso de cambios en la configuración del monitoreo. |
| HU-44 | Como Usuario del sistema (Técnico/Farmacéutico/Administrador), quiero aceptar explícitamente la política de privacidad y los términos de uso al iniciar sesión por primera vez, para cumplir con la Ley N.° 29733 de Protección de Datos Personales en Perú, obteniendo consentimiento informado para el tratamiento de datos sensibles (credenciales, IP, historial de acciones) en la bitácora de auditoría. |
| HU-45 | Como Administrador del Sistema, quiero desactivar o anonimizar un usuario (técnico farmacéutico cesado) sin eliminar su histórico de acciones auditables, para cumplir con el derecho al olvido de la Ley 29733 mientras mantengo la cadena de trazabilidad íntegra para auditorías (21 CFR Part 11). |

### SP-09 · Sprint 9 (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario |
|---|---|
| HU-46 | Como Administrador del Sistema / Administrador IoT, quiero desplegar actualizaciones de firmware cifradas al ESP32 de forma remota (OTA), verificando integridad y permitiendo rollback en caso de fallo, para parchear vulnerabilidades de seguridad detectadas (OWASP IoT v1.0.0) sin requerir intervención física en cada refrigerador farmacéutico. |
| HU-47 | Como Responsable de Base de Datos / Administrador del Sistema, quiero que cuando se detecte corrupción en la cadena hash (un registro alterado), el sistema me notifique, identifique el punto exacto de ruptura y me guíe en la recuperación sin perder integridad futura, para mantener la confiabilidad del sistema ante intentos de manipulación y poder restaurar la continuidad de la auditoría tras incidentes de seguridad. |

### SP-10 · Sprint 10 (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario |
|---|---|
| HU-48 | Como Químico Farmacéutico Responsable, quiero asociar un lote específico de medicamentos a un dispositivo de refrigeración y activar el bloqueo de dispensación virtual ante una excursión térmica crítica sostenida, para evitar que se despachen medicamentos termosensibles que hayan perdido su estabilidad biológica, cumpliendo con el Componente 4 de las BPA. |
| HU-49 | Como Químico Farmacéutico (Regente), quiero registrar y gestionar órdenes de Retiro de Mercado (simuladas o reales) asociadas a lotes que presentaron fallas de conservación, para documentar la eficacia del procedimiento de retiro ante auditorías de DIGEMID bajo el Componente 7 de las BPA. |
| HU-50 | Como Responsable de Auditoría, quiero registrar la firma digitalizada, iniciales y siglas autorizadas de cada usuario en su perfil del sistema, para que se estampen de forma visible en los reportes de control térmico bidiarios e historiales de acciones correctivas, cumpliendo con la validez probatoria exigida por el Componente 5 (Documentación) del manual de BPA. |
| HU-51 | Como Químico Farmacéutico Responsable, quiero configurar la ubicación física exacta del sensor de temperatura interno (DS18B20) basado en el informe del estudio de Mapeo Térmico del refrigerador, para garantizar a los inspectores de DIGEMID que el sensor está midiendo el punto de mayor oscilación de temperatura de la unidad de frío (BPA Componente 3). |
