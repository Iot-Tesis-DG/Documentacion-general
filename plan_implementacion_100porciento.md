# Plan de Implementación al 100% — Frontend + Backend

> IoT (ESP32, sensores, firmware C++/Arduino) lo implementa el equipo directamente. Este documento cubre TODO lo pendiente en Frontend y Backend para completar el alcance oficial declarado en TI, Product Backlog (Anexo5) y los 18 RF / 10 RNF.

---

# BACKEND — Tareas Pendientes

---

## B1. Checklist BPA completo (HU-37) — [ALTO]

**Estado actual**: Cero código relacionado. Ni entidad, ni tabla, ni endpoint, ni caso de uso.

### B1.1. Entidad de dominio

**Archivo nuevo**: `src/domain/entities/checklist_bpa.py`

```python
from dataclasses import dataclass, field
from datetime import datetime
from uuid import UUID, uuid4

@dataclass
class ChecklistBPA:
    id: UUID = field(default_factory=uuid4)
    usuario_id: UUID
    fecha: str  # ISO date "YYYY-MM-DD"
    temperatura_refrigerador_ok: bool
    limpieza_refrigerador_ok: bool
    medicamentos_caducidad_ok: bool
    medicamentos_almacenamiento_ok: bool
    termometro_calibrado: bool
    registro_manual_ok: bool
    observaciones: str | None = None
    created_at: datetime = field(default_factory=datetime.utcnow)
    updated_at: datetime = field(default_factory=datetime.utcnow)

    def validar(self) -> None:
        """Todos los campos booleanos deben estar definidos (no None)."""
        campos = [
            self.temperatura_refrigerador_ok,
            self.limpieza_refrigerador_ok,
            self.medicamentos_caducidad_ok,
            self.medicamentos_almacenamiento_ok,
            self.termometro_calibrado,
            self.registro_manual_ok,
        ]
        if any(c is None for c in campos):
            raise ValueError("Todos los items del checklist deben estar completos")
```

### B1.2. Interfaz de repositorio

**Archivo nuevo**: `src/domain/repositories/ichecklist_repository.py`

```python
from abc import ABC, abstractmethod
from uuid import UUID
from src.domain.entities.checklist_bpa import ChecklistBPA

class IChecklistRepository(ABC):
    @abstractmethod
    async def agregar(self, checklist: ChecklistBPA) -> ChecklistBPA: ...

    @abstractmethod
    async def obtener_ultimo_por_usuario(self, usuario_id: UUID) -> ChecklistBPA | None: ...

    @abstractmethod
    async def listar_por_usuario(self, usuario_id: UUID, limit: int = 50, offset: int = 0) -> list[ChecklistBPA]: ...

    @abstractmethod
    async def listar_por_rango_fechas(self, desde: str, hasta: str) -> list[ChecklistBPA]: ...
```

### B1.3. Modelo SQLAlchemy

**Archivo**: Agregar a `src/infrastructure/database/models.py`

```python
class ChecklistBPAModel(Base):
    __tablename__ = "checklist_bpa"

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    usuario_id: Mapped[str] = mapped_column(String(36), ForeignKey("users.id"), nullable=False, index=True)
    fecha: Mapped[str] = mapped_column(String(10), nullable=False)  # "YYYY-MM-DD"
    temperatura_refrigerador_ok: Mapped[bool] = mapped_column(nullable=False)
    limpieza_refrigerador_ok: Mapped[bool] = mapped_column(nullable=False)
    medicamentos_caducidad_ok: Mapped[bool] = mapped_column(nullable=False)
    medicamentos_almacenamiento_ok: Mapped[bool] = mapped_column(nullable=False)
    termometro_calibrado: Mapped[bool] = mapped_column(nullable=False)
    registro_manual_ok: Mapped[bool] = mapped_column(nullable=False)
    observaciones: Mapped[str | None] = mapped_column(Text, nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    __table_args__ = (
        UniqueConstraint("usuario_id", "fecha", name="uq_checklist_usuario_fecha"),
    )
```

### B1.4. Implementación de repositorio

**Archivo nuevo**: `src/infrastructure/database/repositories/checklist_repository.py`

- Método `agregar`: Si ya existe `(usuario_id, fecha)`, hacer upsert (UPDATE). Si no, INSERT.
- Método `obtener_ultimo_por_usuario`: `SELECT ... WHERE usuario_id = $1 ORDER BY created_at DESC LIMIT 1`
- Método `listar_por_usuario`: con paginación
- Método `listar_por_rango_fechas`: con filtro `fecha BETWEEN $1 AND $2`

### B1.5. Migración Alembic

```bash
cd backend
alembic revision -m "add_checklist_bpa_table"
# Editar el archivo generado en alembic/versions/
alembic upgrade head
```

**Contenido de la migración**:

```python
def upgrade():
    op.create_table(
        "checklist_bpa",
        sa.Column("id", sa.String(36), primary_key=True),
        sa.Column("usuario_id", sa.String(36), sa.ForeignKey("users.id"), nullable=False),
        sa.Column("fecha", sa.String(10), nullable=False),
        sa.Column("temperatura_refrigerador_ok", sa.Boolean(), nullable=False),
        sa.Column("limpieza_refrigerador_ok", sa.Boolean(), nullable=False),
        sa.Column("medicamentos_caducidad_ok", sa.Boolean(), nullable=False),
        sa.Column("medicamentos_almacenamiento_ok", sa.Boolean(), nullable=False),
        sa.Column("termometro_calibrado", sa.Boolean(), nullable=False),
        sa.Column("registro_manual_ok", sa.Boolean(), nullable=False),
        sa.Column("observaciones", sa.Text(), nullable=True),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.Column("updated_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.UniqueConstraint("usuario_id", "fecha", name="uq_checklist_usuario_fecha"),
    )
```

### B1.6. Caso de uso

**Archivo nuevo**: `src/application/use_cases/gestionar_checklist_bpa.py`

**Flujo de `RegistrarChecklistBPAUseCase.execute()`**:

1. Recibir `usuario_id` (del JWT), `fecha`, y los 6 campos booleanos + `observaciones`
2. Validar que ningún campo requerido sea `None` → `DomainError("Todos los ítems deben estar completos")`
3. Crear entidad `ChecklistBPA`
4. Llamar a `checklist_repo.agregar(checklist)` con upsert
5. Registrar en `traceability_records`:
   - `tipo_evento = "CHECKLIST_BPA"`
   - `payload = {usuario_id, fecha, items: {...}}`
   - Hash encadenado con `RegistrarHashEncadenadoUseCase`
6. Registrar en `audit_logs`: acción `"CHECKLIST_BPA_REGISTRADO"`
7. Retornar el checklist persistido

### B1.7. Router + Schemas

**Archivo nuevo**: `src/interface/api/checklist_router.py`

**Schemas** (en `schemas.py`):

```python
class ChecklistBPARequest(BaseModel):
    fecha: str  # "YYYY-MM-DD"
    temperatura_refrigerador_ok: bool
    limpieza_refrigerador_ok: bool
    medicamentos_caducidad_ok: bool
    medicamentos_almacenamiento_ok: bool
    termometro_calibrado: bool
    registro_manual_ok: bool
    observaciones: str | None = None

    @field_validator("fecha")
    @classmethod
    def validar_formato_fecha(cls, v: str) -> str:
        try:
            datetime.strptime(v, "%Y-%m-%d")
        except ValueError:
            raise ValueError("fecha debe tener formato YYYY-MM-DD")
        return v

class ChecklistBPAResponse(BaseModel):
    id: str
    usuario_id: str
    fecha: str
    temperatura_refrigerador_ok: bool
    limpieza_refrigerador_ok: bool
    medicamentos_caducidad_ok: bool
    medicamentos_almacenamiento_ok: bool
    termometro_calibrado: bool
    registro_manual_ok: bool
    observaciones: str | None
    created_at: datetime
    updated_at: datetime
```

