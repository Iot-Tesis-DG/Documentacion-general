# User Stories Finales — Tesis

**Título de la tesis:**  
Sistema IoT con inteligencia artificial y trazabilidad digital verificable para el monitoreo de la cadena de frío de medicamentos termolábiles en farmacias independientes de Lima Metropolitana.

**Cantidad total de User Stories:** 51

> Baseline final para la construcción del Product Backlog definitivo.

---

> Esta sección constituye la versión limpia que debe usarse como entrada para la consolidación posterior de EPICs y Product Backlog. Se mantienen exactamente los 51 IDs originales.

## HU-01 — Captura de temperatura interna (DS18B20)

**EPIC:** EP01 — Adquisición de Datos, Resiliencia y Gestión Edge  
**Rol:** Químico Farmacéutico responsable  
**User Story:**

> Como Químico Farmacéutico responsable, quiero que el sistema capture periódicamente la temperatura interna mediante el DS18B20, para disponer de un registro continuo de la condición térmica cercana a los medicamentos monitoreados.

### Criterios de aceptación

1. Dado que el ESP32 y el DS18B20 están operativos, cuando transcurre el intervalo de muestreo configurado (30 s en la configuración de validación), entonces se registra la temperatura con `device_id`, `reading_id` y `capture_timestamp` UTC.
2. Dado que el DS18B20 no responde o devuelve un código de error reconocido, cuando se ejecuta el muestreo, entonces la lectura se marca como no disponible y no se utiliza como temperatura válida.
3. Dado que se obtiene un valor fuera del rango operativo declarado por el sensor, cuando se valida la lectura, entonces se registra como anomalía de sensor sin detener los siguientes ciclos.
4. La exactitud nominal del sensor se documenta como especificación del fabricante; su desempeño real se comprueba mediante la tarea de validación técnica correspondiente.

**Prioridad:** Alta  
**Sprint actual:** SP-01  
**Story Points actuales:** 5  
**Dependencias:** Sin dependencia funcional estricta.  
**Justificación dentro de la tesis:** Origen primario de temperatura para HU-05, HU-15, HU-17, HU-18, HU-21 y HU-22.

## HU-02 — Captura de temperatura ambiental (SHT31)

**EPIC:** EP01 — Adquisición de Datos, Resiliencia y Gestión Edge  
**Rol:** Químico Farmacéutico responsable  
**User Story:**

> Como Químico Farmacéutico responsable, quiero que el sistema registre la temperatura ambiental medida por el SHT31, para complementar la lectura interna con una variable térmica contextual del entorno de monitoreo.

### Criterios de aceptación

1. Dado que el SHT31 responde correctamente por I²C, cuando se ejecuta el ciclo de muestreo, entonces se registra su temperatura ambiental con `device_id` y `capture_timestamp`.
2. Dado que el SHT31 no responde dentro del tiempo configurado, cuando se agotan los reintentos definidos, entonces la variable se marca como no disponible sin invalidar automáticamente la lectura del DS18B20.
3. Dado que la temperatura SHT31 está fuera de su rango operativo declarado, cuando se valida, entonces se registra como anomalía y no se usa como valor válido.

**Prioridad:** Alta  
**Sprint actual:** SP-01  
**Story Points actuales:** 3  
**Dependencias:** Sin dependencia funcional estricta.  
**Justificación dentro de la tesis:** Complementa HU-01 y comparte captura física con HU-03, pero su variable es temperatura contextual.

## HU-03 — Captura de humedad relativa (SHT31)

**EPIC:** EP01 — Adquisición de Datos, Resiliencia y Gestión Edge  
**Rol:** farmacéutico responsable  
**User Story:**

> Como farmacéutico responsable, quiero que el sistema registre la humedad relativa medida por el SHT31, para disponer de una variable ambiental contextual que pueda utilizarse en el análisis térmico.

### Criterios de aceptación

1. **Escenario 1 (lectura correcta):** Dado que se captura exitosamente la temperatura por I2C, cuando se procesa la respuesta del SHT31, entonces se extrae el % de humedad relativa con precisión de ±2% RH.
2. **Escenario 2 (variable contextual, sin regla de riesgo propia):** Dado que se obtiene una medición válida de humedad, cuando se construye la telemetría, entonces el valor se incorpora como variable contextual sin asignar por sí solo una clase de riesgo térmico, salvo que una regla validada posteriormente en la metodología así lo establezca. La condensación depende de la relación entre temperatura, humedad, punto de rocío y temperatura de superficie, por lo que no se fija un umbral arbitrario de HR como regla funcional.
3. **Escenario 3 (lectura fuera de rango):** Dado que el valor está fuera de 0–100% RH, cuando se valida, entonces se descarta como lectura inválida.

**Prioridad:** Media  
**Sprint actual:** SP-01  
**Story Points actuales:** 3  
**Dependencias:** Sin dependencia funcional estricta.  
**Justificación dentro de la tesis:** Comparte SHT31 con HU-02, pero su responsabilidad exclusiva es humedad contextual.

## HU-04 — Detección de apertura de puerta (MC-38, opcional)

**EPIC:** EP01 — Adquisición de Datos, Resiliencia y Gestión Edge  
**Rol:** técnico de farmacia  
**User Story:**

> Como técnico de farmacia, quiero que el sistema registre los eventos del sensor magnético MC-38 cuando esté instalado, para relacionar aperturas reales de la puerta con fluctuaciones térmicas.

### Criterios de aceptación

1. **Escenario 1 (apertura/cierre real):** Dado que la puerta cambia de estado físicamente, cuando el GPIO se mantiene en el nuevo estado más allá del tiempo mínimo de antirrebote (debounce), entonces se actualiza apertura_refrigerador y se registra el timestamp del cambio.
2. **Escenario 2 (rebote de contacto):** Dado que el GPIO cambia de estado por menos del tiempo de antirrebote, cuando se evalúa la señal, entonces el cambio se ignora y no se reporta.
3. **Escenario 3 (apertura prolongada):** Dado que la puerta permanece abierta más allá de un umbral configurado, cuando se detecta esta condición, entonces se añade al payload un contador de duración de apertura.
4. **Escenario 4 (dispositivo sin MC-38):** Dado que el dispositivo fue configurado sin MC-38, cuando construye la telemetría, entonces door_open se representa como null y mc38_status = "not_installed", sin afectar la captura térmica, MQTT, IA o dashboard.

**Prioridad:** Media  
**Sprint actual:** SP-01  
**Story Points actuales:** 5  
**Dependencias:** Sin dependencia funcional estricta.  
**Justificación dentro de la tesis:** Produce eventos de puerta consumidos por HU-35; es opcional y no bloquea HU-01/HU-18.

## HU-05 — Estructuración versionada de telemetría

**EPIC:** EP01 — Adquisición de Datos, Resiliencia y Gestión Edge  
**Rol:** Administrador del sistema  
**User Story:**

> Como Administrador del sistema, quiero que cada lectura se estructure en un payload interoperable, identificable y versionado antes de su transmisión, para asegurar que el dispositivo y el backend interpreten de forma consistente la telemetría aun cuando algún sensor no esté disponible.

### Criterios de aceptación

1. Dado un ciclo de lectura, cuando se construye la telemetría, entonces el payload incluye como mínimo `schema_version`, `device_id`, `reading_id`, `capture_timestamp`, las variables de sensores habilitados, estados de sensor y versión de firmware.
2. Dado que un sensor está habilitado pero falló, cuando se serializa el payload, entonces su valor se representa de forma explícita como `null` acompañado por el estado de sensor correspondiente.
3. Dado que el payload no cumple el esquema o excede el límite soportado por el firmware/cliente MQTT configurado, cuando se intenta preparar el envío, entonces se registra el error y no se publica una estructura inválida.

**Prioridad:** Alta  
**Sprint actual:** SP-01  
**Story Points actuales:** 3  
**Dependencias:** HU-01, HU-02, HU-03, HU-04  
**Justificación dentro de la tesis:** Contrato de telemetría que alimenta HU-11 y es validado por HU-15.

## HU-06 — Persistencia en Buffer Offline (LittleFS)

**EPIC:** EP01 — Adquisición de Datos, Resiliencia y Gestión Edge  
**Rol:** Químico Farmacéutico responsable  
**User Story:**

> Como Químico Farmacéutico responsable, quiero que el dispositivo conserve localmente las lecturas pendientes durante una pérdida temporal de conectividad, para preservar la continuidad del registro térmico y evidenciar cualquier brecha si la capacidad local llega a agotarse.

### Criterios de aceptación

1. Dado que no existe conectividad Wi-Fi o MQTT, cuando se genera una lectura pendiente, entonces se conserva localmente en LittleFS/SPIFFS manteniendo `reading_id` y timestamp original.
2. Dado que el dispositivo reinicia después de una pérdida de energía, cuando monta el almacenamiento local, entonces recupera los registros válidos pendientes y descarta únicamente archivos comprobablemente incompletos o corruptos.
3. Dado que el almacenamiento alcanza el umbral de capacidad configurado, cuando no puede conservar todas las lecturas, entonces aplica la política de retención definida y registra un evento de saturación con el periodo afectado; ninguna pérdida queda silenciosa.
4. El tamaño de buffer y la pérdida/recuperación efectiva se validan mediante la prueba offline de la tesis y no se presuponen como 'cero pérdida'.

**Prioridad:** Alta  
**Sprint actual:** SP-03  
**Story Points actuales:** 8  
**Dependencias:** HU-05  
**Justificación dentro de la tesis:** Retiene datos; HU-07 los sincroniza y HU-48 hace visible su estado/brechas.

## HU-07 — Sincronización de Buffer Offline con Acuse Lógico

**EPIC:** EP01 — Adquisición de Datos, Resiliencia y Gestión Edge  
**Rol:** farmacéutico responsable  
**User Story:**

> Como farmacéutico responsable, quiero que el sistema reenvíe automáticamente los registros acumulados en LittleFS y los elimine solo cuando el backend confirme su persistencia, para completar el histórico térmico cuando la conexión se recupere, sin duplicidades lógicas y preservando las lecturas disponibles mientras exista capacidad de almacenamiento local.

### Criterios de aceptación

1. **Escenario 1 (sincronización con acuse lógico del backend):** Dado que el ESP32 se reconecta a EMQX y existen archivos de telemetría pendientes, cuando se inicia el proceso de sincronización, entonces se publican en estricto orden FIFO utilizando QoS 1, y cada bloque se elimina de LittleFS **únicamente tras recibir el acuse lógico (ACK) publicado por FastAPI vía MQTT**, que confirma que la lectura hizo `COMMIT` en PostgreSQL. El `PUBACK` de MQTT QoS 1 por sí solo no autoriza el borrado, pues solo confirma que el broker recibió el paquete, no que la base de datos lo persistió.
2. **Escenario 2 (fallo a mitad de sincronización):** Dado que la conexión se interrumpe abruptamente durante la transmisión de los registros, cuando se recupera la red nuevamente, entonces el sistema reanuda la cola de envíos partiendo estrictamente desde el siguiente archivo al último confirmado por el acuse lógico del backend; los bloques enviados pero no acusados se reenvían, y la idempotencia del backend (HU-11) descarta los duplicados resultantes.
3. **Escenario 3 (orden cronológico e integridad):** Dado que existen múltiples archivos pendientes de distintas horas o días, cuando se sincronizan hacia el backend, entonces se publican respetando su marca de tiempo (timestamp) original de captura, permitiendo a la base de datos rearmar la serie temporal correctamente.

