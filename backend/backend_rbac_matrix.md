# RBAC real — matriz servidor por endpoint

Mecanismo: `require_roles(*roles)` (factory de dependencia en `deps.py`) invoca `verificar_permiso()` (`infrastructure/security/rbac.py`), que primero comprueba si `rol_actual == Rol.ADMINISTRADOR` (bypass total), y si no, exige que el rol esté en la tupla de roles permitidos declarada en cada router. **Esto se ejecuta en el servidor, en cada request, antes de tocar la lógica de negocio** — no es una verificación cosmética ni delegada al cliente.

| Endpoint | Administrador | Farmacéutico | Técnico | Código que aplica el control | Estado |
|---|---:|---:|---:|---|---|
| `POST /api/lecturas` | ✅ (bypass) | ✅ | ✅ | `require_roles(Rol.TECNICO, Rol.FARMACEUTICO)` | Real |
| `GET /api/lecturas` (historial) | ✅ | ✅ | ✅ | ídem | Real |
| `GET /api/lecturas/{id}` | ✅ | ✅ | ✅ | ídem | Real |
| `GET /api/alertas` | ✅ | ✅ | ✅ | ídem | Real |
| `PATCH /api/alertas/{id}/revisar` | ✅ (bypass) | ✅ | ❌ | `require_roles(Rol.FARMACEUTICO)` | Real — técnico recibe 403 |
| `POST /api/alertas/{id}/acciones-correctivas` | ✅ | ✅ | ✅ | `require_roles(Rol.TECNICO, Rol.FARMACEUTICO)` | Real |
| `GET /api/trazabilidad` | ✅ | ✅ | ✅ | ídem | Real |
| `GET /api/trazabilidad/verificar` | ✅ | ✅ | ✅ | ídem | Real |
| `GET /api/reportes/bpa` | ✅ (bypass) | ✅ | ❌ | `require_roles(Rol.FARMACEUTICO)` | Real — técnico recibe 403 |
| `GET /api/auditoria` | ✅ | ❌ | ❌ | `require_roles(Rol.ADMINISTRADOR)` | Real — único rol permitido explícito es admin |
| `POST /api/usuarios` | ✅ | ❌ | ❌ | `require_roles(Rol.ADMINISTRADOR)` | Real |
| `GET /api/usuarios` | ✅ | ❌ | ❌ | `require_roles(Rol.ADMINISTRADOR)` | Real |
| `GET /api/ia/modelo` | ✅ (bypass) | ✅ | ❌ | `require_roles(Rol.FARMACEUTICO)` | Real, pero sin consumidor frontend |
| `POST /api/ia/clasificar` | ✅ (bypass) | ✅ | ❌ | `require_roles(Rol.FARMACEUTICO)` | Real, pero sin consumidor frontend |

## Comparación exacta con la matriz de roles del frontend (Fase 2)

Coincidencia perfecta en los dos casos más sensibles verificados en ambos lados:
- `revisar_alerta`: frontend (`puedeRevisar = tienePermiso(rol, ['farmaceutico'])`) ↔ backend (`require_roles(Rol.FARMACEUTICO)`) — **idénticos**.
- `/reportes` (ruta completa protegida en frontend con `roles: ['farmaceutico']`) ↔ backend (`require_roles(Rol.FARMACEUTICO)`) — **idénticos**.
- `/usuarios` y `/auditoria` (frontend `roles: []` → solo admin) ↔ backend (`require_roles(Rol.ADMINISTRADOR)`) — **idénticos**.

**No se detectó ningún endpoint donde el backend sea MÁS permisivo que lo que el frontend deja creer al usuario** (es decir, no hay ningún caso donde la UI oculte un botón pero el backend igual permitiría la acción a un rol no autorizado). Esto es la verificación crítica que Fase 2 dejó pendiente ("la autorización real depende del backend, no verificable aún") — **ahora confirmada: el RBAC es real de extremo a extremo, no solo un adorno visual del frontend.**

## Escalada de privilegios

- **Vertical** (técnico → farmacéutico/administrador): bloqueada en cada endpoint sensible, verificado arriba.
- **Horizontal** (un farmacéutico accediendo a datos de otro farmacéutico/otra sesión): no aplica en este dominio — no hay segmentación por farmacia/tenant en el modelo de datos (`device_id` es global, no asociado a una organización/tenant específico) — el sistema asume una única farmacia validada (consistente con el alcance de TI: un solo escenario representativo). **No hay multi-tenancy, por diseño, no por descuido.**

## Variantes de roles / usuarios con múltiples roles

`RoleModel` en la BD tiene una columna `nombre` única y `UserModel.rol_id` es una FK obligatoria (`nullable=False`) a un único rol — **un usuario solo puede tener un rol a la vez**, sin soporte de roles múltiples. Coherente con el diseño de TI (RF-17: "tres niveles"). No se detectaron variantes de escritura del rol (mayúsculas, guiones, tildes) — el enum `Rol(StrEnum)` en Python y la matriz de `ROLES_INICIALES` en la migración usan exactamente los mismos 3 literales en minúsculas sin tilde, en ambos extremos del sistema (frontend, backend, migración).

## Endpoints administrativos sin protección

No se encontró ningún endpoint mutante (POST/PATCH/DELETE) sin `require_roles(...)` o sin `Depends(get_current_user)` al menos implícito vía `CurrentUserDep`. El único endpoint verdaderamente público es `POST /api/auth/login` (por diseño, es el punto de entrada) y `GET /health` (probe de infraestructura, sin datos sensibles).
