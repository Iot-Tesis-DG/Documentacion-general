# Trazabilidad Product Backlog (Anexo5) vs RF/RNF de TI

Fuente: `Anexo5_PB_Soto_Diego_Gamio_Brenda.xlsx`, hojas "Tareas pendientes" (46 filas), "Historias de Usuario" (42 HU), "Epics" (6 épicas), "Sprints" (7 sprints × 4 semanas = 28 semanas, 1120 horas totales, 208 story points).

## Matriz Historia → Épica → Sprint → RF/RNF

| Historia | Épica | Sprint | Criterio de aceptación (resumen) | RF/RNF relacionado | Estado de coherencia | Observación |
|---|---|---|---|---|---|---|
| HU-01 Captura DS18B20 | EP01 | 1 | 3 escenarios Gherkin (lectura, sensor error, fuera de rango) | RF-02 | Coherente | — |
| HU-02 Captura SHT31 temp | EP01 | 1 | 3 escenarios | RF-01 | Coherente | — |
| HU-03 Humedad SHT31 | EP01 | 1 | 3 escenarios | RF-01 (parcial) | Coherente | Prioridad Media |
| HU-04 Apertura MC-38 | EP01 | 1 | 3 escenarios (debounce, apertura prolongada) | RF-03 | Coherente | Prioridad Media |
| HU-05 Payload JSON | EP01 | 1 | 3 escenarios | RF-04 | Coherente | — |
| HU-06 Buffer LittleFS | EP01 | 2 | 3 escenarios (FIFO, corrupción) | RF-06 | Coherente | — |
| HU-07 Sync buffer | EP01 | 2 | 3 escenarios (QoS1, PUBACK) | RF-06, RNF-07 | Coherente | No menciona explícitamente el límite de 30s de RNF-07 |
| HU-08 Backoff reconexión WiFi | EP01 | 2 | 3 escenarios | (implícito, no RF/RNF directo) | Fuera de alcance explícito de TI pero coherente con arquitectura | — |
| HU-09 TLS handshake | EP02 | 2 | 3 escenarios | RF-05, RNF-05 | Coherente | — |
| HU-10 Auth SNI | EP02 | 2 | 3 escenarios | RNF-05 | Coherente | — |
| HU-11 QoS 1 publish | EP02 | 3 | 3 escenarios | RF-05 | Coherente | — |
| HU-12 Suscripción backend aiomqtt | EP02 | 3 | 3 escenarios | (infraestructura, soporta RF-07) | Coherente | — |
| HU-13 LWT | EP02 | 3 | 3 escenarios | RF-18 | Coherente | — |
| HU-14 Bloqueo puerto 1883 | EP02 | 3 | 3 escenarios | RNF-05 | Coherente | — |
| HU-15 Validación Pydantic | EP03 | 3 | 3 escenarios | RF-07 (validación previa a persistencia) | Coherente | — |
| HU-16 Carga modelo RF (lifespan) | EP03 | 4 | 3 escenarios (RuntimeError si falta) | RF-08 | Coherente | — |
| HU-17 Feature engineering | EP03 | 4 | 3 escenarios | RF-08 (soporte) | Coherente | — |
| HU-18 Inferencia RF | EP03 | 4 | 3 escenarios (baja confianza→flag) | RF-08 | Coherente | **Ninguna HU mide/entrena/valida F1-score — ver gap RNF-04 abajo** |
| HU-19 Emisión SSE backend | EP03 | 4 | 3 escenarios | RF-11 | Coherente | — |
| HU-20 Alerta preventiva | EP03 | 4 | 3 escenarios (anti-spam) | RF-09 | Coherente | — |
| HU-21 Alerta crítica | EP03 | 4 | 3 escenarios | RF-09 | Coherente | — |
| HU-22 Persistencia JSONB | EP03 | 3 | 3 escenarios (concurrencia) | RF-07 | Coherente | — |
| HU-23 Notificación Email/Telegram | EP03 | 7 | 3 escenarios (throttling) | (no está en RF-01..18 de TI — funcionalidad adicional) | **Fuera de alcance declarado en TI** | TI no menciona notificación por correo/Telegram como RF; es una funcionalidad añadida por el equipo en el backlog, sin requisito formal correspondiente. No es necesariamente negativo, pero constituye "elemento implementado(planeado) pero no documentado" en TI |
| HU-24 Hash SHA-256 | EP04 | 5 | 3 escenarios (determinismo) | RF-14 | Coherente | — |
| HU-25 Previous_hash encadenamiento | EP04 | 5 | 3 escenarios (génesis, concurrencia) | RF-14 | Coherente | — |
| HU-26 Verificación O(n) cadena | EP04 | 5 | 3 escenarios (detección alteración) | RF-15 | Coherente | — |
| HU-27 Acción correctiva | EP04 | 5 | 3 escenarios | RF-10 | Coherente | — |
| HU-28 Hash sobre acciones correctivas | EP04 | 5 | 3 escenarios (append-only) | RF-14, RF-16 | Coherente | — |
| HU-29 Backup automático | EP04 | 7 | 3 escenarios | (no RF/RNF explícito — buena práctica operativa) | Fuera de alcance explícito de TI | Prioridad Baja, consistente con ser adicional |
| HU-30 Calibración sensores | EP04 | 5 | 3 escenarios | (no RF/RNF explícito) | Fuera de alcance explícito de TI | Funcionalidad regulatoria BPA añadida, razonable pero no está en RF-01..18 |
| HU-31 Gráfica tendencia térmica | EP05 | 6 | 3 escenarios (estado vacío, downsampling) | RF-11 | Coherente | — |
| HU-32 Dashboard SSE tiempo real | EP05 | 6 | 3 escenarios (reconexión) | RF-11 | Coherente | — |
| HU-33 Tarjetas KPI | EP05 | 6 | 3 escenarios | RF-11 (soporte visual) | Coherente | — |
| HU-34 Semáforo riesgo IA | EP05 | 6 | 3 escenarios | RF-08 (visualización) | Coherente | — |
| HU-35 Alerta puerta abierta UI | EP05 | 6 | 3 escenarios | RF-03 (visualización) | Coherente | — |
| HU-36 Filtros historial | EP05 | 7 | 3 escenarios (rango inválido) | RF-12 | Coherente | — |
| HU-37 Checklist BPA digital | EP05 | 7 | 3 escenarios (borrador local) | RF-13 (parcial) | Coherente | — |
| HU-38 Exportación PDF | EP05 | 7 | 3 escenarios (paginación) | RF-13 | Coherente | — |
| HU-39 Login seguro | EP06 | 1 | 3 escenarios (enumeración, fuerza bruta) | RF-17 | Coherente | — |
| HU-40 JWT en memoria (no localStorage) | EP06 | 1 | 3 escenarios | RNF-06 (seguridad) | Coherente | — |
| HU-41 RBAC 3 roles | EP06 | 7 | 3 escenarios (403, cambio de rol) | RF-17 | Coherente | Roles aquí: Técnico/Farmacéutico/Auditor — **TI define Administrador/Farmacéutico/Técnico** (RF-17) — el backlog sustituye "Administrador" por "Auditor" como tercer rol. Ver inconsistencia abajo |
| HU-42 Bitácora auditoría inmutable | EP06 | 5 | 3 escenarios (append-only) | RF-16 | Coherente | — |
| **HU-43** (huérfana) | **ninguna** | **ninguno** | Copia literal de criterios de HU-41 (RBAC) bajo título de "baja/reemplazo de dispositivo IoT" | — | **Defecto de backlog** | Ver detalle abajo |