**Prioridad:** Alta  
**Sprint actual:** SP-03  
**Story Points actuales:** 8  
**Dependencias:** HU-06, HU-11, HU-22  
**Justificación dentro de la tesis:** Depende de HU-06/HU-11/HU-22; su responsabilidad es reanudación confirmada, no transporte online normal.

## HU-08 — Reconexión automática del nodo

**EPIC:** EP01 — Adquisición de Datos, Resiliencia y Gestión Edge  
**Rol:** Administrador del sistema  
**User Story:**

> Como Administrador del sistema, quiero que el nodo restablezca automáticamente su conexión Wi-Fi después de una interrupción mediante reintentos controlados, para recuperar la transmisión sin saturar la red ni comprometer la continuidad del monitoreo.

### Criterios de aceptación

1. Dado que el nodo pierde Wi-Fi, cuando intenta reconectarse, entonces aplica un backoff progresivo con límite máximo configurable en lugar de reintentos continuos.
2. Dado que la conexión se restablece, cuando el nodo vuelve a estar online, entonces reinicia el contador de backoff y reanuda el flujo de publicación/sincronización.
3. Dado que la reconexión no es posible por credenciales inválidas o ausencia prolongada del AP, cuando se supera el umbral operativo definido, entonces se registra el estado localmente y se expone una señal de diagnóstico sin detener la captura offline.

**Prioridad:** Media  
**Sprint actual:** SP-03  
**Story Points actuales:** 5  
**Dependencias:** Sin dependencia funcional estricta.  
**Justificación dentro de la tesis:** Restablece conectividad para HU-07/HU-09/HU-11, sin sustituir el buffer de HU-06.

## HU-09 — Canal MQTT cifrado con validación del broker

**EPIC:** EP02 — Comunicación MQTT Segura  
**Rol:** Administrador del sistema  
**User Story:**

> Como Administrador del sistema, quiero que el nodo verifique la identidad del broker y establezca un canal MQTT cifrado mediante TLS, para proteger la telemetría y las credenciales frente a exposición o alteración durante el tránsito.

### Criterios de aceptación

1. Dado que el ESP32 inicia una conexión MQTT hacia EMQX Serverless, cuando negocia TLS, entonces valida la cadena de confianza y el nombre del servidor antes de transmitir telemetría.
2. Dado que el certificado es inválido, expirado o no corresponde al host esperado, cuando se realiza la validación, entonces la conexión se rechaza y no existe fallback a MQTT en texto plano.
3. Dado que el handshake TLS excede el timeout configurado, cuando el intento se cancela, entonces se registra el fallo y se activa la estrategia de reconexión.

**Prioridad:** Alta  
**Sprint actual:** SP-03  
**Story Points actuales:** 5  
**Dependencias:** Sin dependencia funcional estricta.  
**Justificación dentro de la tesis:** Protege el canal y valida servidor; HU-10 autentica dispositivo y HU-14 aplica política de endpoint.

## HU-10 — Autenticación de Dispositivo (device_id + token único + ACL)

**EPIC:** EP02 — Comunicación MQTT Segura  
**Rol:** administrador del sistema  
**User Story:**

> Como administrador del sistema, quiero que cada nodo ESP32 se autentique ante el broker con un device_id único y un token/credencial propia, con permisos de publicación restringidos por ACL por tópico, para impedir el acceso de hardware no autorizado conforme al alineamiento con OWASP IoT.

### Criterios de aceptación

1. **Escenario 1 (credenciales válidas):** Dado que un cliente IoT se conecta utilizando un device_id único como usuario y su token/contraseña correspondiente, cuando el broker valida las credenciales, entonces acepta la conexión y la ACL restringe sus permisos de publicación únicamente a sus tópicos designados.
2. **Escenario 2 (credenciales inválidas o revocadas):** Dado que un cliente intenta establecer conexión con un token incorrecto o revocado, cuando el broker evalúa la petición, entonces rechaza la conexión utilizando el reason code correspondiente de MQTT 5.0 (por ejemplo 0x87 *Not authorized* cuando aplique) y deniega cualquier intento de publicación.
3. **Escenario 3 (aplicación de ACL por tópico):** Dado que un dispositivo con credenciales válidas intenta publicar en tópicos de otro dispositivo o en tópicos administrativos, cuando el broker evalúa la ACL, entonces deniega la publicación y registra el evento en los logs de observabilidad de EMQX Cloud para su análisis.

**Prioridad:** Alta  
**Sprint actual:** SP-04  
**Story Points actuales:** 5  
**Dependencias:** HU-09  
**Justificación dentro de la tesis:** Autenticación/autorización normal del nodo; HU-44 gestiona revocación/rotación posterior.

## HU-11 — Publicación de Telemetría (QoS 1) con Idempotencia

**EPIC:** EP02 — Comunicación MQTT Segura  
**Rol:** Administrador del sistema  
**User Story:**

> Como Administrador del sistema, quiero que la telemetría se envíe con QoS 1 y se procese de forma idempotente en el backend, para reducir pérdidas durante la transmisión sin generar registros duplicados ante retransmisiones.

### Criterios de aceptación

1. Dado que el nodo publica una lectura, cuando utiliza MQTT QoS 1, entonces mantiene el mensaje pendiente hasta recibir el `PUBACK` correspondiente del broker.
2. Dado que no se recibe `PUBACK` dentro del tiempo configurado, cuando se reintenta la publicación, entonces se conserva el mismo contenido lógico de la lectura y su `reading_id`.
3. Dado que el backend recibe más de una vez el mismo `device_id + reading_id`, cuando intenta persistirlo, entonces la operación es idempotente y no duplica la lectura.
4. Dado que se agotan los reintentos de transmisión en línea, cuando el nodo declara la desconexión, entonces la lectura queda sujeta al mecanismo de buffer offline.

**Prioridad:** Alta  
**Sprint actual:** SP-04  
**Story Points actuales:** 5  
**Dependencias:** HU-05, HU-09, HU-10  
**Justificación dentro de la tesis:** Entrega MQTT e idempotencia; HU-07 añade acuse lógico posterior al commit.

## HU-12 — Recepción continua de telemetría en backend

**EPIC:** EP02 — Comunicación MQTT Segura  
**Rol:** Administrador del sistema  
**User Story:**

> Como Administrador del sistema, quiero que el backend reciba de forma continua la telemetría publicada por los dispositivos autorizados, para centralizar el procesamiento de lecturas sin bloquear la atención de otros mensajes o solicitudes.

### Criterios de aceptación

1. Dado que FastAPI inicia correctamente, cuando se habilita el consumidor MQTT, entonces queda suscrito al patrón de tópicos autorizado para recibir telemetría de los dispositivos.
2. Dado que llegan mensajes de varios dispositivos, cuando el backend los consume, entonces un mensaje no bloquea el procesamiento de los demás.
3. Dado que la conexión backend–broker se interrumpe, cuando se restablece, entonces el consumidor recupera su suscripción sin requerir reinicio manual del servicio.
4. El uso de `aiomqtt` y del ciclo asíncrono queda como implementación del criterio, no como valor principal de la historia.

**Prioridad:** Alta  
**Sprint actual:** SP-04  
**Story Points actuales:** 3  
**Dependencias:** HU-09, HU-10  
**Justificación dentro de la tesis:** Puente broker→backend; HU-15 valida el contenido después de recibirlo.

## HU-13 — Estado de conectividad del dispositivo

**EPIC:** EP02 — Comunicación MQTT Segura  
**Rol:** Administrador del sistema  
**User Story:**

> Como Administrador del sistema, quiero que el sistema conozca y actualice el estado de conectividad de cada nodo IoT, para detectar oportunamente desconexiones abruptas y distinguir equipos online, offline o recuperados.

### Criterios de aceptación

1. Dado que un nodo pierde energía o red sin desconexión ordenada, cuando el broker vence el keep-alive, entonces publica el LWT que identifica al dispositivo como offline.
2. Dado que el nodo se desconecta de forma ordenada, cuando envía `DISCONNECT`, entonces no se interpreta la salida como una caída abrupta.
3. Dado que el nodo vuelve a conectarse, cuando completa la sesión segura, entonces publica su estado online y el dashboard puede actualizar la conectividad.

**Prioridad:** Media  
**Sprint actual:** SP-04  
**Story Points actuales:** 3  
**Dependencias:** HU-09, HU-10  
**Justificación dentro de la tesis:** Produce estado de conectividad consumido por HU-33/HU-48.

## HU-14 — Uso exclusivo de endpoints cifrados (EMQX Cloud Serverless)

**EPIC:** EP02 — Comunicación MQTT Segura  
**Rol:** Administrador del sistema  
**User Story:**

> Como Administrador del sistema, quiero que la configuración operativa de los dispositivos permita únicamente endpoints MQTT cifrados compatibles con EMQX Cloud Serverless, para evitar que una configuración incorrecta habilite transmisión de telemetría o credenciales sin TLS.

### Criterios de aceptación

1. Dado que se revisa la configuración productiva del firmware, cuando se inspecciona el endpoint MQTT, entonces utiliza únicamente el endpoint TLS soportado por EMQX Serverless.
2. Dado que un dispositivo intenta establecer la conexión configurada, cuando usa el puerto/endpoint seguro autorizado, entonces continúa a la validación del certificado y de credenciales.
3. Dado que se detectan intentos de conexión no autorizados en el broker, cuando el administrador consulta la observabilidad disponible, entonces puede identificar dichos eventos sin habilitar un canal de texto plano.

**Prioridad:** Media  
**Sprint actual:** SP-04  
**Story Points actuales:** 2  
**Dependencias:** HU-09  
**Justificación dentro de la tesis:** Política de endpoint cifrado; no reemplaza la validación criptográfica de HU-09.

## HU-15 — Validación de telemetría entrante

**EPIC:** EP03 — Procesamiento e Inteligencia Artificial  
**Rol:** Administrador del sistema  
**User Story:**

> Como Administrador del sistema, quiero que el backend acepte únicamente telemetría con estructura y tipos válidos, diferenciando una falla de sensor de un mensaje malformado, para evitar que datos corruptos contaminen el histórico sin perder evidencia de fallas reales de los sensores.

### Criterios de aceptación

1. Dado un payload que cumple el esquema y los tipos esperados, cuando el backend lo valida con Pydantic, entonces continúa al procesamiento de dominio.
2. Dado un campo de sensor presente como `null` y acompañado por un estado de falla válido, cuando se valida, entonces el mensaje se acepta para registrar la falla del sensor.
3. Dado que falta un campo estructural obligatorio o existe un tipo incompatible/JSON malformado, cuando se valida, entonces el mensaje se rechaza y el error se registra sin detener el consumo de los demás mensajes.
4. Dado un `schema_version` no soportado, cuando se recibe, entonces el backend lo rechaza explícitamente o lo enruta a una compatibilidad versionada previamente definida.

