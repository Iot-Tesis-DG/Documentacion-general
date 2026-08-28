# Historias de Usuario — Product Backlog (Anexo 5)

**Informe de tareas pendientes de productos (Product Backlog)**

Sistema de monitoreo de cadena de frío para refrigeradores farmacéuticos (ESP32 + EMQX Cloud + FastAPI + React + PostgreSQL), con **integridad verificable mediante hash encadenado SHA-256** (registro append-only) y cumplimiento del marco normativo peruano: **Ley N.° 29733** de Protección de Datos Personales (y su D.S. 016-2024-JUS), **RM 554-2022/MINSA** y **RM 810-2024/MINSA** (Buenas Prácticas de Oficina Farmacéutica), lineamientos **DIGEMID** y guías de seguridad **OWASP IoT / WSTG**.

> **Alcance del backlog:** este Product Backlog contiene únicamente **51 Historias de Usuario de Producto** orientadas a los roles operativos del sistema (Farmacéutico, Técnico, Administrador, Auditor). Las actividades de experimentación e investigación (preparación del dataset, entrenamiento y evaluación del Random Forest, pruebas OWASP, pruebas de estrés offline y validación en campo) **no son historias de producto**: se gestionan en el Capítulo de Metodología y Validación Técnica de la tesis como *Technical Enablers* (TE) y *Validation Tasks* (VT).

---

## 1. Resumen general

| Concepto | Valor |
|---|---|
| Historias de usuario de producto | 51 |
| Épicas | 6 (EP01 – EP06) |
| Sprints | 10 (SP-01 – SP-10), de 4 semanas cada uno |
| Duración total | 40 semanas · 200 días · 1600 horas |
| **Total Story Points (reestimado)** | **262** |
| Estado general | No se ha iniciado |

**Distribución por prioridad:**

| Prioridad | Cantidad | Historias |
|---|---|---|
| Alta | 31 | HU-01, 02, 05, 06, 07, 09, 10, 11, 12, 15, 16, 17, 18, 21, 22, 23, 24, 25, 26, 31, 32, 34, 39, 40, 41, 43, 44, 45, 46, 47, 50 |
| Media | 19 | HU-03, 04, 08, 13, 14, 19, 20, 27, 28, 30, 33, 35, 36, 37, 38, 42, 48, 49, 51 |
| Baja | 1 | HU-29 |

**Criterios de reestimación (escala 1/2/3/5/8/13):** los Story Points se reestimaron desde cero valorando complejidad técnica, incertidumbre e integración entre capas, con énfasis en los tres ejes que justifican el esfuerzo ante el jurado: **controles de seguridad transversal** (TLS, credenciales únicas, RBAC, bitácora), **observabilidad del modelo de IA** (versionado del modelo, trazabilidad de la inferencia) y **trazabilidad verificable** (hash encadenado SHA-256, comportamiento append-only).

---

## 2. Épicas

| ID | Épica | Descripción | HU incluidas |
|---|---|---|---|
| **EP01** | Adquisición de Datos, Resiliencia y Gestión Edge | Captura de variables térmicas/ambientales/apertura en el nodo ESP32, estructuración en JSON interoperable, continuidad del registro ante cortes de energía o conectividad (buffer LittleFS con reanudación verificada) y gestión del ciclo de vida y metadatos de instalación del hardware. Base de toda la cadena de datos. | HU-01, 02, 03, 04, 05, 06, 07, 08, 43, 51 |
| **EP02** | Comunicación MQTT Segura | Transporte cifrado y autenticado de telemetría hacia EMQX Cloud: handshake TLS con validación del certificado del servidor, autenticación de dispositivo por device_id + token único + ACL, QoS 1 con manejo de duplicados, LWT y bloqueo de canales no cifrados. Primera línea de defensa contra manipulación de datos en tránsito. | HU-09, 10, 11, 12, 13, 14, 44 |
| **EP03** | Procesamiento e Inteligencia Artificial | Validación de payloads con Pydantic, extracción de características, inferencia de riesgo con Random Forest, gobierno y versionado operativo del modelo, trazabilidad de la inferencia, emisión SSE y generación de alertas preventivas/críticas. Componente central de clasificación del riesgo térmico. | HU-15, 16, 17, 18, 19, 20, 21, 46, 47 |
| **EP04** | Persistencia y Trazabilidad Verificable | Persistencia asíncrona en PostgreSQL (JSONB), hash encadenado SHA-256 con comportamiento append-only, verificación de integridad O(n), registro auditable de acciones correctivas y de cambios de configuración del hardware, backups y controles operativos verificables. Reemplaza registros manuales por evidencia digital verificable ante auditorías internas e inspecciones. | HU-22, 24, 25, 26, 27, 28, 29, 30, 37, 49 |
| **EP05** | Monitoreo Web, Alertas y Reportes | Dashboard React en tiempo real (SSE): tendencia térmica, KPIs, semáforo de riesgo efectivo, alertas de puerta, notificación asíncrona de excursiones críticas, estado de sincronización del nodo, filtros de historial y exportación de evidencia en PDF. Capa de interacción operativa del personal de farmacia. | HU-23, 31, 32, 33, 34, 35, 36, 38, 48 |
| **EP06** | Autenticación, Autorización y Seguridad Transversal | Login seguro (bcrypt + JWT en memoria volátil), RBAC de mínimo privilegio, bitácora append-only con hash encadenado, desactivación/revocación de acceso de usuarios y supervisión de eventos de seguridad. Componente transversal que protege las operaciones críticas de las demás épicas. | HU-39, 40, 41, 42, 45, 50 |

---

## 3. Historias de usuario por épica

> Formato de cada historia: **Como** [rol], **quiero** [capacidad], **para** [beneficio].

---

# Épica EP01 · Adquisición de Datos, Resiliencia y Gestión Edge

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

1. **Escenario 1 (payload completo y ajustado a la arquitectura):** Dado que las variables de los sensores se capturaron correctamente, cuando se preparan para envío, entonces se genera un JSON con device_id, reading_id, timestamp (ISO 8601 UTC), estado_conectividad y firmware_version.
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
2. **Escenario 2 (buffer lleno):** Dado que el espacio en LittleFS alcanza el umbral máximo configurado, cuando se intenta guardar una nueva lectura, entonces se aplica política FIFO descartando el archivo más antiguo y se registra un evento de saturación que el backend podrá visualizar como brecha temporal de datos (HU-48).
3. **Escenario 3 (corte de energía durante escritura):** Dado que ocurre una pérdida de energía mientras se escribe un archivo, cuando el ESP32 reinicia, entonces se descarta cualquier archivo incompleto o corrupto detectado por verificación de integridad antes de reanudar la operación.

---

### HU-07 · Sincronización de Buffer Offline con Acuse Lógico

- **Como** farmacéutico responsable
- **Quiero** que el sistema reenvíe automáticamente los registros acumulados en LittleFS y los elimine solo cuando el backend confirme su persistencia
- **Para** completar el histórico térmico cuando la conexión se recupere, sin duplicidades ni pérdidas
- **Prioridad:** Alta · **Sprint:** 2 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (sincronización con acuse lógico del backend):** Dado que el ESP32 se reconecta a EMQX y existen archivos de telemetría pendientes, cuando se inicia el proceso de sincronización, entonces se publican en estricto orden FIFO utilizando QoS 1, y cada bloque se elimina de LittleFS **únicamente tras recibir el acuse lógico (ACK) publicado por FastAPI vía MQTT**, que confirma que la lectura hizo `COMMIT` en PostgreSQL. El `PUBACK` de MQTT QoS 1 por sí solo no autoriza el borrado, pues solo confirma que el broker recibió el paquete, no que la base de datos lo persistió.
2. **Escenario 2 (fallo a mitad de sincronización):** Dado que la conexión se interrumpe abruptamente durante la transmisión de los registros, cuando se recupera la red nuevamente, entonces el sistema reanuda la cola de envíos partiendo estrictamente desde el siguiente archivo al último confirmado por el acuse lógico del backend; los bloques enviados pero no acusados se reenvían, y la idempotencia del backend (HU-11) descarta los duplicados resultantes.
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

### HU-43 · Gestión del Ciclo de Vida de Hardware (Alta/Baja de Dispositivos)

- **Como** Administrador del Sistema / Técnico de Mantenimiento
- **Quiero** registrar la baja de un dispositivo IoT (ESP32/sensores) cuando se malogre o sea reemplazado, sin corromper la cadena de trazabilidad histórica
- **Para** mantener la consistencia administrativa, auditar el reemplazo de hardware y cumplir con DIGEMID en caso de cambios en la configuración del monitoreo
- **Prioridad:** Alta · **Sprint:** 10 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (Baja ordenada de dispositivo funcional):** Dado que un técnico decide retirar un ESP32 activo para mantenimiento preventivo o reemplazo, cuando accede al módulo de "Gestión de Dispositivos" en el dashboard administrativo, entonces encuentra la opción "Dar de baja" con un formulario que solicita: motivo de baja (falla de hardware, mantenimiento, reemplazo, retiro de servicio), descripción detallada del incidente, nuevo device_id del reemplazo (si aplica). Y al confirmar, el sistema registra un evento especial en la tabla traceability_records con: tipo_evento: "BAJA_HARDWARE", previous_hash del último evento del dispositivo original, payload: {device_id_anterior, motivo_baja, device_id_reemplazo_si_existe, timestamp_baja}, su respectivo hash SHA-256 encadenado. Y el device_id anterior se marca como "inactivo" en la tabla devices sin eliminar registros históricos, y se genera una notificación para el Administrador informando el evento.
2. **Escenario 2 (Reemplazo de dispositivo con vinculación de historial):** Dado que un técnico reporta que el sensor DS18B20 del ESP32 "pharmacy_fridge_01" falló, cuando registra la baja y especifica que será reemplazado por "pharmacy_fridge_01_v2", entonces el sistema crea una "cadena de custodia" virtual donde: el nuevo device_id hereda acceso a los datos históricos del anterior (lectura únicamente), se registra una vinculación bidireccional en la tabla devices para auditoría, los futuros registros del nuevo dispositivo comienzan su propia cadena hash independiente, y en reportes y dashboards aparece una nota: "Dispositivo reemplazado de [device_id_anterior] el [fecha]".
3. **Escenario 3 (Baja de dispositivo sin reemplazo - Fin de servicio):** Dado que un refrigerador farmacéutico es retirado de operación permanentemente, cuando se registra la baja con motivo "Fin de servicio / Decommissioning", entonces el dispositivo queda marcado como inactivo pero su histórico completo permanece íntegro en la base de datos. Y si alguien intenta consultarlo desde el dashboard, ve un estado "Dispositivo retirado" con la fecha y razón, y su cadena hash se marca como "cerrada" pero verificable para auditorías futuras.
4. **Escenario 4 (Validación de permisos e integridad):** Dado que un Técnico (rol = TECNICO) intenta dar de baja un dispositivo, cuando hace clic en "Dar de baja", entonces el sistema verifica que solo ADMINISTRADOR o TECNICO con permiso especial pueden ejecutar esta acción. Y si falta permiso, rechaza la operación con HTTP 403 y registra el intento fallido en audit_logs.

