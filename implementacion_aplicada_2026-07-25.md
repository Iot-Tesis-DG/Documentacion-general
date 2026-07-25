# Informe de implementación — plan al 100 % aplicado

**Fecha:** 25 de julio de 2026
**Alcance:** ejecución de `plan_implementacion_100porciento.md` sobre `backend/` y `frontend/`.
**Estado de pruebas:** backend **340 pruebas en verde** (línea base antes de este trabajo: 178) y frontend **60 pruebas en verde** (línea base: 0).
**Dependencias vulnerables:** 0 en backend (`pip-audit`), 0 en producción de frontend (`npm audit --omit=dev`).
**Cobertura del plan:** las 27 tareas, incluidas las dos marcadas como OPCIONAL. No queda nada sin aplicar.

---

## 1. Resumen ejecutivo

Se implementaron las cuatro tareas de prioridad **ALTA** que el propio plan señalaba
como determinantes del cierre de alcance (checklist BPA, exportación PDF,
notificación ante excursión crítica y registro de calibración), más todas las de
prioridad media y baja, y también las dos marcadas como OPCIONAL (sección 6).
Durante la ejecución se detectaron y corrigieron **cinco defectos que el plan
no contemplaba** (sección 5).

Varias tareas del plan ya estaban implementadas en el repositorio: el plan se
redactó contra un estado anterior del frontend. Se verificaron una a una antes
de tocar nada (sección 4).

---

## 2. Backend

| # | Tarea | HU/RF | Estado | Evidencia |
|---|-------|-------|--------|-----------|
| B1 | Checklist BPA completo | HU-37 | ✅ | `tests/integration/test_checklist_bpa_api.py` (11) |
| B2 | Exportación PDF de reportes | HU-38, RF-13 | ✅ | `tests/integration/test_reporte_pdf_api.py` (7) |
| B9 | Notificación email/Telegram | HU-23 | ✅ | `tests/unit/test_notificacion_service.py` (10) + `test_notificacion_episodio.py` (5) |
| B10 | Registro de calibración | HU-30 | ✅ | `tests/integration/test_calibracion_api.py` (7) |
| B3 | SSE desde ingesta HTTP | B-06 | ✅ | `test_ingesta_timestamp_y_sse.py` |
| B4 | Manejador MQTT `/eventos` | B-09 | ✅ | `tests/integration/test_mqtt_eventos.py` (8) |
| B5 | Validación de timestamp | B-10 | ✅ | `test_ingesta_timestamp_y_sse.py` (12) |
| B6 | Índices de base de datos | B-11 | ✅ | migración 0006 + `test_migraciones.py` |
| B7 | `password_min_length` | B-12 | ✅ ya existía | `test_seguridad_api.py::test_crear_usuario_rechaza_passwords_debiles` |
| B8 | Pruebas adicionales | — | ✅ | migraciones (5) + cobertura RBAC (79) |
| B11 | Escaneo de dependencias | — | ✅ | `pip-audit`: sin vulnerabilidades |
| B12 | Verificación de CSP | — | ✅ | validado en navegador real; SSE, Tailwind y ECharts funcionan |
| B13 | Rate limiting por endpoint | — | ✅ | `tests/integration/test_rate_limit_endpoints.py` (11) |

### Decisiones de diseño que se apartan del plan

**PDF con ReportLab, no WeasyPrint.** El plan proponía WeasyPrint. Se descartó
porque exige `libpango` y `libcairo` instaladas en el sistema operativo, y en el
equipo de desarrollo no están (`ctypes.util.find_library` devuelve `None` para
ambas): el backend no habría arrancado. ReportLab se instala como *wheel* sin
dependencias nativas, que es el requisito real — el reporte debe poder generarse
en cualquier equipo donde corra el backend, incluido el de la sustentación.

**Checklist de 10 ítems, no de 6.** El plan definía seis campos booleanos. El
frontend ya mostraba **diez** ítems, traducidos y anclados al Manual de BPA
(RM N.º 132-2015/MINSA). Se implementó el backend sobre esos diez: es un
superconjunto del plan, evita una regresión en la interfaz y mantiene la
trazabilidad documental con la norma citada.