**Endpoints**:

| Método | Ruta | Roles | Descripción |
|--------|------|-------|-------------|
| POST | `/api/checklist-bpa` | farmaceutico, admin | Guardar/actualizar checklist del día |
| GET | `/api/checklist-bpa` | farmaceutico, admin | Obtener último checklist del usuario autenticado |
| GET | `/api/checklist-bpa/historial` | farmaceutico, admin | Listar historial de checklists del usuario |

### B1.8. Registrar router en main.py

Agregar en `src/interface/main.py`:
```python
from src.interface.api.checklist_router import router as checklist_router
app.include_router(checklist_router, prefix="/api")
```

---

## B2. Exportación PDF de reportes (HU-38 / RF-13) — [ALTO]

**Estado actual**: `GET /api/reportes/bpa` devuelve JSON. El campo `archivo_url` en tabla `report_exports` existe pero nunca se puebla. `requirements.txt` no tiene librería PDF.

### B2.1. Dependencia

Agregar a `requirements.txt`:
```
weasyprint>=60.0
```

Instalar: `pip install weasyprint`

### B2.2. Template HTML del reporte

**Archivo nuevo**: `src/infrastructure/pdf/templates/reporte_bpa.html`

Template Jinja2 con:
- Encabezado: "Reporte BPA — Monitoreo de Cadena de Frío", nombre farmacia, rango de fechas
- Sección 1: Resumen estadístico (total lecturas, alertas por nivel, % tiempo en rango 2-8°C)
- Sección 2: Tabla de lecturas térmicas paginada (timestamp, temp_interna, temp_ambiental, humedad, nivel_riesgo)
- Sección 3: Tabla de alertas generadas en el período
- Sección 4: Cadena de trazabilidad (últimos N hashes con previous_hash/hash_actual truncados)
- Sección 5: Estado de integridad de cadena (resultado de verificación)
- Pie de página en cada hoja: "ThermoTrace — Trazabilidad digital verificable (SHA-256) — Documento generado {fecha_generacion}"

### B2.3. Servicio de generación PDF

**Archivo nuevo**: `src/infrastructure/pdf/generador_pdf.py`

```python
import io
from weasyprint import HTML
from jinja2 import Environment, FileSystemLoader

class GeneradorPDF:
    def __init__(self, template_dir: str = "src/infrastructure/pdf/templates"):
        self.env = Environment(loader=FileSystemLoader(template_dir))

    async def generar_reporte_bpa(
        self,
        lecturas: list[dict],
        alertas: list[dict],
        trazabilidad: list[dict],
        estadisticas: dict,
        veredicto_integridad: dict,
        fecha_desde: str,
        fecha_hasta: str,
        usuario: str,
    ) -> bytes:
        template = self.env.get_template("reporte_bpa.html")
        html_content = template.render(
            lecturas=lecturas,
            alertas=alertas,
            trazabilidad=trazabilidad,
            estadisticas=estadisticas,
            veredicto_integridad=veredicto_integridad,
            fecha_desde=fecha_desde,
            fecha_hasta=fecha_hasta,
            usuario=usuario,
        )
        pdf_bytes = HTML(string=html_content).write_pdf()
        return pdf_bytes
```

### B2.4. Caso de uso

**Archivo nuevo**: `src/application/use_cases/exportar_reporte_bpa_pdf.py`

**Flujo**:

1. Validar fechas (`desde <= hasta`, formato ISO)
2. Obtener lecturas del rango (`lectura_repo.listar(desde, hasta)`)
3. Obtener alertas del rango (`alerta_repo.listar(desde, hasta)`)
4. Obtener registros de trazabilidad del rango
5. Calcular estadísticas agregadas:
   - Total lecturas
   - Count por nivel_riesgo (normal / riesgo_preventivo / excursion_critica)
   - % tiempo en rango 2-8°C vs fuera
   - Temp mínima, máxima, promedio
6. Ejecutar verificación de integridad de cadena
7. Llamar a `GeneradorPDF.generar_reporte_bpa(...)` con todos los datos
8. Guardar PDF en sistema de archivos (ej. `exported_pdfs/reporte_bpa_{uuid}.pdf`)
9. Insertar registro en `report_exports` con `archivo_url` poblado
10. Registrar en `traceability_records`: `tipo_evento = "REPORTE_BPA_EXPORTADO"`
11. Retornar URL del PDF generado (o streaming response directa)

### B2.5. Endpoint nuevo

En `src/interface/api/reportes_router.py`:

```python
@router.get("/reportes/bpa/pdf")
async def exportar_reporte_bpa_pdf(
    desde: str = Query(...),
    hasta: str = Query(...),
    current_user: Usuario = Depends(require_roles("farmaceutico", "administrador")),
):
    """Genera y descarga reporte BPA en formato PDF con trazabilidad verificable."""
    use_case = ExportarReporteBPAPDFUseCase(...)
    pdf_bytes = await use_case.execute(desde, hasta, current_user)
    return StreamingResponse(
        io.BytesIO(pdf_bytes),
        media_type="application/pdf",
        headers={"Content-Disposition": f'attachment; filename="reporte_bpa_{desde}_{hasta}.pdf"'}
    )
```

### B2.6. Poblar `archivo_url` en endpoint existente

En el `GET /api/reportes/bpa` existente, el campo `archivo_url` del modelo `ReportExportModel` debe reflejar la URL real. Agregar endpoints:

- `GET /api/reportes/bpa/pdf/{export_id}` → descargar un PDF generado previamente
- Asegurar que `archivo_url` se pueble al generar

---

## B3. Emitir SSE desde ingesta HTTP (B-06) — [MEDIO]

**Problema**: `POST /api/lecturas` (ingesta por HTTP) persiste la lectura pero NO emite evento SSE. Solo el camino MQTT lo hace. Dashboard no se actualiza si la lectura entra por HTTP.

### Solución

**Archivo**: `src/interface/api/lecturas_router.py`

Modificar `ingestar_lectura()`:

```python
@router.post("/lecturas")
async def ingestar_lectura(
    payload: LecturaPayload,
    current_user: Usuario = Depends(require_roles("tecnico", "farmaceutico")),
):
    # ... validación, autorización del dispositivo, pipeline de ingesta ...

    registro = await registrar_lectura_use_case.execute(payload, current_user)

    # NUEVO: emitir SSE para que el dashboard se actualice en tiempo real
    from src.interface.api.sse_broadcaster import broadcaster

    await broadcaster.publicar(
        LecturaResponse.from_entity(registro).model_dump(mode="json")
    )

    return registro
```

### Alternativa más limpia

Mover la emisión SSE al propio `RegistrarLecturaTermicaUseCase.execute()` como último paso (después del commit). Así ambos caminos (MQTT y HTTP) emiten automáticamente sin duplicar código:

```python
# En registrar_lectura_termica.py
async def execute(self, ...):
    # ... pipeline completo ...
    await self.uow.commit()

    # Emitir SSE para dashboard
    await self.broadcaster.publicar(lectura_response)

    return lectura
```

---

## B4. Manejador dedicado para topic MQTT `farmacias/+/eventos` (B-09) — [MEDIO]

**Problema**: Backend se suscribe a `farmacias/+/eventos` pero todo mensaje que no sea `LecturaPayload` falla validación y se descarta silenciosamente. Los eventos LWT del ESP32 nunca se procesan.

### B4.1. Schema Pydantic para eventos

**Archivo**: Agregar a `src/infrastructure/mqtt/payload_schema.py`

```python
from enum import StrEnum

class TipoEventoDispositivo(StrEnum):
    LWT_ONLINE = "lwt_online"
    LWT_OFFLINE = "lwt_offline"
    ERROR_SENSOR = "error_sensor"
    FIRMWARE_UPDATE = "firmware_update"

class EventoDispositivoPayload(BaseModel):
    device_id: str
    tipo_evento: TipoEventoDispositivo
    timestamp: datetime
    detalle: str | None = None  # ej. "sensor DS18B20 no responde en bus 1-Wire"
    firmware_version: str | None = None
    modelo_config: ConfigDict = {"extra": "forbid"}
```