**Prioridad:** Alta  
**Sprint actual:** SP-04  
**Story Points actuales:** 3  
**Dependencias:** HU-05, HU-12  
**Justificación dentro de la tesis:** Valida el contrato de HU-05 antes de HU-17/HU-18/HU-22.

## HU-16 — Disponibilidad segura del modelo IA

**EPIC:** EP03 — Procesamiento e Inteligencia Artificial  
**Rol:** Administrador del sistema  
**User Story:**

> Como Administrador del sistema, quiero que el servicio cargue únicamente la versión aprobada e íntegra del modelo Random Forest y continúe monitoreando si el modelo no está disponible, para mantener la clasificación cuando sea posible sin convertir una falla de IA en una caída del monitoreo térmico.

### Criterios de aceptación

1. Dado que existe una versión de modelo marcada como activa y aprobada, cuando FastAPI inicia mediante `lifespan`, entonces verifica la integridad del artefacto antes de deserializarlo y lo carga una sola vez para reutilizarlo.
2. Dado que el artefacto falta, está corrupto o no coincide con el hash aprobado, cuando inicia el servicio, entonces `model_status` queda como no disponible y no se ejecuta inferencia con ese archivo.
3. Dado que la IA está no disponible, cuando llegan lecturas válidas, entonces la ingesta, persistencia, regla directa 2–8 °C, alertas críticas basadas en rango e histórico continúan operando.
4. Dado que se activa otra versión aprobada, cuando se realiza la transición operativa, entonces el cambio queda auditado y no se acepta un artefacto no verificado.

**Prioridad:** Alta  
**Sprint actual:** SP-05  
**Story Points actuales:** 5  
**Dependencias:** HU-46  
**Justificación dentro de la tesis:** Disponibilidad/integridad del artefacto; HU-46 gobierna versión y HU-18 ejecuta inferencia.

## HU-17 — Consideración de la evolución térmica y variables contextuales en la clasificación

**EPIC:** EP03 — Procesamiento e Inteligencia Artificial  
**Rol:** farmacéutico responsable  
**User Story:**

> Como farmacéutico responsable, quiero que la clasificación considere la evolución térmica reciente y las variables contextuales disponibles, para identificar condiciones preventivas antes de que ocurra una excursión térmica.

### Criterios de aceptación

1. **Escenario 1 (cálculo completo de la matriz operativa):** Dado que se recibe un modelo de datos validado por Pydantic, cuando se invoca el servicio de extracción de dominio, entonces se calcula la matriz completa de variables operativas (gradiente térmico, ventana móvil, delta térmico, vector de características) necesarias para alimentar al clasificador, manteniendo paridad entre entrenamiento y servicio (*training-serving parity*).
2. **Escenario 2 (historial insuficiente — sin imputación de valores no entrenados):** Dado que el nodo acaba de conectarse y no hay suficiente historial térmico reciente para calcular una variable derivada indispensable (ej. tendencia térmica móvil), cuando se intenta construir la matriz, entonces el sistema **no inyecta valores neutros por defecto** (pues producirían *training-serving skew* frente a un modelo que no fue entrenado con esos valores); en su lugar, marca explícitamente el estado como `no_clasificable` y la lectura se evalúa únicamente por la regla directa de rango térmico 2–8 °C (HU-21).
3. **Escenario 3 (valores atípicos / outliers):** Dado que una variable de entrada (ej. 80 °C en el refrigerador) está fuera de los rangos físicamente plausibles para una operación normal, cuando se construye la matriz de características, entonces el registro se marca preventivamente como anomalía de hardware y se excluye de la inferencia estándar, aplicándose la regla directa de rango térmico.

**Prioridad:** Alta  
**Sprint actual:** SP-05  
**Story Points actuales:** 5  
**Dependencias:** HU-15  
**Justificación dentro de la tesis:** Prepara las características; HU-18 clasifica.

## HU-18 — Clasificación de Riesgo Térmico (Random Forest)

**EPIC:** EP03 — Procesamiento e Inteligencia Artificial  
**Rol:** Químico Farmacéutico responsable  
**User Story:**

> Como Químico Farmacéutico responsable, quiero que el sistema clasifique cada lectura elegible mediante el modelo Random Forest, para obtener un nivel de riesgo térmico Normal, Riesgo Preventivo o Excursión Crítica junto con la evidencia de la inferencia.

### Criterios de aceptación

1. Dado un vector de características válido, cuando el Random Forest realiza `predict_proba`/predicción, entonces se almacena la clase del modelo, las probabilidades disponibles, `model_version` y timestamp de inferencia.
2. Dado que la entrada no es apta para el modelo, cuando se intenta clasificar, entonces el estado IA queda como `no_clasificable` sin perder la lectura original.
3. Dado que la confianza está por debajo del umbral operativo documentado, cuando se presenta el resultado, entonces se marca para revisión sin falsificar una mayor certeza.
4. La política que transforma `model_class` y la regla directa de 2–8 °C en `effective_risk` debe ser determinista y permanecer versionada/documentada.

**Prioridad:** Alta  
**Sprint actual:** SP-05  
**Story Points actuales:** 5  
**Dependencias:** HU-16, HU-17  
**Justificación dentro de la tesis:** Produce `model_class`; HU-20 genera prevención, HU-34 presenta `effective_risk` y HU-47 audita inferencia.

## HU-19 — Emisión de Eventos SSE (Flujo Unidireccional)

**EPIC:** EP03 — Procesamiento e Inteligencia Artificial  
**Rol:** técnico de farmacia  
**User Story:**

> Como técnico de farmacia, quiero ver en tiempo real la lectura procesada y la clasificación de riesgo en el dashboard, para monitorear incidentes sin recargar la interfaz.

### Criterios de aceptación

1. **Escenario 1 (cliente conectado):** Dado que se persiste la lectura y su clasificación, cuando existen clientes web activos (técnicos/farmacéuticos), entonces el servidor despacha el evento JSON vía SSE a todos los suscriptores al instante.
2. **Escenario 2 (sin clientes conectados):** Dado que no hay clientes suscritos al canal SSE al momento de procesar la telemetría, cuando se intenta emitir el evento, entonces el despacho se omite para ahorrar recursos, pero el dato no se pierde ya que queda disponible en PostgreSQL para consultas históricas.
3. **Escenario 3 (desconexión abrupta del cliente):** Dado que un cliente cierra el navegador o pierde red abruptamente, cuando el servidor FastAPI detecta la ruptura del canal SSE, entonces libera los recursos de memoria y cierra el generador asíncrono sin afectar la transmisión a los demás clientes conectados.

**Prioridad:** Media  
**Sprint actual:** SP-05  
**Story Points actuales:** 3  
**Dependencias:** HU-18, HU-22  
**Justificación dentro de la tesis:** Emisión servidor; HU-32 consumo/actualización cliente.

## HU-20 — Generación de Alerta Preventiva

**EPIC:** EP03 — Procesamiento e Inteligencia Artificial  
**Rol:** Químico Farmacéutico responsable  
**User Story:**

> Como Químico Farmacéutico responsable, quiero que el sistema genere una alerta cuando la clasificación efectiva indique Riesgo Preventivo, para intervenir antes de que la condición evolucione hacia una excursión térmica.

### Criterios de aceptación

1. Dado que `effective_risk` resulta Riesgo Preventivo, cuando se procesa la lectura, entonces se crea o actualiza una alerta preventiva asociada al dispositivo y lectura que la originó.
2. Dado que ya existe una alerta preventiva activa para el mismo incidente, cuando llegan nuevas clasificaciones equivalentes, entonces se evita crear alertas duplicadas y se actualiza su seguimiento.
3. Dado que el riesgo vuelve a Normal durante la ventana de normalización configurada, cuando se cumple esa condición, entonces la alerta preventiva cambia al estado de resolución definido conservando su historial.
4. Cada transición de estado relevante queda auditada/trazada según las historias de trazabilidad correspondientes.

**Prioridad:** Media  
**Sprint actual:** SP-07  
**Story Points actuales:** 5  
**Dependencias:** HU-18, HU-22  
**Justificación dentro de la tesis:** Genera alerta preventiva; HU-23 gestiona seguimiento de críticas, no preventivas.

## HU-21 — Generación de Alerta Crítica (Regla directa 2–8 °C)

**EPIC:** EP03 — Procesamiento e Inteligencia Artificial  
**Rol:** Químico Farmacéutico responsable  
**User Story:**

> Como Químico Farmacéutico responsable, quiero que el sistema registre una excursión térmica crítica cuando una temperatura válida esté fuera del rango de 2 °C a 8 °C, para activar el seguimiento y las acciones correctivas correspondientes sin depender de la predicción del modelo.

### Criterios de aceptación

1. Dado que una temperatura válida es menor que 2 °C o mayor que 8 °C, cuando el motor de dominio la evalúa, entonces registra una excursión crítica confirmada independientemente de la clase devuelta por la IA.
2. Dado que la temperatura retorna al rango 2–8 °C, cuando llega una nueva lectura válida, entonces se registra el retorno sin borrar la excursión original.
3. Dado que la excursión permanece sin atención más allá del tiempo operativo configurado, cuando se evalúa su seguimiento, entonces el dashboard mantiene la alerta como pendiente de acción.
4. Una clasificación IA crítica con temperatura aún dentro del rango se conserva como resultado del modelo, pero no se etiqueta como 'excursión confirmada' salvo que la regla directa también se cumpla.

**Prioridad:** Alta  
**Sprint actual:** SP-07  
**Story Points actuales:** 3  
**Dependencias:** HU-15, HU-22  
**Justificación dentro de la tesis:** Confirma excursión por rango; HU-23 sigue su atención y HU-27 documenta acción.

## HU-22 — Persistencia del historial térmico

**EPIC:** EP04 — Persistencia y Trazabilidad Verificable  
**Rol:** Químico Farmacéutico responsable  
**User Story:**

> Como Químico Farmacéutico responsable, quiero que toda telemetría válida se conserve en PostgreSQL junto con su clasificación de IA cuando esté disponible, para mantener un histórico térmico completo que siga siendo utilizable incluso durante una indisponibilidad temporal del modelo.

### Criterios de aceptación

1. Dado un payload válido, cuando el backend lo acepta, entonces persiste la telemetría en PostgreSQL con `device_id`, `reading_id`, timestamps, valores de sensores y estados de calidad, aunque la IA esté temporalmente no disponible.
2. Dado que existe clasificación IA, cuando se persiste o actualiza el registro asociado, entonces se conserva también la clase, probabilidades pertinentes y `model_version` sin sustituir la telemetría original.
3. Dado que PostgreSQL no está disponible, cuando falla el commit, entonces no se envía el acuse lógico de persistencia al dispositivo y la lectura puede volver a sincronizarse desde LittleFS.
4. Dado que se recibe un `reading_id` ya persistido para el mismo dispositivo, cuando se procesa nuevamente, entonces no se genera una lectura duplicada.

