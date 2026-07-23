# Validación de las 13 matrices de benchmarking (Anexo2_IB_Soto_Diego_Gamio_Brenda.docx)

Metodología de verificación: se leyeron íntegramente las 3110 líneas del documento convertido, localizando para cada uno de los 13 componentes: tabla de propuestas, tabla de criterios/escalas 1-3-5, matriz de enfrentamiento (pesos), matriz de benchmarking (puntaje×peso) y matriz de riesgos. Se recalculó manualmente la aritmética de la matriz de benchmarking del componente 1 como muestra de control (ver abajo) y se extrajo programáticamente el resultado final (`Promedio Total`) de los 13 componentes para contrastarlos contra la Tabla 9 de TI.

## Verificación aritmética (muestra de control — Componente 1, Microcontrolador Edge)

Pesos: Precisión temporización 20%, Compatibilidad multisensor 30%, Conectividad segura 30%, Costo/disponibilidad 10%, Consumo energético 10% (suma 100%, matriz de enfrentamiento con subtotales 2/3/3/1/1 sobre 10 = correcto).

| Alternativa | Cálculo (puntaje×peso por criterio) | Suma declarada | Suma verificada |
|---|---|---|---|
| ESP32 DevKitC V4 | 5×.20+5×.30+5×.30+4×.10+4×.10 = 1.00+1.50+1.50+0.40+0.40 | 4.80 | ✅ 4.80 |
| ESP32 con IA embebida | 5×.20+5×.30+5×.30+4×.10+3×.10 = 1.00+1.50+1.50+0.40+0.30 | 4.70 | ✅ 4.70 |
| Arduino en monitoreo IoT | 3×.20+3×.30+3×.30+4×.10+3×.10 = 0.60+0.90+0.90+0.40+0.30 | 3.10 | ✅ 3.10 |
| Nodo IoT bajo consumo | 4×.20+4×.30+3×.30+4×.10+5×.10 = 0.80+1.20+0.90+0.40+0.50 | 3.80 | ✅ 3.80 |

Aritmética exacta, sin errores de redondeo. Se asume la misma metodología (puntaje×peso, pesos suman 100%) es aplicada consistentemente en los 12 componentes restantes — estructura de tabla idéntica confirmada en los 13.

## Tabla comparativa: Anexo2 (detalle) vs TI Tabla 9 (resumen)

| Componente | Ganador Anexo2 | Score Anexo2 | Runner-up | Score runner-up | Margen | Score TI Tabla 9 | Coherencia |
|---|---|---|---|---|---|---|---|
| Microcontrolador edge | ESP32 DevKitC V4 | 4.80 | ESP32 con IA embebida | 4.70 | **0.10 (2%)** | 4.80 | ✅ coincide, ⚠️ margen estrecho no declarado |
| Sensores térmicos/ambientales | SHT31+DS18B20+MC-38 | 5.00 | Sonda de refrigerador evaluada | 4.20 | 0.80 | 5.00 | ✅ coincide |
| Firmware y buffer offline | Arduino Core+LittleFS | 4.90 | Registro integrado cadena de frío | 4.10 | 0.80 | 4.90 | ✅ coincide |
| Comunicación IoT | MQTT sobre TLS | 4.70 | MQTT con monitoreo de transporte | 4.60 | **0.10 (2%)** | 4.70 | ✅ coincide, TI sí reconoce este caso como "el más bajo" pero no como "margen estrecho" |
| Broker MQTT | Broker gestionado tipo EMQX | 4.80 | Arquitectura IoT segura c/tácticas | 3.90 (nota: 3.9 vs 3.8 auth ligera, revisar orden) | 0.90 | 4.80 | ✅ coincide |
| Backend y arquitectura | FastAPI+DDD | 5.00 | Servidor plataforma multisensor | 3.70 | 1.30 | 5.00 | ✅ coincide |
| Base de datos | PostgreSQL+JSONB | 4.90 | Registro cloud seguimiento IoT | 3.90 | 1.00 | 4.90 | ✅ coincide |
| Modelo de IA | Random Forest | 4.80 | Clasificación ML fallas enfriamiento | 4.30 | 0.50 | 4.80 | ✅ coincide |
| Trazabilidad digital | SHA-256+previous_hash | 5.00 | Blockchain privada calidad medicamentos | 4.10 | 0.90 | 5.00 | ✅ coincide |
| Frontend y dashboard | React+Vite+TS+shadcn/ui | 5.00 | Interfaz web farmacéutica | 3.40 | 1.60 | 5.00 | ✅ coincide |
| Visualización y tiempo real | Apache ECharts+SSE | 4.90 | Monitoreo inalámbrico tiempo real | 3.90 | 1.00 | 4.90 | ✅ coincide |
| Despliegue cloud | Railway+Vercel+broker gestionado | 4.80 | Cloud IoT completo | 3.90 | 0.90 | 4.80 | ✅ coincide |
| Controles de seguridad | JWT/RBAC+MQTT/TLS+auditoría | 5.00 | (empate 4.10 x2) | 4.10 | 0.90 | 5.00 | ✅ coincide |

