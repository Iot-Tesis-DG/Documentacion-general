# 08 — MQTT simulado

PostgreSQL 18 temporal: payload aliases SHT31/DS18B20, normal, crítico, sensor
DS18B20 ausente, 0.0 C y duplicado. Resultado: 4 lecturas, 1 alerta, 6 hashes,
7 eventos SSE. Duplicado no agregó lectura ni evento. Sensor ausente: riesgo y
confianza `null`, origen `fallo_sensor`, inferencia `omitida`. Base eliminada.
