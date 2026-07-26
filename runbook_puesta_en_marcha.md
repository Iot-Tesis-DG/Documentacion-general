# Runbook — Puesta en marcha con el ESP32 físico

> **Para**: Diego / Brenda, con el hardware ya armado.
> **Fecha**: 2026-07-25
> **Qué es**: la lista exacta de pasos que faltan. Todo lo que era código ya
> está hecho y compila; lo de aquí abajo son datos que solo tú tienes
> (credenciales, hostname) y comprobaciones sobre hardware real.

---

## 0. Antes de enchufar nada — lo que NO está hecho y te bloquearía

| # | Bloqueador | Por qué te para | Dónde |
|---|-----------|-----------------|-------|
| 1 | `MQTT_HOST` es `"tu-instancia.emqx.cloud"` | El nodo no resuelve el broker | `config.h:27` o `build_flags` |
| 2 | `MQTT_TOKEN` es `"token_generado_en_emqx"` | Error MQTT 4 (credenciales) | `config.h` / `build_flags` |
| 3 | `WIFI_SSID` / `WIFI_PASSWORD` de ejemplo | No hay red | `config.h:18,21` |
| 4 | **`root_ca.pem` no está en el repo** | Handshake TLS falla → error -1 | `data/certs/` (está en `.gitignore`) |
| 5 | **`FARM-01-CDL` no existe en la tabla `devices`** | El backend **descarta todo**: lecturas (403) y eventos | BD del backend |

El punto 5 es el que más cuesta diagnosticar: el nodo dirá "publicado" y el
dashboard seguirá vacío. `device_registry_estricto = True` por defecto
(`config.py:50`).

---

## 1. Certificado CA

```bash
cd iot-firmware
curl -o data/certs/root_ca.pem https://letsencrypt.org/certs/isrgrootx1.pem
head -1 data/certs/root_ca.pem   # -----BEGIN CERTIFICATE-----
```

## 2. EMQX Cloud

1. Instancia Serverless en https://www.emqx.com/en/cloud
2. **Authentication** → añadir usuario: `FARM-01-CDL` / token (`openssl rand -hex 16`)
3. **ACL** → `pub farmacias/FARM-01-CDL/#` para ese usuario
4. Otro usuario `backend_service` con `sub farmacias/+/lecturas` y `sub farmacias/+/eventos`
5. Copiar el hostname

## 3. Credenciales del nodo — sin tocar `config.h`

`MQTT_HOST` ya admite inyección (antes no: no tenía guarda `#ifndef`).

```ini
; platformio.ini
build_flags =
    -DCORE_DEBUG_LEVEL=0
    -DWIFI_SSID='"TuRed"'
    -DWIFI_PASSWORD='"tu_password"'
    -DMQTT_HOST='"xxxxx.emqx.cloud"'
    -DMQTT_TOKEN='"tu_token"'
```

> Si usas varios ESP32, cambia `DEVICE_ID` por nodo y crea credenciales y ACL
> separadas en EMQX.

## 4. Registrar el dispositivo en el backend

**Sin esto no funciona nada.** El `device_id` debe existir antes de que el nodo
publique.

```bash
cd backend && .venv/bin/python -m scripts.seed_dev   # revisa qué device crea
```

Si el seed no crea `FARM-01-CDL`, dalo de alta por la API (rol admin) o insértalo
en `devices`. Verifica:

```bash
curl -H "Authorization: Bearer $TOKEN" https://<backend>/api/dispositivos
```

## 5. Backend

```bash
# backend/.env
MQTT_ENABLED=true
MQTT_HOST=xxxxx.emqx.cloud
MQTT_PORT=8883
MQTT_USERNAME=backend_service
MQTT_PASSWORD=<token backend>
MQTT_TLS_ENABLED=true
```

## 6. Ensayo SIN hardware (recomendado antes de flashear)

El simulador ahora emite **exactamente** el payload del firmware, así que valida
la cadena completa salvo los sensores:

```bash
cd backend && .venv/bin/python -m scripts.simulador_esp32 --device-id FARM-01-CDL
```

Si el dashboard se mueve, todo lo que no es hardware está bien. Si falla aquí,
falla también con el ESP32 — y es mucho más fácil de depurar.

## 7. Flashear

```bash
cd iot-firmware
pio run --target uploadfs    # PRIMERO: sube root_ca.pem a LittleFS
pio run --target upload
pio device monitor
```

