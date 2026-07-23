# Modelo de amenazas

## Overview

El sistema recibe telemetría MQTT de dispositivos, la persiste en PostgreSQL, la clasifica con Random Forest, genera alertas/trazabilidad SHA-256 y la entrega a usuarios autenticados por API/SSE y React. Los activos principales son credenciales JWT, datos térmicos, alertas, cadena de hashes, modelo y artefactos de IA.

## Threat Model, Trust Boundaries, and Assumptions

Entradas no confiables: cuerpo HTTP, credenciales, ticket SSE, payload y tópico MQTT, valores de sensores y parámetros de reporte. Entradas de operador: variables de entorno, ACL MQTT, artefacto del modelo y despliegue. Se asume PostgreSQL local controlado durante pruebas. TLS protege tránsito MQTT, pero no reemplaza ACL/certificados por dispositivo.

## Attack Surface, Mitigations, and Attacker Stories

Controles existentes: JWT con issuer/audience/JTI, RBAC backend, contraseñas hasheadas, CORS/TrustedHost/HSTS en producción, CSP, headers, límite de tasa, cuerpo máximo, tickets SSE de un uso, validación Pydantic, restricciones PostgreSQL, deduplicación y checksum del modelo. Se añadieron límite ASGI para cuerpos chunked, contratos MQTT cerrados, cachés de seguridad acotadas, y neutralización de fórmulas CSV.

Historias relevantes: envío MQTT no autorizado/repetido, agotamiento HTTP, robo/reuso de ticket, manipulación local del modelo, XSS desde datos de telemetría, inyección de fórmula al exportar y configuración insegura de producción.

## Severity Calibration

- Crítica: una ACL MQTT ausente que permita publicar como cualquier dispositivo o artefacto de modelo modificable por un usuario menos privilegiado.
- Alta: estado de revocación/SSE no compartido al escalar a varios workers, permitiendo reuso de ticket o token revocado.
- Media: agotamiento de recursos por rutas costosas, replay MQTT con identidad ya comprometida o dependencia vulnerable.
- Baja: configuración local insegura sin exposición pública o degradación de disponibilidad sin acceso a datos.
