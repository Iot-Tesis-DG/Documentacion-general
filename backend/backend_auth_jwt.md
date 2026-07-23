# Autenticación JWT — backend

## Flujo real (verificado en código)

1. `POST /api/auth/login` recibe `OAuth2PasswordRequestForm` (username=email, password).
2. `AutenticarUsuarioUseCase` (no leído línea por línea en detalle pero su firma y uso confirman: busca usuario por email, verifica password).
3. `verify_password()` usa `passlib.CryptContext(schemes=["bcrypt"])` — **hashing real con bcrypt**, no MD5/SHA plano.
4. Si falla: se registra el fallo en `SlidingWindowRateLimiter` (por IP) y se audita `LOGIN_FALLIDO` con el email intentado (nota: el email se registra en la auditoría incluso si no existe la cuenta — no es enumeración de usuarios porque el mensaje de error no lo revela, pero el dato queda en `audit_logs`, accesible solo a administrador).
5. Si tiene éxito: se resetea el limitador para esa IP, se audita `LOGIN_EXITOSO`, se emite el JWT.

## Estructura del JWT — verificado real

Claims exigidos explícitamente en la decodificación (`options={"require": [...]}`): `exp`, `iat`, `sub`, `iss`, `aud`, `jti`. Si falta cualquiera, se rechaza.

- `sub`: UUID del usuario (no el email — buena práctica, evita filtrar el email en cada request innecesariamente aunque también se incluye `email` como claim adicional).
- `rol`: string del enum `Rol`.
- `iss`: `"cadena-frio-backend"`, `aud`: `"cadena-frio-api"` — **verificados en la decodificación** (`jwt.decode(..., audience=..., issuer=...)`), no solo generados y nunca comprobados. Esto es una capa de defensa real contra confusión de tokens entre servicios.
- `jti`: UUID único por token, habilita revocación.
- Algoritmo: HS256 (simétrico) — configurable vía `JWT_ALGORITHM`, con secreto validado en producción (≥32 caracteres, no el valor por defecto — `config.py::_validar_secretos_en_produccion`).

## Revocación — implementada realmente, no solo declarada

`POST /api/auth/logout` registra el `jti` del token actual en `JtiStore` (`app.state.token_revocation`) hasta su `exp` natural. `get_current_user()` verifica en CADA request si el `jti` está en el store revocado, y rechaza con 401 si es así. **Esto es una revocación server-side real**, superior a la mayoría de implementaciones JWT ingenuas que solo confían en la expiración natural.

**Limitación real documentada por el propio código**: el `JtiStore` es en memoria, por proceso — en un despliegue multi-worker o multi-instancia (Railway con >1 réplica), la revocación de un worker no sería visible para otro. El comentario del código lo reconoce explícitamente ("en despliegue multi-instancia se sustituiría por Redis con TTL"). Es una limitación real del prototipo, no un error, y está honestamente documentada en el propio código.

## Refresh token: ausente

No existe endpoint de refresh. El JWT expira a los 60 minutos (default `JWT_ACCESS_TOKEN_EXPIRE_MINUTES`) y el usuario debe volver a autenticarse. Coherente con la estrategia "JWT en memoria" del frontend (recargar página ya cierra sesión de todos modos).

## Tickets SSE — mecanismo separado y correctamente aislado

- Audiencia distinta (`aud: "cadena-frio-api:sse"`), evita que un ticket SSE se use como token de API normal o viceversa (`jwt.decode` valida audiencia estrictamente).
- Vida corta configurable (`sse_ticket_expire_seconds`, default 60s).
- **Consumo de un solo uso real**: `JtiStore.consumir()` es atómico (con `Lock()`), devuelve `False` si el `jti` ya fue usado — el router SSE rechaza con 401 "Ticket SSE ya utilizado" en ese caso. Verificado en código, no solo declarado.

## Credenciales hardcodeadas / secretos por defecto

- `_SECRETO_POR_DEFECTO` existe como valor de desarrollo (`"clave_secreta_larga_y_aleatoria_cambiar_en_produccion"`), pero **el propio `Settings` valida en `model_validator` que este valor NO puede usarse si `environment == "production"`** (lanza `ValueError` al arrancar). Esto es una salvaguarda real contra el error clásico de dejar el secreto por defecto en producción — no un hallazgo, es una buena práctica confirmada.
- `mqtt_password` por defecto (`"token_seguro"`) tiene la misma protección: se rechaza en producción si no se cambió.
- No se encontraron usuarios seed con contraseñas hardcodeadas en el código de `src/` (el seed de desarrollo vive en `scripts/seed_dev.py`, no ejecutado en esta auditoría para no alterar `dev.db`, pero por convención de nombre es exclusivamente para desarrollo local).

## Prevención de enumeración de usuarios

El mensaje de error de login no distingue "usuario no existe" de "contraseña incorrecta" (mismo `CredencialesInvalidasError` en ambos casos, mismo HTTP 401 genérico) — coincide con lo que el frontend espera (`t('login.errorCredenciales')` genérico) y con HU-26 (Escenario 2). Confirmado correcto.

## Rate limiting

`SlidingWindowRateLimiter` por IP, 5 intentos fallidos en 300 segundos por defecto (`login_max_intentos`, `login_ventana_segundos`). Solo cuenta fallos (un login exitoso resetea el contador), evitando penalizar usuarios legítimos detrás de IPs compartidas (NAT de oficina/farmacia). Devuelve 429 con header `Retry-After`. **Real, no cosmético.**

## ¿El frontend puede mantener su estrategia de JWT en memoria?

**Sí, es compatible.** El backend no exige cookies, no fuerza `Set-Cookie`, y el flujo `Bearer` + body JSON es exactamente lo que `apiClient.ts` (interceptor `Authorization: Bearer <token>`) espera. La revocación server-side por `jti` incluso mejora la postura de seguridad que el frontend por sí solo no podría lograr (el frontend "olvida" el token al recargar, pero sin el backend, un token robado antes de esa recarga seguiría siendo válido hasta expirar; con la revocación real, un logout explícito lo invalida de inmediato — **aunque el frontend actualmente nunca llama a `/api/auth/logout`**, ver hallazgo).