---

### HU-51 · Registro de Ubicación y Metadatos de Instalación del Sensor

- **Como** administrador del sistema
- **Quiero** documentar la posición física del sensor DS18B20 dentro del refrigerador y las notas de instalación técnica asociadas a la metadata del dispositivo
- **Para** que las lecturas térmicas queden contextualizadas con el punto exacto de medición y el histórico de instalación del equipo
- **Prioridad:** Media · **Sprint:** 2 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (registro de la ubicación del sensor):** Dado que el Administrador registra o edita un dispositivo de refrigeración en el sistema, cuando ingresa la posición física del sensor DS18B20 dentro del refrigerador (ej. "estante medio, zona cercana a la puerta"), entonces el sistema persiste este dato como metadata del dispositivo y lo muestra en la ficha del equipo.
2. **Escenario 2 (notas de instalación técnica):** Dado que el técnico concluye la instalación física del nodo, cuando registra las notas de instalación (fecha de instalación, técnico responsable, observaciones de montaje), entonces quedan asociadas al device_id y consultables desde la ficha del dispositivo.
3. **Escenario 3 (metadata visible en reportes):** Dado que se genera un reporte histórico del dispositivo (HU-38), cuando se renderiza el encabezado del documento, entonces se incluye la ubicación registrada del sensor, dando contexto a la autoridad que revisa la evidencia sobre el punto exacto donde se midió la temperatura.

---

# Épica EP02 · Comunicación MQTT Segura

---

### HU-09 · Handshake TLS 1.2/1.3 en el Borde (Validación del Servidor)

- **Como** administrador del sistema
- **Quiero** que el nodo valide el certificado de la Autoridad Certificadora (CA) del servidor mediante One-way TLS 1.2/1.3
- **Para** cifrar los datos en tránsito y evitar ataques Man-in-the-Middle
- **Prioridad:** Alta · **Sprint:** 3 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (handshake exitoso):** Dado que el ESP32 inicia conexión hacia el broker EMQX en el puerto seguro 8883, cuando el certificado del servidor es validado como confiable contra la CA raíz almacenada, entonces se establece un canal cifrado y la sesión MQTT puede continuar de forma segura.
2. **Escenario 2 (certificado inválido o expirado):** Dado que el certificado presentado por el servidor no es válido o no coincide con el dominio esperado, cuando se evalúa durante el handshake criptográfico, entonces el ESP32 rechaza la conexión inmediatamente y no publica ningún dato térmico en texto plano.
3. **Escenario 3 (timeout del handshake):** Dado que la negociación asimétrica TLS no se completa dentro del tiempo límite configurado por latencia de red, cuando se agota dicho tiempo, entonces el intento se cancela de forma segura, se registra como fallo de conexión en el log local y se activa el flujo de reconexión con backoff exponencial.

> **Nota:** esta historia cubre la validación del certificado del **servidor** y el cifrado del canal. La autenticación del **dispositivo** se gestiona en la HU-10 (device_id + token único + ACL).

---

### HU-10 · Autenticación de Dispositivo (device_id + token único + ACL)

- **Como** administrador del sistema
- **Quiero** que cada nodo ESP32 se autentique ante el broker con un device_id único y un token/credencial propia, con permisos de publicación restringidos por ACL por tópico
- **Para** impedir el acceso de hardware no autorizado conforme a OWASP IoT
- **Prioridad:** Alta · **Sprint:** 3 · **Story Points:** 5 · **Estado:** No se ha iniciado

> **Aclaración técnica:** el SNI (*Server Name Indication*) forma parte del handshake TLS y sirve para indicar el nombre del servidor y enrutar la conexión segura al host correcto (junto con la validación del certificado, HU-09); **no es un mecanismo de autenticación del dispositivo**. La autenticación del dispositivo se realiza mediante `device_id + token/credencial única + ACL` en el broker.

**Criterios de aceptación:**

1. **Escenario 1 (credenciales válidas):** Dado que un cliente IoT se conecta utilizando un device_id único como usuario y su token/contraseña correspondiente, cuando el broker valida las credenciales, entonces acepta la conexión y la ACL restringe sus permisos de publicación únicamente a sus tópicos designados.
2. **Escenario 2 (credenciales inválidas o revocadas):** Dado que un cliente intenta establecer conexión con un token incorrecto o revocado, cuando el broker evalúa la petición, entonces se rechaza la conexión enviando el código de respuesta MQTT 0x05 (Not Authorized) y se deniega cualquier intento de publicación.
3. **Escenario 3 (aplicación de ACL por tópico):** Dado que un dispositivo con credenciales válidas intenta publicar en tópicos de otro dispositivo o en tópicos administrativos, cuando el broker evalúa la ACL, entonces deniega la publicación y registra el evento en los logs de observabilidad de EMQX Cloud para su análisis.

---

### HU-11 · Publicación de Telemetría (QoS 1) con Idempotencia

- **Como** administrador del sistema
- **Quiero** que las lecturas térmicas se publiquen con QoS 1 (al menos una vez) y que el backend trate los duplicados de red de forma idempotente
- **Para** confirmar la recepción en el broker y evitar vacíos en la serie temporal sin duplicar registros
- **Prioridad:** Alta · **Sprint:** 3 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (publicación confirmada):** Dado que el ESP32 publica un payload JSON con QoS 1 hacia el tópico correspondiente, cuando el broker EMQX lo recibe e ingesta con éxito, entonces responde con el paquete PUBACK y el ESP32 libera de forma segura la memoria RAM asociada a esa lectura.
2. **Escenario 2 (PUBACK no recibido):** Dado que el paquete PUBACK no llega dentro del tiempo de espera configurado debido a intermitencias en la red, cuando se agota el timeout, entonces el ESP32 reintenta la publicación de la misma lectura con el flag DUP (Duplicate) activado, alertando al broker de un posible reenvío.
3. **Escenario 3 (reintentos agotados y contingencia):** Dado que se agota el número máximo de reintentos de publicación QoS 1 sin respuesta del servidor, cuando esto ocurre, entonces se declara la caída de la conexión y la lectura térmica se conserva intacta en el buffer persistente (LittleFS) para su sincronización posterior, en lugar de descartarse.
4. **Escenario 4 (duplicados de red e idempotencia):** Dado que QoS 1 es, por definición del protocolo MQTT, entrega "al menos una vez" y puede generar duplicados de red (ej. reenvíos con flag DUP o re-sincronizaciones del buffer offline), cuando el backend recibe la misma lectura más de una vez, entonces la restricción `UNIQUE(device_id, reading_id)` en PostgreSQL descarta las inserciones duplicadas, garantizando idempotencia estricta sin vacíos ni doble conteo en la serie temporal.

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
- **Prioridad:** Media · **Sprint:** 3 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (desconexión abrupta):** Dado que el nodo pierde energía o conectividad sin enviar un cierre ordenado, cuando el broker detecta el timeout del keep-alive, entonces EMQX publica el LWT preconfigurado enviando un payload JSON con el campo estado_conectividad: offline al tópico de alertas del dispositivo.
2. **Escenario 2 (desconexión ordenada):** Dado que el dispositivo se desconecta limpiamente (ej. mantenimiento), cuando se envía el paquete de red DISCONNECT, entonces el broker descarta el mensaje LWT, ya que la desconexión fue intencional.
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

### HU-44 · Rotación y Revocación de Credenciales IoT

- **Como** administrador del sistema
- **Quiero** revocar o rotar el token MQTT de un ESP32 cuyas credenciales se consideren comprometidas, sin alterar su historial de telemetría
- **Para** cortar de inmediato el acceso del nodo comprometido preservando la integridad del histórico ya registrado
- **Prioridad:** Alta · **Sprint:** 4 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (revocación de credencial comprometida):** Dado que el Administrador detecta que el token de un ESP32 pudo haber sido comprometido, cuando ejecuta la revocación desde la gestión de dispositivos, entonces el broker invalida la credencial de inmediato y todo intento posterior de conexión con ese token es rechazado con código MQTT 0x05 (Not Authorized).
2. **Escenario 2 (rotación con continuidad de servicio):** Dado que se genera un nuevo token para el dispositivo y se aprovisiona en el ESP32, cuando el nodo se reconecta con la nueva credencial, entonces la telemetría histórica ya persistida permanece intacta y asociada al mismo device_id, sin modificaciones ni interrupciones en la cadena de trazabilidad.
3. **Escenario 3 (auditoría del evento):** Dado que se ejecuta una rotación o revocación, cuando se confirma la operación, entonces se registra un evento append-only en la bitácora de auditoría con user_id del administrador, device_id, motivo y timestamp, encadenado con hash SHA-256.

---

# Épica EP03 · Procesamiento e Inteligencia Artificial

---

### HU-15 · Validación de esquema entrante (Pydantic)

- **Como** administrador del sistema
- **Quiero** que el backend valide los payloads MQTT entrantes con Pydantic v2
- **Para** rechazar estructuras corruptas o maliciosas antes de persistir los datos
- **Prioridad:** Alta · **Sprint:** 4 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (payload válido):** Dado que llega un mensaje MQTT con todos los campos obligatorios y tipos de datos correctos desde la capa Edge, cuando se valida contra el esquema de Pydantic, entonces el modelo de datos se acepta y el flujo asíncrono continúa hacia la capa de Dominio.
2. **Escenario 2 (campo faltante):** Dado que el payload carece de una variable crítica (ej. temperatura_interna), cuando se ejecuta la validación estructural, entonces se rechaza el mensaje, se genera un log de error detallando el campo faltante y no se persiste en la base de datos.
3. **Escenario 3 (tipo de dato incorrecto o JSON malformado):** Dado que un campo tiene un tipo no esperado (ej. string en lugar de float) o el JSON está malformado, cuando se intenta deserializar, entonces Pydantic captura la excepción de validación, descarta el mensaje y registra la advertencia, garantizando que el event loop de aiomqtt no se bloquee ni se detenga el consumo de telemetría de otros dispositivos.

---

### HU-16 · Carga en memoria del Modelo IA (Lifespan)

- **Como** administrador del sistema
- **Quiero** que el backend cargue el modelo Random Forest al iniciar el servicio
- **Para** procesar inferencias de riesgo sin latencia de lectura de disco por mensaje
- **Prioridad:** Alta · **Sprint:** 5 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (carga exitosa en RAM):** Dado que el servidor Uvicorn inicia su ejecución, cuando se dispara el contexto asíncrono de lifespan, entonces el archivo .pkl del modelo se deserializa mediante scikit-learn y queda inyectado globalmente en la memoria de la aplicación.
2. **Escenario 2 (archivo de modelo no encontrado o corrupto):** Dado que el archivo del modelo predictivo no existe en la ruta especificada o está corrupto, cuando se intenta cargar en el arranque, entonces el servicio registra un error crítico y lanza un RuntimeError interrumpiendo el inicio de FastAPI, evitando así operar en un estado inconsistente sin IA.
3. **Escenario 3 (consistencia con la versión registrada):** Dado que el modelo cargado debe corresponder a la versión activa aprobada, cuando finaliza la carga, entonces el backend verifica el hash SHA-256 del archivo contra el registro de versionado operativo (HU-46); si no coinciden, registra un error crítico y no sirve clasificaciones.

