# Arquitectura real del backend

## Veredicto de arquitectura: DDD real, no solo nombres de carpeta

Verificado por dirección de dependencias real, no solo por nombres:

```
backend/src/
├── domain/                     ← sin imports de FastAPI, SQLAlchemy ni ningún framework
│   ├── entities/                (LecturaTermica, AlertaTermica, RegistroTrazabilidad, Usuario, AccionCorrectiva — dataclasses puras)
│   ├── value_objects/            (Rol, NivelRiesgo — StrEnum; RangoTermico, HashEncadenado — dataclasses frozen con lógica propia)
│   ├── repositories/              (8 interfaces ABC/Protocol: puertos puros, sin implementación)
│   └── exceptions.py               (jerarquía propia: DomainError → 6 excepciones específicas)
│
├── application/                ← depende de domain (interfaces) + infrastructure (servicios), nunca de FastAPI directamente
│   └── use_cases/                (11 casos de uso, uno por operación de negocio: registrar lectura, clasificar riesgo,
│                                   generar alerta, registrar hash, verificar integridad, auditar acción, autenticar,
│                                   gestionar usuarios, consultar alertas/historial, registrar acción correctiva,
│                                   exportar reporte BPA)
│
├── infrastructure/              ← implementaciones concretas de los puertos de domain
│   ├── security/                 (jwt_handler, password_hasher, rbac, rate_limiter, revocation_store)
│   ├── ai/                        (features, reglas_riesgo, random_forest_service, train_model + models/*.pkl,*.json)
│   ├── hash/                      (sha256_service — wrapper delgado sobre el VO de dominio)
│   ├── mqtt/                      (mqtt_client, payload_schema)
│   ├── database/                  (models — ORM SQLAlchemy —, session, base, repositories/* — 8 implementaciones concretas de las interfaces de domain)
│   └── config.py                  (Settings pydantic-settings)
│
└── interface/                   ← capa HTTP, único punto que conoce FastAPI
    ├── main.py                    (create_app, lifespan MQTT, middlewares, routers)
    └── api/                        (9 routers, deps.py con inyección de dependencias, schemas Pydantic de I/O, mappers dominio→schema,
                                     api_protection, security_headers, sse_broadcaster)
```

## Verificación de la dirección de dependencias (evidencia, no inferencia)

- `domain/entities/lectura_termica.py`: solo importa `dataclasses`, `datetime`, `uuid`, y su propio VO `NivelRiesgo`. **Cero acoplamiento a FastAPI/SQLAlchemy** — confirmado leyendo el archivo completo.
- `domain/repositories/i_trazabilidad_repository.py` y las otras 7 interfaces: son puertos abstractos; las implementaciones concretas (`SQLAlchemyTrazabilidadRepository`, etc.) viven en `infrastructure/database/repositories/`, e implementan la interfaz de dominio — patrón de puertos y adaptadores correctamente aplicado.
- `application/use_cases/registrar_lectura_termica.py`: recibe repositorios como interfaces inyectadas por constructor (`ILecturaRepository`, no `SQLAlchemyLecturaRepository`), coherente con Inversión de Dependencias.
- `interface/api/lecturas_router.py`: es el único lugar que instancia las clases concretas de SQLAlchemy y las inyecta en los casos de uso — correcto, la capa de interface es la que "cablea" (composition root a nivel de request).

**Conclusión**: esta es una arquitectura DDD/hexagonal genuina, no una imitación cosmética de carpetas. La dirección de dependencias señala consistentemente hacia adentro (Domain no depende de nada externo).

## Entidades vs Value Objects

- **Entidades** (con identidad, mutables en memoria vía `@dataclass(slots=True)` sin `frozen`): `LecturaTermica`, `AlertaTermica`, `Usuario`, `RegistroTrazabilidad`, `AccionCorrectiva`. Tienen comportamiento propio no trivial (ej. `LecturaTermica.es_lectura_valida()`, `.diferencia_sensores()`) — **no son anémicas**, encapsulan lógica de negocio real.
- **Value Objects** (`@dataclass(frozen=True, slots=True)`, inmutables, sin identidad): `RangoTermico` (con `contiene()`, `distancia_al_limite()`), `HashEncadenado` (con `calcular_hash()`, `encadenar()`, `verificar()`). Comportamiento rico, no solo contenedores de datos.

## Casos de uso vs lógica en routers

Verificado que los routers (`interface/api/*.py`) son delgados: reciben el request, resuelven dependencias (sesión DB, usuario autenticado), instancian el caso de uso correspondiente, y traducen excepciones de dominio a HTTP. **No se encontró lógica de negocio (reglas, cálculos, validaciones de dominio) directamente en ningún router** — toda vive en `application/use_cases/` o `domain/`. Esto es lo opuesto de un "fat controller" y confirma disciplina arquitectónica real.

## Ciclo de vida (lifespan) y composición

`interface/main.py::lifespan()`: al arrancar, crea el `SSEBroadcaster` en `app.state`, y si `mqtt_enabled` y no es entorno `test`, abre una sesión MQTT persistente que consume mensajes en background y los enruta al mismo pipeline que usan los endpoints HTTP de ingesta (`RegistrarLecturaTermicaUseCase`). Si el broker no responde (`aiomqtt.MqttError`), el backend **continúa funcionando sin MQTT** en vez de fallar el arranque — decisión de resiliencia razonable para un prototipo (aunque significa que un fallo de conectividad al broker es silencioso más allá de un log de warning).

## Inyección de dependencias

FastAPI `Depends()` se usa consistentemente para: sesión de BD (`DbSessionDep`), settings (`SettingsDep`), usuario autenticado (`CurrentUserDep`), handler JWT (`JWTHandlerDep`), y control de rol (`require_roles(...)` como factory de dependencia). Patrón limpio y sin duplicación.

## No se detectaron

- Importaciones circulares (estructura de capas estrictamente unidireccional).
- Entidades anémicas (todas tienen comportamiento propio).
- Lógica de negocio filtrada a los routers.
- Unit of Work explícito como patrón nombrado, pero la sesión de SQLAlchemy async (`AsyncSession`) cumple ese rol implícitamente (un `session.commit()` por request en el router, después de que todos los casos de uso relacionados usan la misma sesión) — funcionalmente equivalente, aunque no está nombrado como patrón UoW explícito en el código.
- Eventos de dominio explícitos (no hay un bus de eventos ni patrón Observer formal) — la "propagación" de una lectura hacia alerta+hash+SSE ocurre por orquestación directa y secuencial dentro de `RegistrarLecturaTermicaUseCase`, no por eventos de dominio publicados/suscritos. Es una decisión de diseño válida para este tamaño de sistema, no un defecto.