### B4.2. Manejador de eventos

**Archivo**: Modificar `src/interface/main.py::_procesar_mensaje_mqtt()`

```python
async def _procesar_mensaje_mqtt(message: Message):
    topic = message.topic.value

    if "/lecturas" in topic:
        return await _procesar_lectura_mqtt(message)

    if "/eventos" in topic:
        return await _procesar_evento_mqtt(message)


async def _procesar_evento_mqtt(message: Message):
    try:
        payload = json.loads(message.payload)
        evento = EventoDispositivoPayload.model_validate(payload)
    except (json.JSONDecodeError, ValidationError):
        logger.warning(f"Evento MQTT inválido en topic {message.topic}")
        return

    match evento.tipo_evento:
        case TipoEventoDispositivo.LWT_OFFLINE:
            await device_repo.actualizar_estado_conectividad(evento.device_id, False)
            await audit_log_repo.registrar("DISPOSITIVO_OFFLINE", evento.device_id)
            await broadcaster.publicar({
                "tipo": "desconexion",
                "device_id": evento.device_id,
                "timestamp": evento.timestamp.isoformat(),
            })

        case TipoEventoDispositivo.LWT_ONLINE:
            await device_repo.actualizar_estado_conectividad(evento.device_id, True)
            await audit_log_repo.registrar("DISPOSITIVO_ONLINE", evento.device_id)
            await broadcaster.publicar({
                "tipo": "reconexion",
                "device_id": evento.device_id,
                "timestamp": evento.timestamp.isoformat(),
            })

        case TipoEventoDispositivo.ERROR_SENSOR:
            await audit_log_repo.registrar(
                "ERROR_SENSOR",
                evento.device_id,
                detalle={"sensor": evento.detalle},
            )
            await broadcaster.publicar({
                "tipo": "fallo_sensor",
                "device_id": evento.device_id,
                "detalle": evento.detalle,
            })

        case TipoEventoDispositivo.FIRMWARE_UPDATE:
            await device_repo.actualizar_firmware_version(
                evento.device_id, evento.firmware_version
            )
            await audit_log_repo.registrar(
                "FIRMWARE_ACTUALIZADO",
                evento.device_id,
                detalle={"version": evento.firmware_version},
            )
```

---

## B5. Validación de timestamp futuro/antiguo en ingesta (B-10) — [MEDIO]

**Archivo**: `src/domain/entities/lectura_termica.py`

Agregar validación en `es_lectura_valida()`:

```python
from datetime import datetime, timedelta, timezone

MAX_FUTURO = timedelta(minutes=10)
MAX_PASADO = timedelta(hours=2)

@staticmethod
def es_timestamp_valido(timestamp: datetime) -> tuple[bool, str]:
    ahora = datetime.now(timezone.utc)
    if timestamp.tzinfo is None:
        timestamp = timestamp.replace(tzinfo=timezone.utc)

    if timestamp > ahora + MAX_FUTURO:
        return False, "timestamp_futuro"
    if timestamp < ahora - MAX_PASADO:
        return False, "timestamp_demasiado_antiguo"
    return True, "ok"
```

**Archivo**: `src/application/use_cases/registrar_lectura_termica.py`

Agregar en el paso de validación (paso 2 del pipeline):

```python
valido, motivo = LecturaTermica.es_timestamp_valido(payload.timestamp)
if not valido:
    await self.audit_log_repo.registrar(
        "LECTURA_RECHAZADA_TIMESTAMP",
        payload.device_id,
        detalle={
            "motivo": motivo,
            "timestamp_recibido": payload.timestamp.isoformat(),
            "servidor_utc": datetime.now(timezone.utc).isoformat(),
        }
    )
    raise LecturaInvalidaError(f"Timestamp rechazado: {motivo}")
```

---

## B6. Índices de base de datos (B-11) — [MEDIO]

**Migración Alembic nueva** (`alembic revision -m "add_performance_indexes"`):

```python
def upgrade():
    op.create_index(
        "idx_thermal_readings_device_ts",
        "thermal_readings",
        ["device_id", sa.text("timestamp DESC")],
    )
    op.create_index(
        "idx_traceability_records_created",
        "traceability_records",
        [sa.text("created_at DESC")],
    )
    op.create_index(
        "idx_audit_logs_created",
        "audit_logs",
        [sa.text("created_at DESC")],
    )
    op.create_index(
        "idx_thermal_alerts_device_created",
        "thermal_alerts",
        ["device_id", sa.text("created_at DESC")],
    )

def downgrade():
    op.drop_index("idx_thermal_readings_device_ts")
    op.drop_index("idx_traceability_records_created")
    op.drop_index("idx_audit_logs_created")
    op.drop_index("idx_thermal_alerts_device_created")
```

---

## B7. Validar `password_min_length` en schema (B-12) — [BAJO]

**Archivo**: `src/interface/api/schemas.py`

Verificar que `CrearUsuarioRequest` tenga:

```python
class CrearUsuarioRequest(BaseModel):
    nombre: str = Field(..., min_length=1, max_length=100)
    email: EmailStr
    password: str = Field(..., min_length=settings.PASSWORD_MIN_LENGTH)
    rol: Rol

    @field_validator("password")
    @classmethod
    def validar_complejidad(cls, v: str) -> str:
        if not any(c.isalpha() for c in v):
            raise ValueError("La contraseña debe contener al menos una letra")
        if not any(c.isdigit() for c in v):
            raise ValueError("La contraseña debe contener al menos un número")
        return v
```

Si ya existe pero sin `min_length`, agregarlo. Si no se usa `settings.PASSWORD_MIN_LENGTH`, crear la constante en `config.py` con valor por defecto `10`.

**Test nuevo** (`tests/unit/test_password_validation.py`):

```python
async def test_contrasena_corta_rechazada(client):
    response = await client.post("/api/usuarios", json={
        "nombre": "Test", "email": "test@test.com",
        "password": "Ab1", "rol": "tecnico",
    })
    assert response.status_code == 422
```

---

## B8. Tests adicionales — [MEDIO]

### B8.1. Test de integración MQTT

**Archivo nuevo**: `tests/integration/test_mqtt_ingestion.py`

- Simular mensaje MQTT válido → verificar que se persiste en `thermal_readings`
- Simular mensaje MQTT con device_id que no coincide con topic → verificar rechazo (anti-spoofing)
- Simular mensaje MQTT duplicado (mismo device_id+timestamp) → verificar que no genera duplicado

### B8.2. Test de migraciones

**Archivo nuevo**: `tests/integration/test_migrations.py`

- `alembic upgrade head` contra SQLite en memoria → verificar que no hay error
- Verificar que las 10+ tablas existen (`thermal_readings`, `thermal_alerts`, `traceability_records`, `audit_logs`, `users`, `devices`, `checklist_bpa`, etc.)

### B8.3. Test de revocación JWT

**Archivo nuevo**: `tests/integration/test_jwt_revocation.py`

- Login → obtener token → llamar logout → mismo token rechazado en siguiente request
- Ticket SSE consumido una vez → rechazado en segundo uso
- Ticket SSE expirado (>60s) → rechazado

### B8.4. Test de cobertura RBAC

**Archivo nuevo**: `tests/integration/test_rbac_coverage.py`

- Para cada uno de los 33+ endpoints, verificar que requieren JWT (401 sin token)
- Para cada endpoint con restricción de rol, verificar que rol incorrecto recibe 403
- Verificar que ningún endpoint mutable escapó sin `require_roles`

---

## B9. Notificación email/Telegram ante excursión crítica (HU-23) — [ALTO]

**Estado actual**: Cero código en backend. NI servicio SMTP, ni integración Telegram, ni background task, ni throttling de notificaciones.

