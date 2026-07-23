# Trazabilidad SHA-256 encadenada — análisis exhaustivo bajo concurrencia

## Estructura exacta verificada

```
hash_actual = SHA256(previous_hash + timestamp_canónico_UTC_ISO8601 + json.dumps(payload, sort_keys=True, default=str))
```

Confirmado en `domain/value_objects/hash_encadenado.py::HashEncadenado.calcular_hash()`. Esto corresponde exactamente a la estructura conceptual pedida: `SHA-256(previous_hash + timestamp + payload)`, con dos detalles de implementación correctos:
- **Serialización canónica del payload**: `json.dumps(..., sort_keys=True)` — garantiza que el mismo payload lógico (con claves en cualquier orden) produzca siempre el mismo hash. Verificado con test unitario (`test_orden_de_claves_en_payload_no_afecta_el_hash`).
- **Timestamp canónico**: `timestamp_canonico()` fuerza cualquier datetime naive a UTC explícito antes de serializar — evita que SQLite (que devuelve naive datetimes) rompa la verificación posterior. Documentado y con test.

## Primer registro (génesis)

`GENESIS_HASH = "0" * 64` — valor fijo, determinista, sin depender de ningún dato previo. Confirmado usado tanto al insertar el primer registro (`obtener_ultimo_hash()` devuelve `GENESIS_HASH` si la tabla está vacía) como al verificar (`VerificarIntegridadRegistroUseCase` arranca con `previous_hash = GENESIS_HASH`).

## Verificación de cadena (RF-15)

`VerificarIntegridadRegistroUseCase.execute()`: trae TODOS los registros ordenados por `created_at` ascendente, recalcula secuencialmente el hash esperado de cada uno con los mismos datos almacenados, y compara tanto `previous_hash` como `hash_actual` contra lo esperado. Si algo no coincide, retorna `integra=False` y el índice del primer registro inconsistente. **Detecta alteración real**: un test unitario (`test_verificar_detecta_alteracion_del_payload`) confirma que cambiar cualquier campo del payload cambia el hash y la verificación lo detecta.

Complejidad O(n) — coherente con lo declarado en TI y en el backlog (HU-26).

## Auditoría integrada

Cada evento de trazabilidad importante (`LECTURA_TERMICA`, `ALERTA_TERMICA`, y potencialmente otros tipos vistos en el frontend demo: `ACCION_CORRECTIVA`, `REPORTE_BPA`, `CONECTIVIDAD`, `AUDITORIA`) usa el mismo mecanismo de hash encadenado vía `RegistrarHashEncadenadoUseCase`, invocado desde múltiples casos de uso (`RegistrarLecturaTermicaUseCase`, `RegistrarAccionCorrectivaUseCase` — confirmado por import en `alertas_router.py`). Es una cadena **global única**, no separada por dispositivo ni por establecimiento — coherente con el alcance de TI (un solo escenario de validación).

## Terminología: correctamente NO usa "blockchain"

Confirmado por `grep` mental de todo el código de hash/trazabilidad leído: nunca aparece la palabra "blockchain", "block", "chain of blocks", ni ninguna librería de blockchain en `requirements.txt`. Coherente con la posición académica declarada en TI (trazabilidad digital verificable ≠ blockchain).

## Inmutabilidad — verificado por ausencia de endpoints de modificación

**No existe ningún endpoint PATCH/PUT/DELETE sobre `/api/trazabilidad`** en `trazabilidad_router.py` (solo 2 rutas GET). A nivel de base de datos, no hay ninguna restricción `CHECK` o trigger que impida un `UPDATE`/`DELETE` directo sobre la tabla si alguien tuviera acceso SQL directo — la inmutabilidad es **una garantía a nivel de aplicación (API), no a nivel de motor de base de datos**. Esto es consistente con la mayoría de sistemas de este tipo (Postgres no ofrece tablas "append-only" nativas sin configuración adicional de permisos GRANT/REVOKE a nivel de rol de BD, que no se evidencia configurada aquí) — es una limitación real a documentar: **un DBA o un atacante con acceso directo a PostgreSQL podría alterar registros sin que la aplicación lo impida a nivel de esquema**, aunque sí lo detectaría la próxima verificación de integridad (RF-15).