**Conclusión de coherencia**: los 13/13 ganadores y puntajes del Anexo2 coinciden exactamente con la Tabla 9 de TI. No hay discrepancia numérica entre el anexo detallado y el resumen ejecutivo citado en el cuerpo de la tesis.

## Inconsistencia detectada [MEDIO]

TI (sección Benchmarking, ~línea 1242) afirma: *"Ninguna selección quedó definida por un margen estrecho"*, y solo reconoce a "MQTT sobre TLS con 4.70" como el puntaje más bajo relativo. Sin embargo, el propio Anexo2 muestra que **dos** componentes tienen margen ganador-vs-segundo de apenas **0.10 sobre 5.00 (2%)**: Microcontrolador edge (ESP32 DevKitC 4.80 vs ESP32 con IA embebida 4.70) y Comunicación IoT (MQTT/TLS 4.70 vs MQTT con monitoreo de transporte 4.60). El primero de estos dos casos (microcontrolador) **no es mencionado en absoluto** en el texto de TI como caso de margen estrecho. Esto contradice la afirmación categórica "ninguna selección quedó definida por un margen estrecho" y es un punto que un jurado riguroso podría señalar como imprecisión en la redacción de conclusiones del benchmarking.

## Sesgos/circularidad de criterios (observación cualitativa)

Los criterios de "Fuente (cita APA 7)" en la Tabla 2 (y equivalentes en los demás componentes) citan como fuente de los propios umbrales cuantitativos (1/3/5) a los mismos autores cuyas propuestas compiten en el benchmarking (p. ej., componente 1 usa a Harrabi et al., Cabezas et al., Frontera-Bergas et al. tanto como fuente de los criterios COMO como una de las alternativas evaluadas). Esto no es necesariamente incorrecto metodológicamente (los criterios se derivan de la literatura relevante), pero introduce un riesgo de circularidad: los umbrales de puntaje pudieron haberse calibrado post-hoc para favorecer la alternativa elegida por el equipo. No se encontró evidencia de que los umbrales se hayan definido *antes* de conocer el desempeño de cada alternativa (no hay fecha ni versión previa registrada en el documento). Riesgo [BAJO-MEDIO] a mencionar si el jurado pregunta por objetividad del benchmarking.

## Tecnologías ganadoras vs stack definitivo

Confirmado: las 13 tecnologías ganadoras del Anexo2 son exactamente las que TI describe como stack definitivo del prototipo (ESP32 DevKitC V4, SHT31+DS18B20+MC-38, Arduino Core+LittleFS, MQTT/TLS, EMQX Cloud, FastAPI+DDD, PostgreSQL+JSONB, Random Forest, SHA-256+previous_hash, React+Vite+TS+shadcn/ui, ECharts+SSE, Railway+Vercel, JWT/RBAC+MQTT/TLS+auditoría). Sin discrepancias.

## Componentes evaluados con criterios no comparables entre sí

No se detectaron criterios evidentemente no comparables entre alternativas dentro de un mismo componente (todos los criterios de cada componente se aplican de forma homogénea a las 4 alternativas comparadas). Los cinco criterios varían de componente a componente (esperado y correcto, ya que cada componente technológico requiere criterios distintos).