**Prioridad:** Alta  
**Sprint actual:** SP-06  
**Story Points actuales:** 5  
**Dependencias:** HU-15  
**Justificación dentro de la tesis:** Persistencia fuente para HU-31/HU-36/HU-38 y soporte del ACK lógico de HU-07.

## HU-23 — Acuse y Seguimiento Operativo de Alertas Críticas

**EPIC:** EP05 — Monitoreo Web, Alertas y Reportes  
**Rol:** farmacéutico responsable  
**User Story:**

> Como farmacéutico responsable, quiero reconocer una alerta crítica y hacerle seguimiento en el dashboard hasta que se registre una acción correctiva, para asegurar que ninguna excursión quede sin atención.

### Criterios de aceptación

1. **Escenario 1 (reconocimiento de la alerta):** Dado que se genera una excursión crítica (HU-21) en estado PENDIENTE, cuando el farmacéutico la visualiza en el dashboard y la marca como reconocida, entonces el estado pasa a RECONOCIDA y se registra quién la reconoció y cuándo (identidad extraída del JWT), trazado en la cadena global.
2. **Escenario 2 (seguimiento hasta la atención):** Dado que una alerta está RECONOCIDA, mientras no se registre una acción correctiva, entonces se muestra como pendiente de atención; cuando se registra la acción correctiva (HU-27), entonces el estado pasa a ATENDIDA.
3. **Escenario 3 (máquina de estados y visibilidad):** Dado que el flujo de estados es PENDIENTE → RECONOCIDA → ATENDIDA, cuando hay alertas en PENDIENTE o RECONOCIDA, entonces el dashboard las destaca para garantizar que ninguna excursión quede sin atención.

**Prioridad:** Alta  
**Sprint actual:** SP-08  
**Story Points actuales:** 8  
**Dependencias:** HU-21, HU-39, HU-41  
**Justificación dentro de la tesis:** Gestiona estado de una alerta crítica; HU-27 registra la acción concreta que la atiende.

## HU-24 — Hash SHA-256 canónico por evento

**EPIC:** EP04 — Persistencia y Trazabilidad Verificable  
**Rol:** Responsable de auditoría  
**User Story:**

> Como Responsable de auditoría, quiero que cada evento trazable disponga de un hash SHA-256 calculado sobre una representación canónica y reproducible de sus datos, para verificar posteriormente si el contenido almacenado fue alterado.

### Criterios de aceptación

1. Dado un evento trazable válido, cuando se crea su registro, entonces se calcula `current_hash = SHA-256(previous_hash + timestamp_UTC_normalizado + canonical_json(payload))` con una serialización canónica documentada.
2. Dado el mismo `previous_hash`, timestamp y payload canónico, cuando se recalcula el hash, entonces se obtiene exactamente el mismo resultado.
3. Dado que un valor del payload cambia después del registro, cuando se recalcula, entonces el hash resultante difiere del almacenado.
4. Los valores nulos se representan de acuerdo con el esquema canónico del evento; no se transforman arbitrariamente entre `null`, vacío o ausencia durante la verificación.

**Prioridad:** Alta  
**Sprint actual:** SP-06  
**Story Points actuales:** 5  
**Dependencias:** HU-22  
**Justificación dentro de la tesis:** Define hash de cada evento; HU-25 define encadenamiento y HU-28 su aplicación a acciones humanas.

## HU-25 — Encadenamiento con Previous Hash (Ledger Centralizado)

**EPIC:** EP04 — Persistencia y Trazabilidad Verificable  
**Rol:** responsable de auditoría  
**User Story:**

> Como responsable de auditoría, quiero que cada nuevo registro incluya como previous_hash el hash resultante del evento anterior en traceability_records, para formar un registro encadenado cuya integridad sea verificable secuencialmente.

### Criterios de aceptación

1. **Escenario 1 (encadenamiento correcto):** Dado que se guarda un nuevo evento validado, cuando se calcula su hash, entonces el registro toma como `previous_hash(N)` el **hash resultante (`current_hash`) del registro inmediatamente anterior (N-1)** —es decir, `previous_hash(N) = current_hash(N-1)`— y calcula su propio `current_hash(N) = SHA-256(previous_hash(N) + timestamp(N) + canonical_payload(N))`, garantizando la dependencia temporal. No se encadena el previous_hash del registro anterior.
2. **Escenario 2 (primer evento de la cadena / Génesis):** Dado que el sistema se inicializa y no existe un registro anterior en la tabla, cuando se calcula su hash, entonces se utiliza un valor semilla inicial predefinido de 256 bits (bloque génesis) en lugar de un previous_hash real.
3. **Escenario 3 (resolución de inserciones concurrentes):** Dado que dos o más eventos (de distintos refrigeradores) intentan registrarse casi simultáneamente en PostgreSQL, cuando se determina el orden de la cadena por transacciones atómicas, entonces el sistema garantiza que cada nuevo hash referencie de forma única y consistente al hash inmediatamente anterior consolidado, evitando bifurcaciones de la cadena.

**Prioridad:** Alta  
**Sprint actual:** SP-06  
**Story Points actuales:** 8  
**Dependencias:** HU-24  
**Justificación dentro de la tesis:** Encadena registros; HU-26 verifica globalmente y HU-37 verifica un segmento consultado.

## HU-26 — Verificación global de integridad

**EPIC:** EP04 — Persistencia y Trazabilidad Verificable  
**Rol:** Responsable de auditoría  
**User Story:**

> Como Responsable de auditoría, quiero poder ejecutar una verificación completa de la cadena global de trazabilidad, para identificar el primer punto de ruptura y determinar si la evidencia registrada conserva integridad verificable.

### Criterios de aceptación

1. Dado que el responsable de auditoría inicia una verificación global, cuando el backend recorre la cadena desde el génesis o ancla definida, entonces recalcula hashes y valida cada relación `previous_hash`.
2. Dado que se detecta una inconsistencia, cuando concluye el análisis, entonces se informa al menos el primer registro afectado, su timestamp y el tipo de discrepancia sin ocultar los resultados posteriores.
3. Dado que la verificación se interrumpe, cuando se reintenta, entonces puede reiniciarse o reanudarse de forma controlada sin bloquear la ingesta normal.
4. El resultado se presenta como verificación de integridad de registros, no como certificación sanitaria.

**Prioridad:** Alta  
**Sprint actual:** SP-06  
**Story Points actuales:** 8  
**Dependencias:** HU-24, HU-25  
**Justificación dentro de la tesis:** Verificación global; HU-37 ofrece vista acotada por dispositivo/periodo.

## HU-27 — Persistencia de Acción Correctiva

**EPIC:** EP04 — Persistencia y Trazabilidad Verificable  
**Rol:** Químico Farmacéutico (Regente)  
**User Story:**

> Como Químico Farmacéutico (Regente), quiero registrar la acción correctiva aplicada tras una alerta, para dejar constancia operativa trazable de la atención de la alerta conforme a las Buenas Prácticas de Oficina Farmacéutica (BPOF).

### Criterios de aceptación

1. **Escenario 1 (registro exitoso):** Dado que hay una alerta activa en el dashboard, cuando el usuario ingresa la justificación y la acción tomada, entonces el sistema extrae su identidad desde el token JWT, asocia el evento al ID de la alerta y la marca como atendida.
2. **Escenario 2 (alerta ya resuelta):** Dado que el usuario intenta registrar una acción sobre una alerta que acaba de ser cerrada por otro compañero, cuando envía la solicitud, entonces el sistema la rechaza mediante concurrencia e informa visualmente que la alerta ya no está vigente.
3. **Escenario 3 (campos incompletos):** Dado que no se completa la descripción mínima requerida de la acción, cuando intenta enviar el formulario, entonces el sistema bloquea el envío en el frontend y solicita completar la información obligatoria.

**Prioridad:** Media  
**Sprint actual:** SP-07  
**Story Points actuales:** 5  
**Dependencias:** HU-21, HU-23, HU-39, HU-41  
**Justificación dentro de la tesis:** Contenido de la acción; HU-28 preserva su auditabilidad y rectificaciones.

## HU-28 — Trazabilidad de acciones correctivas

**EPIC:** EP04 — Persistencia y Trazabilidad Verificable  
**Rol:** Responsable de auditoría  
**User Story:**

> Como Responsable de auditoría, quiero que las acciones correctivas y sus eventuales rectificaciones conserven el registro original y queden incorporadas a la cadena de trazabilidad, para detectar modificaciones posteriores y reconstruir la secuencia real de decisiones operativas.

### Criterios de aceptación

1. Dado que se registra una acción correctiva, cuando se incorpora a la evidencia, entonces genera un evento de trazabilidad asociado a la acción y a la alerta correspondiente.
2. Dado que se necesita corregir una justificación previamente registrada, cuando el usuario solicita la corrección, entonces se crea un nuevo evento que referencia al anterior en lugar de sobrescribirlo.
3. Dado que el responsable de auditoría verifica el ciclo de una alerta, cuando consulta sus acciones y rectificaciones, entonces puede reconstruir el orden temporal y comprobar su integridad mediante la cadena global.

**Prioridad:** Media  
**Sprint actual:** SP-07  
**Story Points actuales:** 3  
**Dependencias:** HU-24, HU-25, HU-27  
**Justificación dentro de la tesis:** Especialización de trazabilidad para acciones correctivas, apoyada en HU-24/HU-25.

## HU-29 — Respaldo Programado y Prueba de Restauración Controlada

**EPIC:** EP04 — Persistencia y Trazabilidad Verificable  
**Rol:** administrador de infraestructura  
**User Story:**

> Como administrador de infraestructura, quiero configurar el respaldo programado de la base de datos PostgreSQL y ejecutar pruebas de restauración controladas, para asegurar la continuidad operativa y verificar que la evidencia histórica puede recuperarse efectivamente ante caídas o desastres.

### Criterios de aceptación

1. **Escenario 1 (configuración del respaldo disponible):** Dado que la base de datos PostgreSQL se aloja en Railway, cuando el Administrador configura el mecanismo de respaldo disponible en el plan contratado, entonces el respaldo queda habilitado y se documenta la periodicidad y retención efectivas (los respaldos diarios de Railway tienen su propia retención, p. ej. 6 días; los esquemas Weekly, Monthly y PITR dependen de la configuración). La política concreta no se fija en esta historia.
2. **Escenario 2 (prueba de restauración controlada):** Dado que se requiere verificar la efectividad del respaldo, cuando se ejecuta periódicamente un simulacro de restauración en un entorno aislado, entonces se verifica que las tablas thermal_readings, traceability_records y audit_logs, los usuarios y la metadata crítica se recuperan al 100 %, y se registra el resultado de la prueba.
3. **Escenario 3 (fallo del backup):** Dado que el proceso automático de respaldo falla (ej. por límites de almacenamiento), cuando el servidor detecta el error de la tarea programada, entonces se registra una alerta operativa crítica en el log del backend para intervención manual del administrador.