### B9.1. Servicio de notificación

**Archivo nuevo**: `src/infrastructure/notifications/notificacion_service.py`

```python
import smtplib
import asyncio
import logging
from email.mime.text import MIMEText
from datetime import datetime, timedelta, timezone

import httpx
from src.infrastructure.config import settings

logger = logging.getLogger(__name__)

class NotificacionService:
    """Envía notificaciones por email y/o Telegram. No bloquea el flujo principal."""

    def __init__(self):
        self._ultima_notificacion: dict[str, datetime] = {}
        self._cooldown = timedelta(minutes=15)

    async def notificar_excursion_critica(
        self, device_id: str, temperatura: float, timestamp: str
    ) -> None:
        if not self._debe_notificar(device_id):
            return
        self._ultima_notificacion[device_id] = datetime.now(timezone.utc)

        if settings.SMTP_ENABLED:
            await self._enviar_email(device_id, temperatura, timestamp)
        if settings.TELEGRAM_ENABLED:
            await self._enviar_telegram(device_id, temperatura, timestamp)

    def _debe_notificar(self, device_id: str) -> bool:
        ahora = datetime.now(timezone.utc)
        ultima = self._ultima_notificacion.get(device_id)
        if ultima and (ahora - ultima) < self._cooldown:
            logger.debug(f"Throttling notificación para {device_id}")
            return False
        return True

    async def _enviar_email(self, device_id: str, temp: float, ts: str) -> None:
        msg = MIMEText(
            f"ALERTA CRÍTICA — Cadena de Frío\n\n"
            f"Dispositivo: {device_id}\n"
            f"Temperatura registrada: {temp}°C\n"
            f"Timestamp: {ts}\n"
            f"Acción requerida: verificar refrigerador inmediatamente."
        )
        msg["Subject"] = f"[ThermoTrace] EXCURSIÓN CRÍTICA — {device_id}"
        msg["From"] = settings.SMTP_FROM
        msg["To"] = settings.SMTP_TO

        loop = asyncio.get_event_loop()
        try:
            await loop.run_in_executor(None, self._smtp_send, msg)
            logger.info(f"Email enviado para {device_id}")
        except Exception as e:
            logger.error(f"Fallo envío email: {e}")

    def _smtp_send(self, msg: MIMEText) -> None:
        with smtplib.SMTP_SSL(settings.SMTP_HOST, settings.SMTP_PORT) as server:
            server.login(settings.SMTP_USER, settings.SMTP_PASSWORD)
            server.send_message(msg)

    async def _enviar_telegram(self, device_id: str, temp: float, ts: str) -> None:
        mensaje = (
            f"🚨 *EXCURSIÓN CRÍTICA — Cadena de Frío*\n\n"
            f"Dispositivo: `{device_id}`\n"
            f"Temperatura: *{temp}°C*\n"
            f"Timestamp: {ts}\n"
            f"Verificar refrigerador inmediatamente."
        )
        try:
            async with httpx.AsyncClient() as client:
                await client.post(
                    f"https://api.telegram.org/bot{settings.TELEGRAM_BOT_TOKEN}/sendMessage",
                    json={
                        "chat_id": settings.TELEGRAM_CHAT_ID,
                        "text": mensaje,
                        "parse_mode": "Markdown",
                    },
                )
            logger.info(f"Telegram enviado para {device_id}")
        except Exception as e:
            logger.error(f"Fallo envío Telegram: {e}")
```

### B9.2. Variables de entorno

Agregar a `src/infrastructure/config.py`:

```python
# Notificaciones
SMTP_ENABLED: bool = False
SMTP_HOST: str = ""
SMTP_PORT: int = 465
SMTP_USER: str = ""
SMTP_PASSWORD: str = ""
SMTP_FROM: str = ""
SMTP_TO: str = ""

TELEGRAM_ENABLED: bool = False
TELEGRAM_BOT_TOKEN: str = ""
TELEGRAM_CHAT_ID: str = ""
```

### B9.3. Integrar en GenerarAlertaUseCase

**Modificar** `src/application/use_cases/generar_alerta.py`:

```python
class GenerarAlertaUseCase:
    def __init__(self, ..., notificacion_service: NotificacionService):
        ...
        self.notificacion_service = notificacion_service

    async def execute(self, lectura: LecturaTermica) -> AlertaTermica | None:
        alerta = await self._generar_o_actualizar(lectura)

        if alerta and alerta.nivel_riesgo == NivelRiesgo.EXCURSION_CRITICA:
            asyncio.create_task(
                self.notificacion_service.notificar_excursion_critica(
                    device_id=lectura.device_id,
                    temperatura=lectura.temperatura_interna or 0,
                    timestamp=lectura.timestamp.isoformat(),
                )
            )

        return alerta
```

---

## B10. Registro de calibración de sensores con trazabilidad (HU-30) — [ALTO]

**Estado actual**: Cero código en backend y frontend.

### B10.1. Migración — agregar columnas a devices

```bash
alembic revision -m "add_calibracion_to_devices"
```

```python
def upgrade():
    op.add_column("devices", sa.Column("fecha_ultima_calibracion", sa.Date(), nullable=True))
    op.add_column("devices", sa.Column("numero_certificado_calibracion", sa.String(100), nullable=True))
    op.add_column("devices", sa.Column("fecha_proxima_calibracion", sa.Date(), nullable=True))
    op.add_column("devices", sa.Column("observaciones_calibracion", sa.Text(), nullable=True))

def downgrade():
    op.drop_column("devices", "fecha_ultima_calibracion")
    op.drop_column("devices", "numero_certificado_calibracion")
    op.drop_column("devices", "fecha_proxima_calibracion")
    op.drop_column("devices", "observaciones_calibracion")
```

### B10.2. Actualizar modelo SQLAlchemy

Agregar a `DeviceModel` en `src/infrastructure/database/models.py`:

```python
fecha_ultima_calibracion: Mapped[date | None] = mapped_column(Date, nullable=True)
numero_certificado_calibracion: Mapped[str | None] = mapped_column(String(100), nullable=True)
fecha_proxima_calibracion: Mapped[date | None] = mapped_column(Date, nullable=True)
observaciones_calibracion: Mapped[str | None] = mapped_column(Text, nullable=True)
```

### B10.3. Endpoint de calibración

**Modificar** `src/interface/api/dispositivos_router.py`:

```python
class CalibracionRequest(BaseModel):
    fecha_calibracion: date
    numero_certificado: str = Field(..., min_length=1, max_length=100)
    observaciones: str | None = None

    @field_validator("fecha_calibracion")
    @classmethod
    def no_futura(cls, v: date) -> date:
        if v > date.today():
            raise ValueError("La fecha de calibración no puede ser futura")
        return v

@router.patch("/dispositivos/{device_id}/calibracion")
async def registrar_calibracion(
    device_id: str,
    body: CalibracionRequest,
    current_user: Usuario = Depends(require_roles("farmaceutico", "administrador")),
):
    """Registra calibración de sensores con trazabilidad SHA-256."""
    device = await device_repo.obtener(device_id)
    if not device:
        raise RecursoNoEncontradoError("Dispositivo no encontrado")

    fecha_proxima = body.fecha_calibracion.replace(year=body.fecha_calibracion.year + 1)
    device.fecha_ultima_calibracion = body.fecha_calibracion
    device.numero_certificado_calibracion = body.numero_certificado
    device.fecha_proxima_calibracion = fecha_proxima
    device.observaciones_calibracion = body.observaciones
    await device_repo.actualizar(device)

    # Trazabilidad
    await registrar_hash_encadenado.execute(
        tipo_evento="CALIBRACION_SENSORES",
        payload={
            "device_id": device_id,
            "fecha_calibracion": body.fecha_calibracion.isoformat(),
            "numero_certificado": body.numero_certificado,
            "fecha_proxima": fecha_proxima.isoformat(),
        },
        device_id=device_id,
        usuario_id=current_user.id,
    )

    await audit_log_repo.registrar("CALIBRACION_REGISTRADA", device_id, detalle={
        "certificado": body.numero_certificado,
        "usuario": current_user.id,
    })

    return DeviceResponse.from_entity(device)
```