---

### HU-17 · Extracción de Características (Feature Engineering)

- **Como** farmacéutico responsable
- **Quiero** que el sistema calcule métricas derivadas como gradiente térmico y derivadas de tiempo
- **Para** alimentar correctamente el modelo Random Forest que clasifica el riesgo
- **Prioridad:** Alta · **Sprint:** 5 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (cálculo completo):** Dado que se recibe un modelo de datos validado por Pydantic, cuando se invoca el servicio de extracción de dominio, entonces se calcula la matriz completa de variables operativas (ej. delta térmico) necesarias para alimentar al clasificador.
2. **Escenario 2 (historial insuficiente — sin imputación de valores no entrenados):** Dado que el nodo acaba de conectarse y no hay suficiente historial térmico reciente para calcular una variable derivada indispensable (ej. tendencia térmica móvil), cuando se intenta construir la matriz, entonces el sistema **no inyecta valores neutros por defecto** (pues producirían *training-serving skew* frente a un modelo que no fue entrenado con esos valores); en su lugar, marca explícitamente el estado como `no_clasificable` y la lectura se evalúa únicamente por la regla directa de rango térmico 2–8 °C (HU-21).
3. **Escenario 3 (valores atípicos / outliers):** Dado que una variable de entrada (ej. 80 °C en el refrigerador) está fuera de los rangos físicamente plausibles para una operación normal, cuando se construye la matriz de características, entonces el registro se marca preventivamente como anomalía de hardware y se excluye de la inferencia estándar, aplicándose la regla directa de rango térmico.

---

### HU-18 · Inferencia de Riesgo Térmico (Random Forest)

- **Como** farmacéutico responsable
- **Quiero** que el sistema evalúe la matriz de características operativas
- **Para** clasificar el estado térmico en Normal, Riesgo Preventivo o Excursión Crítica
- **Prioridad:** Alta · **Sprint:** 5 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (clasificación exitosa en RAM):** Dado que se ingresan las features correctamente desde el módulo de dominio, cuando el modelo Random Forest (cargado previamente vía joblib) procesa la matriz, entonces retorna una de las 3 clases junto con la probabilidad (confianza) de la predicción en milisegundos, y el resultado queda asociado a la versión del modelo utilizada (HU-46/HU-47).
2. **Escenario 2 (entrada incompleta):** Dado que la matriz contiene valores faltantes no imputables (ej. sensor desconectado), cuando se intenta inferir, entonces el sistema evita generar una clasificación no confiable, registra el estado como "no_clasificable" y delega la evaluación a la regla directa de rango térmico 2–8 °C.
3. **Escenario 3 (baja confianza en la predicción):** Dado que la probabilidad de la clase predicha está por debajo del umbral mínimo configurado, cuando se evalúa el resultado, entonces la lectura se marca adicionalmente con un flag de revisión manual, sin impedir el registro de la clase de mayor probabilidad.

---

### HU-19 · Emisión de Eventos SSE (Flujo Unidireccional)

- **Como** técnico de farmacia
- **Quiero** ver en tiempo real la lectura procesada y la clasificación de riesgo en el dashboard
- **Para** monitorear incidentes sin recargar la interfaz
- **Prioridad:** Media · **Sprint:** 5 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (cliente conectado):** Dado que se persiste la lectura y su clasificación, cuando existen clientes web activos (técnicos/farmacéuticos), entonces el servidor despacha el evento JSON vía SSE a todos los suscriptores al instante.
2. **Escenario 2 (sin clientes conectados):** Dado que no hay clientes suscritos al canal SSE al momento de procesar la telemetría, cuando se intenta emitir el evento, entonces el despacho se omite para ahorrar recursos, pero el dato no se pierde ya que queda disponible en PostgreSQL para consultas históricas.
3. **Escenario 3 (desconexión abrupta del cliente):** Dado que un cliente cierra el navegador o pierde red abruptamente, cuando el servidor FastAPI detecta la ruptura del canal SSE, entonces libera los recursos de memoria y cierra el generador asíncrono sin afectar la transmisión a los demás clientes conectados.

---

### HU-20 · Generación de Alerta Preventiva

- **Como** farmacéutico responsable
- **Quiero** que el sistema genere una alerta cuando el modelo detecte Riesgo Preventivo
- **Para** actuar antes de que se rompa la cadena de frío
- **Prioridad:** Media · **Sprint:** 7 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (alerta generada):** Dado que la IA retorna la clase riesgo_preventivo, cuando el motor de reglas procesa la salida, entonces se inserta un nuevo registro en la tabla thermal_alerts de PostgreSQL con nivel, timestamp y dispositivo, y se emite un aviso urgente por el canal SSE.
2. **Escenario 2 (riesgo preventivo sostenido):** Dado que ya existe una alerta preventiva activa y sin atender para el mismo dispositivo, cuando llega otra clasificación de riesgo_preventivo, entonces se actualiza la fecha de la alerta existente en lugar de duplicar registros, evitando la saturación (spam) en el dashboard.
3. **Escenario 3 (resolución automática):** Dado que el dispositivo vuelve a clasificarse como normal de forma sostenida tras una alerta preventiva, cuando se confirma esta estabilización, entonces la alerta activa se marca como "resuelta automáticamente" en la base de datos.

---

### HU-21 · Generación de Alerta Crítica (Regla directa 2–8 °C)

- **Como** farmacéutico responsable
- **Quiero** que el sistema registre una excursión térmica crítica si la temperatura sale del rango objetivo de 2 °C a 8 °C definido por la investigación
- **Para** ejecutar acciones correctivas auditables conforme a las Buenas Prácticas de Almacenamiento
- **Prioridad:** Alta · **Sprint:** 7 · **Story Points:** 3 · **Estado:** No se ha iniciado

> **Nota:** el rango **2 °C a 8 °C** es el rango objetivo de conservación definido por la investigación y opera como regla crítica directa e independiente de la predicción del modelo de IA.

**Criterios de aceptación:**

1. **Escenario 1 (excursión detectada por regla directa):** Dado que la lectura térmica es < 2 °C o > 8 °C, cuando el motor de reglas la evalúa, entonces se marca y persiste la alerta como excursion_critica de inmediato, independientemente del resultado predictivo del modelo de IA, despachando la alarma visual al frontend.
2. **Escenario 2 (retorno al rango seguro):** Dado que existe una excursión crítica activa y la siguiente lectura vuelve al rango de 2–8 °C, cuando se procesa, entonces se registra el evento de retorno al rango seguro, pero no se elimina ni se cierra la excursión original hasta que reciba una justificación manual del farmacéutico.
3. **Escenario 3 (excursión sin acción correctiva):** Dado que una excursión crítica permanece sin una acción correctiva asociada (ej. traslado de productos) tras un tiempo prudencial, cuando se evalúa su estado, entonces queda anclada como "Pendiente de Acción" forzando al dashboard a destacarla en rojo hasta su resolución.

---

### HU-46 · Gestión y Versionado Operativo del Modelo IA

- **Como** administrador del sistema
- **Quiero** registrar en base de datos la versión activa del modelo Random Forest, su hash SHA-256 y la versión del esquema de features, asegurando que solo modelos aprobados clasifiquen en producción
- **Para** garantizar el gobierno y la observabilidad del modelo de IA, evitando que una versión no autorizada clasifique el riesgo térmico
- **Prioridad:** Alta · **Sprint:** 5 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (registro de una nueva versión aprobada):** Dado que se dispone de un modelo entrenado y aprobado, cuando el Administrador registra la nueva versión, entonces el backend almacena: identificador de versión, hash SHA-256 del archivo del modelo, versión del esquema de features asociado y fecha de aprobación, y la marca como versión activa de producción.
2. **Escenario 2 (bloqueo de modelos no aprobados):** Dado que se intenta activar un modelo cuyo registro de aprobación no existe, cuando el sistema valida la activación, entonces rechaza la operación, mantiene activa la versión aprobada vigente y registra el evento en la bitácora.
3. **Escenario 3 (consistencia entre modelo cargado y modelo registrado):** Dado que el servicio FastAPI carga el modelo en memoria (HU-16), cuando finaliza el arranque, entonces el backend verifica que el hash SHA-256 del archivo cargado coincide con el hash registrado para la versión activa; si difiere, registra un error crítico y no sirve clasificaciones.
4. **Escenario 4 (historial de cambios de versión):** Dado que se cambia la versión activa, cuando se confirma el cambio, entonces se registra un evento append-only en traceability_records con versión anterior, versión nueva, usuario que realizó el cambio y timestamp, encadenado con hash SHA-256.

---

### HU-47 · Consulta de Trazabilidad de la Inferencia IA

- **Como** responsable de auditoría / farmacéutico
- **Quiero** auditar qué versión específica del modelo y qué probabilidades generaron la clasificación de una lectura térmica determinada
- **Para** explicar y verificar las decisiones de la IA ante auditorías internas o consultas de la autoridad sanitaria
- **Prioridad:** Alta · **Sprint:** 6 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (consulta de la inferencia asociada a una lectura):** Dado que el auditor selecciona una lectura térmica del historial, cuando abre el detalle de la inferencia, entonces el sistema muestra: versión del modelo utilizada, vector de características de entrada, probabilidades por clase (normal, riesgo_preventivo, excursion_critica), clasificación final y timestamp de la inferencia.
2. **Escenario 2 (coherencia con el versionado del modelo):** Dado que una lectura fue clasificada con la versión X del modelo, cuando la auditoría cruza el dato con el versionado operativo (HU-46), entonces se confirma que la versión X era la versión activa y aprobada en el momento de la clasificación.
3. **Escenario 3 (acceso de solo lectura):** Dado que el auditor consulta la trazabilidad de la inferencia, cuando intenta modificar cualquier campo del registro, entonces el sistema lo impide (modo solo lectura) y registra el intento en la bitácora de auditoría.

---

# Épica EP04 · Persistencia y Trazabilidad Verificable

---

### HU-22 · Consolidación de Historial en Capa de Infraestructura

- **Como** farmacéutico responsable
- **Quiero** que la telemetría se almacene en PostgreSQL usando JSONB
- **Para** conservar un histórico térmico flexible junto con su clasificación asociada
- **Prioridad:** Alta · **Sprint:** 4 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (persistencia asíncrona exitosa):** Dado que el payload es válido y el riesgo térmico ya fue clasificado por la IA, cuando se mapea el objeto al ORM, entonces se persiste de forma asíncrona en la tabla thermal_readings, incluyendo el campo JSONB con la telemetría completa de los sensores.
2. **Escenario 2 (fallo temporal de conexión a BD en Railway):** Dado que la instancia de PostgreSQL en la nube (Railway) no está disponible temporalmente al momento de persistir, cuando ocurre el intento de guardado, entonces el repositorio reintenta la conexión un número limitado de veces (política de reintentos) antes de registrar el fallo en el log, sin perder el mensaje validado en memoria.
3. **Escenario 3 (concurrencia de escrituras IoT):** Dado que llegan múltiples lecturas casi simultáneas desde distintos dispositivos (refrigeradores/farmacias), cuando el backend invoca la capa de persistencia, entonces cada registro se guarda en la base de datos de forma asíncrona e independiente, sin bloquear el event loop del servidor ni retrasar la ingesta de los demás dispositivos.