`uploadfs` antes que `upload`: sin el CA en LittleFS el TLS falla con error -1.

## 8. Qué debes ver en el monitor serie

```
[LittleFS] Montado. 0 lecturas pendientes. ~1560 KB libres.
[WiFi] Conectado! IP: ..., RSSI: -45 dBm.
[NTP] Hora sincronizada: 2026-07-25T...Z        ← DEBE aparecer
[MQTT] Certificado CA cargado (1934 bytes).
[MQTT] Conectado a EMQX. Publicando LWT 'online'...
[Core0] Ciclo completado. Pendientes: 1 en RAM, 1 en Flash.
[Core1] Publicadas 1 lecturas. Quedan 0 en Flash.   ← DEBE aparecer
```

**Las dos líneas marcadas son el criterio de éxito**:

- Sin `[NTP] Hora sincronizada` el nodo usa la hora de compilación; pasadas 48 h
  el backend rechaza todo por `timestamp_demasiado_antiguo`, en silencio.
- Sin `[Core1] Publicadas N lecturas` la cola crece y nada llega al dashboard.
  Este era el bug V-09: ese log no existía porque el código tampoco.

## 9. Validación de tesis (§4.4 del análisis, OE4)

| # | Prueba | Criterio |
|---|--------|----------|
| 1 | Lectura de los 3 sensores | Sin `NAN` en el JSON |
| 2 | Precisión DS18B20 vs termómetro patrón | MAE ≤ 0.5 °C |
| 3 | Latencia captura → dashboard | ≤ 5 s (RNF-01) |
| 4 | **Apertura corta de puerta (~10 s), cerrarla, y esperar al siguiente reporte** | `apertura_refrigerador: true` y `duracion_apertura_segundos ≈ 10`, **aunque la puerta ya esté cerrada al publicar**. Antes esto se perdía por completo (V-17): era el fallo más silencioso de todos |
| 5 | **Corte de Wi-Fi 10 min** | La cola crece en Flash; al volver, se drena en orden FIFO |
| 6 | Sincronización tras reconexión | ≤ 30 s (RNF-07) |
| 7 | Apagón brusco | El broker publica LWT → badge offline en el dashboard |
| 8 | Excursión térmica (sacar del refri) | Alerta + notificación + cadena hash |
| 9 | Verificar integridad | Endpoint devuelve `integra: true` |
| 10 | Reinicio del nodo | LittleFS conserva la cola pendiente |

La prueba **5** es la más valiosa para la defensa: es RF-06 + RNF-07 y es donde
se ve el offline-first, que es la aportación diferencial del prototipo.

---

## 10. Diagnóstico rápido

| Síntoma | Causa más probable |
|---------|--------------------|
| Bucle de reinicio, `[LittleFS] Fallo al montar` | No ejecutaste `uploadfs`, o la tabla de particiones no es `partitions_thermotrace.csv` |
| `[MQTT] ERROR: -1` | Falta `root_ca.pem` en LittleFS, o no es ISRG Root X1 |
| `[MQTT] ERROR: 4` | Token incorrecto |
| `[MQTT] ERROR: 5` | Falta la regla ACL de publicación |
| Nodo dice "Publicadas N" pero el dashboard vacío | **`device_id` no registrado** (paso 4). Mira `audit_logs`: `EVENTO_DISPOSITIVO_DESCONOCIDO` |
| Timestamps de 1970 o del día del flasheo | NTP no sincronizó: sin salida UDP/123 |
| El dashboard funciona en demo pero no contra el backend real | `connect-src 'self'` del CSP de Vercel — ver §4.7 del análisis |

---

## 11. Estado honesto

**Hecho y verificado por máquina**: el firmware compila (Flash 78.3 %), backend
346 pruebas en verde, frontend 63, tsc y ESLint limpios, builds de producción
OK, el simulador emite el payload real del firmware y valida contra los dos
esquemas.

**No verificado — necesita tu hardware**: que los drivers DS18B20 / SHT31 /
MC-38 lean de verdad, el handshake TLS contra EMQX, el montaje efectivo de
LittleFS, la cadencia real de 30 s y el comportamiento del watchdog. Compilar no
es ejecutar; nadie ha visto todavía a este firmware corriendo.

**Riesgo residual conocido**: los tres drivers de sensores nunca se han
ejecutado. Es el punto donde es más probable que aparezca algo — por eso el
paso 6 (simulador) importa: aísla "¿falla el sensor?" de "¿falla la cadena?".