**Prioridad:** Baja  
**Sprint actual:** SP-08  
**Story Points actuales:** 3  
**Dependencias:** HU-22  
**Justificación dentro de la tesis:** Recuperabilidad de PostgreSQL; no sustituye la verificación de integridad HU-26.

## HU-30 — Alerta de Vencimiento de Calibración de Sensores

**EPIC:** EP04 — Persistencia y Trazabilidad Verificable  
**Rol:** Químico Farmacéutico responsable  
**User Story:**

> Como Químico Farmacéutico responsable, quiero que el sistema registre la vigencia de calibración de los sensores y advierta antes de su vencimiento, para evitar utilizar sin advertencia mediciones provenientes de sensores con documentación de calibración vencida.

### Criterios de aceptación

1. Dado un sensor registrado, cuando el responsable ingresa `calibrated_at`, `valid_until` y referencia del certificado, entonces la información queda asociada al dispositivo y trazada.
2. Dado que se aproxima `valid_until` dentro del margen de aviso configurado, cuando el sistema evalúa la vigencia, entonces muestra una advertencia operativa diferenciada de las alertas térmicas.
3. Dado que la calibración está vencida, cuando se visualizan nuevas lecturas, entonces se identifica claramente esa condición documental hasta que se registre una nueva calibración.
4. El margen de aviso es configurable; la periodicidad de calibración proviene del certificado/procedimiento aplicable y no se inventa como obligación fija.

**Prioridad:** Media  
**Sprint actual:** SP-07  
**Story Points actuales:** 5  
**Dependencias:** HU-43  
**Justificación dentro de la tesis:** Estado documental de calibración; HU-49 conserva el historial de sus cambios.

## HU-31 — Visualización de Tendencia Térmica

**EPIC:** EP05 — Monitoreo Web, Alertas y Reportes  
**Rol:** técnico de farmacia  
**User Story:**

> Como técnico de farmacia, quiero visualizar una gráfica de temperatura con umbrales fijos de 2 °C a 8 °C, para evaluar rápidamente la tendencia térmica del refrigerador.

### Criterios de aceptación

1. **Escenario 1 (carga con datos):** Dado que abro el dashboard de un refrigerador y existen lecturas históricas en la base de datos, cuando el componente de React 19 carga la instancia de ECharts, entonces se renderiza la línea histórica superpuesta sobre zonas de colores (rojo/verde) que delimitan visualmente el rango objetivo de 2–8 °C.
2. **Escenario 2 (sin datos históricos):** Dado que un dispositivo IoT recién registrado no ha transmitido lecturas aún, cuando se abre su vista de gráfica, entonces el sistema muestra un "estado vacío" (empty state) informativo indicando que está a la espera de datos, en lugar de un lienzo en blanco confuso.
3. **Escenario 3 (gran volumen de puntos):** Dado que el rango de fechas seleccionado contiene miles de lecturas continuas, cuando ECharts renderiza el gráfico, entonces se aplica muestreo o agregación visual (data downsampling) de forma automática para mantener la fluidez de la interfaz sin perder la forma real de la tendencia térmica.

**Prioridad:** Alta  
**Sprint actual:** SP-08  
**Story Points actuales:** 8  
**Dependencias:** HU-22  
**Justificación dentro de la tesis:** Visualiza histórico persistido por HU-22; no sustituye filtros avanzados de HU-36.

## HU-32 — Actualización en Tiempo Real del Dashboard (SSE autenticado)

**EPIC:** EP05 — Monitoreo Web, Alertas y Reportes  
**Rol:** técnico de farmacia  
**User Story:**

> Como técnico de farmacia, quiero que el dashboard reciba actualizaciones del backend mediante Server-Sent Events (SSE) autenticados, para ver el último punto de la gráfica y los KPI en tiempo real.

### Criterios de aceptación

1. **Escenario 1 (actualización asíncrona en tiempo real):** Dado que la conexión SSE hacia la API FastAPI está activa, cuando llega un nuevo evento JSON con telemetría fresca, entonces el gestor de estado global del frontend actualiza automáticamente los componentes dependientes sin necesidad de recargar la página.
2. **Escenario 2 (pérdida de conexión SSE):** Dado que la red local de la farmacia o el enlace con el servidor falla, cuando el cliente React detecta la ruptura del flujo continuo, entonces muestra un indicador visual en amarillo de "reconectando..." e intenta restablecer el canal SSE automáticamente mediante un backoff de reintentos.
3. **Escenario 3 (múltiples pestañas):** Dado que el usuario tiene el dashboard abierto en más de una pestaña o dispositivo, cuando el servidor despacha un evento SSE, entonces todas las instancias del cliente reflejan la actualización instantáneamente de forma independiente y concurrente.

**Prioridad:** Alta  
**Sprint actual:** SP-09  
**Story Points actuales:** 5  
**Dependencias:** HU-19, HU-39, HU-40  
**Justificación dentro de la tesis:** Cliente en tiempo real que consume la emisión HU-19.

## HU-33 — Tarjetas de KPI (Estado Actual y Tolerancia a Fallo)

**EPIC:** EP05 — Monitoreo Web, Alertas y Reportes  
**Rol:** técnico de farmacia  
**User Story:**

> Como técnico de farmacia, quiero visualizar el último valor de temperatura (DS18B20) y humedad ambiental (SHT31) en tarjetas KPI, para obtener una lectura rápida del estado del equipo a un solo vistazo.

### Criterios de aceptación

1. **Escenario 1 (datos disponibles y vigentes):** Dado que el dashboard web se renderiza y la caché cuenta con datos válidos, cuando se cargan los componentes de las tarjetas KPI, entonces se muestran los valores numéricos actuales exactos en °C y % HR.
2. **Escenario 2 (última lectura desactualizada por red):** Dado que el último timestamp recibido supera el umbral máximo configurable de inactividad (freshness), cuando se muestran las tarjetas, entonces se oscurecen ligeramente o incluyen un ícono de advertencia indicando la antigüedad del dato.
3. **Escenario 3 (lectura en error de sensor):** Dado que el flujo de datos indica que un sensor en particular está fallando (valor null), cuando React intenta renderizar su KPI correspondiente, entonces muestra el texto "Sin lectura disponible" o "Falla de sensor" en lugar de mostrar información potencialmente engañosa que confunda al personal.

**Prioridad:** Media  
**Sprint actual:** SP-09  
**Story Points actuales:** 3  
**Dependencias:** HU-13, HU-22, HU-32  
**Justificación dentro de la tesis:** Resumen actual/frescura; HU-31 ofrece tendencia histórica y HU-34 riesgo.

## HU-34 — Visualización del riesgo efectivo

**EPIC:** EP05 — Monitoreo Web, Alertas y Reportes  
**Rol:** Químico Farmacéutico responsable  
**User Story:**

> Como Químico Farmacéutico responsable, quiero que el dashboard muestre un estado visual de riesgo efectivo que combine la regla crítica de 2–8 °C con la clasificación del Random Forest, para priorizar la atención operativa sin ocultar cuándo una excursión fue confirmada directamente por la temperatura.

### Criterios de aceptación

1. Dado que la temperatura está dentro de 2–8 °C y el modelo clasifica Normal, cuando se actualiza el dashboard, entonces el riesgo efectivo se muestra como Normal.
2. Dado que la temperatura está dentro de 2–8 °C y el modelo clasifica Riesgo Preventivo, cuando se actualiza, entonces se muestra Riesgo Preventivo.
3. Dado que la temperatura válida está fuera de 2–8 °C, cuando se calcula el riesgo efectivo, entonces el estado es Excursión Crítica confirmada independientemente de la predicción IA.
4. Dado que el modelo devuelve Excursión Crítica con temperatura todavía dentro del rango, cuando se presenta el resultado, entonces se muestra la clasificación IA de forma diferenciada y no se afirma que exista una excursión confirmada hasta que la regla directa se cumpla.

**Prioridad:** Alta  
**Sprint actual:** SP-09  
**Story Points actuales:** 5  
**Dependencias:** HU-18, HU-21, HU-32  
**Justificación dentro de la tesis:** Presenta `effective_risk`; se alimenta de HU-18 y HU-21.

## HU-35 — Notificación UI de Puerta Abierta (MC-38, opcional)

**EPIC:** EP05 — Monitoreo Web, Alertas y Reportes  
**Rol:** Técnico de farmacia  
**User Story:**

> Como Técnico de farmacia, quiero que el dashboard advierta cuando una puerta monitoreada por MC-38 permanezca abierta más allá del umbral configurado, para reducir aperturas prolongadas que puedan contribuir a una desviación térmica.

### Criterios de aceptación

1. Dado que el dispositivo tiene MC-38 instalado y la puerta se abre, cuando el evento llega al dashboard, entonces se muestra el estado de puerta abierta.
2. Dado que la apertura supera el umbral de duración configurado, cuando continúa abierta, entonces la interfaz destaca la advertencia y muestra el tiempo transcurrido.
3. Dado que la puerta se cierra, cuando se recibe el nuevo estado, entonces la advertencia activa desaparece conservando el evento histórico.
4. Dado que el dispositivo no tiene MC-38, cuando se muestra su tarjeta, entonces no se presenta una falsa alerta de puerta.

**Prioridad:** Media  
**Sprint actual:** SP-09  
**Story Points actuales:** 3  
**Dependencias:** HU-04, HU-32  
**Justificación dentro de la tesis:** Presentación de evento producido por HU-04.

## HU-36 — Panel de Filtros de Historial y Auditoría

**EPIC:** EP05 — Monitoreo Web, Alertas y Reportes  
**Rol:** Químico Farmacéutico (Regente)  
**User Story:**

> Como Químico Farmacéutico (Regente), quiero consultar y filtrar los datos históricos por fecha y hora, para analizar eventos retrospectivos, excursiones pasadas y generar evidencia para auditorías.

### Criterios de aceptación

1. **Escenario 1 (filtro aplicado exitosamente):** Dado que entro al módulo de historial, cuando selecciono un rango temporal en el DatePicker (basado en shadcn/ui) y ejecuto la búsqueda, entonces el frontend lanza una petición REST a la API de FastAPI, que es la que consulta PostgreSQL; la respuesta alimenta tanto la gráfica de Apache ECharts como la tabla de datos con la información exacta de ese periodo.
2. **Escenario 2 (rango temporal sin datos):** Dado que el rango de fechas seleccionado no contiene telemetría registrada (ej. equipo apagado), cuando se responde la consulta, entonces el sistema captura la matriz vacía y muestra un componente ilustrativo con el mensaje "Sin datos registrados en el rango seleccionado".
3. **Escenario 3 (validación de rango inválido en frontend y backend):** Dado que selecciono un rango incongruente donde la fecha de fin es cronológicamente anterior a la fecha de inicio, cuando intento aplicar el filtro, entonces la validación del formulario (Frontend) bloquea la consulta y muestra un mensaje de error; y el mismo rango temporal se valida nuevamente en FastAPI antes de ejecutar la consulta, de modo que una llamada directa a la API también sea rechazada.