---

### HU-24 · Cálculo de Hash Criptográfico (SHA-256)

- **Como** responsable de auditoría
- **Quiero** que el sistema calcule un hash SHA-256 por cada evento térmico u operativo
- **Para** asegurar su integridad verificable sin depender de infraestructuras blockchain de terceros
- **Prioridad:** Alta · **Sprint:** 4 · **Story Points:** 5 · **Estado:** No se ha iniciado

> **Nota de alcance:** el hash encadenado **detecta manipulaciones retrospectivas** (cualquier alteración posterior cambia el hash recalculado), pero **no impide físicamente** que un administrador con acceso directo a la base de datos modifique una tabla; por ello se complementa con comportamiento append-only, control de acceso (HU-41) y bitácora de auditoría (HU-42).

**Criterios de aceptación:**

1. **Escenario 1 (cálculo exitoso con insumos exactos):** Dado que ocurre un evento relevante (lectura, alerta o acción correctiva), cuando se genera su registro en el backend, entonces la librería hashlib de Python calcula una firma SHA-256 de 256 bits concatenando estrictamente los campos (previous_hash + payload + timestamp).
2. **Escenario 2 (campo vacío o nulo):** Dado que algún campo del evento llega vacío o nulo de forma inesperada, cuando se calcula el hash criptográfico, entonces el campo se normaliza a un valor representable estándar (ej. string vacío o "null") antes del cálculo, evitando que un campo ausente rompa la reproducibilidad de la firma.
3. **Escenario 3 (determinismo matemático):** Dado que se recalcula el hash de un evento ya almacenado ingresando exactamente los mismos datos originales, cuando se compara con la firma original, entonces ambos valores alfanuméricos coinciden con exactitud matemática, demostrando el determinismo del algoritmo.

---

### HU-25 · Encadenamiento con Previous Hash (Ledger Centralizado)

- **Como** responsable de auditoría
- **Quiero** que cada nuevo registro incluya el hash del evento anterior en traceability_records
- **Para** formar un registro encadenado cuya integridad sea verificable secuencialmente
- **Prioridad:** Alta · **Sprint:** 4 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (encadenamiento normal secuencial):** Dado que se guarda un nuevo evento validado, cuando se calcula su firma hash, entonces el algoritmo concatena el valor de la columna previous_hash del registro N-1 con los datos actuales, garantizando la dependencia temporal.
2. **Escenario 2 (primer evento de la cadena / Génesis):** Dado que el sistema se inicializa y no existe un registro anterior en la tabla, cuando se calcula su hash, entonces se utiliza un valor semilla inicial predefinido de 256 bits (bloque génesis) en lugar de un previous_hash real.
3. **Escenario 3 (resolución de inserciones concurrentes):** Dado que dos o más eventos (de distintos refrigeradores) intentan registrarse casi simultáneamente en PostgreSQL, cuando se determina el orden de la cadena por transacciones atómicas, entonces el sistema garantiza que cada nuevo hash referencie de forma única y consistente al hash inmediatamente anterior consolidado, evitando bifurcaciones de la cadena.

---

### HU-26 · Verificación O(n) de Cadena de Integridad

- **Como** Responsable de Auditoría o Fiscalización interna
- **Quiero** solicitar una verificación algorítmica en bloque de toda la tabla de trazabilidad
- **Para** detectar manipulaciones en los registros y presentar evidencias confiables ante inspecciones de DIGEMID
- **Prioridad:** Alta · **Sprint:** 6 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (cadena íntegra confirmada):** Dado que el auditor solicita la verificación en el dashboard, cuando el backend recalcula los hashes secuencialmente (complejidad O(n)), entonces el sistema confirma que cada firma coincide con la almacenada y retorna un estado global "Íntegro / Válido".
2. **Escenario 2 (detección de registro alterado):** Dado que un registro fue modificado a nivel de base de datos tras su creación, cuando se recalcula su hash, entonces el valor resultante no coincide con el almacenado y la cadena se rompe, señalando el ID del bloque específico como "Corrupto", sin detener la evaluación de los registros restantes.
3. **Escenario 3 (interrupción del proceso pesado):** Dado que la verificación O(n) se interrumpe (por timeout o cierre de conexión) antes de recorrer toda la extensa tabla, cuando el auditor reintenta la acción, entonces el proceso puede reanudarse o reiniciarse de forma controlada sin bloquear otras operaciones concurrentes de lectura en la base de datos.

---

### HU-27 · Persistencia de Acción Correctiva

- **Como** Químico Farmacéutico (Regente)
- **Quiero** registrar la acción correctiva aplicada tras una alerta
- **Para** cumplir con el soporte documental exigido por la normativa de Buenas Prácticas de Almacenamiento de DIGEMID
- **Prioridad:** Media · **Sprint:** 6 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (registro exitoso):** Dado que hay una alerta activa en el dashboard, cuando el usuario ingresa la justificación y la acción tomada, entonces el sistema extrae su identidad desde el token JWT, asocia el evento al ID de la alerta y la marca como atendida.
2. **Escenario 2 (alerta ya resuelta):** Dado que el usuario intenta registrar una acción sobre una alerta que acaba de ser cerrada por otro compañero, cuando envía la solicitud, entonces el sistema la rechaza mediante concurrencia e informa visualmente que la alerta ya no está vigente.
3. **Escenario 3 (campos incompletos):** Dado que no se completa la descripción mínima requerida de la acción, cuando intenta enviar el formulario, entonces el sistema bloquea el envío en el frontend y solicita completar la información obligatoria.

---

### HU-28 · Auditoría Automática de Acciones Correctivas

- **Como** responsable de auditoría
- **Quiero** que el sistema calcule hash SHA-256 también sobre las acciones manuales de los usuarios
- **Para** detectar cambios posteriores en las justificaciones registradas
- **Prioridad:** Media · **Sprint:** 6 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (acción trazada criptográficamente):** Dado que se envía una acción correctiva desde el dashboard, cuando se persiste en la tabla traceability_records, entonces se genera un evento encadenado tipo ACCION_CORRECTIVA con su propio hash SHA-256, utilizando el previous_hash del último movimiento del sistema.
2. **Escenario 2 (intento de edición posterior / Append-Only):** Dado que un usuario intenta modificar una justificación ya registrada en la base de datos, cuando se procesa la solicitud, entonces la base de datos rechaza la alteración del registro original (comportamiento append-only), exigiendo que se cree un nuevo evento que referencie la corrección.
3. **Escenario 3 (verificación cruzada):** Dado que el auditor verifica la cadena completa, cuando se incluyen eventos de tipo ACCION_CORRECTIVA, entonces el sistema los valida secuencialmente con el mismo mecanismo algorítmico que las lecturas y alertas, garantizando la integridad verificable de todo el ciclo de vida del dato.

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
- **Prioridad:** Media · **Sprint:** 7 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (registro trazable y auditable):** Dado que el Farmacéutico ingresa la fecha de última calibración y el número de certificado de un nodo, cuando guarda el registro, entonces el sistema lo almacena asociado al device_id y genera un evento en la tabla de trazabilidad con su respectiva firma SHA-256 encadenada, para detectar alteraciones futuras.
2. **Escenario 2 (vencimiento próximo):** Dado que faltan 30 días o menos para el vencimiento anual del certificado, cuando el backend evalúa diariamente las fechas registradas, entonces genera un "Riesgo Operativo" visible en el dashboard, con un color e iconografía distintivos para no confundirlo con una alerta de excursión térmica.
3. **Escenario 3 (certificado vencido):** Dado que la fecha de vencimiento superó el límite sin que se haya registrado una recalibración, cuando se evalúa el estado del sensor, entonces el riesgo operativo escala a nivel crítico y las lecturas de ese dispositivo se marcan visualmente con una advertencia de "calibración vencida" hasta que se actualice la documentación.

---

### HU-37 · Verificación de Integridad de Telemetría por Dispositivo y Periodo

> **Historia reenfocada:** la versión original (check-list BPA completo como autoinspección digital) se descartó por desviación de producto. El slot HU-37 se reescribió como una historia de **trazabilidad verificable** alineada a EP04, centrada en la integridad de la telemetría.

- **Como** farmacéutico responsable
- **Quiero** solicitar la verificación de integridad de la telemetría térmica de un dispositivo en un periodo determinado y obtener el estado de cada lectura
- **Para** confirmar que los registros de cadena de frío no han sido alterados y contar con evidencia verificable acotada a la consulta
- **Prioridad:** Media · **Sprint:** 9 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (verificación acotada por dispositivo y periodo):** Dado que el farmacéutico selecciona un dispositivo y un rango temporal en el dashboard, cuando ejecuta la verificación, entonces el backend (FastAPI) recalcula los hashes SHA-256 de los eventos térmicos de ese rango y retorna el estado de integridad (íntegro / alterado) de cada lectura.
2. **Escenario 2 (detección de lectura alterada en el rango):** Dado que alguna lectura del rango seleccionado fue modificada tras su registro, cuando se recalcula su hash, entonces el sistema la señala como "Corrupta" con su ID y timestamp, y continúa evaluando el resto de lecturas del rango.
3. **Escenario 3 (constancia verificable de la revisión):** Dado que la verificación concluye, cuando se confirma el resultado, entonces se registra un evento append-only tipo VERIFICACION_INTEGRIDAD en traceability_records con dispositivo, periodo, resultado global y usuario solicitante, encadenado con hash SHA-256.

---

### HU-49 · Historial Auditable de Cambios de Configuración del Hardware

- **Como** responsable de auditoría
- **Quiero** consultar la bitácora de reemplazos de sensores, cambios de umbral y mantenimiento del hardware
- **Para** auditar la evolución de la configuración del monitoreo sin alterar los registros pasados
- **Prioridad:** Media · **Sprint:** 9 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (consulta de la bitácora de hardware):** Dado que el auditor accede al módulo de cambios de configuración, cuando filtra por dispositivo, fecha o tipo de evento, entonces el sistema muestra los eventos de reemplazo de sensor, cambio de umbral y mantenimiento registrados en traceability_records, en modo solo lectura.
2. **Escenario 2 (registro de cada cambio de configuración):** Dado que el Administrador modifica un parámetro de configuración del hardware (ej. umbral de alerta térmica o datos de calibración, HU-30), cuando confirma el cambio, entonces se registra un evento con valor anterior, valor nuevo, usuario responsable y timestamp, encadenado con hash SHA-256.
3. **Escenario 3 (registros pasados inalterados):** Dado que se consulta un cambio de configuración histórico, cuando se revisa el historial, entonces la telemetría registrada antes del cambio conserva sus valores y hashes originales; el evento de cambio solo documenta desde cuándo aplica la nueva configuración.

---