## Gaps detectados (requisito TI sin historia correspondiente)

**[ALTO] RNF-04 (F1-Score ponderado ≥0.85) no tiene historia de usuario.** Las HU-16/17/18 cubren carga del modelo, feature engineering e inferencia, pero ninguna historia contempla el **entrenamiento**, la **validación con conjunto de prueba**, ni el **cálculo/registro de métricas** (F1, precisión, recall, accuracy por clase) que TI exige como criterio de validación técnica (RNF-04) y que además Product Backlog "Estado" = "No se ha iniciado" en las 46 tareas confirma que, a la fecha de creación del Anexo5, ni siquiera existía still el modelo entrenado. Esto es un vacío de planificación: sin una historia que planifique explícitamente el entrenamiento/validación del modelo, existe riesgo real de que la métrica RNF-04 nunca se mida formalmente ni se documente con evidencia reproducible.

**RNF-02 (disponibilidad ≥95%) y RNF-10 (carga dashboard ≤3s) tampoco tienen historia de usuario dedicada** — son atributos de calidad transversales que normalmente se miden en fase de pruebas de validación técnica (Sprint 8/TP2, fuera del backlog de 7 sprints de construcción), lo cual es razonable y no se marca como defecto.

## Historia huérfana / corrupta: HU-43

En la hoja "Tareas pendientes" (fila B29) existe una historia con ID `HU-43`, título "Administrador/Técnico — registrar baja o reemplazo de dispositivo IoT sin corromper trazabilidad", pero sus 3 escenarios de aceptación son **copia literal palabra por palabra** de los escenarios de HU-41 (RBAC: "Dado que un usuario con rol de Administrador navega a la ruta de gestión de usuarios..."). Esta historia:
- **No aparece** en la hoja "Historias de Usuario" (que termina en HU-42).
- **No aparece** en ninguna Épica (EP01-EP06 solo referencian HU-01 a HU-42).
- **No aparece** en ningún Sprint (los 7 sprints suman exactamente 42 historias).