**Ventana de timestamp: 48 h hacia atrás, no 2 h.** El plan proponía rechazar
lecturas de más de 2 horas de antigüedad. Eso habría descartado en silencio el
caso de uso real más importante: el ESP32 almacena lecturas cuando pierde
conectividad y las reenvía al reconectar — precisamente durante una caída
nocturna, que es cuando la evidencia térmica más importa. Hacia el futuro sí se
mantiene una tolerancia mínima (10 min, solo deriva de reloj): un dispositivo no
puede reportar el futuro, y aceptarlo permitiría desplazar el orden temporal de
la cadena.

**La verificación de integridad dentro del PDF es de solo lectura.** El endpoint
`GET /api/trazabilidad/verificar` tiene efectos colaterales por diseño (evento de
emergencia, snapshot forense, flag global). Reutilizarlo tal cual en la
exportación habría hecho que *descargar un reporte* disparara eventos forenses.
El caso de uso del PDF instancia el verificador sin repositorio de corrupción.
Cubierto por `test_descargar_pdf_no_dispara_evento_de_corrupcion`.

---

## 3. Frontend

| # | Tarea | HU/RF | Estado |
|---|-------|-------|--------|
| F1 | Checklist BPA contra backend real | HU-37 | ✅ reemplaza `localStorage` |
| F2 | Botón exportar PDF | HU-38, RF-13 | ✅ acción primaria en Reportes |
| F4 | Vista de métricas del modelo IA | RNF-04 | ✅ página nueva `/metricas-ia` |
| F5 | Logout real contra la API | B-07 | ✅ revoca el `jti` en el servidor |
| F6 | ErrorBoundary global | F-09 | ✅ |
| F7 | Code-splitting por ruta | F-11 | ✅ ECharts (606 KB) fuera del paquete inicial |
| F8 | ESLint | F-07 | ✅ sin incidencias en `src/` |
| F3 | Indicador de conectividad | RF-18 | ✅ ya existía |
| FS1 | Pantalla de privacidad | HU-44 | ✅ ya existía (`PrivacyConsentModal`) |
| FS2 | Gestión de dispositivos | HU-43 | ✅ ya existía; calibración añadida en backend |
| FS3 | Banner de integridad de cadena | — | ✅ ya existía |
| FS4 | Botón «aislar corrupción» | HU-47 | ✅ ya existía |
| F9 | Pruebas con Vitest | F-06 | ✅ 60 pruebas en 7 archivos |
| FS5 | Aviso de sesión próxima a expirar | — | ✅ verificado en navegador |

---

## 4. Validación de extremo a extremo

Se levantó el sistema completo (backend + frontend + base de datos migrada desde
cero) y se recorrió con un navegador real controlado por Playwright:

- Inicio de sesión y consentimiento de privacidad (Ley N.° 29733).
- Ingesta de **90 lecturas** con un perfil térmico realista (normal, riesgo
  preventivo y una excursión crítica sostenida).
- Dashboard: KPIs, curva térmica y clasificación de IA con confianza y versión
  de modelo visibles.
- Checklist BPA: se comprobó que la página **carga desde el backend** el estado
  exacto guardado (9/10, «con observaciones», texto de la observación y sello de
  última modificación).
- Métricas IA: F1 ponderado 0.9671, validación cruzada 0.9727 ± 0.0023,
  desempeño por clase, peso de cada variable y hashes de procedencia.
- Reportes: 90 lecturas, 3 alertas, 123 registros de trazabilidad; **descarga de
  PDF verificada desde la interfaz**.
- Registro del servidor durante toda la sesión: **solo códigos 200 y 201**,
  ninguna excepción.

El PDF resultante se inspeccionó visualmente: portada, resumen con porcentaje de
tiempo en rango (82.2 %), veredicto de integridad sobre 123 registros
encadenados, checklists declarados, historial de lecturas, alertas y la cadena de
hashes.

---

## 5. Defectos encontrados durante la implementación

Ninguno de estos figuraba en el plan; se detectaron al escribir las pruebas.