# Épica EP05 · Monitoreo Web, Alertas y Reportes

---

### HU-23 · Notificaciones Asíncronas de Excursión Crítica

- **Como** farmacéutico responsable
- **Quiero** recibir una notificación automática por correo electrónico o Telegram cuando se genere una excursión crítica
- **Para** enterarme del incidente de forma inmediata aunque no tenga el dashboard web abierto
- **Prioridad:** Alta · **Sprint:** 7 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (excursión fuera de horario):** Dado que se genera una excursión crítica térmica, cuando el backend (FastAPI) registra la alerta, entonces despacha una tarea en segundo plano (Background Task) que envía automáticamente una notificación vía SMTP o API de Telegram al responsable designado, sin bloquear el hilo principal.
2. **Escenario 2 (fallo en el servicio de terceros):** Dado que el servicio de envío externo (SMTP/Telegram) no responde o presenta latencia, cuando se intenta notificar, entonces el sistema reintenta un número limitado de veces y registra el fallo en el log, asegurando que la persistencia de la alerta en PostgreSQL no se vea afectada.
3. **Escenario 3 (Throttling / prevención de spam):** Dado que se generan múltiples lecturas de excursión crítica seguidas para el mismo dispositivo, cuando el sistema evalúa el envío de mensajes, entonces aplica una política de limitación de frecuencia (throttling) para enviar un solo resumen, evitando saturar al responsable con notificaciones duplicadas del mismo incidente.

---

### HU-31 · Visualización de Tendencia Térmica

- **Como** técnico de farmacia
- **Quiero** visualizar una gráfica de temperatura con umbrales fijos de 2 °C a 8 °C
- **Para** evaluar rápidamente la tendencia térmica del refrigerador
- **Prioridad:** Alta · **Sprint:** 8 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (carga con datos):** Dado que abro el dashboard de un refrigerador y existen lecturas históricas en la base de datos, cuando el componente de React 19 carga la instancia de ECharts, entonces se renderiza la línea histórica superpuesta sobre zonas de colores (rojo/verde) que delimitan visualmente el rango objetivo de 2–8 °C.
2. **Escenario 2 (sin datos históricos):** Dado que un dispositivo IoT recién registrado no ha transmitido lecturas aún, cuando se abre su vista de gráfica, entonces el sistema muestra un "estado vacío" (empty state) informativo indicando que está a la espera de datos, en lugar de un lienzo en blanco confuso.
3. **Escenario 3 (gran volumen de puntos):** Dado que el rango de fechas seleccionado contiene miles de lecturas continuas, cuando ECharts renderiza el gráfico, entonces se aplica muestreo o agregación visual (data downsampling) de forma automática para mantener la fluidez de la interfaz sin perder la forma real de la tendencia térmica.

---

### HU-32 · Actualización en Tiempo Real del Dashboard

- **Como** técnico de farmacia
- **Quiero** que el dashboard reciba actualizaciones del backend mediante Server-Sent Events (SSE)
- **Para** ver el último punto de la gráfica y los KPI en tiempo real
- **Prioridad:** Alta · **Sprint:** 8 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (actualización asíncrona en tiempo real):** Dado que la conexión SSE hacia la API FastAPI está activa, cuando llega un nuevo evento JSON con telemetría fresca, entonces el gestor de estado global del frontend actualiza automáticamente los componentes dependientes sin necesidad de recargar la página.
2. **Escenario 2 (pérdida de conexión SSE):** Dado que la red local de la farmacia o el enlace con el servidor falla, cuando el cliente React detecta la ruptura del flujo continuo, entonces muestra un indicador visual en amarillo de "reconectando..." e intenta restablecer el canal SSE automáticamente mediante un backoff de reintentos.
3. **Escenario 3 (múltiples pestañas):** Dado que el usuario tiene el dashboard abierto en más de una pestaña o dispositivo, cuando el servidor despacha un evento SSE, entonces todas las instancias del cliente reflejan la actualización instantáneamente de forma independiente y concurrente.

---

### HU-33 · Tarjetas de KPI (Estado Actual y Tolerancia a Fallo)

- **Como** técnico de farmacia
- **Quiero** visualizar el último valor de temperatura (DS18B20) y humedad ambiental (SHT31) en tarjetas KPI
- **Para** obtener una lectura rápida del estado del equipo a un solo vistazo
- **Prioridad:** Media · **Sprint:** 8 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (datos disponibles y vigentes):** Dado que el dashboard web se renderiza y la caché cuenta con datos válidos, cuando se cargan los componentes de las tarjetas KPI, entonces se muestran los valores numéricos actuales exactos en °C y % HR.
2. **Escenario 2 (última lectura desactualizada por red):** Dado que el último timestamp recibido supera el umbral máximo esperado de inactividad (ej. más de 5 minutos sin datos del edge), cuando se muestran las tarjetas, entonces se oscurecen ligeramente o incluyen un ícono de advertencia indicando la antigüedad del dato.
3. **Escenario 3 (lectura en error de sensor):** Dado que el flujo de datos indica que un sensor en particular está fallando (valor null), cuando React intenta renderizar su KPI correspondiente, entonces muestra el texto "Sin lectura disponible" o "Falla de sensor" en lugar de mostrar información potencialmente engañosa que confunda al personal.

---

### HU-34 · Semáforo de Riesgo Efectivo (Regla 2–8 °C + Random Forest)

- **Como** farmacéutico
- **Quiero** ver un semáforo visual (Verde, Amarillo, Rojo) en la tarjeta del equipo que refleje el riesgo efectivo, combinando la regla crítica directa de 2–8 °C con la predicción preventiva del Random Forest
- **Para** identificar de inmediato el nivel de atención clínica o técnica requerido
- **Prioridad:** Alta · **Sprint:** 8 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (estado normal y riesgo preventivo):** Dado que el dashboard recibe una actualización vía SSE, cuando se evalúa el riesgo efectivo, entonces el semáforo se muestra en Verde si la lectura está dentro de 2–8 °C y la clase IA es normal; y muta dinámicamente a Amarillo si el modelo predice riesgo_preventivo aunque la temperatura siga dentro del rango.
2. **Escenario 2 (excursión crítica por regla directa):** Dado que la lectura sale del rango objetivo de 2–8 °C (regla crítica directa, HU-21), cuando la interfaz (UI) se actualiza, entonces el semáforo cambia a Rojo con alerta visual independientemente de la predicción del modelo de IA, y permanece en este estado hasta que la temperatura retorne al rango de forma sostenida.
3. **Escenario 3 (clasificación no disponible por fallo):** Dado que la última lectura fue rechazada o quedó catalogada como "no_clasificable" (ej. datos insuficientes), cuando React intenta actualizar el semáforo, entonces se muestra un estado neutro (ej. color gris) y la evaluación se apoya únicamente en la regla directa de 2–8 °C, evitando proporcionar una falsa sensación de seguridad.

---

### HU-35 · Notificación UI de Puerta Abierta (MC-38)

- **Como** personal técnico de farmacia
- **Quiero** visualizar una alerta intermitente en la interfaz si el refrigerador permanece abierto
- **Para** cerrarlo rápidamente y evitar fuga térmica y condensación
- **Prioridad:** Media · **Sprint:** 8 · **Story Points:** 3 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (apertura en tiempo real):** Dado que el sensor magnético (MC-38) registra apertura y envía el estado true empaquetado en el flujo SSE, cuando la aplicación web procesa el evento, entonces renderiza un ícono de puerta abierta en rojo con una animación intermitente (pulsing) en la tarjeta del dispositivo.
2. **Escenario 2 (cierre confirmado):** Dado que la puerta física se cierra y el estado del sensor vuelve a false, cuando el frontend recibe la actualización asíncrona, entonces la notificación de alerta y el ícono desaparecen inmediatamente sin necesidad de recargar la página.
3. **Escenario 3 (apertura prolongada crítica):** Dado que la puerta permanece abierta más allá del tiempo máximo configurado de seguridad (ej. 2 minutos), cuando el estado global detecta esta prolongación, entonces la alerta visual roja se acompaña de un contador que muestra el tiempo exacto transcurrido desde la apertura.

---

### HU-36 · Panel de Filtros de Historial y Auditoría

- **Como** Químico Farmacéutico (Regente)
- **Quiero** consultar y filtrar los datos históricos por fecha y hora
- **Para** analizar eventos retrospectivos, excursiones pasadas y generar evidencia para auditorías
- **Prioridad:** Media · **Sprint:** 9 · **Story Points:** 5 · **Estado:** No se ha iniciado

> **Nota de arquitectura:** la consulta sigue el patrón en capas `React (Frontend) → REST API FastAPI (Backend) → PostgreSQL (BD)`. El frontend nunca consulta la base de datos directamente.

**Criterios de aceptación:**

1. **Escenario 1 (filtro aplicado exitosamente):** Dado que entro al módulo de historial, cuando selecciono un rango temporal en el DatePicker (basado en shadcn/ui) y ejecuto la búsqueda, entonces el frontend lanza una petición REST a la API de FastAPI, que es la que consulta PostgreSQL; la respuesta alimenta tanto la gráfica de Apache ECharts como la tabla de datos con la información exacta de ese periodo.
2. **Escenario 2 (rango temporal sin datos):** Dado que el rango de fechas seleccionado no contiene telemetría registrada (ej. equipo apagado), cuando se responde la consulta, entonces el sistema captura la matriz vacía y muestra un componente ilustrativo con el mensaje "Sin datos registrados en el rango seleccionado".
3. **Escenario 3 (validación de rango inválido):** Dado que selecciono un rango incongruente donde la fecha de fin es cronológicamente anterior a la fecha de inicio, cuando intento aplicar el filtro, entonces la validación del formulario (Frontend) bloquea la consulta y muestra un mensaje de error indicando que se debe ajustar la selección.

---

### HU-38 · Exportación de Reporte PDF (Evidencia Sanitaria)

- **Como** responsable de auditoría o auditor interno
- **Quiero** generar y exportar un reporte en formato PDF del historial térmico y las alertas
- **Para** adjuntarlo como evidencia digital verificable del cumplimiento de la cadena de frío y entregarlo a la autoridad de inspección sanitaria (DIRIS/DIGEMID) cuando lo requiera
- **Prioridad:** Media · **Sprint:** 9 · **Story Points:** 8 · **Estado:** No se ha iniciado

> **Nota de roles:** el inspector sanitario (DIRIS/DIGEMID) no necesita cuenta ni acceso al sistema; el PDF es emitido y entregado por el Químico Farmacéutico o el auditor interno.

**Criterios de aceptación:**

1. **Escenario 1 (exportación exitosa con trazabilidad Hash):** Dado que visualizo un rango de fechas filtrado en el dashboard, cuando presiono el botón "Exportar PDF", entonces el navegador descarga un documento formal que incluye el resumen gráfico de ECharts, la tabla de telemetría y las firmas criptográficas SHA-256 asociadas a cada registro para respaldar su integridad verificable.
2. **Escenario 2 (rango temporal muy extenso):** Dado que el rango de fechas solicitado abarca un gran volumen de datos (ej. todo un mes de registros continuos), cuando el frontend procesa la generación del reporte, entonces el sistema pagina adecuadamente las tablas dentro del PDF y resume los KPIs sin omitir el periodo crítico solicitado.
3. **Escenario 3 (error en la generación del documento):** Dado que ocurre un error inesperado al intentar compilar o descargar el archivo PDF por falta de memoria o red, cuando el sistema detecta la falla, entonces muestra un mensaje de error claro al usuario (ej. Toast notification) y permite reintentar la descarga sin perder los filtros de fecha previamente aplicados.