## CASO CONCRETO DE CONCURRENCIA QUE ROMPE LA CADENA — confirmado, no solo teórico

**Ubicación exacta**: `infrastructure/database/repositories/trazabilidad_repository.py::obtener_ultimo_hash()` + `application/use_cases/registrar_hash_encadenado.py::execute()`.

**Secuencia del problema**:
1. Transacción A (p. ej. procesando una lectura MQTT del dispositivo 1) llama `obtener_ultimo_hash()` → obtiene hash `H`.
2. Antes de que A haga `INSERT`+`commit`, Transacción B (p. ej. procesando una lectura MQTT del dispositivo 2, o una acción correctiva de un usuario, llegando casi simultáneamente) también llama `obtener_ultimo_hash()` → **también obtiene el mismo hash `H`** (porque A todavía no ha hecho commit, así que su INSERT no es visible para B bajo el nivel de aislamiento típico READ COMMITTED de PostgreSQL).
3. A calcula `hash_actual_A = SHA256(H + ...)` e inserta.
4. B calcula `hash_actual_B = SHA256(H + ...)` (con su propio timestamp/payload, distinto del de A) e inserta.
5. **Resultado**: dos registros distintos, ambos con `previous_hash = H`, cada uno con su propio `hash_actual` distinto (la restricción `unique=True` sobre `hash_actual` no los bloquea, porque sus valores de hash SÍ son diferentes entre sí al incluir payloads/timestamps distintos). **La cadena se bifurca**: ya no hay un único "último hash" lineal, hay dos ramas paralelas desde `H`.
6. La siguiente inserción (Transacción C) leerá "el último por `created_at` DESC" — es decir, tomará arbitrariamente UNA de las dos ramas (la que tenga el `created_at` más reciente, que depende del orden real de commit, no necesariamente del orden en que A o B leyeron el hash). La otra rama queda "huérfana": su registro sigue en la tabla, pero ningún registro futuro lo referencia como `previous_hash`.
7. **Efecto en `VerificarIntegridadRegistroUseCase`**: como este caso de uso ordena TODOS los registros por `created_at` ascendente y espera una cadena estrictamente lineal, al llegar al segundo de los dos registros bifurcados (el que tenga `created_at` mayor entre A y B), su `previous_hash` almacenado (`H`) **no coincidirá** con el `hash_actual` del registro inmediatamente anterior en el orden de `created_at` (que sería el otro registro bifurcado, con un `hash_actual` distinto de `H`) — **la verificación de integridad reportaría `integra=False`**, marcando como "corrupta" una cadena que en realidad nunca fue alterada maliciosamente, solo bifurcada por una condición de carrera legítima del sistema.

**Es decir**: bajo tráfico concurrente real (varias lecturas MQTT casi simultáneas de dispositivos distintos, o una lectura y una acción correctiva del mismo instante), el sistema puede **auto-generar un falso positivo de "cadena rota"**, sin que haya habido ninguna manipulación real. Esto es un defecto de diseño concreto y demostrable, no solo un riesgo teórico — se sigue directamente de la ausencia de bloqueo (`SELECT ... FOR UPDATE`) o de una transacción serializable en `obtener_ultimo_hash()`.

## Recomendación técnica concreta

- Usar `SELECT ... FOR UPDATE` sobre la última fila de `traceability_records` (o una fila "puntero" dedicada) dentro de la misma transacción que hace el `INSERT`, para serializar el acceso al "último hash" entre transacciones concurrentes.
- Alternativa: aislamiento de transacción `SERIALIZABLE` en PostgreSQL para las operaciones de escritura en esta tabla, con reintento automático ante `SerializationFailure`.
- Alternativa más simple: una columna `secuencia` autoincremental (`BIGSERIAL`) con una restricción única, usada como el verdadero "puntero" de orden en vez de `created_at`, combinada con un candado explícito.

## Rendimiento de la verificación

O(n) sobre el total de registros — coherente con lo declarado. Para el volumen de un prototipo (miles de registros), es trivialmente rápido; a escala de producción con millones de registros, sería una operación costosa ejecutada bajo demanda (no cacheada, no incremental) — aceptable para un prototipo académico, mencionable como deuda técnica de escalabilidad futura.
