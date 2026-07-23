# MQTT y EMQX — análisis real

Librería: `aiomqtt>=2.3.0` (wrapper asíncrono sobre `paho-mqtt`).

| Topic | Dirección | Payload | QoS | Retain | Validación | Caso de uso | Persistencia |
|---|---|---|---|---|---|---|---|
| `farmacias/+/lecturas` | Entrante (suscripción) | `LecturaPayload` (Pydantic): device_id, timestamp, temperaturas, humedad, apertura, conectividad, firmware_version | No especificado explícitamente en `client.subscribe()` (usa default de aiomqtt/paho, típicamente QoS 0) | No verificado (no se lee/fuerza flag retain) | `LecturaPayload.model_validate_json()` (Pydantic v2, con rangos físicos `ge`/`le`) + verificación anti-suplantación device_id-vs-topic | `RegistrarLecturaTermicaUseCase` (pipeline completo) | `thermal_readings`, `thermal_alerts`, `traceability_records` |
| `farmacias/+/eventos` | Entrante (suscripción) | No procesado — **suscrito pero sin manejador específico visible**; el mismo `manejador` genérico (`_procesar_mensaje_mqtt`) se aplica a ambos topics, y ese manejador solo sabe parsear `LecturaPayload` | — | — | Ninguna específica para "eventos" distintos de lecturas | Ninguno diferenciado | Ninguna |

## Hallazgo: topic `eventos` suscrito pero sin lógica propia

El backend se suscribe a `farmacias/+/eventos` (línea `TOPIC_EVENTOS` en `mqtt_client.py`) pero el único manejador de mensajes (`_procesar_mensaje_mqtt` en `main.py`) intenta parsear TODO mensaje recibido como `LecturaPayload`. Si el firmware publicara un evento distinto (p. ej. `CONECTIVIDAD` o `ERROR_SENSOR`, mencionados en la trazabilidad demo del frontend) en el topic `eventos` con una estructura diferente, `LecturaPayload.model_validate_json()` fallaría con una excepción de validación Pydantic — que **no está capturada explícitamente** en `_procesar_mensaje_mqtt` (solo `consumir_mensajes()` en `mqtt_client.py` tiene un `except Exception: continue` genérico que evita tumbar el consumidor, pero silenciosamente descarta cualquier evento no-lectura sin registrar nada). **Conclusión: el topic `eventos` está declarado pero no tiene lógica de procesamiento diferenciada — es una suscripción funcionalmente inerte para cualquier payload que no sea una lectura térmica.**

## TLS

`build_ssl_context(tls_enabled)`: usa `ssl.create_default_context()` (validación estándar del sistema contra CAs conocidas) cuando `mqtt_tls_enabled=True` (default). Coherente con EMQX Cloud Serverless (que usa certificados de CA públicas, no self-signed). En producción, `config.py` fuerza `MQTT_TLS_ENABLED=true` (valida y falla el arranque si no). **RNF-05 cumplido a nivel de configuración.**

## Autenticación de dispositivo ante el broker

El backend se conecta con `mqtt_username`/`mqtt_password` **únicos para el servicio backend completo** (`backend_service`), no una credencial por dispositivo IoT individual. La autenticación por-dispositivo (que cada ESP32 tenga sus propias credenciales) sería responsabilidad de la configuración del broker EMQX Cloud (fuera de este repositorio, gestionado por el proveedor) — **no verificable desde el código del backend**, ya que el backend actúa como un suscriptor central, no como el punto de autenticación de cada dispositivo. La autorización real de "qué dispositivo puede publicar en qué topic" recaería en la configuración de EMQX (ACLs del broker), no visible en este repositorio.

## Autorización de dispositivo — sí implementada a nivel de aplicación

Aunque el broker no se audita aquí, el backend **sí implementa una capa propia de autorización de aplicación**: `DEVICE_REGISTRY_ESTRICTO` (config), verificado en `RegistrarLecturaTermicaUseCase._autorizar_dispositivo()` — en modo estricto, solo `device_id` ya provisionados en la tabla `devices` pueden persistir lecturas; de lo contrario se lanza `DispositivoNoAutorizadoError`, se **audita el rechazo** (`DISPOSITIVO_RECHAZADO` en `audit_logs`), y el mensaje se descarta. Esto es un control de mínimo privilegio real, no cosmético, implementado tanto en el flujo MQTT como en el flujo HTTP (`POST /api/lecturas`).