---

### HU-48 · Visualización del Estado de Sincronización y Brechas Temporales

- **Como** farmacéutico
- **Quiero** ver en el dashboard si un ESP32 tiene datos pendientes en LittleFS o si ocurrió una pérdida irrecuperable de datos por buffer lleno tras un corte prolongado
- **Para** conocer la completitud del registro térmico y actuar frente a brechas de información
- **Prioridad:** Media · **Sprint:** 9 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (datos pendientes de sincronización):** Dado que un ESP32 mantiene lecturas acumuladas en LittleFS pendientes de sincronizar (HU-06/HU-07), cuando el dashboard muestra el estado del dispositivo, entonces presenta un indicador de "datos pendientes" con el número de registros retenidos y el rango temporal que cubren.
2. **Escenario 2 (brecha irrecuperable por buffer lleno):** Dado que el nodo descartó bloques antiguos por saturación de LittleFS durante una desconexión prolongada (HU-06, Escenario 2), cuando el backend procesa el evento de saturación, entonces el dashboard muestra una brecha temporal señalizada en la gráfica con la causa "pérdida de datos por desconexión prolongada" y el periodo afectado.
3. **Escenario 3 (sincronización completada):** Dado que el nodo sincroniza todos los bloques pendientes con acuse lógico del backend (HU-07), cuando el dashboard se actualiza, entonces el indicador de "datos pendientes" desaparece y la serie gráfica queda completa.

---

# Épica EP06 · Autenticación, Autorización y Seguridad Transversal

---

### HU-39 · Autenticación de Usuarios (Login seguro)

- **Como** usuario del sistema (Técnico/Farmacéutico)
- **Quiero** ingresar mi correo y contraseña de forma segura
- **Para** obtener acceso al dashboard cerrado conforme a OWASP Web Security Testing Guide v4.2
- **Prioridad:** Alta · **Sprint:** 1 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (credenciales correctas):** Dado que envío credenciales válidas al endpoint /api/login, cuando el backend (FastAPI) las verifica contra el hash bcrypt almacenado en PostgreSQL, entonces recibo un token JWT firmado criptográficamente que incluye mi rol operativo autorizado.
2. **Escenario 2 (mitigación de enumeración de usuarios):** Dado que envío una contraseña incorrecta o un correo inexistente, cuando el sistema procesa la validación, entonces se rechaza el acceso con un mensaje genérico (ej. "Credenciales inválidas") sin revelar si el correo existe en la base de datos, y se registra el evento en la bitácora.
3. **Escenario 3 (protección contra fuerza bruta):** Dado que se superan varios intentos fallidos consecutivos para la misma cuenta en una ventana de tiempo corta, cuando ocurre el siguiente intento de login, entonces se aplica una restricción temporal (rate limiting / bloqueo de cuenta) antes de permitir un nuevo intento, mitigando ataques automatizados.

---

### HU-40 · Gestión Segura de JWT en Memoria

- **Como** usuario autenticado
- **Quiero** que mi token JWT se almacene solo en la memoria volátil del navegador y nunca en localStorage
- **Para** reducir el riesgo de robo de sesión por ataques XSS
- **Prioridad:** Alta · **Sprint:** 2 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (sesión activa y segura):** Dado que me autentico correctamente en la plataforma, cuando navego por la SPA (Single Page Application), entonces el token JWT vive únicamente en el gestor de estado de la aplicación (ej. Zustand o React Context), sin persistir en localStorage ni sessionStorage.
2. **Escenario 2 (cierre de pestaña o navegador):** Dado que cierro la pestaña actual o el navegador por completo, cuando intento volver a acceder a la URL protegida del dashboard, entonces el token ya no está disponible en memoria y el sistema me redirige obligatoriamente a la vista de login.
3. **Escenario 3 (recarga manual defensiva):** Dado que recargo manualmente la página (F5) estando autenticado, cuando la aplicación React se reinicializa, entonces se me solicita iniciar sesión nuevamente, ya que el estado en memoria se purga de forma esperada garantizando que no queden rastros del token en el cliente.

---

### HU-41 · Autorización por Roles (RBAC y Principio de Mínimo Privilegio)

- **Como** Administrador del sistema
- **Quiero** restringir el acceso a rutas visuales y funciones de la API según el rol del usuario autenticado (Técnico, Farmacéutico, Auditor)
- **Para** aplicar el principio de mínimo privilegio y cumplir con las normativas de seguridad OWASP
- **Prioridad:** Alta · **Sprint:** 10 · **Story Points:** 8 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (acceso autorizado):** Dado que un usuario con rol de Administrador navega a la ruta de gestión de usuarios, cuando React Router en el frontend y FastAPI en el backend verifican el claim de rol dentro del JWT, entonces el sistema permite renderizar la vista y retornar los datos solicitados.
2. **Escenario 2 (acceso no autorizado y protección de API):** Dado que un Técnico intenta acceder a la ruta /usuarios o invoca directamente el endpoint protegido, cuando se verifica el rol del JWT, entonces React Router bloquea la vista con "Acceso Denegado" y FastAPI rechaza la petición con un código HTTP 403 (Forbidden), sin exponer información sensible.
3. **Escenario 3 (cambio de rol en sesión activa):** Dado que los privilegios de un usuario son modificados por un Administrador mientras el usuario tiene una sesión activa, cuando el usuario realiza su siguiente acción y el token se refresca o revalida, entonces el sistema aplica inmediatamente los permisos actualizados, invalidando los privilegios originales del inicio de sesión.

---

### HU-42 · Bitácora Append-Only de Auditoría (Audit Logs con SHA-256)

- **Como** Responsable de Auditoría
- **Quiero** tener un registro automatizado con comportamiento append-only de cada inicio de sesión, cambio de estado y exportación de datos
- **Para** cumplir con el seguimiento estricto de identidades ante inspecciones sanitarias
- **Prioridad:** Media · **Sprint:** 6 · **Story Points:** 8 · **Estado:** No se ha iniciado

> **Nota de alcance:** la bitácora append-only con hash encadenado **detecta** alteraciones retrospectivas mediante verificación (HU-26); no impide físicamente la modificación por parte de un actor con acceso directo a la base de datos, riesgo que se mitiga con control de acceso (HU-41) y monitoreo de eventos de seguridad (HU-50).

**Criterios de aceptación:**

1. **Escenario 1 (evento registrado criptográficamente):** Dado que un usuario inicia sesión, falla un intento de login o exporta evidencia, cuando el backend (FastAPI) detecta la acción, entonces guarda un registro asíncrono en la base de datos (PostgreSQL) con el ID de usuario, IP, timestamp y su firma criptográfica SHA-256.
2. **Escenario 2 (intento de modificación / Append-only):** Dado que un actor malicioso o administrador intenta alterar o borrar un registro de auditoría ya almacenado, cuando se procesa la solicitud a nivel de base de datos, entonces se rechaza la operación, ya que la tabla de trazabilidad está configurada estrictamente como solo escritura (append-only) y rompería la cadena del previous_hash.
3. **Escenario 3 (consulta filtrada y segura):** Dado que el Auditor necesita revisar los eventos de seguridad de un periodo o usuario específico, cuando aplica los filtros en el módulo de auditoría de React, entonces obtiene únicamente los registros coincidentes en modo de solo lectura, sin ninguna posibilidad técnica de editarlos desde la interfaz.

---

### HU-45 · Desactivación de Usuarios y Revocación de Acceso

- **Como** Administrador del Sistema
- **Quiero** desactivar las cuentas de personal cesado impidiendo nuevos logins, pero preservando su user_id en los logs de auditoría
- **Para** revocar el acceso sin romper la cadena de trazabilidad ni perder el histórico de acciones del usuario
- **Prioridad:** Alta · **Sprint:** 10 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (desactivación impide nuevos logins):** Dado que un miembro del personal cesa en sus funciones, cuando el Administrador desactiva su cuenta, entonces el campo users.is_active pasa a false y todo intento posterior de login se rechaza con el mensaje genérico de credenciales inválidas, registrándose el intento en la bitácora.
2. **Escenario 2 (preservación del historial auditable):** Dado que el usuario desactivado tiene acciones registradas en audit_logs y traceability_records, cuando se ejecuta la desactivación, entonces esos registros se preservan intactos (solo lectura) conservando su user_id, de modo que la cadena de hash no se rompe ni se elimina evidencia histórica.
3. **Escenario 3 (auditoría de la desactivación):** Dado que se confirma la desactivación, cuando se registra la operación, entonces se genera un evento append-only tipo DESACTIVACION_USUARIO con user_id del usuario desactivado, user_id del administrador que ejecutó la acción, motivo y timestamp, encadenado con hash SHA-256.

---

### HU-50 · Gestión y Visualización de Eventos de Seguridad

- **Como** Administrador del Sistema
- **Quiero** un panel para supervisar intentos de login fallidos, bloqueos por fuerza bruta y accesos denegados por RBAC
- **Para** detectar y gestionar oportunamente intentos de ataque contra el sistema
- **Prioridad:** Alta · **Sprint:** 10 · **Story Points:** 5 · **Estado:** No se ha iniciado

**Criterios de aceptación:**

1. **Escenario 1 (panel de eventos de seguridad):** Dado que el Administrador accede al módulo de eventos de seguridad, cuando filtra por periodo, tipo de evento o usuario, entonces el sistema muestra los intentos de login fallidos, bloqueos por fuerza bruta y denegaciones de acceso RBAC (HTTP 403) registrados en audit_logs, en modo solo lectura.
2. **Escenario 2 (alerta por umbral de intentos):** Dado que se superan los intentos fallidos configurados para una misma cuenta o dirección IP en una ventana de tiempo, cuando el backend evalúa los eventos, entonces genera una alerta de seguridad visible en el panel y la despacha vía SSE a los dashboards administrativos.
3. **Escenario 3 (integración con los mecanismos de protección):** Dado que ocurre un bloqueo por fuerza bruta (HU-39) o una denegación de acceso por rol (HU-41), cuando el evento se persiste, entonces aparece en el panel con usuario/IP, timestamp y tipo de evento, permitiendo su correlación para análisis de intrusión.

---

# 4. Plan de Sprints

> Cada sprint dura 4 semanas (20 días · 160 horas). Total: 40 semanas · 200 días · 1600 horas · 51 historias de usuario distribuidas en 10 sprints. La distribución respeta dependencias técnicas: fundamentos edge y autenticación → transporte MQTT seguro → ingesta y trazabilidad → IA y su gobierno → verificación y auditoría → alertas → frontend → reportes y hardening de seguridad.