### B10.4. Tarea de verificación de vencimientos

**Archivo nuevo**: `src/infrastructure/scheduler/verificador_calibraciones.py`

Ejecutar en el `lifespan` de FastAPI como tarea de fondo cada 24h:

```python
async def verificar_calibraciones_vencidas():
    """Tarea programada: verifica dispositivos con calibración vencida o próxima."""
    while True:
        try:
            hoy = date.today()
            # Vencidas
            vencidos = await device_repo.listar_calibracion_vencida(hoy)
            for d in vencidos:
                await alerta_repo.generar_alerta_operativa(
                    device_id=d.id,
                    tipo="calibracion_vencida",
                    mensaje=f"Calibración vencida desde {d.fecha_proxima_calibracion}",
                )

            # Próximas (30 días)
            limite = hoy + timedelta(days=30)
            proximos = await device_repo.listar_calibracion_proxima(hoy, limite)
            for d in proximos:
                await alerta_repo.generar_alerta_operativa(
                    device_id=d.id,
                    tipo="calibracion_proxima",
                    mensaje=f"Calibración vence el {d.fecha_proxima_calibracion}",
                )
        except Exception as e:
            logger.error(f"Error en verificación de calibraciones: {e}")

        await asyncio.sleep(86400)  # 24 horas
```

Agregar en `src/interface/main.py`:
```python
async def lifespan(app: FastAPI):
    # ... existing setup ...
    asyncio.create_task(verificar_calibraciones_vencidas())
    yield
    # ... shutdown ...
```

### B10.5. Nuevos métodos en repositorio de dispositivos

```python
async def listar_calibracion_vencida(self, hoy: date) -> list[Device]:
    stmt = select(DeviceModel).where(
        DeviceModel.activo == True,
        DeviceModel.fecha_proxima_calibracion.isnot(None),
        DeviceModel.fecha_proxima_calibracion <= hoy,
    )
    ...

async def listar_calibracion_proxima(self, desde: date, hasta: date) -> list[Device]:
    stmt = select(DeviceModel).where(
        DeviceModel.activo == True,
        DeviceModel.fecha_proxima_calibracion.isnot(None),
        DeviceModel.fecha_proxima_calibracion.between(desde, hasta),
    )
    ...
```

---

## B11. Escaneo de dependencias vulnerables — [BAJO]

```bash
cd backend
.venv312/bin/pip install pip-audit
.venv312/bin/pip-audit
```

Si hay vulnerabilidades, actualizar `requirements.txt`.

```bash
cd frontend
npm audit
npm audit fix
```

---

## B12. Verificar CSP contra frontend real — [BAJO]

Backend emite `Content-Security-Policy: default-src 'none'; ...` en `src/interface/api/security_headers.py`.

Verificar que no rompe:
- **SSE** (`EventSource`) → necesita `connect-src` a la URL del backend
- **Tailwind / shadcn/ui** → genera estilos inline, necesita `style-src 'unsafe-inline'` o nonce
- **ECharts** → renderiza SVG via DOM, sin problema

Si algo se rompe, ajustar en `security_headers.py`.

---

## B13. Rate limiting adicional (opcional) — [OPCIONAL]

Backend YA tiene: login (5 intentos/5min por IP) + global (240 req/min por IP).

Agregar en `src/interface/api/api_protection.py`:

```python
# Rate limiting específico para ingesta de lecturas (anti-flood ESP32)
@router.post("/lecturas")
@rate_limit(max_requests=120, window_seconds=60, by="ip")  # 2 lecturas/segundo máximo por IP
async def ingestar_lectura(...): ...

# Rate limiting para verificación de cadena (operación O(n) costosa)
@router.get("/trazabilidad/verificar")
@rate_limit(max_requests=5, window_seconds=60, by="user")  # 5 verificaciones/minuto por usuario
async def verificar_integridad(...): ...
```

---

# SEGURIDAD — Tareas Pendientes en Frontend

---

## FS1. Pantalla de aceptación de privacidad post-login (HU-44) — [MEDIO]

**Backend YA tiene**:
- `POST /api/auth/privacidad/aceptar` → marca `privacy_accepted=True`, registra en hash chain
- `POST /api/auth/privacidad/rechazar` → revoca token, registra en hash chain
- Middleware `get_current_user` verifica `privacy_accepted` → 403 si no aceptado
- Login response incluye flag `require_privacy_consent: bool`

**Frontend NO tiene**: pantalla de aceptación.

### Qué implementar

**Archivo nuevo**: `src/presentation/pages/PrivacidadPage.tsx`

```tsx
import { useAuthStore } from "@/application/stores/authStore";
import { apiClient } from "@/infrastructure/api/apiClient";
import { useNavigate } from "react-router-dom";

export function PrivacidadPage() {
  const { usuario } = useAuthStore();
  const navigate = useNavigate();
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleAceptar = async () => {
    setLoading(true);
    try {
      await apiClient.post("/api/auth/privacidad/aceptar");
      navigate("/dashboard", { replace: true });
    } catch {
      setError("Error al procesar la aceptación");
    } finally {
      setLoading(false);
    }
  };

  const handleRechazar = async () => {
    setLoading(true);
    try {
      await apiClient.post("/api/auth/privacidad/rechazar");
      useAuthStore.getState().logout();
      navigate("/login", { replace: true });
    } catch {
      setError("Error al procesar el rechazo");
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="flex flex-col items-center justify-center min-h-screen p-8 max-w-2xl mx-auto">
      <h1 className="text-2xl font-bold mb-6">
        Política de Privacidad — Ley N° 29733
      </h1>
      <div className="prose prose-sm mb-8">
        {/* Texto completo de la política de privacidad */}
        <p>ThermoTrace recopila datos de temperatura...</p>
      </div>
      {error && <p className="text-red-500 mb-4">{error}</p>}
      <div className="flex gap-4">
        <Button onClick={handleRechazar} variant="outline" disabled={loading}>
          Rechazar
        </Button>
        <Button onClick={handleAceptar} disabled={loading}>
          {loading ? "Procesando..." : "Aceptar y continuar"}
        </Button>
      </div>
    </div>
  );
}
```

### Modificar authStore para redirigir a privacidad

```typescript
interface LoginResponse {
    access_token: string;
    token_type: string;
    require_privacy_consent: boolean;
    usuario: Usuario;
}

// En login():
if (data.require_privacy_consent) {
    set({ token: data.access_token, usuario: data.usuario, isAuthenticated: true });
    navigate("/privacidad");
} else {
    set({ token: data.access_token, usuario: data.usuario, isAuthenticated: true });
    navigate("/dashboard");
}
```

### Agregar ruta

```tsx
<Route path="/privacidad" element={
  <ProtectedRoute roles={["tecnico", "farmaceutico", "administrador"]}>
    <PrivacidadPage />
  </ProtectedRoute>
} />
```

---

## FS2. Gestión de dispositivos: listar + dar de baja + calibración (HU-43 + HU-30) — [MEDIO]

**Backend YA tiene**:
- `GET /api/dispositivos` → listado con estado_conectividad, firmware_version, calibración
- `POST /api/dispositivos/{id}/baja` → validación motivo, vínculo de reemplazo, hash chain `BAJA_HARDWARE`
- `PATCH /api/dispositivos/{id}/calibracion` → (nuevo, implementado en B10.3)

**Frontend NO tiene**: UI de gestión de dispositivos.

### Qué implementar

**Archivo nuevo**: `src/presentation/pages/DispositivosPage.tsx`

Tabla con columnas:
| Dispositivo | Ubicación | Firmware | Estado | Última calibración | Vencimiento | Acciones |
|-------------|-----------|----------|--------|--------------------|-------------|----------|