**Prioridad:** Media  
**Sprint actual:** SP-09  
**Story Points actuales:** 5  
**Dependencias:** HU-22, HU-39, HU-41  
**Justificación dentro de la tesis:** Consulta histórica; HU-37 añade verificación criptográfica al periodo.

## HU-37 — Verificación de Integridad de Telemetría por Dispositivo y Periodo

**EPIC:** EP04 — Persistencia y Trazabilidad Verificable  
**Rol:** farmacéutico responsable  
**User Story:**

> Como farmacéutico responsable, quiero solicitar la verificación de integridad de la telemetría térmica de un dispositivo en un periodo determinado, para confirmar que los registros de cadena de frío no han sido alterados y contar con evidencia verificable acotada a la consulta.

### Criterios de aceptación

1. **Escenario 1 (verificación de un segmento de la cadena global):** Dado que el farmacéutico selecciona un dispositivo y un rango temporal en el dashboard, cuando ejecuta la verificación, entonces el backend identifica el segmento temporal correspondiente en la cadena global, ancla la verificación en el registro inmediatamente anterior a dicho segmento, verifica secuencialmente la consistencia matemática de todos los bloques del intervalo y filtra en la interfaz el estado de las lecturas pertenecientes al dispositivo consultado. No se verifican eventos aislados ignorando los bloques intermedios de la cadena.
2. **Escenario 2 (detección de lectura alterada en el rango):** Dado que alguna lectura del rango seleccionado fue modificada tras su registro, cuando se recalcula su hash dentro de la verificación del segmento, entonces el sistema la señala como "Corrupta" con su ID y timestamp, y continúa evaluando el resto del segmento.
3. **Escenario 3 (constancia verificable de la revisión):** Dado que la verificación concluye, cuando se confirma el resultado, entonces se registra un evento append-only tipo VERIFICACION_INTEGRIDAD en traceability_records con dispositivo, periodo, resultado global y usuario solicitante, encadenado con hash SHA-256.

**Prioridad:** Media  
**Sprint actual:** SP-09  
**Story Points actuales:** 5  
**Dependencias:** HU-24, HU-25, HU-26, HU-36  
**Justificación dentro de la tesis:** Vista de integridad acotada que se apoya en HU-24/HU-25/HU-26.

## HU-38 — Reporte PDF de evidencia del monitoreo

**EPIC:** EP05 — Monitoreo Web, Alertas y Reportes  
**Rol:** Responsable de auditoría  
**User Story:**

> Como Responsable de auditoría, quiero poder exportar un reporte PDF del monitoreo de un dispositivo y periodo seleccionado, para disponer de evidencia revisable de lecturas, alertas, acciones y resultado de verificación de integridad sin afirmar cumplimiento sanitario integral.

### Criterios de aceptación

1. Dado un dispositivo y periodo autorizados, cuando el responsable solicita el reporte, entonces FastAPI genera un PDF con identificación del equipo, periodo, resumen térmico, alertas, acciones correctivas y metadatos de instalación pertinentes.
2. Dado que el periodo fue verificado, cuando se genera el PDF, entonces incluye el resultado y fecha de la verificación de integridad y una referencia/digest reproducible; no es obligatorio imprimir el hash de cada una de decenas de miles de lecturas en el cuerpo principal.
3. Dado un volumen elevado de datos, cuando se genera el reporte, entonces el documento pagina o resume de forma consistente sin omitir las excursiones y eventos relevantes.
4. El documento se presenta como evidencia del monitoreo y la trazabilidad; no declara por sí mismo cumplimiento sanitario integral.

**Prioridad:** Media  
**Sprint actual:** SP-10  
**Story Points actuales:** 8  
**Dependencias:** HU-22, HU-23, HU-27, HU-37, HU-39, HU-41  
**Justificación dentro de la tesis:** Integra resultados de HU-22/HU-23/HU-27/HU-37 en una evidencia exportable.

## HU-39 — Autenticación de Usuarios (Login seguro)

**EPIC:** EP06 — Autenticación, Autorización y Seguridad Transversal  
**Rol:** Usuario autorizado  
**User Story:**

> Como Usuario autorizado, quiero poder autenticarse con sus credenciales antes de acceder a la plataforma, para impedir el acceso al dashboard y a las funciones protegidas a personas no autenticadas.

### Criterios de aceptación

1. Dado que un usuario registrado presenta credenciales válidas, cuando el backend las verifica contra una contraseña almacenada con un algoritmo apropiado como Argon2id, entonces emite una sesión/JWT con identidad y rol autorizado.
2. Dado un correo inexistente o contraseña incorrecta, cuando se rechaza el login, entonces el mensaje no revela cuál dato fue incorrecto y el intento queda auditado.
3. Dado que se supera el umbral configurado de intentos fallidos, cuando ocurren nuevos intentos, entonces se aplica el control de rate limiting/bloqueo temporal definido.
4. Dado un usuario desactivado, cuando intenta autenticarse, entonces no obtiene una sesión válida.

**Prioridad:** Alta  
**Sprint actual:** SP-01  
**Story Points actuales:** 8  
**Dependencias:** Sin dependencia funcional estricta.  
**Justificación dentro de la tesis:** Crea sesión; HU-40 la maneja y HU-41 autoriza operaciones.

## HU-40 — Gestión de sesión JWT en memoria

**EPIC:** EP06 — Autenticación, Autorización y Seguridad Transversal  
**Rol:** Usuario autorizado  
**User Story:**

> Como Usuario autorizado, quiero que mi sesión web reduzca la exposición del JWT evitando su persistencia en almacenamiento web permanente, para disminuir el riesgo de reutilización del token si el navegador o la aplicación cliente se ven comprometidos.

### Criterios de aceptación

1. Dado que el usuario inicia sesión, cuando la SPA conserva el JWT, entonces el token permanece en memoria de la aplicación y no se almacena en `localStorage` ni `sessionStorage`.
2. Dado que el contexto del navegador se cierra o la memoria de la aplicación se pierde, cuando el usuario vuelve a una ruta protegida, entonces debe autenticarse nuevamente.
3. Dado que el usuario cierra sesión, cuando se procesa el logout, entonces la aplicación elimina el token en memoria y las llamadas posteriores protegidas no lo reutilizan.
4. La estrategia de sesión se prueba mediante los casos de gestión de sesión definidos en el plan de seguridad; la historia no afirma inmunidad frente a XSS.

**Prioridad:** Alta  
**Sprint actual:** SP-02  
**Story Points actuales:** 5  
**Dependencias:** HU-39  
**Justificación dentro de la tesis:** Gestión de sesión JWT; HU-39 autentica y HU-45 revoca acceso.

## HU-41 — Gestión de Usuarios y Autorización RBAC

**EPIC:** EP06 — Autenticación, Autorización y Seguridad Transversal  
**Rol:** Administrador del sistema  
**User Story:**

> Como Administrador del sistema, quiero poder gestionar usuarios y roles y que cada operación protegida respete los permisos RBAC vigentes, para aplicar el principio de mínimo privilegio y evitar escaladas de privilegios.

### Criterios de aceptación

1. Dado un Administrador autorizado, cuando crea un usuario o asigna/modifica un rol, entonces la operación respeta la matriz `ADMINISTRADOR/FARMACEUTICO/TECNICO/AUDITOR` y queda auditada.
2. Dado un usuario con permiso suficiente, cuando accede a una función protegida, entonces tanto frontend como FastAPI permiten la operación conforme al rol vigente.
3. Dado un usuario sin permiso, cuando intenta invocar directamente una ruta/API protegida, entonces el backend rechaza la operación con el código correspondiente sin depender solo de React.
4. Dado que cambia el rol o estado de una cuenta con sesión activa, cuando realiza una nueva operación protegida, entonces el backend aplica el estado vigente mediante revalidación/versionado de sesión u otro mecanismo equivalente.

**Prioridad:** Alta  
**Sprint actual:** SP-02  
**Story Points actuales:** 8  
**Dependencias:** HU-39  
**Justificación dentro de la tesis:** Gobierno RBAC transversal usado por todas las funciones protegidas.

## HU-42 — Bitácora auditable vinculada a trazabilidad global

**EPIC:** EP06 — Autenticación, Autorización y Seguridad Transversal  
**Rol:** Responsable de auditoría  
**User Story:**

> Como Responsable de auditoría, quiero que las operaciones sensibles de los usuarios queden registradas en una bitácora de solo adición vinculada a la trazabilidad global, para reconstruir quién realizó cada acción, cuándo ocurrió y detectar alteraciones posteriores.

### Criterios de aceptación

1. Dado que ocurre una operación sensible definida —por ejemplo login, fallo de autenticación, cambio de rol, desactivación, exportación o cambio de configuración—, cuando FastAPI la procesa, entonces registra actor, timestamp, acción, resultado y recurso afectado.
2. Dado que el evento de auditoría se consolida, cuando se incorpora a la evidencia, entonces queda vinculado a la cadena global de trazabilidad mediante el mecanismo SHA-256/previous_hash.
3. Dado que un usuario intenta modificar o eliminar un evento ya consolidado mediante la aplicación, cuando se procesa la solicitud, entonces se rechaza y cualquier rectificación requiere un nuevo evento.
4. Dado que el auditor consulta la bitácora, cuando filtra por periodo, actor o tipo de acción, entonces obtiene una vista de solo lectura.

**Prioridad:** Media  
**Sprint actual:** SP-07  
**Story Points actuales:** 8  
**Dependencias:** HU-24, HU-25, HU-39, HU-41  
**Justificación dentro de la tesis:** Auditoría de acciones humanas; utiliza HU-24/HU-25 para integridad global.

## HU-43 — Gestión del ciclo de vida de dispositivos IoT

**EPIC:** EP01 — Adquisición de Datos, Resiliencia y Gestión Edge  
**Rol:** Administrador del sistema  
**User Story:**

> Como Administrador del sistema, quiero poder gestionar el alta, reemplazo y baja de los dispositivos IoT sin eliminar su historial previo, para mantener identificable el origen de las lecturas y conservar la continuidad administrativa y trazable ante cambios de hardware.

### Criterios de aceptación

1. Dado que se incorpora un nuevo nodo, cuando el Administrador lo da de alta, entonces se registra un `device_id` único, estado activo, sensores habilitados y metadatos básicos sin reutilizar la identidad de otro equipo.
2. Dado que un dispositivo es reemplazado, cuando se registra el cambio, entonces el nuevo `device_id` queda relacionado con el anterior mediante metadatos de reemplazo y el histórico previo permanece intacto.
3. Dado que un dispositivo se retira, cuando se confirma la baja, entonces queda inactivo y sus credenciales pueden revocarse sin eliminar telemetría ni trazabilidad histórica.
4. Dado cualquier alta, reemplazo o baja, cuando la operación se confirma, entonces genera un evento auditado/trazable y respeta RBAC.