| Sprint | Story Points | Historias de usuario |
|---|---|---|
| **SP-01** | 27 | HU-39 · HU-01 · HU-02 · HU-03 · HU-04 · HU-05 |
| **SP-02** | 29 | HU-06 · HU-07 · HU-08 · HU-40 · HU-51 |
| **SP-03** | 23 | HU-09 · HU-10 · HU-11 · HU-12 · HU-13 · HU-14 |
| **SP-04** | 26 | HU-15 · HU-22 · HU-24 · HU-25 · HU-44 |
| **SP-05** | 26 | HU-16 · HU-17 · HU-18 · HU-19 · HU-46 |
| **SP-06** | 29 | HU-26 · HU-27 · HU-28 · HU-42 · HU-47 |
| **SP-07** | 24 | HU-20 · HU-21 · HU-23 · HU-29 · HU-30 |
| **SP-08** | 24 | HU-31 · HU-32 · HU-33 · HU-34 · HU-35 |
| **SP-09** | 28 | HU-36 · HU-37 · HU-38 · HU-48 · HU-49 |
| **SP-10** | 26 | HU-41 · HU-43 · HU-45 · HU-50 |
| **TOTAL** | **262** | **51 historias de usuario** |

## Detalle por sprint

### SP-01 · Sprint 1 — Fundamentos: acceso seguro y adquisición edge (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario | Prioridad | Pts |
|---|---|---|---|
| HU-39 | Como usuario del sistema (Técnico/Farmacéutico), quiero ingresar mi correo y contraseña de forma segura, para obtener acceso al dashboard cerrado conforme a OWASP Web Security Testing Guide v4.2. | Alta | 8 |
| HU-01 | Como farmacéutico responsable, quiero que el sistema capture la temperatura del sensor DS18B20 cada 30 segundos, para vigilar con precisión la condición térmica cercana al medicamento. | Alta | 5 |
| HU-02 | Como farmacéutico responsable, quiero que el sistema capture la temperatura ambiental mediante el sensor SHT31, para complementar el control de la temperatura interna del refrigerador. | Alta | 3 |
| HU-03 | Como farmacéutico responsable, quiero que el sistema registre la humedad relativa del sensor SHT31, para detectar condiciones ambientales de riesgo adicionales a la temperatura. | Media | 3 |
| HU-04 | Como técnico de farmacia, quiero que el sistema registre los eventos del sensor magnético MC-38, para relacionar aperturas reales de la puerta con fluctuaciones térmicas. | Media | 5 |
| HU-05 | Como administrador del sistema, quiero que el sistema estructure las métricas en un payload JSON estándar, para asegurar interoperabilidad con el backend aunque alguna variable no esté disponible. | Alta | 3 |

### SP-02 · Sprint 2 — Resiliencia edge y seguridad de sesión (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario | Prioridad | Pts |
|---|---|---|---|
| HU-06 | Como farmacéutico responsable, quiero que el sistema conserve las lecturas en memoria local cuando se pierda conectividad, para no perder trazabilidad térmica durante cortes prolongados. | Alta | 8 |
| HU-07 | Como farmacéutico responsable, quiero que el sistema reenvíe automáticamente los registros acumulados en LittleFS y los elimine solo cuando el backend confirme su persistencia, para completar el histórico térmico cuando la conexión se recupere, sin duplicidades ni pérdidas. | Alta | 8 |
| HU-08 | Como administrador del sistema, quiero que el nodo se reconecte usando una estrategia de backoff exponencial, para restablecer el servicio sin saturar la red local ni el microcontrolador. | Media | 5 |
| HU-40 | Como usuario autenticado, quiero que mi token JWT se almacene solo en la memoria volátil del navegador y nunca en localStorage, para reducir el riesgo de robo de sesión por ataques XSS. | Alta | 5 |
| HU-51 | Como administrador del sistema, quiero documentar la posición física del sensor DS18B20 dentro del refrigerador y las notas de instalación técnica asociadas a la metadata del dispositivo, para que las lecturas térmicas queden contextualizadas con el punto exacto de medición y el histórico de instalación del equipo. | Media | 3 |

### SP-03 · Sprint 3 — Transporte MQTT seguro (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario | Prioridad | Pts |
|---|---|---|---|
| HU-09 | Como administrador del sistema, quiero que el nodo valide el certificado de la Autoridad Certificadora (CA) del servidor mediante One-way TLS 1.2/1.3, para cifrar los datos en tránsito y evitar ataques Man-in-the-Middle. | Alta | 5 |
| HU-10 | Como administrador del sistema, quiero que cada nodo ESP32 se autentique ante el broker con un device_id único y un token/credencial propia, con permisos de publicación restringidos por ACL por tópico, para impedir el acceso de hardware no autorizado conforme a OWASP IoT. | Alta | 5 |
| HU-11 | Como administrador del sistema, quiero que las lecturas térmicas se publiquen con QoS 1 (al menos una vez) y que el backend trate los duplicados de red de forma idempotente, para confirmar la recepción en el broker y evitar vacíos en la serie temporal sin duplicar registros. | Alta | 5 |
| HU-12 | Como administrador del sistema, quiero que el backend se suscriba al tópico farmacias/+/lecturas mediante aiomqtt, para consumir telemetría de cualquier refrigerador sin bloquear el servidor. | Alta | 3 |
| HU-13 | Como administrador del sistema, quiero que el nodo configure un mensaje Last Will and Testament (LWT) en el broker, para notificar caídas abruptas de energía o red al backend y al dashboard. | Media | 3 |
| HU-14 | Como administrador del sistema, quiero que el broker MQTT gestionado bloquee conexiones por el puerto 1883 en texto plano, para forzar comunicaciones seguras y evitar sniffers de red. | Media | 2 |

### SP-04 · Sprint 4 — Ingesta, persistencia y base de trazabilidad (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario | Prioridad | Pts |
|---|---|---|---|
| HU-15 | Como administrador del sistema, quiero que el backend valide los payloads MQTT entrantes con Pydantic v2, para rechazar estructuras corruptas o maliciosas antes de persistir los datos. | Alta | 3 |
| HU-22 | Como farmacéutico responsable, quiero que la telemetría se almacene en PostgreSQL usando JSONB, para conservar un histórico térmico flexible junto con su clasificación asociada. | Alta | 5 |
| HU-24 | Como responsable de auditoría, quiero que el sistema calcule un hash SHA-256 por cada evento térmico u operativo, para asegurar su integridad verificable sin depender de infraestructuras blockchain de terceros. | Alta | 5 |
| HU-25 | Como responsable de auditoría, quiero que cada nuevo registro incluya el hash del evento anterior en traceability_records, para formar un registro encadenado cuya integridad sea verificable secuencialmente. | Alta | 8 |
| HU-44 | Como administrador del sistema, quiero revocar o rotar el token MQTT de un ESP32 cuyas credenciales se consideren comprometidas, sin alterar su historial de telemetría, para cortar de inmediato el acceso del nodo comprometido preservando la integridad del histórico ya registrado. | Alta | 5 |

### SP-05 · Sprint 5 — Inteligencia artificial y gobierno del modelo (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario | Prioridad | Pts |
|---|---|---|---|
| HU-16 | Como administrador del sistema, quiero que el backend cargue el modelo Random Forest al iniciar el servicio, para procesar inferencias de riesgo sin latencia de lectura de disco por mensaje. | Alta | 5 |
| HU-17 | Como farmacéutico responsable, quiero que el sistema calcule métricas derivadas como gradiente térmico y derivadas de tiempo, para alimentar correctamente el modelo Random Forest que clasifica el riesgo. | Alta | 5 |
| HU-18 | Como farmacéutico responsable, quiero que el sistema evalúe la matriz de características operativas, para clasificar el estado térmico en Normal, Riesgo Preventivo o Excursión Crítica. | Alta | 5 |
| HU-19 | Como técnico de farmacia, quiero ver en tiempo real la lectura procesada y la clasificación de riesgo en el dashboard, para monitorear incidentes sin recargar la interfaz. | Media | 3 |
| HU-46 | Como administrador del sistema, quiero registrar en base de datos la versión activa del modelo Random Forest, su hash SHA-256 y la versión del esquema de features, asegurando que solo modelos aprobados clasifiquen en producción, para garantizar el gobierno y la observabilidad del modelo de IA, evitando que una versión no autorizada clasifique el riesgo térmico. | Alta | 8 |

### SP-06 · Sprint 6 — Verificación de integridad y auditoría (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario | Prioridad | Pts |
|---|---|---|---|
| HU-26 | Como Responsable de Auditoría o Fiscalización interna, quiero solicitar una verificación algorítmica en bloque de toda la tabla de trazabilidad, para detectar manipulaciones en los registros y presentar evidencias confiables ante inspecciones de DIGEMID. | Alta | 8 |
| HU-27 | Como Químico Farmacéutico (Regente), quiero registrar la acción correctiva aplicada tras una alerta, para cumplir con el soporte documental exigido por la normativa de Buenas Prácticas de Almacenamiento de DIGEMID. | Media | 5 |
| HU-28 | Como responsable de auditoría, quiero que el sistema calcule hash SHA-256 también sobre las acciones manuales de los usuarios, para detectar cambios posteriores en las justificaciones registradas. | Media | 3 |
| HU-42 | Como Responsable de Auditoría, quiero tener un registro automatizado con comportamiento append-only de cada inicio de sesión, cambio de estado y exportación de datos, para cumplir con el seguimiento estricto de identidades ante inspecciones sanitarias. | Media | 8 |
| HU-47 | Como responsable de auditoría / farmacéutico, quiero auditar qué versión específica del modelo y qué probabilidades generaron la clasificación de una lectura térmica determinada, para explicar y verificar las decisiones de la IA ante auditorías internas o consultas de la autoridad sanitaria. | Alta | 5 |

### SP-07 · Sprint 7 — Alertas, notificaciones y continuidad operativa (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario | Prioridad | Pts |
|---|---|---|---|
| HU-20 | Como farmacéutico responsable, quiero que el sistema genere una alerta cuando el modelo detecte Riesgo Preventivo, para actuar antes de que se rompa la cadena de frío. | Media | 5 |
| HU-21 | Como farmacéutico responsable, quiero que el sistema registre una excursión térmica crítica si la temperatura sale del rango objetivo de 2 °C a 8 °C definido por la investigación, para ejecutar acciones correctivas auditables conforme a las Buenas Prácticas de Almacenamiento. | Alta | 3 |
| HU-23 | Como farmacéutico responsable, quiero recibir una notificación automática por correo electrónico o Telegram cuando se genere una excursión crítica, para enterarme del incidente de forma inmediata aunque no tenga el dashboard web abierto. | Alta | 8 |
| HU-29 | Como administrador de infraestructura, quiero que la base de datos PostgreSQL se respalde de forma automatizada, para asegurar la continuidad operativa y proteger la evidencia histórica ante caídas o desastres. | Baja | 3 |
| HU-30 | Como Químico Farmacéutico, quiero registrar la fecha de última calibración y el certificado del sensor de temperatura, para recibir alertas antes del vencimiento y mantener el cumplimiento regulatorio. | Media | 5 |

