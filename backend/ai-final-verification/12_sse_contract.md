# 12 — Contrato SSE backend vs frontend (comparación campo por campo, verificado en código real)

## Backend — `LecturaResponse` (`schemas.py`, mapeado por `mappers.py::lectura_to_response`)

Campos emitidos en el evento SSE (mismo schema que la respuesta HTTP `POST /api/lecturas`): `id`, `device_id`, `timestamp`, `temperatura_ambiental`, `humedad_ambiental`, `temperatura_interna`, `apertura_refrigerador`, `estado_conectividad`, `nivel_riesgo`, **`confianza_ia`**, **`modelo_version`** (los 2 últimos añadidos en la serie de sesiones de IA — corrige hallazgo AI-07 previo).

## Frontend — `LecturaTermica` (interfaz TypeScript, `frontend/src/domain/entities/LecturaTermica.ts`)

```typescript
export interface LecturaTermica {
  id: string | null
  device_id: string
  timestamp: string
  temperatura_ambiental: number | null
  humedad_ambiental: number | null
  temperatura_interna: number | null
  apertura_refrigerador: boolean
  estado_conectividad: string
  nivel_riesgo: NivelRiesgo | null
}
```

**No declara `confianza_ia` ni `modelo_version` ni `origen_clasificacion`.**

## Consumo real en runtime (`sseClient.ts`, líneas 64-69)

```typescript
source.onmessage = (event) => {
  try {
    onLectura(JSON.parse(event.data) as LecturaTermica)
  } catch { /* ... */ }
}
```

`JSON.parse` + `as LecturaTermica` es un **cast de TypeScript, no una validación en runtime** (no hay `zod`/`io-ts` ni verificación de forma). Esto significa:

- Los campos `confianza_ia`/`modelo_version`/`origen_clasificacion` **sí llegan físicamente** en el JSON del evento SSE (el backend los serializa, JS no descarta propiedades desconocidas), pero **son invisibles para el compilador TypeScript** y **ningún componente de la UI los lee o los muestra** (confirmado por búsqueda: 0 referencias a `confianza_ia`/`modelo_version` fuera del propio backend y de `demoAdapter.ts`/`datosDemo.ts`, que son la capa de datos simulados para demo, no consumo real del contrato backend).

## Tabla de paridad

| Campo | Backend emite | Frontend tipa | Frontend consume/muestra |
|---|---|---|---|
| `id` | Sí | Sí | Sí |
| `device_id` | Sí | Sí | Sí |
| `timestamp` | Sí | Sí | Sí |
| `temperatura_ambiental` | Sí | Sí | Sí |
| `humedad_ambiental` | Sí | Sí | Sí |
| `temperatura_interna` | Sí | Sí | Sí |
| `apertura_refrigerador` | Sí | Sí | Sí |
| `estado_conectividad` | Sí | Sí | Sí |
| `nivel_riesgo` | Sí | Sí | Sí |
| `confianza_ia` | **Sí** | **No** | **No** |
| `modelo_version` | **Sí** | **No** | **No** |
| `origen_clasificacion` | **No** (verificado por grep directo: `schemas.py` y `mappers.py` solo exponen `confianza_ia`/`modelo_version`, `origen_clasificacion` NO aparece en ninguno de los dos) | **No** | **No** |

**Confirmado por grep directo** (`grep -n "origen_clasificacion\|confianza_ia\|modelo_version" schemas.py mappers.py`): `origen_clasificacion` existe en la entidad de dominio y en el ORM/BD, pero **no llega ni al schema Pydantic ni al payload SSE/HTTP** — es un dato persistido y auditable solo vía acceso directo a la base de datos, nunca vía API pública. Esto significa que un consumidor externo de la API que reciba `confianza_ia=0.0` **no puede distinguir** "el modelo decidió con confianza 0" de "no hubo inferencia por sensor caído" sin consultar la base de datos directamente — agrava la ambigüedad ya señalada en `10_persistence_migrations.md`. Hallazgo real, no hipotético.

## Conclusión

El pipeline de **evidencia IA llega hasta el borde del backend y hasta el payload de red**, pero **no hay ninguna interfaz de usuario que la muestre al farmacéutico o técnico en tiempo real** — la trazabilidad de qué generó cada clasificación (`modelo` vs `regla_salvaguarda` vs `sin_dato_sensor`) es auditable vía API/base de datos pero no visible en el frontend actual. Esto es consistente con el veredicto ya fijado de fases previas ("Parcialmente implementado") — el backend expone más de lo que el frontend consume.