## Anti-suplantación (device_id del payload vs. topic)

`_device_id_del_topic()` extrae el segmento intermedio de `farmacias/{device_id}/lecturas` y `_procesar_mensaje_mqtt` compara ese valor contra el `device_id` declarado dentro del payload JSON — si no coinciden, el mensaje se descarta con un warning en log, **sin persistir nada**. Esto asume que el broker ya autorizó al cliente MQTT a publicar únicamente bajo su propio topic (vía ACL de EMQX) — el backend añade una segunda verificación de consistencia interna, defensa en profundidad razonable.

## QoS, Last Will, Retain — evaluación

- **QoS de suscripción**: no se especifica explícitamente (`client.subscribe(TOPIC_LECTURAS)` sin parámetro `qos`), lo que en aiomqtt/paho por defecto es QoS 0 del lado suscriptor. El backlog (HU-11) describe que el **ESP32 publica con QoS 1**; el QoS efectivo de una entrega end-to-end depende del mínimo entre publicador y suscriptor según el broker — **no verificable con certeza sin acceso al broker real**, pero la ausencia de un QoS explícito del lado suscriptor es una omisión notable frente al diseño declarado.
- **Last Will and Testament (LWT)**: HU-13 describe que el firmware ESP32 configura un LWT para notificar desconexiones abruptas. **El backend (código Python) no configura ningún LWT propio** (LWT es responsabilidad del cliente publicador, es decir, del firmware ESP32 — fuera de este repositorio). El backend tampoco tiene lógica visible para procesar específicamente un mensaje LWT recibido como un tipo de evento diferenciado (cae en el mismo problema del topic `eventos` genérico arriba descrito).
- **Retain**: no se usa ni se lee explícitamente en ningún punto del código backend.
- **Reconexión**: `aiomqtt.Client(..., reconnect=True)` delega la reconexión a la librería subyacente; no hay backoff exponencial custom del lado backend (a diferencia del firmware, que sí lo implementa según HU-08). Razonable, dado que el backend es un servicio de larga duración, no un dispositivo con recursos limitados.

## Orden, timestamps, mensajes retrasados/inválidos

- El **timestamp de la lectura** proviene del payload del dispositivo (`payload.timestamp`), no se genera server-side al recibir el mensaje — correcto para preservar el momento real de captura, pero **no hay validación de que el timestamp no sea futuro ni excesivamente antiguo** (`LecturaTermica.es_lectura_valida()` solo valida rangos físicos de temperatura/humedad, no el timestamp). Un mensaje con timestamp corrupto o desincronizado (reloj del ESP32 mal configurado) se aceptaría sin objeción.
- **Deduplicación**: no implementada (ver también `backend_database_schema.md`) — un reenvío MQTT por PUBACK perdido generaría un registro duplicado real en `thermal_readings`, cada uno con su propio hash de trazabilidad (dos eslabones distintos para el "mismo" evento físico).
- **Mensajes fuera de orden**: el pipeline no reordena — cada mensaje se procesa e inserta en el orden de llegada de la red, no en el orden de su `timestamp`. Si dos lecturas llegan fuera de orden cronológico (posible tras una reconexión que sincroniza un buffer LittleFS retrasado), la cadena de trazabilidad (que se ordena por `created_at`, el momento de inserción en BD, no por `timestamp` del evento) seguiría siendo internamente consistente (el hash encadenado no se rompe), pero el orden cronológico visible en el historial no sería estrictamente el orden real de captura si hay lecturas retroactivas del buffer offline. Esto es coherente con cómo funciona un buffer offline en general (los datos sincronizados llegan "tarde" pero deben insertarse igual) — no es un defecto de la cadena hash, es una característica esperable del diseño.

## No se intentó conexión al broker de producción

Toda esta sección se basó en lectura estática de código; no se intentó abrir una conexión real a EMQX Cloud (no hay credenciales de producción disponibles ni se debe usarlas).