### SP-08 · Sprint 8 — Monitoreo web en tiempo real (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario | Prioridad | Pts |
|---|---|---|---|
| HU-31 | Como técnico de farmacia, quiero visualizar una gráfica de temperatura con umbrales fijos de 2 °C a 8 °C, para evaluar rápidamente la tendencia térmica del refrigerador. | Alta | 8 |
| HU-32 | Como técnico de farmacia, quiero que el dashboard reciba actualizaciones del backend mediante Server-Sent Events (SSE), para ver el último punto de la gráfica y los KPI en tiempo real. | Alta | 5 |
| HU-33 | Como técnico de farmacia, quiero visualizar el último valor de temperatura (DS18B20) y humedad ambiental (SHT31) en tarjetas KPI, para obtener una lectura rápida del estado del equipo a un solo vistazo. | Media | 3 |
| HU-34 | Como farmacéutico, quiero ver un semáforo visual (Verde, Amarillo, Rojo) en la tarjeta del equipo que refleje el riesgo efectivo, combinando la regla crítica directa de 2–8 °C con la predicción preventiva del Random Forest, para identificar de inmediato el nivel de atención clínica o técnica requerido. | Alta | 5 |
| HU-35 | Como personal técnico de farmacia, quiero visualizar una alerta intermitente en la interfaz si el refrigerador permanece abierto, para cerrarlo rápidamente y evitar fuga térmica y condensación. | Media | 3 |

### SP-09 · Sprint 9 — Reportes, verificación y administración operativa (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario | Prioridad | Pts |
|---|---|---|---|
| HU-36 | Como Químico Farmacéutico (Regente), quiero consultar y filtrar los datos históricos por fecha y hora, para analizar eventos retrospectivos, excursiones pasadas y generar evidencia para auditorías. | Media | 5 |
| HU-37 | Como farmacéutico responsable, quiero solicitar la verificación de integridad de la telemetría térmica de un dispositivo en un periodo determinado y obtener el estado de cada lectura, para confirmar que los registros de cadena de frío no han sido alterados y contar con evidencia verificable acotada a la consulta. | Media | 5 |
| HU-38 | Como responsable de auditoría o auditor interno, quiero generar y exportar un reporte en formato PDF del historial térmico y las alertas, para adjuntarlo como evidencia digital verificable del cumplimiento de la cadena de frío y entregarlo a la autoridad de inspección sanitaria (DIRIS/DIGEMID) cuando lo requiera. | Media | 8 |
| HU-48 | Como farmacéutico, quiero ver en el dashboard si un ESP32 tiene datos pendientes en LittleFS o si ocurrió una pérdida irrecuperable de datos por buffer lleno tras un corte prolongado, para conocer la completitud del registro térmico y actuar frente a brechas de información. | Media | 5 |
| HU-49 | Como responsable de auditoría, quiero consultar la bitácora de reemplazos de sensores, cambios de umbral y mantenimiento del hardware, para auditar la evolución de la configuración del monitoreo sin alterar los registros pasados. | Media | 5 |

### SP-10 · Sprint 10 — Hardening de seguridad transversal y administración de dispositivos (4 semanas · 20 días · 160 horas)

| ID | Historia de usuario | Prioridad | Pts |
|---|---|---|---|
| HU-41 | Como Administrador del sistema, quiero restringir el acceso a rutas visuales y funciones de la API según el rol del usuario autenticado (Técnico, Farmacéutico, Auditor), para aplicar el principio de mínimo privilegio y cumplir con las normativas de seguridad OWASP. | Alta | 8 |
| HU-43 | Como Administrador del Sistema / Técnico de Mantenimiento, quiero registrar la baja de un dispositivo IoT (ESP32/sensores) cuando se malogre o sea reemplazado, sin corromper la cadena de trazabilidad histórica, para mantener la consistencia administrativa, auditar el reemplazo de hardware y cumplir con DIGEMID en caso de cambios en la configuración del monitoreo. | Alta | 8 |
| HU-45 | Como Administrador del Sistema, quiero desactivar las cuentas de personal cesado impidiendo nuevos logins, pero preservando su user_id en los logs de auditoría, para revocar el acceso sin romper la cadena de trazabilidad ni perder el histórico de acciones del usuario. | Alta | 5 |
| HU-50 | Como Administrador del Sistema, quiero un panel para supervisar intentos de login fallidos, bloqueos por fuerza bruta y accesos denegados por RBAC, para detectar y gestionar oportunamente intentos de ataque contra el sistema. | Alta | 5 |

---

# 5. Notas metodológicas — cambios aplicados respecto a la versión revisada

## 5.1 Historias eliminadas/reemplazadas y motivo

| Elemento eliminado | HU anterior | Motivo técnico / metodológico |
|---|---|---|
| Actualización de Firmware OTA | HU-46 (previa) | **Scope creep masivo:** requiere particionado flash dual, cifrado de binarios, gestión de BLOBs, anti-downgrade y rollback por hardware. No es el problema de la tesis (monitoreo térmico e IA). Reemplazada por la nueva HU-46 (versionado operativo del modelo IA). |
| Cuarentena y Bloqueo de Lotes | HU-48 (previa) | Obliga a crear un subsistema de inventario (lotes, stock, vencimiento, dispensación). Error conceptual: una excursión de 9 °C por 15 min no demuestra pérdida de estabilidad biológica; depende de la cinética de degradación y la ficha técnica del fabricante. Reemplazada por la nueva HU-48 (estado de sincronización y brechas). |
| Gestión de Retiro de Mercado (Recall) | HU-49 (previa) | Fuera de alcance: el sistema es un monitor IoT de cadena de frío, no una plataforma de logística inversa para recalls de DIGEMID. Reemplazada por la nueva HU-49 (historial de cambios de configuración del hardware). |
| Firma digitalizada en imagen (PNG) | HU-50 (previa) | Error legal y técnico: una imagen PNG no es una firma digital; la Ley N.° 27269 (Firmas y Certificados Digitales) exige criptografía asimétrica acreditada. La validez probatoria se garantiza con JWT + RBAC + audit_logs + hash encadenado. Reemplazada por la nueva HU-50 (eventos de seguridad). |
| Referencias a GDPR y 21 CFR Part 11 | HU-44/45 (previas) | Incongruencia jurisdiccional: 21 CFR Part 11 es de la FDA (EE. UU.) y el GDPR es de la Unión Europea. El marco legal aplicable es la Ley N.° 29733 (y su D.S. 016-2024-JUS) y las RM 554-2022/MINSA / RM 810-2024/MINSA. Reemplazadas por las nuevas HU-44 (credenciales IoT) y HU-45 (desactivación de usuarios). |
| Formulario Checklist BPA completo | HU-37 (original) | Desviación de producto: convertía el software en un gestor de autoinspecciones en papel digitalizado. El slot se reenfocó en trazabilidad: verificación de integridad de telemetría por dispositivo y periodo (nueva HU-37). |
| Rol "Inspector Sanitario / DIGEMID" como usuario | HU-38 | Inconsistencia de usuario: un inspector de DIRIS no tiene cuenta en el sistema; quien emite y entrega el PDF de evidencia es el Químico Farmacéutico o el auditor interno. |
| Manejo de corrupción y recuperación de cadena hash | HU-47 (previa) | Su esencia (detección de corrupción) permanece en la HU-26; la recuperación ante corrupción se trasladó al protocolo de validación VT-02 del capítulo de metodología. Reemplazada por la nueva HU-47 (trazabilidad de la inferencia IA). |

## 5.2 Correcciones técnicas aplicadas a historias existentes

| HU | Corrección |
|---|---|
| HU-07 | El ESP32 borra bloques de LittleFS únicamente tras el **acuse lógico (ACK) de FastAPI vía MQTT** (COMMIT en PostgreSQL), no tras el PUBACK de MQTT QoS 1, que solo confirma recepción del broker. |
| HU-09 / HU-10 | El **SNI no autentica al dispositivo**: SNI/TLS validan el certificado del servidor y enrutan la conexión segura; la autenticación del dispositivo es `device_id + token único + ACL` en el broker. |
| HU-11 | QoS 1 ("al menos una vez") puede generar duplicados de red por definición del protocolo; FastAPI aplica **idempotencia estricta** con `UNIQUE(device_id, reading_id)` en PostgreSQL. |
| HU-17 | Sin imputación de "valores neutros por defecto" (producen *training-serving skew*): si faltan variables indispensables, el estado se marca `no_clasificable` y se evalúa solo la regla de rango térmico 2–8 °C. |
| HU-21 / HU-34 | 2 °C a 8 °C definido como **rango objetivo de la investigación** (regla crítica directa); el semáforo muestra el **riesgo efectivo** (regla directa + predicción preventiva del Random Forest). |
| HU-24 / HU-25 / HU-42 | Se eliminaron los términos "inmutabilidad algorítmica" y "cadena inviolable"; se usa **"integridad verificable mediante hash encadenado SHA-256"** con comportamiento **append-only**. El hash detecta manipulaciones retrospectivas, no impide físicamente que un administrador con acceso directo a la BD altere una tabla. |
| HU-36 | Redacción corregida a arquitectura en capas: `React (Frontend) → REST API FastAPI (Backend) → PostgreSQL (BD)`; el frontend nunca consulta la base de datos directamente. |
| HU-38 | Rol corregido: emite el reporte el responsable de auditoría o auditor interno; el inspector sanitario recibe el PDF, no inicia sesión. |

## 5.3 Actividades de investigación trasladadas al Capítulo de Metodología y Validación Técnica

Las siguientes actividades **no son historias de usuario de producto**; se gestionan como *Technical Enablers* (TE) y *Validation Tasks* (VT) en la sección de Metodología y Protocolos de Validación del documento de tesis:

| Código | Actividad |
|---|---|
| TE-01 | Preparación, limpieza y versionado del dataset térmico. |
| TE-02 | Entrenamiento, ajuste de hiperparámetros y evaluación del Random Forest (Precision, Recall, F1 por clase). |
| VT-01 | Protocolo de prueba de corte de red, saturación de LittleFS y recuperación offline. |
| VT-02 | Protocolo de inyección de corrupción en PostgreSQL y prueba de detección O(n) del hash. |
| VS-01 | Plan de pruebas de seguridad estática y dinámica basado en OWASP IoT STG y WSTG. |
| VT-03 | Protocolo de validación técnica integral desplegado en la farmacia del Cercado de Lima. |

## 5.4 Marco normativo de referencia

- **Ley N.° 29733** — Protección de Datos Personales (Perú) y su reglamento **D.S. 016-2024-JUS**.
- **RM 554-2022/MINSA** y **RM 810-2024/MINSA** — Buenas Prácticas de Oficina Farmacéutica.
- **DIGEMID / DIRIS** — supervisión e inspección sanitaria.
- **OWASP IoT Top 10 / IoT STG** y **OWASP WSTG v4.2** — seguridad del sistema.
- No aplican a este alcance: RM 132-2015-MINSA (droguerías/laboratorios), GDPR (Unión Europea), 21 CFR Part 11 (FDA, EE. UU.).
