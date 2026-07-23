# Autenticación y RBAC

## Flujo de login

1. `LoginPage` envía `POST /api/auth/login` con body `application/x-www-form-urlencoded` (`username`, `password`) — formato compatible con `OAuth2PasswordRequestForm` de FastAPI (indicio de diseño consciente del backend real, no arbitrario).
2. Respuesta esperada: `{access_token, token_type}`.
3. `decodificarSesion(token)` decodifica el payload JWT (**sin verificar firma** — comentario explícito: "eso lo hace el backend", correcto, el cliente nunca debe validar firmas).
4. Extrae `{id: sub, email, rol}` del payload.

## Almacenamiento del token — verificado real, no solo declarado

**El JWT vive únicamente en una variable de módulo en memoria** (`accessToken` en `apiClient.ts`), nunca en `localStorage` ni `sessionStorage`, en el build de producción real. Esto coincide exactamente con HU-40 y con el criterio de seguridad RNF declarado en TI (mitigación XSS). Verificado leyendo el código fuente directamente, no solo el comentario.

**Excepción documentada y limitada al modo demo**: cuando `MODO_DEMO=true`, el token (falso, sin valor real, firmado con `alg: none`) se guarda en `sessionStorage` bajo la clave `cf_demo_token`, exclusivamente para que la demo pública sobreviva a un F5 sin pedir login de nuevo. Esto está bien aislado (`if (MODO_DEMO) sessionStorage.setItem(...)`) y no aplica al build de producción real (`npm run build`, sin `--mode demo`).

## Expiración y logout

- No hay decodificación de `exp` en el cliente para detectar expiración proactiva — el mecanismo real es reactivo: el interceptor de Axios detecta un 401 en cualquier petición y dispara `onSesionExpirada()` → `logout()` + redirect a `/login`. Funcional pero no proactivo (el usuario descubre la expiración solo al intentar una acción).
- No existe refresh token ni endpoint de renovación consumido.
- Logout: limpia el token en memoria (`setAccessToken(null)`), limpia flags de `sessionStorage`, resetea el store Zustand. Correcto y completo.

## Protección de rutas — verificado, no solo visual

`RequireAuth` (en `RouteGuards.tsx`) redirige a `/login` si `!autenticado`. Se aplica como wrapper de layout en `App.tsx`, envolviendo TODAS las rutas protegidas — **la navegación directa por URL a `/dashboard` sin sesión sí es bloqueada** (redirect real vía React Router, no solo ocultamiento de botón).

`RequireRoles` bloquea el `Outlet` (contenido) si `!tienePermiso(usuario.rol, roles)`, mostrando una pantalla de "sin permiso" en vez de renderizar la página. **Esto también es protección real de ruta, no solo ocultación de menú** — si un técnico navega manualmente a `/usuarios`, la ruta existe pero el componente `UsuariosPage` nunca se renderiza (el `Outlet` retorna el mensaje de acceso denegado en su lugar).

**Limitación real (esperada en cualquier SPA)**: esta protección es client-side. Un atacante que interceptara/fabricara peticiones HTTP directas al backend (`/api/usuarios`, `/api/auditoria`) sin pasar por la UI **no está protegido por este código** — la autorización real y definitiva debe existir en el backend (dependency de FastAPI verificando el rol del JWT en cada endpoint). **Esto no se puede confirmar como implementado hasta Fase 3.** El frontend, por sí solo, nunca puede ser la única barrera de RBAC — es correcto que exista aquí como UX, pero es insuficiente como control de seguridad si el backend no replica la misma verificación.

## Roles: 3 confirmados, coincide con TI RF-17

`Rol.ts`: `'administrador' | 'farmaceutico' | 'tecnico'` — exactamente los 3 roles de TI RF-17. El admin tiene acceso implícito a todo (`tienePermiso`: `rol === 'administrador' || permitidos.includes(rol)`), patrón de "superusuario" razonable y explícito.

**Nota de coherencia con backlog**: el Anexo5 (Fase 1) usaba de facto un cuarto rol narrativo ("Auditor") en varias historias de usuario (HU-41, HU-42, HU-44). El código real del frontend **no implementa un rol "auditor" separado** — la ruta `/auditoria` está reservada a `administrador` únicamente (`roles: []`). Esto **resuelve/aclara** la ambigüedad detectada en Fase 1: el frontend deja claro que solo hay 3 roles reales, y "auditor" en el backlog era terminología narrativa para "quien tiene acceso a auditoría" (el administrador), no un cuarto rol de sistema. Se recomienda que el jurado no encuentre esto como contradicción si se les explica así.

## Prevención de navegación directa

Confirmado arriba (RouteGuards se aplican a nivel de árbol de rutas, no de botones individuales).

## Credenciales hardcodeadas

**Ninguna credencial hardcodeada en el código de producción real.** Las 3 cuentas demo (`farmaceutico@demo.pe`, `admin@demo.pe`, `tecnico@demo.pe`) están en `datosDemo.ts`, usadas solo si `MODO_DEMO=true`, y la contraseña que usan (`'demo-2026'`) es arbitraria porque en modo demo el `demoAdapter` no verifica contraseña alguna (ver `demoAdapter.ts`: el login demo genera el token solo a partir del email, ignorando el password recibido). Esto es aceptable para una demo pública sin backend real, pero **si por error `MODO_DEMO` quedara activo en un despliegue que sí apunta a un backend real, cualquier password sería aceptada client-side** — riesgo mitigado porque `MODO_DEMO` solo se activa vía variable de entorno de build (`VITE_MODO_DEMO=true`), no en runtime, y el build de producción normal (`npm run build`) no la define. Ver hallazgo de severidad correspondiente.

## Conclusión de la sección

RBAC del frontend: **implementado de forma real** (no solo cosmético) a nivel de rutas y menú, con las limitaciones inherentes a cualquier SPA (la autorización definitiva depende del backend, no verificable aún). No se detectó "RBAC solo visual" — hay bloqueo real de renderizado, no solo ocultamiento de botones.
