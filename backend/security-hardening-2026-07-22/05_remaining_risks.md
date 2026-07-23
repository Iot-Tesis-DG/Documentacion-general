# Riesgos que requieren infraestructura o decisión de despliegue

1. **Multiworker/multiinstancia (alto):** revocación JWT, tickets SSE y rate limit están acotados pero aún son locales al proceso. Antes de más de un worker/instancia se requiere Redis o PostgreSQL con TTL compartido y prueba de dos procesos.
2. **Identidad MQTT (alto):** el backend valida tópico/payload, pero la prueba de identidad depende de ACL o certificados por dispositivo configurados en el broker. No se ha usado EMQX de producción ni se afirma esa validación.
3. **Replay con `message_id` (medio):** el contrato acepta `message_id`, pero la garantía persistida es `(device_id, timestamp)`. Para convertir el ID en nonce se requiere una migración nueva, campo único y política de retención; no se cambia el esquema sin esa decisión de dominio.
4. **Modelo joblib (medio condicionado):** checksum detecta alteración accidental, no sustituye firma de procedencia si un actor puede modificar a la vez modelo y metadata. Mantener directorio de modelos de solo lectura para el proceso y permisos solo del dueño de despliegue.
5. **Rutas costosas:** verificar toda la cadena de trazabilidad y exportar reportes grandes deben paginarse o ejecutarse como trabajo asíncrono antes de exponerlos a tráfico intenso.
6. **Supply chain backend:** `requirements*.txt` usa rangos sin lock con hashes; agregar lock reproducible y `pip-audit` en CI. `pip-audit` no estaba instalado, por lo que no se declara un veredicto CVE de Python.

No se encontraron secretos reales versionados, claves privadas, firmware ESP32, CI, Docker/Compose ni configuración productiva en este directorio.
