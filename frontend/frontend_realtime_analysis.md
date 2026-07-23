# Análisis de datos en tiempo real (RF-11)

## Mecanismo real: SSE genuino, no WebSocket ni polling

Verificado en `infrastructure/sse/sseClient.ts`: el frontend usa **`EventSource` nativo del navegador**, exactamente como declara TI. No hay WebSocket (`ws://`/`wss://`), no hay `setInterval`/polling HTTP disfrazado de tiempo real en el camino de producción.

## Flujo de autenticación SSE (solución elegante a una limitación real)

`EventSource` no puede enviar headers HTTP personalizados (no hay forma de mandar `Authorization: Bearer <jwt>` en una petición SSE nativa). El frontend resuelve esto con un patrón de **ticket efímero**:
1. Antes de abrir el `EventSource`, hace `POST /api/auth/sse-ticket` (autenticado con el JWT normal vía Axios/Bearer).
2. El backend (presumiblemente) devuelve un ticket de corta duración.
3. El `EventSource` se abre a `${SSE_URL}?ticket=<ticket>`.
4. Si la conexión falla (`onerror`) se asume ticket expirado o red caída, se cierra y se reintenta pidiendo un ticket nuevo tras 5000ms.

Esto es un diseño técnicamente sólido y más seguro que las alternativas ingenuas (JWT completo en query string, sin autenticación alguna). **No verificado en esta fase si el backend realmente implementa `/api/auth/sse-ticket`** — pendiente de Fase 3.

## URL y eventos

- URL: `import.meta.env.VITE_SSE_URL` o fallback `${API_BASE_URL}/api/sse/lecturas`.
- Formato de mensaje: `event.data` se parsea como JSON directo a `LecturaTermica` (sin campo `event:` custom, usa el evento `message` por defecto de SSE).
- Mensajes malformados o keep-alive: capturados en `try/catch` silencioso (`// Mensaje keep-alive o malformado: se ignora`) — correcto, evita que un ping del servidor rompa el parseo.

## Reconexión y backoff

- Reintento fijo cada 5000ms (`REINTENTO_MS`), **no exponencial** — a diferencia del backoff exponencial que sí implementa el firmware ESP32 según HU-08 del backlog (1s→2s→4s→8s→16s). El frontend usa un intervalo fijo simple. No es necesariamente un defecto (los patrones de reconexión de cliente web vs firmware embebido suelen diferir), pero es una asimetría de diseño a mencionar.
- Limpieza de conexión: la función devuelta por `suscribirseLecturas()` cierra `source` y limpia el `setTimeout` pendiente — se invoca correctamente en el `return` del `useEffect` de `useMonitoreoTermico`. **No hay fuga de memoria detectada.**

## Modo demo

`simularStream()` sustituye el `EventSource` real por un `setInterval` de 8000ms que invoca `generarLecturaEnVivo()` (generador determinista con deriva suave hacia el centro del rango 2-8°C). Esto es correcto y honesto: solo se activa si `MODO_DEMO=true`, y el propio código lo etiqueta sin ambigüedad.

## Duplicación de eventos

No se detectó lógica de deduplicación de eventos SSE por ID — si el backend reenviara un evento repetido, el frontend lo agregaría dos veces a la serie (`setSerie((previa) => [...previa, lectura].slice(-60))`). Riesgo bajo (SSE no suele duplicar salvo reconexión mal implementada en backend), pero no hay guardia explícita del lado cliente.

## Sincronización con HTTP

`useMonitoreoTermico` primero hace `GET /api/lecturas?limite=60` (carga inicial) y luego se suscribe a SSE, agregando lecturas nuevas al final del array. Consistente y correcto: no hay condición de carrera visible (el array HTTP inicial ya viene invertido a orden cronológico ascendente, y las lecturas SSE se agregan al final, preservando el orden).

## Estado offline

`sseConectado` (booleano) se expone a `DashboardPage`, que muestra un indicador visual (punto verde animado "Conectado" / punto gris "Desconectado"). No hay lógica de "modo offline" más allá de esta indicación visual — no se persisten lecturas localmente si SSE cae (a diferencia del firmware ESP32, que sí tiene buffer offline LittleFS documentado en TI/backlog; el frontend no necesita esto, ya que su rol es solo mostrar, no capturar).

## Clasificación de correspondencia con la tesis

| Aspecto | Estado |
|---|---|
| Mecanismo SSE vs WebSocket declarado en TI | **Coincide** |
| Autenticación del canal en tiempo real | **Coincide** (no explícitamente detallado en TI, pero es consistente con el uso de JWT+RBAC declarado) |
| Reconexión | **Coincide parcialmente** — existe pero es intervalo fijo, no backoff exponencial (TI no exige backoff específico para el cliente web, solo para el firmware) |
| Backend real (`/api/sse/lecturas`, `/api/auth/sse-ticket`) | **No comprobado** — pendiente de verificar existencia real en Fase 3 |