- **Estado** → `<ConnectivityBadge estado={d.estado_conectividad} />`
- **Vencimiento** → badge verde (vigente), amarillo (≤30 días), rojo (vencido)
- **Acciones**:
  - Botón "Registrar calibración" → modal con campos fecha + certificado + observaciones
  - Botón "Dar de baja" → modal con motivo (falla_hardware/mantenimiento/reemplazo/fin_de_servicio) + device_id de reemplazo (opcional)

**Ruta protegida**: solo `farmaceutico` y `administrador`.

**Hook**: `useDispositivos.ts` consumiendo `GET /api/dispositivos`.

**Agregar enlace en sidebar**: "Dispositivos" (visible solo para farmacéutico y admin).

---

## FS3. Banner de estado de integridad de cadena en Trazabilidad — [BAJO]

**Backend YA tiene**: `GET /api/trazabilidad/estado` → `{ cadena_comprometida: bool }`.

### Qué implementar

**Modificar** `src/presentation/pages/TrazabilidadPage.tsx`:

Agregar banner al inicio de la tabla:

```tsx
const { data: estado } = useQuery({
  queryKey: ["trazabilidad-estado"],
  queryFn: () => apiClient.get("/api/trazabilidad/estado").then(r => r.data),
  refetchInterval: 30000,  // cada 30 segundos
});

{estado?.cadena_comprometida ? (
  <Alert variant="destructive">
    ALERTA: Cadena de trazabilidad comprometida. Se detectó corrupción en uno o más registros.
    {esAdmin && (
      <Button variant="outline" size="sm" className="ml-4">
        Aislar corrupción y restaurar cadena
      </Button>
    )}
  </Alert>
) : (
  <Alert>
    Cadena de trazabilidad íntegra — 0 registros comprometidos.
  </Alert>
)}
```

---

## FS4. Botón "Aislar corrupción" para admin (HU-47) — [BAJO]

**Backend YA tiene**: `POST /api/trazabilidad/corrupcion/{id}/aislar`:
- Marca registro corrupto + posteriores como afectados
- Inicia nueva cadena génesis
- Restaura flag `cadena_comprometida`

### Qué implementar

En `TrazabilidadPage`, cuando `cadena_comprometida === true` y `usuario.rol === "administrador"`:

1. Mostrar el registro corrupto resaltado en rojo en la tabla de hashes
2. Botón "Aislar registro y restaurar cadena" junto al banner de alerta
3. Modal de confirmación:

```tsx
<Dialog>
  <DialogTrigger asChild>
    <Button variant="destructive">Aislar corrupción y restaurar cadena</Button>
  </DialogTrigger>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>¿Restaurar cadena de trazabilidad?</DialogTitle>
    </DialogHeader>
    <p className="text-sm">
      Se marcará el registro #{registroCorruptoId} y todos los posteriores como
      afectados. Se iniciará una nueva cadena génesis.
      Esta acción es irreversible y quedará registrada en auditoría.
    </p>
    <DialogFooter>
      <Button variant="outline" onClick={() => setOpen(false)}>Cancelar</Button>
      <Button variant="destructive" onClick={handleAislar}>
        Confirmar aislamiento
      </Button>
    </DialogFooter>
  </DialogContent>
</Dialog>
```

Llamar `POST /api/trazabilidad/corrupcion/{id}/aislar`, refrescar tabla, invalidar query de estado.

---

## FS5. Indicador visual de token próximo a expirar — [OPCIONAL, suma UX]

En `authStore`, verificar `exp` del JWT decodificado. Si faltan <5 minutos, mostrar toast "Tu sesión expirará pronto". Si ya expiró, forzar logout.

---

---

## F1. Checklist BPA con persistencia real (HU-37) — [ALTO]

**Archivos actuales**: `src/presentation/pages/ChecklistBPAPage.tsx` — usa `localStorage`. Cero llamadas API.

### F1.1. Hook nuevo

**Archivo nuevo**: `src/application/hooks/useChecklistBPA.ts`

```typescript
import { useState, useCallback } from "react";
import { apiClient } from "@/infrastructure/api/apiClient";
import { ChecklistBPA, ChecklistBPARequest } from "@/domain/entities/ChecklistBPA";

export function useChecklistBPA() {
  const [checklist, setChecklist] = useState<ChecklistBPA | null>(null);
  const [loading, setLoading] = useState(false);
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const cargarUltimo = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const { data } = await apiClient.get<ChecklistBPA>("/api/checklist-bpa");
      setChecklist(data);
    } catch (err) {
      setError("No se pudo cargar el checklist");
    } finally {
      setLoading(false);
    }
  }, []);

  const guardar = useCallback(async (payload: ChecklistBPARequest) => {
    setSaving(true);
    setError(null);
    try {
      const { data } = await apiClient.post<ChecklistBPA>("/api/checklist-bpa", payload);
      setChecklist(data);
      return data;
    } catch (err) {
      setError("Error al guardar el checklist");
      throw err;
    } finally {
      setSaving(false);
    }
  }, []);

  return { checklist, loading, saving, error, cargarUltimo, guardar };
}
```

### F1.2. Entidad de dominio

**Archivo nuevo** (o modificar): `src/domain/entities/ChecklistBPA.ts`

```typescript
export interface ChecklistBPA {
  id: string;
  usuario_id: string;
  fecha: string;
  temperatura_refrigerador_ok: boolean;
  limpieza_refrigerador_ok: boolean;
  medicamentos_caducidad_ok: boolean;
  medicamentos_almacenamiento_ok: boolean;
  termometro_calibrado: boolean;
  registro_manual_ok: boolean;
  observaciones: string | null;
  created_at: string;
  updated_at: string;
}

export interface ChecklistBPARequest {
  fecha: string;
  temperatura_refrigerador_ok: boolean;
  limpieza_refrigerador_ok: boolean;
  medicamentos_caducidad_ok: boolean;
  medicamentos_almacenamiento_ok: boolean;
  termometro_calibrado: boolean;
  registro_manual_ok: boolean;
  observaciones?: string | null;
}

export const ITEMS_CHECKLIST_BPA = [
  { key: "temperatura_refrigerador_ok" as const, label: "Temperatura del refrigerador dentro del rango 2-8°C" },
  { key: "limpieza_refrigerador_ok" as const, label: "Limpieza y orden del refrigerador" },
  { key: "medicamentos_caducidad_ok" as const, label: "Medicamentos sin fecha de caducidad vencida" },
  { key: "medicamentos_almacenamiento_ok" as const, label: "Medicamentos correctamente almacenados" },
  { key: "termometro_calibrado" as const, label: "Termómetro calibrado y certificado vigente" },
  { key: "registro_manual_ok" as const, label: "Registro manual de temperatura al día" },
] as const;
```

### F1.3. Modificar página

**Archivo**: `src/presentation/pages/ChecklistBPAPage.tsx`

Reemplazar `localStorage` por el hook `useChecklistBPA`:

```tsx
export function ChecklistBPAPage() {
  const { checklist, loading, saving, error, cargarUltimo, guardar } = useChecklistBPA();
  const [items, setItems] = useState<Record<string, boolean | null>>({});
  const [observaciones, setObservaciones] = useState("");
  const [guardado, setGuardado] = useState(false);

  useEffect(() => {
    cargarUltimo();
  }, []);

  useEffect(() => {
    if (checklist) {
      setItems({
        temperatura_refrigerador_ok: checklist.temperatura_refrigerador_ok,
        limpieza_refrigerador_ok: checklist.limpieza_refrigerador_ok,
        medicamentos_caducidad_ok: checklist.medicamentos_caducidad_ok,
        medicamentos_almacenamiento_ok: checklist.medicamentos_almacenamiento_ok,
        termometro_calibrado: checklist.termometro_calibrado,
        registro_manual_ok: checklist.registro_manual_ok,
      });
      setObservaciones(checklist.observaciones ?? "");
    }
  }, [checklist]);

  const handleGuardar = async () => {
    const todosCompletos = Object.values(items).every((v) => v !== null);
    if (!todosCompletos) {
      alert("Todos los ítems deben ser marcados como Cumple o No Cumple");
      return;
    }
    await guardar({
      fecha: new Date().toISOString().slice(0, 10),
      ...items as Record<string, boolean>,
      observaciones: observaciones || null,
    });
    setGuardado(true);
    setTimeout(() => setGuardado(false), 3000);
  };

  // Render:
  // - Loading spinner si loading=true
  // - Items del checklist con radio buttons: Cumple / No Cumple
  // - Campo de observaciones (textarea)
  // - Botón "Guardar" con loading spinner si saving=true
  // - Mensaje "Guardado exitosamente" si guardado=true
  // - Mensaje de error si error != null
  // - Timestamp de última modificación y usuario (checklist.updated_at)
}
```

---

## F2. Exportación de PDF (HU-38 / RF-13) — [ALTO]

**Archivos actuales**: `src/application/hooks/useReportesBPA.ts` — solo CSV/JSON.

### F2.1. Agregar botón y hook

**Archivo**: Modificar `src/presentation/pages/ReportesBPAPage.tsx`

Agregar botón "Exportar PDF" junto a los existentes "Exportar CSV" y "Exportar JSON".

**Flujo**:

```tsx
const [exportandoPDF, setExportandoPDF] = useState(false);

const handleExportarPDF = async () => {
  setExportandoPDF(true);
  try {
    const response = await apiClient.get(
      `/api/reportes/bpa/pdf?desde=${fechaDesde}&hasta=${fechaHasta}`,
      { responseType: "blob" }
    );
    const url = window.URL.createObjectURL(new Blob([response.data], { type: "application/pdf" }));
    const link = document.createElement("a");
    link.href = url;
    link.download = `reporte_bpa_${fechaDesde}_${fechaHasta}.pdf`;
    link.click();
    window.URL.revokeObjectURL(url);
  } catch (err) {
    setError("Error al generar el PDF");
  } finally {
    setExportandoPDF(false);
  }
};
```

**UI del botón**:
- Texto: "Exportar PDF" con ícono de documento
- Loading spinner mientras se genera
- Deshabilitado si no hay fechas seleccionadas

### F2.2. Alternativa client-side (si backend no implementa B2)

Si el backend no genera PDF, usar `jspdf` + `html2canvas` en el frontend:

```bash
npm install jspdf html2canvas
```

Generar PDF renderizando el dashboard actual como imagen y embeber la tabla:

```tsx
import jsPDF from "jspdf";
import html2canvas from "html2canvas";

const exportarPDFClientSide = async () => {
  const pdf = new jsPDF("p", "mm", "a4");
  const dashboardEl = document.getElementById("dashboard-chart");

  if (dashboardEl) {
    const canvas = await html2canvas(dashboardEl);
    const imgData = canvas.toDataURL("image/png");
    pdf.addImage(imgData, "PNG", 10, 10, 190, 80);
  }

  // Agregar tabla de lecturas
  pdf.autoTable({
    head: [["Timestamp", "Temp. Interna", "Temp. Ambiente", "Humedad", "Riesgo"]],
    body: lecturas.map((l) => [
      l.timestamp, `${l.temp_interna}°C`, `${l.temp_ambiental}°C`,
      `${l.humedad}%`, l.nivel_riesgo,
    ]),
    startY: 100,
  });

  pdf.save(`reporte_bpa_${fechaDesde}_${fechaHasta}.pdf`);
};
```

**Recomendación**: Implementar server-side (backend B2) — es más robusto y permite incluir la cadena de trazabilidad con hashes verificables.

---

## F3. Indicador de conectividad de dispositivo (RF-18) — [MEDIO]

**Archivos nuevo/modificado**:

### F3.1. Componente ConnectivityBadge

**Archivo nuevo**: `src/presentation/components/ConnectivityBadge.tsx`

```tsx
import { Badge } from "@/presentation/components/ui/badge";

type Props = {
  estado: "en_linea" | "offline" | "desconocido";
  ultimaVez?: string;
};

export function ConnectivityBadge({ estado, ultimaVez }: Props) {
  const variants = {
    en_linea: { variant: "success" as const, label: "En línea" },
    offline: { variant: "destructive" as const, label: "Offline" },
    desconocido: { variant: "secondary" as const, label: "Desconocido" },
  };
  const { variant, label } = variants[estado];

  return (
    <Badge variant={variant} title={ultimaVez ? `Última conexión: ${ultimaVez}` : undefined}>
      {label}
    </Badge>
  );
}
```

### F3.2. Agregar a DashboardPage

En la sección de tarjetas KPI o junto al nombre/dispositivo, renderizar `<ConnectivityBadge estado={lectura.estado_conectividad} />`.

### F3.3. Agregar filtro en HistorialPage

El backend ya soporta `?estado_conectividad=online` como query param. Agregar un `<Select>` con opciones: Todos, En línea, Offline.

---

## F4. Vista de métricas del modelo IA (RNF-04) — [MEDIO]

**Backend ya expone**: `GET /api/ia/modelo` con metadata + métricas. Frontend nunca lo consume.

### F4.1. Hook

**Archivo nuevo**: `src/application/hooks/useModeloIA.ts`

```typescript
export interface MetricasModelo {
  accuracy: number;
  f1_weighted: number;
  f1_por_clase: {
    excursion_critica: number;
    riesgo_preventivo: number;
    normal: number;
  };
  precision_por_clase: Record<string, number>;
  recall_por_clase: Record<string, number>;
  cross_val_mean: number;
  cross_val_std: number;
  trained_at: string;
  model_version: string;
  n_samples: number;
}

export function useModeloIA() {
  const [metricas, setMetricas] = useState<MetricasModelo | null>(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    setLoading(true);
    apiClient.get("/api/ia/modelo")
      .then(({ data }) => setMetricas(data))
      .finally(() => setLoading(false));
  }, []);

  return { metricas, loading };
}
```

### F4.2. Página nueva

**Archivo nuevo**: `src/presentation/pages/MetricasIAPage.tsx`

Componente con:
- Título: "Métricas del Modelo de IA — Random Forest"
- Tarjetas superiores:
  - F1-Score Ponderado: `0.9659` (verde si ≥0.85)
  - Accuracy: `96.56%`
  - Validación Cruzada 5-Fold: `0.9615 ± 0.0031`
  - Fecha de entrenamiento: `2026-07-11`
- Tabla de métricas por clase:

| Clase | F1 | Precisión | Recall |
|-------|-----|-----------|--------|
| excursion_critica | 0.9815 | 0.99 | 0.97 |
| riesgo_preventivo | 0.9563 | 0.93 | 0.98 |
| normal | 0.9226 | 0.88 | 0.97 |

- Metadata del modelo: versión, número de muestras de entrenamiento, features utilizadas

### F4.3. Ruta y RBAC

En `src/App.tsx`, agregar ruta protegida:

```tsx
<Route path="/ia-metrics" element={
  <ProtectedRoute roles={["farmaceutico", "administrador"]}>
    <MetricasIAPage />
  </ProtectedRoute>
} />
```

Agregar enlace en sidebar/nav: "Métricas IA" (visible solo para farmacéutico y admin).

---

## F5. Conectar logout real (B-07) — [MEDIO]

**Archivo**: `src/application/stores/authStore.ts`

Modificar función `logout()`:

```typescript
logout: async () => {
  try {
    await apiClient.post("/api/auth/logout");
  } catch {
    // Si falla la red, limpiar estado local igual
    console.warn("No se pudo revocar el token en el servidor");
  } finally {
    set({ token: null, usuario: null, isAuthenticated: false });
  }
},
```

**Archivo**: `src/presentation/components/UserMenu.tsx` (o donde esté el botón de logout)