**5.1 — Las migraciones solo funcionaban en PostgreSQL.**
`alembic upgrade head` fallaba desde cero con
`NotImplementedError: No support for ALTER of constraints in SQLite dialect`.
Las migraciones 0002, 0004 y 0005 usaban `create_unique_constraint` y
`create_foreign_key` directos. La suite nunca lo detectó porque crea el esquema
con `Base.metadata.create_all`, que no ejecuta ni una migración: un modelo podía
divergir de su migración sin que nadie se enterara hasta desplegar. Se
convirtieron a `op.batch_alter_table` (en PostgreSQL emite exactamente los mismos
`ALTER`) y se añadió `tests/integration/test_migraciones.py`.

**5.2 — Condición de carrera en el cierre de sesión.**
Al conectar el `logout` real, la petición de revocación salía **sin cabecera de
autorización**: el interceptor de Axios lee el token en un microtask, y
`setAccessToken(null)` se ejecuta antes. El servidor habría respondido 401 y el
`jti` habría quedado sin revocar — es decir, el arreglo habría sido cosmético. Se
fija la cabecera explícitamente con el token capturado antes de limpiarlo.

**5.3 — El checklist no permitía registrar un hallazgo.**
El botón de guardar quedaba deshabilitado mientras no estuvieran los diez ítems
marcados. Un farmacéutico que encontrara una no conformidad —justo el caso que
hay que documentar— no podía guardarla. El backend sí lo permitía. Se eliminó el
bloqueo: un ítem sin marcar es una declaración de no conformidad, y la
verificación se registra como «con observaciones».

**5.4 — Pruebas con fechas de calendario fijas.**
Cinco pruebas usaban timestamps absolutos (`datetime(2026, 7, 22, …)`). Eran
bombas de tiempo latentes que la validación B-10 hizo estallar. Se anclaron al
reloj actual, conservando las propiedades que verifican.

**5.5 — `agregar()` de usuarios descartaba la mitad de la entidad.**
`SQLAlchemyUsuarioRepository.agregar` construía el `UserModel` solo con nombre,
email, hash y rol. Los campos de privacidad (HU-44, Ley N.° 29733) y de ciclo de
vida (HU-45) los rellenaban los `default` del modelo: crear un usuario como
inactivo lo guardaba **activo**, y crearlo con un consentimiento ya registrado
lo guardaba **sin** ese consentimiento. En silencio, porque `agregar` devolvía
la entidad releída del propio modelo y por tanto siempre coherente consigo
misma.

La suite no lo detectaba porque creaba usuarios siempre con los valores por
defecto, que coinciden con los del modelo. Se detectó al ver aparecer el modal
de privacidad durante la validación de FS5 con un usuario creado como ya
consentido.

Arreglado persistiendo los trece campos. `_to_entity` leía trece y `agregar`
escribía cuatro: la prueba `test_agregar_no_pierde_ningun_campo_de_la_entidad`
recorre los campos del dataclass de forma genérica, así que un campo nuevo
olvidado en `agregar` la rompe sin escribir nada más. Verificado por mutación:
reintroducido el defecto, 6 de las 7 pruebas caen.

---

## 6. Segunda pasada: las tareas opcionales

Las tres tareas que habían quedado fuera en la primera pasada se aplicaron
después. No queda ninguna pendiente.

### 6.1 F9 — pruebas de frontend con Vitest

**60 pruebas en 7 archivos**, bajo `src/tests/`. Configuración propia en
`vitest.config.ts` (sin Tailwind ni troceado de bundle: no cambian un resultado
y sí alargan cada corrida).

| Archivo | Qué fija |
|---------|----------|
| `application/stores/authStore.test.ts` | login, logout, revocación del `jti`, consentimiento HU-44 |
| `application/hooks/useChecklistBPA.test.ts` | carga, `null` como «aún sin verificar», guardado y sus fallos |
| `application/hooks/useExpiracionSesion.test.ts` | cuenta atrás, expiración, suspensión del equipo |
| `infrastructure/apiClient.test.ts` | cabecera de autorización, 401 vs 403 |
| `presentation/pages/LoginPage.test.tsx` | formulario, credenciales, 429, aviso de sesión |
| `presentation/pages/ChecklistBPAPage.test.tsx` | los diez ítems, rehidratación, no conformidades |
| `presentation/guards/RouteGuards.test.tsx` | RBAC, jerarquía del administrador, `roles={[]}` |

