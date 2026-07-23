# SSE server-side — verificación del flujo real

| Evento SSE | Productor backend | Payload | Consumidor frontend | Compatible | Riesgo |
|---|---|---|---|---|---|
| `message` (default SSE, sin `event:` custom) | `SSEBroadcaster.publicar()`, invocado solo desde `_procesar_mensaje_mqtt` (camino MQTT) | `lectura_to_response(...).model_dump(mode="json")` — mismo shape que `LecturaResponse` de `GET /api/lecturas` | `sseClient.ts::onmessage` → `JSON.parse(event.data)` | **Compatible** | Ver hallazgo: ingesta HTTP no emite este evento |
| `: connected` (comentario SSE) | `sse_router.py::_event_stream`, primer chunk | — | Ignorado por `EventSource` (los comentarios `:` no disparan `onmessage`), pero fuerza el flush de headers | Compatible (uso estándar del protocolo SSE) | Ninguno |
| `: keep-alive` (comentario SSE, cada 15s de inactividad) | ídem | — | Ignorado, mantiene la conexión viva a través de proxies/balanceadores | Compatible | Ninguno |

## Flujo completo verificado

1. Cliente pide `POST /api/auth/sse-ticket` (autenticado con Bearer) → recibe ticket JWT de audiencia `...:sse`, vida 60s.
2. Cliente abre `GET /api/sse/lecturas?ticket=...`.
3. Backend valida el ticket (`jwt_handler.validar_ticket_sse`) — audiencia, firma, expiración.
4. Backend **consume el ticket de un solo uso** (`sse_ticket_store.consumir`) — un ticket reutilizado (p. ej. capturado en logs de un proxy intermedio) es rechazado con 401 incluso si aún no expiró.
5. Si válido: `StreamingResponse` con `_event_stream`, que se suscribe al `SSEBroadcaster` (cola `asyncio.Queue(maxsize=100)`) y reenvía cada mensaje publicado.
6. Al desconectarse el cliente (`request.is_disconnected()`), el generador termina y el `finally` desuscribe la cola.

## TTL y reutilización

TTL: 60s configurable (`sse_ticket_expire_seconds`). Reutilización: bloqueada por diseño (confirmado, ver arriba). Revocación explícita de tickets no consumidos: no existe (si un ticket se emite y nunca se usa, simplemente expira solo a los 60s — no hay endpoint para invalidarlo antes).

## Memoria compartida entre instancias / multi-worker

**Limitación real, documentada honestamente en el propio código**: tanto `JtiStore` (revocación/tickets) como `SSEBroadcaster` (colas de suscripción) son estructuras **en memoria de un solo proceso**. En un despliegue Railway con más de un worker/réplica, un ticket emitido por el worker A no sería reconocido como "consumido" por el worker B, y un evento publicado por el worker que procesa el mensaje MQTT solo llegaría a los clientes SSE conectados a ESE MISMO worker — **los clientes conectados a otros workers no recibirían la actualización en tiempo real**. Esto es una limitación de escalabilidad horizontal real, no un bug funcional para el escenario de validación (una sola farmacia, presumiblemente un solo worker/instancia en el plan Hobby de Railway declarado en TI), pero debe entenderse como tal si se plantea escalar el sistema.

## CORS

`CORSMiddleware` configurado con `allow_origins=settings.cors_origins` (lista explícita, default solo `http://localhost:5173` en desarrollo) — **no usa `"*"`**, y `config.py` prohíbe explícitamente `"*"` en producción (`_validar_secretos_en_produccion`). Correcto para un endpoint SSE con credenciales.

## Formato SSE y compatibilidad

El formato `data: <json>\n\n` es el estándar correcto que `EventSource` espera. Confirmado que el frontend parsea `event.data` directamente como JSON de una `LecturaTermica` — coincide exactamente con lo que `lectura_to_response()` serializa.

## Backpressure

La cola tiene `maxsize=100`; si un cliente lento no consume, `publicar()` hace `if queue.full(): continue` — **descarta silenciosamente eventos para ese cliente específico** en vez de bloquear al publicador o desconectar al cliente lento. Es una decisión de diseño razonable (evita que un cliente lento bloquee la difusión a los demás), aunque significa que un cliente con red intermitente podría perder lecturas intermedias sin ningún aviso (se enteraría solo al re-consultar `/api/lecturas`).

## Hallazgo cruzado con Fase 2

El endpoint SSE real existe y es compatible exactamente con lo que el frontend espera (confirmando la pregunta pendiente de Fase 2: "¿el backend implementa realmente `/api/auth/sse-ticket` y `/api/sse/lecturas`?" → **Sí, ambos reales y correctamente diseñados**).
