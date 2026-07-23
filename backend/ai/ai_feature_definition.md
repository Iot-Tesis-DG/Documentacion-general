# Definición de features — fuente única confirmada

| Orden | Feature | Fórmula | Unidad | Ventana temporal | Origen |
|---:|---|---|---|---|---|
| 0 | `temperatura_ambiental` | Lectura directa SHT31 | °C | instantánea | Sensor |
| 1 | `humedad_ambiental` | Lectura directa SHT31 | %HR | instantánea | Sensor |
| 2 | `temperatura_interna` | Lectura directa DS18B20 | °C | instantánea | Sensor |
| 3 | `diferencia_sensores` | `temperatura_ambiental - temperatura_interna` | °C | instantánea | Calculada |
| 4 | `duracion_fuera_rango` | Minutos desde que la última racha de lecturas fuera de [2,8]°C comenzó (retrocede en el historial hasta encontrar una lectura dentro de rango) | minutos | hasta 20 lecturas recientes | Calculada del historial real |
| 5 | `frecuencia_desviaciones` | Conteo de lecturas del historial reciente fuera de [2,8]°C | conteo | hasta 20 lecturas recientes | Calculada del historial real |
| 6 | `tendencia_termica` | Pendiente de regresión lineal (`np.polyfit` grado 1) sobre las temperaturas internas del historial | °C/lectura | hasta 20 lecturas recientes | Calculada del historial real |
| 7 | `apertura_refrigerador` | Estado del reed switch MC-38 (booleano→float) | — | instantánea | Sensor |
| 8 | `hora_evento` | `lectura.timestamp.hour` | hora [0-23] | instantánea | Derivada del timestamp |
| 9 | `estado_conectividad_online` | `estado_conectividad == "online"` (booleano→float) | — | instantánea | Reportado por el dispositivo |

## Única función responsable — confirmado

`FeaturesRiesgoTermico.to_array()` (`features.py` líneas 32-44) es el único punto que serializa el vector en orden fijo. Tanto **entrenamiento** (`train_model.py::generar_dataset`, vía `FeaturesRiesgoTermico(...).to_array()` implícito en `filas.append(medidas.to_array())`, línea 97) como **inferencia en producción** (`ClasificarRiesgoTermicoUseCase._construir_features()`, que construye el mismo dataclass y lo pasa a `servicio.inferir()` → `features.to_array()` en `random_forest_service.py` línea 106) usan exactamente la misma clase y el mismo método de serialización. **No hay una implementación duplicada o divergente del orden de features entre entrenamiento e inferencia** — esto es correcto y evita un error clásico de "feature skew".

## Variables NO añadidas (correctamente, según lo que el dispositivo puede producir)

Media móvil, desviación estándar y velocidad de incremento explícitas no se implementan como features separadas — la tendencia (`tendencia_termica`, vía regresión lineal) ya captura una noción de velocidad de cambio, y no se justificó una feature adicional redundante. El "tiempo desde la última lectura" tampoco es una feature explícita (el sistema no maneja lecturas espaciadas irregularmente como un caso distinto) — limitación menor, no crítica.