Es una historia fantasma: fue dada de alta con un ID y una intención (gestión de baja de dispositivos IoT, relevante para trazabilidad DIGEMID) pero su contenido de aceptación real nunca se escribió — quedó con contenido copiado de otra historia por error, y en consecuencia no fue planificada ni asignada. **Esto significa que la funcionalidad "dar de baja/reemplazar un dispositivo IoT sin corromper trazabilidad" — relevante y mencionada en la propia HU-43 — no está realmente planificada en ningún sprint**, pese a existir una fila que aparenta cubrirla.

## RBAC: discrepancia de roles TI vs Backlog

TI (RF-17): *"autorización por roles (RBAC) con tres niveles: administrador, farmacéutico y técnico."*
Backlog (HU-41, HU-28, HU-37, HU-42 combinados): los roles que aparecen en las historias de usuario son **Técnico de farmacia, Farmacéutico responsable / Químico Farmacéutico (Regente), Administrador del sistema, y Responsable de Auditoría / Auditor** — es decir, el backlog opera de facto con **4 roles/perfiles narrativos** (Técnico, Farmacéutico/Regente, Administrador, Auditor), no 3. El propio texto de HU-41 dice literalmente "según el rol del usuario autenticado (Técnico, Farmacéutico, Auditor)" — omitiendo "Administrador" en su propia lista de 3, aunque "Administrador" es quien ejecuta la acción en el escenario 1 de esa misma historia. Esto es una inconsistencia real de nomenclatura de roles entre TI (3 roles fijos) y el backlog (uso inconsistente de 3-4 nombres de rol a lo largo de las historias). **A verificar en el backend (Fase 3) cuántos roles existen realmente en el enum/tabla de roles del código.**

## Funcionalidades del backlog sin requisito formal en TI (posible scope creep documentado)

- HU-23 (notificación email/Telegram) — no está en RF-01 a RF-18.
- HU-29 (backup automático PostgreSQL) — no está en RF-01 a RF-18 (aunque es buena práctica operativa razonable).
- HU-30 (alerta de vencimiento de calibración de sensores) — no está en RF-01 a RF-18, aunque se conecta con BPA/DIGEMID.

Estas tres son adiciones razonables del equipo de desarrollo más allá del documento formal de requisitos, no necesariamente un problema, pero deben mencionarse explícitamente en la sustentación como "funcionalidades adicionales no derivadas directamente de RF-01..18" para evitar que el jurado las perciba como items no planificados.

## Historias duplicadas o ambiguas

Ninguna historia duplica el ID de otra (IDs únicos HU-01 a HU-42, más la huérfana HU-43). No se detectaron historias con texto idéntico entre sí salvo el caso ya descrito de HU-43 (copia de HU-41).

## Estado de ejecución declarado

Las 46 filas de "Tareas pendientes" muestran **Estado = "No se ha iniciado"** para el 100% de las historias, a la fecha de creación del archivo. Esto es un dato crítico de línea base: **el Product Backlog documenta que, en el momento de su elaboración, ninguna historia había comenzado desarrollo.** Se deberá contrastar en Fase 2/3 si el código real de frontend/backend ha avanzado más allá de este estado documentado (lo más probable, dado que existen carpetas `frontend/` y `backend/` con contenido), lo cual indicaría que el Product Backlog no se actualizó para reflejar el progreso real — o que se trabajó sin actualizar la hoja de cálculo.