**Prioridad:** Alta  
**Sprint actual:** SP-02  
**Story Points actuales:** 8  
**Dependencias:** HU-39, HU-41  
**Justificación dentro de la tesis:** Ciclo de vida del dispositivo; HU-51 agrega metadatos de instalación y HU-49 audita cambios.

## HU-44 — Rotación y Revocación de Credenciales IoT

**EPIC:** EP02 — Comunicación MQTT Segura  
**Rol:** administrador del sistema  
**User Story:**

> Como administrador del sistema, quiero revocar o rotar el token MQTT de un ESP32 cuyas credenciales se consideren comprometidas, sin alterar su historial de telemetría, para cortar de inmediato el acceso del nodo comprometido preservando la integridad del histórico ya registrado.

### Criterios de aceptación

1. **Escenario 1 (revocación de credencial comprometida):** Dado que el Administrador detecta que el token de un ESP32 pudo haber sido comprometido, cuando ejecuta la revocación desde la gestión de dispositivos, entonces el broker invalida la credencial de inmediato y todo intento posterior de conexión con ese token es rechazado con el reason code correspondiente de MQTT 5.0 (por ejemplo 0x87 *Not authorized* cuando aplique).
2. **Escenario 2 (rotación con continuidad de servicio):** Dado que se genera un nuevo token para el dispositivo y se aprovisiona en el ESP32, cuando el nodo se reconecta con la nueva credencial, entonces la telemetría histórica ya persistida permanece intacta y asociada al mismo device_id, sin modificaciones ni interrupciones en la cadena de trazabilidad.
3. **Escenario 3 (auditoría del evento):** Dado que se ejecuta una rotación o revocación, cuando se confirma la operación, entonces se registra un evento append-only en la bitácora de auditoría con user_id del administrador, device_id, motivo y timestamp, encadenado con hash SHA-256.

**Prioridad:** Alta  
**Sprint actual:** SP-04  
**Story Points actuales:** 5  
**Dependencias:** HU-10, HU-43, HU-39, HU-41  
**Justificación dentro de la tesis:** Ciclo de vida de credenciales IoT tras HU-10.

## HU-45 — Desactivación de Usuarios y Revocación de Acceso

**EPIC:** EP06 — Autenticación, Autorización y Seguridad Transversal  
**Rol:** Administrador del Sistema  
**User Story:**

> Como Administrador del Sistema, quiero desactivar las cuentas de personal cesado impidiendo nuevos logins y revocando las sesiones activas, pero preservando su user_id en los logs de auditoría, para revocar el acceso de forma efectiva sin romper la cadena de trazabilidad ni perder el histórico de acciones del usuario.

### Criterios de aceptación

1. **Escenario 1 (desactivación impide nuevos logins):** Dado que un miembro del personal cesa en sus funciones, cuando el Administrador desactiva su cuenta, entonces el campo users.is_active pasa a false y todo intento posterior de login se rechaza con el mensaje genérico de credenciales inválidas, registrándose el intento en la bitácora.
2. **Escenario 2 (revocación de sesiones activas):** Dado que el usuario desactivado aún tiene un JWT válido en una sesión abierta, cuando intenta cualquier operación protegida, entonces el backend vuelve a comprobar is_active (o la versión de sesión) en cada request y rechaza inmediatamente el token de una cuenta revocada.
3. **Escenario 3 (preservación del historial auditable):** Dado que el usuario desactivado tiene acciones registradas en audit_logs y traceability_records, cuando se ejecuta la desactivación, entonces esos registros se preservan intactos (solo lectura) conservando su user_id, de modo que la cadena de hash no se rompe ni se elimina evidencia histórica.
4. **Escenario 4 (auditoría de la desactivación):** Dado que se confirma la desactivación, cuando se registra la operación, entonces se genera un evento append-only tipo DESACTIVACION_USUARIO con user_id del usuario desactivado, user_id del administrador que ejecutó la acción, motivo y timestamp, encadenado con hash SHA-256.

**Prioridad:** Alta  
**Sprint actual:** SP-10  
**Story Points actuales:** 5  
**Dependencias:** HU-39, HU-41, HU-42  
**Justificación dentro de la tesis:** Ciclo de vida de cuentas humanas tras HU-39/HU-41.

## HU-46 — Gestión y Versionado Operativo del Modelo IA

**EPIC:** EP03 — Procesamiento e Inteligencia Artificial  
**Rol:** administrador del sistema  
**User Story:**

> Como administrador del sistema, quiero registrar en base de datos la versión activa del modelo Random Forest, su hash SHA-256 y los metadatos de reproducibilidad, asegurando que solo modelos aprobados clasifiquen en producción, para garantizar el gobierno y la observabilidad del modelo de IA, evitando que una versión no autorizada clasifique el riesgo térmico.

### Criterios de aceptación

1. **Escenario 1 (registro de una nueva versión aprobada):** Dado que se dispone de un modelo entrenado y aprobado, cuando el Administrador registra la nueva versión, entonces el backend almacena: identificador de versión, hash SHA-256 del archivo del modelo (model_hash), feature_schema_version, dataset_version, scikit_learn_version, python_version, trained_at y fecha de aprobación, y la marca como versión activa de producción.
2. **Escenario 2 (bloqueo de modelos no aprobados):** Dado que se intenta activar un modelo cuyo registro de aprobación no existe, cuando el sistema valida la activación, entonces rechaza la operación, mantiene activa la versión aprobada vigente y registra el evento en la bitácora.
3. **Escenario 3 (verificación de integridad antes de deserializar):** Dado que el servicio FastAPI va a cargar el modelo en memoria (HU-16), cuando se prepara la carga, entonces el backend calcula el hash SHA-256 de los bytes del artefacto y lo compara con el hash registrado para la versión activa **antes** de deserializar; si difiere, no carga el modelo, registra un error crítico y no sirve clasificaciones.
4. **Escenario 4 (historial de cambios de versión):** Dado que se cambia la versión activa, cuando se confirma el cambio, entonces se registra un evento append-only en traceability_records con versión anterior, versión nueva, usuario que realizó el cambio y timestamp, encadenado con hash SHA-256.

**Prioridad:** Alta  
**Sprint actual:** SP-05  
**Story Points actuales:** 8  
**Dependencias:** HU-39, HU-41  
**Justificación dentro de la tesis:** Gobierno de artefactos IA; HU-16 carga y HU-47 explica inferencias.

## HU-47 — Consulta de Trazabilidad de la Inferencia IA

**EPIC:** EP03 — Procesamiento e Inteligencia Artificial  
**Rol:** responsable de auditoría / farmacéutico  
**User Story:**

> Como responsable de auditoría / farmacéutico, quiero auditar qué versión específica del modelo y qué probabilidades generaron la clasificación de una lectura térmica determinada, para explicar y verificar las decisiones de la IA ante auditorías internas o consultas de la autoridad sanitaria.

### Criterios de aceptación

1. **Escenario 1 (consulta de la inferencia asociada a una lectura):** Dado que el auditor selecciona una lectura térmica del historial, cuando abre el detalle de la inferencia, entonces el sistema muestra: versión del modelo utilizada, vector de características de entrada, probabilidades por clase (normal, riesgo_preventivo, excursion_critica), clasificación final y timestamp de la inferencia.
2. **Escenario 2 (coherencia con el versionado del modelo):** Dado que una lectura fue clasificada con la versión X del modelo, cuando la auditoría cruza el dato con el versionado operativo (HU-46), entonces se confirma que la versión X era la versión activa y aprobada en el momento de la clasificación.
3. **Escenario 3 (acceso de solo lectura):** Dado que el auditor consulta la trazabilidad de la inferencia, cuando intenta modificar cualquier campo del registro, entonces el sistema lo impide (modo solo lectura) y registra el intento en la bitácora de auditoría.

**Prioridad:** Alta  
**Sprint actual:** SP-08  
**Story Points actuales:** 5  
**Dependencias:** HU-18, HU-46, HU-36  
**Justificación dentro de la tesis:** Consulta explicativa de una inferencia generada en HU-18 y versionada en HU-46.

## HU-48 — Visualización del Estado de Sincronización y Brechas Temporales

**EPIC:** EP05 — Monitoreo Web, Alertas y Reportes  
**Rol:** farmacéutico  
**User Story:**

> Como farmacéutico, quiero ver en el dashboard si un ESP32 tiene datos pendientes en LittleFS o si ocurrió una pérdida irrecuperable de datos por buffer lleno tras un corte prolongado, para conocer la completitud del registro térmico y actuar frente a brechas de información.

### Criterios de aceptación

1. **Escenario 1 (datos pendientes de sincronización):** Dado que un ESP32 mantiene lecturas acumuladas en LittleFS pendientes de sincronizar (HU-06/HU-07), cuando el dashboard muestra el estado del dispositivo, entonces presenta un indicador de "datos pendientes" con el número de registros retenidos y el rango temporal que cubren.
2. **Escenario 2 (brecha irrecuperable por buffer lleno):** Dado que el nodo descartó bloques antiguos por saturación de LittleFS durante una desconexión prolongada (HU-06, Escenario 2), cuando el backend procesa el evento de saturación, entonces el dashboard muestra una brecha temporal señalizada en la gráfica con la causa "pérdida de datos por desconexión prolongada" y el periodo afectado.
3. **Escenario 3 (sincronización completada):** Dado que el nodo sincroniza todos los bloques pendientes con acuse lógico del backend (HU-07), cuando el dashboard se actualiza, entonces el indicador de "datos pendientes" desaparece y la serie gráfica queda completa.

**Prioridad:** Media  
**Sprint actual:** SP-10  
**Story Points actuales:** 5  
**Dependencias:** HU-06, HU-07, HU-13, HU-32  
**Justificación dentro de la tesis:** Observabilidad de HU-06/HU-07; evita confundir brecha con estado térmico normal.

## HU-49 — Historial auditable de configuración del hardware

**EPIC:** EP04 — Persistencia y Trazabilidad Verificable  
**Rol:** Responsable de auditoría  
**User Story:**

> Como Responsable de auditoría, quiero poder consultar el historial de cambios de configuración, calibración, instalación y reemplazo del hardware, para determinar desde cuándo estuvo vigente cada configuración sin alterar la telemetría registrada previamente.

### Criterios de aceptación

1. Dado que se registra una modificación de instalación, calibración, intervalo de muestreo, sensores habilitados o reemplazo de hardware, cuando se confirma, entonces se conserva valor anterior, valor nuevo, actor y timestamp según corresponda.
2. Dado que el auditor consulta un dispositivo, cuando filtra el historial de configuración, entonces puede identificar desde cuándo estuvo vigente cada cambio.
3. Dado que existe telemetría anterior a un cambio, cuando se consulta, entonces sus datos y hashes originales no se recalculan ni reescriben.
4. El rango térmico objetivo de 2–8 °C definido por la investigación no se trata como un parámetro libre editable dentro de esta historia.

**Prioridad:** Media  
**Sprint actual:** SP-10  
**Story Points actuales:** 5  
**Dependencias:** HU-24, HU-25, HU-30, HU-43, HU-51  
**Justificación dentro de la tesis:** Audita cambios provenientes de HU-30/HU-43/HU-51.