Asegurar que el botón llame a `authStore.logout()` (que ahora es async) y muestre feedback visual mientras se ejecuta.

---

## F6. ErrorBoundary global (F-09) — [BAJO]

**Archivo nuevo**: `src/presentation/components/ErrorBoundary.tsx`

```tsx
import { Component, type ReactNode } from "react";

interface Props {
  children: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error("ErrorBoundary:", error, info.componentStack);
    // Opcional: enviar a servicio de logging
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="flex flex-col items-center justify-center min-h-screen gap-4">
          <h1 className="text-2xl font-bold">Algo sali&oacute; mal</h1>
          <p className="text-muted-foreground">
            Ocurri&oacute; un error inesperado. Por favor, intent&aacute; de nuevo.
          </p>
          <pre className="text-sm text-red-500 max-w-lg overflow-auto">
            {this.state.error?.message}
          </pre>
          <div className="flex gap-2">
            <Button onClick={() => window.location.reload()}>Recargar p&aacute;gina</Button>
            <Button variant="outline" onClick={() => window.location.href = "/"}>
              Volver al inicio
            </Button>
          </div>
        </div>
      );
    }
    return this.props.children;
  }
}
```

**Archivo**: `src/App.tsx` — envolver todo el árbol:

```tsx
<ErrorBoundary>
  <RouterProvider router={router} />
</ErrorBoundary>
```

---

## F7. Code-splitting por ruta (F-11) — [BAJO]

**Archivo**: `src/App.tsx` o archivo de rutas

```tsx
import { lazy, Suspense } from "react";

const DashboardPage = lazy(() => import("@/presentation/pages/DashboardPage"));
const HistorialPage = lazy(() => import("@/presentation/pages/HistorialPage"));
const MetricasIAPage = lazy(() => import("@/presentation/pages/MetricasIAPage"));

function LazyPage({ children }: { children: ReactNode }) {
  return (
    <Suspense fallback={<div className="p-8"><Spinner /></div>}>
      {children}
    </Suspense>
  );
}

// En rutas:
<Route path="/dashboard" element={<LazyPage><DashboardPage /></LazyPage>} />
```

Esto saca ECharts (~568KB) del bundle inicial — solo se carga cuando el usuario navega al dashboard.

---

## F8. ESLint (F-07) — [BAJO]

```bash
npm install -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin \
  eslint-plugin-react-hooks eslint-plugin-react-refresh
```

**Archivo nuevo**: `eslint.config.js`

```javascript
import tseslint from "@typescript-eslint/eslint-plugin";
import tsparser from "@typescript-eslint/parser";
import reactHooks from "eslint-plugin-react-hooks";
import reactRefresh from "eslint-plugin-react-refresh";

export default [
  {
    ignores: ["dist/", "node_modules/"],
  },
  {
    files: ["src/**/*.{ts,tsx}"],
    languageOptions: {
      parser: tsparser,
      parserOptions: { ecmaVersion: "latest", sourceType: "module" },
    },
    plugins: {
      "@typescript-eslint": tseslint,
      "react-hooks": reactHooks,
      "react-refresh": reactRefresh,
    },
    rules: {
      "react-hooks/rules-of-hooks": "error",
      "react-hooks/exhaustive-deps": "warn",
      "react-refresh/only-export-components": ["warn", { allowConstantExport: true }],
      "@typescript-eslint/no-explicit-any": "error",
    },
  },
];
```

Agregar script en `package.json`:

```json
{
  "scripts": {
    "lint": "eslint src/ --ext .ts,.tsx",
    "lint:fix": "eslint src/ --ext .ts,.tsx --fix"
  }
}
```

---

## F9. Tests automatizados (F-06) — [BAJO]

```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom \
  @testing-library/user-event jsdom
```

**Archivo nuevo**: `vitest.config.ts`

```typescript
import { defineConfig } from "vitest/config";
import path from "path";

export default defineConfig({
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./src/tests/setup.ts"],
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, "./src") },
  },
});
```

**Archivo nuevo**: `src/tests/setup.ts`

```typescript
import "@testing-library/jest-dom";
```

**Tests mínimos**:

1. `src/tests/application/stores/authStore.test.ts` — login, logout, token expired
2. `src/tests/presentation/pages/LoginPage.test.tsx` — renderiza formulario, submit con credenciales
3. `src/tests/presentation/pages/ChecklistBPAPage.test.tsx` — renderiza items, validación todos completos
4. `src/tests/presentation/guards/ProtectedRoute.test.tsx` — sin token redirige a /login, con rol incorrecto muestra 403

Agregar script:

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run"
  }
}
```

---

# RESUMEN — Orden de implementación

## Backend

| # | Tarea | HU/RF | Prioridad | Archivos |
|---|-------|-------|-----------|----------|
| B1 | Checklist BPA completo | HU-37 | **ALTO** | 6 nuevos + 2 mod + 1 migración |
| B2 | Exportación PDF reportes | HU-38, RF-13 | **ALTO** | 3 nuevos + 1 mod + 1 dep (weasyprint) |
| B9 | Notificación email/Telegram | HU-23 | **ALTO** | 1 nuevo + 2 mod + 6 vars env |
| B10 | Registro calibración sensores | HU-30 | **ALTO** | 1 nuevo + 3 mod + 1 migración |
| B3 | SSE desde ingesta HTTP | — | MEDIO | 1 mod |
| B4 | Manejador topic MQTT eventos | — | MEDIO | 2 nuevos + 1 mod |
| B5 | Validación timestamp | — | MEDIO | 2 mod |
| B6 | Índices BD | — | MEDIO | 1 migración |
| B8 | Tests adicionales | — | MEDIO | 4 nuevos archivos test |
| B7 | password_min_length | — | BAJO | 1 mod + 1 test |
| B11 | pip-audit dependencias | — | BAJO | comando único |
| B12 | Verificar CSP vs frontend | — | BAJO | revisión + posible ajuste |
| B13 | Rate limiting extra | — | OPCIONAL | 1 mod |

## Frontend

| # | Tarea | HU/RF | Prioridad | Archivos |
|---|-------|-------|-----------|----------|
| F1 | Checklist BPA → backend real | HU-37 | **ALTO** | 2 nuevos + 1 mod |
| F2 | Botón exportar PDF | HU-38, RF-13 | **ALTO** | 1 mod |
| F3 | ConnectivityBadge + filtro | RF-18 | MEDIO | 1 nuevo + 2 mod |
| F4 | Vista métricas IA | RNF-04 | MEDIO | 2 nuevos + 1 mod |
| F5 | Logout real → llamar API | — | MEDIO | 1 mod |
| FS1 | Pantalla privacidad post-login | HU-44 | MEDIO | 1 nuevo + 1 mod + 1 ruta |
| FS2 | Gestión dispositivos (listar + baja + calibración) | HU-43, HU-30 | MEDIO | 2 nuevos + 1 mod + 1 ruta |
| FS3 | Banner integridad cadena | — | BAJO | 1 mod |
| FS4 | Botón aislar corrupción (admin) | HU-47 | BAJO | 1 mod |
| F6 | ErrorBoundary global | — | BAJO | 1 nuevo + 1 mod |
| F7 | Code-splitting por ruta | — | BAJO | 1 mod |
| F8 | ESLint | — | BAJO | 1 nuevo + 1 mod package.json |
| F9 | Tests frontend (Vitest) | — | BAJO | 1 nuevo config + 4 test files |
| FS5 | Indicador expiración token JWT | — | OPCIONAL | 1 mod |

---

**Nota para el desarrollador**: Las tareas B1, B2, B9 y B10 (Checklist BPA + PDF + Notificaciones + Calibración) son las más grandes y definen si el sistema llega al 100% del alcance documentado. Empezar por esas. Las tareas MEDIO/BAJO son correcciones, integraciones de endpoints ya existentes, y mejoras de calidad que suman solidez para la sustentación.