En vez de simular el módulo `apiClient`, las pruebas sustituyen el **adaptador**
de Axios. La petición atraviesa así los interceptores reales — que es justo
donde vivía el defecto 5.2 — y se puede afirmar sobre lo que saldría por la red.

Dos pruebas están marcadas como `REGRESIÓN` porque fijan defectos que ya
ocurrieron (5.2 y 5.3). **Se verificó que fallan**: se reintrodujeron ambos
defectos a propósito y las dos pruebas se pusieron en rojo, junto con dos
colaterales. Una prueba verde que no puede romperse no demuestra nada.

### 6.2 B13 — cuotas por endpoint

El motivo por el que se había aplazado —el riesgo de estrangular el volcado del
buffer del ESP32— **no se sostiene**: ese reenvío viaja por MQTT (RF-05/RF-06) y
no atraviesa la pila HTTP, así que el RNF-07 (sincronización ≤30 s tras
reconectar) es indiferente a un límite REST. Revisado esto, se aplicó.

| Endpoint | Cuota | Clave | Motivo |
|----------|-------|-------|--------|
| `GET /api/trazabilidad/verificar` | 5 / min | usuario | Rehashea la cadena entera: O(n) sobre una tabla que solo crece |
| `POST /api/lecturas` | 120 / min (configurable) | IP | Vía REST secundaria; el buffer del ESP32 va por MQTT |

La verificación se limita **por usuario y no por IP** porque en una farmacia
todos los puestos salen por la misma dirección: con clave por IP, el primero en
verificar dejaría sin cuota a los demás. Cubierto por
`test_la_cuota_es_por_usuario_no_por_ip`.

Los limitadores viven en `app.state`, no a nivel de módulo, para que cada
instancia de la aplicación arranque con la cuota limpia. Un límite global
haría que las pruebas heredaran cuota agotada y fallaran de forma intermitente
según el orden de ejecución.

### 6.3 FS5 — aviso de sesión próxima a expirar

`decodificarSesion` ahora normaliza el `exp` del JWT a milisegundos, y
`useExpiracionSesion` avisa a los 5 minutos y fuerza el cierre al vencer.

Se usa un intervalo de un segundo en vez de un `setTimeout` calculado de una
vez: con temporizador único, suspender el equipo retrasaría el disparo y la
sesión seguiría viva en pantalla mucho después de haber caducado el token.
Cubierto por `test('expira aunque el equipo se suspenda…')`.

Sin este aviso el usuario descubría la caducidad al pulsar «guardar»: la
petición devolvía 401 y el trabajo a medio escribir —una verificación BPA, una
acción correctiva— se perdía sin explicación.

**Verificado en navegador real** (Playwright, token de 4 minutos): aparece la
banda con la cuenta atrás, decrece correctamente, y al vencer el plazo la
aplicación redirige sola a `/login` con el aviso «Tu sesión expiró». Cero
errores de consola.

---

## 7. Notas para la sustentación

**Nota sobre `npm audit`.** Tras instalar ESLint, `npm audit` reporta 5
vulnerabilidades altas en su árbol transitivo (`minimatch`). Son
**dependencias de desarrollo**: nunca se empaquetan ni llegan al navegador.
`npm audit --omit=dev` —lo que realmente se despliega— reporta **0
vulnerabilidades**. Conviene tener presente esta distinción durante la
sustentación.

---

## 8. Cómo reproducir la verificación

```bash
# Backend: suite completa
cd backend && .venv312/bin/python -m pytest -q          # 340 en verde

# Backend: dependencias
.venv312/bin/pip-audit                                   # sin vulnerabilidades

# Frontend
cd frontend
npm run test:run                                         # 60 en verde
npm run typecheck                                        # sin errores
npm run lint                                             # sin incidencias
npm run build                                            # compila
npm audit --omit=dev                                     # 0 vulnerabilidades
```