## HU-50 — Supervisión de eventos de seguridad

**EPIC:** EP06 — Autenticación, Autorización y Seguridad Transversal  
**Rol:** Administrador del sistema  
**User Story:**

> Como Administrador del sistema, quiero poder consultar eventos de autenticación fallida, bloqueos y denegaciones de acceso, para identificar patrones de acceso anómalos y verificar que los controles de seguridad están operando.

### Criterios de aceptación

1. Dado que el Administrador abre el panel de seguridad, cuando filtra por periodo, usuario o tipo, entonces puede consultar fallos de login, bloqueos temporales y denegaciones RBAC registrados.
2. Dado que se supera un umbral configurado de intentos fallidos, cuando el backend detecta el patrón, entonces genera un evento/alerta administrativa visible sin afirmar automáticamente que se trata de un ataque confirmado.
3. Dado que HU-39, HU-41 o HU-45 rechazan una operación por seguridad, cuando el evento queda auditado, entonces puede correlacionarse en el panel mediante actor/IP cuando corresponda, timestamp y resultado.
4. El panel es de supervisión y evidencia técnica; no constituye certificación de seguridad.

**Prioridad:** Alta  
**Sprint actual:** SP-10  
**Story Points actuales:** 5  
**Dependencias:** HU-39, HU-41, HU-42, HU-45  
**Justificación dentro de la tesis:** Supervisa eventos originados principalmente por HU-39/HU-41/HU-45.

## HU-51 — Registro de Ubicación y Metadatos de Instalación del Sensor

**EPIC:** EP01 — Adquisición de Datos, Resiliencia y Gestión Edge  
**Rol:** Administrador del sistema  
**User Story:**

> Como Administrador del sistema, quiero poder documentar la ubicación física del DS18B20 y los metadatos de instalación asociados al dispositivo, para contextualizar cada lectura con el punto de medición y conservar el historial técnico de la instalación.

### Criterios de aceptación

1. Dado un dispositivo registrado, cuando el Administrador documenta la ubicación del DS18B20, entonces la descripción del punto de medición queda asociada al `device_id` y visible en su ficha.
2. Dado que se realiza la instalación o una modificación física, cuando se registran fecha, responsable y observaciones técnicas, entonces se conservan como metadatos de instalación.
3. Dado que cambia la ubicación del sensor, cuando se confirma la nueva configuración, entonces se crea un evento histórico y no se sobrescribe silenciosamente la ubicación previa.
4. Dado que se genera un reporte de monitoreo, cuando corresponde, entonces incluye la ubicación vigente durante el periodo reportado o la referencia al historial de cambios.

**Prioridad:** Media  
**Sprint actual:** SP-02  
**Story Points actuales:** 3  
**Dependencias:** HU-43  
**Justificación dentro de la tesis:** Metadata de instalación dentro del ciclo de vida HU-43 y auditable por HU-49.

## 6. Control matemático obligatorio

**Cantidad inicial de User Stories: 51**  
**Cantidad revisada: 51**  
**Cantidad final de User Stories: 51**  

**Control:** `51 originales → 51 revisadas → 51 finales` ✅

**Total Story Points preservado:** 262.

## 7. Matriz de cobertura final

| Componente tesis | HU asociadas | Cobertura | Observaciones |
|---|---|---|---|
| IoT | HU-01–HU-08, HU-43, HU-51 | Completa | Captura, estructuración, resiliencia y ciclo de vida del nodo. |
| Sensores | HU-01, HU-02, HU-03, HU-04, HU-30, HU-51 | Completa | DS18B20 y SHT31 cubiertos; MC-38 permanece opcional; calibración e instalación contextualizadas. |
| MQTT/TLS | HU-09–HU-14, HU-44 | Completa | TLS, autenticación, autorización por tópico, QoS 1, LWT, endpoint seguro y rotación de credenciales. |
| Resiliencia offline | HU-06, HU-07, HU-08, HU-11, HU-13, HU-48 | Completa con límite explícito | Se evita prometer cero pérdida; saturación de buffer genera brecha observable. |
| Backend | HU-12, HU-15, HU-16, HU-19, HU-22 | Completa | Recepción, validación, degradación de IA, eventos y persistencia. |
| IA / Random Forest | HU-16, HU-17, HU-18, HU-20, HU-34, HU-46, HU-47 | Completa | IA restringida a clasificación de riesgo térmico; se conserva versionado y trazabilidad de inferencia. |
| Alertas | HU-20, HU-21, HU-23, HU-27, HU-30, HU-35 | Completa | Prevención, excursión confirmada, acuse, atención y alertas operativas diferenciadas. |
| Dashboard | HU-19, HU-31–HU-36, HU-48, HU-50 | Completa | Tiempo real, tendencia, KPI, riesgo, historial, brechas y eventos de seguridad. |
| Trazabilidad | HU-24, HU-25, HU-26, HU-28, HU-37, HU-42, HU-46, HU-47, HU-49 | Completa | SHA-256 + previous_hash + timestamp + payload + PostgreSQL; una cadena global, sin blockchain. |
| Seguridad | HU-09, HU-10, HU-14, HU-39–HU-45, HU-50 | Completa como eje transversal | Seguridad sustentada como alineamiento técnico, no certificación ni variable central del título. |
| Auditoría | HU-26, HU-28, HU-37, HU-38, HU-42, HU-47, HU-49, HU-50 | Completa | Auditoría de integridad, acciones, IA, configuración y accesos. |
| Reportes | HU-38 | Suficiente | Reporte de evidencia del monitoreo; evita declarar cumplimiento sanitario integral. |
| Validación técnica | HU-01, HU-06, HU-07, HU-18, HU-21, HU-26, HU-29, HU-38, HU-48, HU-51 | Soportada por el producto | La ejecución experimental se mantiene fuera del backlog como VT/TE y se realiza en el Cercado de Lima. |

## 8. Observaciones para construcción posterior del Product Backlog

1. **No volver a cambiar los 51 IDs** al trasladar estas historias al Excel; las modificaciones posteriores deben ser de prioridad, estimación, dependencias o criterios, no de alcance improvisado.
2. **Reestimar Story Points con el equipo** antes de tratarlos como compromiso. Los 262 puntos se conservan por trazabilidad con el archivo actual, pero son una estimación preliminar.
3. **Mantener una columna Dependencias** usando la relación definida en cada HU.
4. **No convertir TE/VT en HU.** Entrenamiento del Random Forest, evaluación ML, estrés offline, manipulación controlada de hashes, pruebas OWASP y validación técnica en el Cercado de Lima pertenecen a metodología/validación.
5. **Distinguir tres conceptos en modelo de datos y UI:** `model_class`, `effective_risk` y `excursion_confirmed`. Esto evita que una predicción crítica se confunda con una excursión físicamente confirmada por una lectura <2 °C o >8 °C.
6. **Persistencia primero como evidencia.** Una falla del modelo no debe eliminar telemetría válida ni impedir el histórico.
7. **Una única cadena global.** `audit_logs` puede ser una vista/registro operativo, pero los eventos que deban tener integridad verificable se incorporan al mismo mecanismo global de `traceability_records`; no crear cadenas paralelas sin justificación.
8. **Rango 2–8 °C no configurable libremente.** Es una condición central de la investigación; HU-49 audita otros cambios de configuración, no cambia el rango aprobado.
9. **Seguridad como transversal.** EP02/EP06 son defendibles porque habilitan operación segura y auditada, pero no deben presentarse como una nueva variable central de investigación ni como certificación ISO/OWASP.
10. **Reporte PDF como evidencia**, no como certificado de cumplimiento sanitario.

## 9. Veredicto final

### **APTO PARA CONSTRUIR EL PRODUCT BACKLOG DEFINITIVO**

El conjunto final conserva exactamente 51 User Stories y cubre de forma coherente el flujo **Sensor → ESP32 → validación → buffer offline → MQTT/TLS → EMQX → FastAPI → Random Forest/regla térmica → PostgreSQL → trazabilidad → alertas → dashboard → usuario**.

La calificación es técnicamente justificable **después de aplicar las reformulaciones contenidas en este documento**. El archivo de partida aún tenía ajustes relevantes, por lo que no era aconsejable consolidarlo sin esta pasada final. La versión resultante sí puede utilizarse como base para revisar EPICs, dependencias, prioridades, Story Points y orden definitivo del Product Backlog.

## Fuentes metodológicas y técnicas verificadas

La revisión se contrastó con fuentes oficiales y literatura reciente. Las fuentes se utilizan para validar criterios, no para convertir el backlog en un documento normativo.

1. Scrum.org. (2024). *The User Story Format*. https://www.scrum.org/resources/blog/user-story-format
2. Scrum.org. (2024). *How to Use Acceptance Criteria?* https://www.scrum.org/resources/blog/how-use-acceptance-criteria
3. Schwaber, K., & Sutherland, J. (2020). *The Scrum Guide*. https://scrumguides.org/scrum-guide.html
4. Agile Alliance. *Why you need your user stories to fit into one sprint*. https://agilealliance.org/why-you-need-your-user-stories-to-fit-into-one-sprint/
5. Amna, A. R., Wautelet, Y., Poelmans, S., Heng, S., & Poels, G. (2025). The AmbiTRUS framework for identifying potential ambiguity in user stories. *Journal of Systems and Software, 223*, 112357. https://doi.org/10.1016/j.jss.2025.112357
6. ISO/IEC. (2024). *ISO/IEC 30141:2024 — Internet of Things (IoT) — Reference architecture*. https://www.iso.org/standard/88800.html
7. OWASP Foundation. (2024). *IoT Security Testing Guide — First Release*. https://owasp.org/blog/2024/03/01/iot-security-testing-guide
8. OWASP Foundation. *Web Security Testing Guide v4.2*. https://owasp.org/www-project-web-security-testing-guide/
9. OASIS. (2019). *MQTT Version 5.0*. https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html
10. EMQX. *Serverless Connection Guide*. https://docs.emqx.com/en/cloud/latest/deployments/port_guide_serverless.html
11. EMQX. *Authentication / Authorization*. https://docs.emqx.com/en/cloud/latest/deployments/auth_overview.html
12. FastAPI. *Lifespan Events*. https://fastapi.tiangolo.com/advanced/events/
13. scikit-learn. *Model persistence*. https://scikit-learn.org/stable/model_persistence.html
14. OWASP. *Password Storage Cheat Sheet*. https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
15. Railway. *Backups*. https://docs.railway.com/volumes/backups
16. Ministerio de Salud del Perú. (2024). *Resolución Ministerial N.° 810-2024-MINSA*. https://www.gob.pe/institucion/minsa/normas-legales/6215483-810-2024-
17. DIGEMID. (2022). *Resolución Ministerial N.° 554-2022/MINSA — Manual de Buenas Prácticas de Oficina Farmacéutica*. https://www.digemid.minsa.gob.pe/webDigemid/normas-legales/2022/resolucion-ministerial-n-554-2022-minsa/
