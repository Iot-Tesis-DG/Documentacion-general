# Riesgos para la sustentación

## Riesgo 1 — Demostración en vivo de la trazabilidad puede fallar (CRÍTICO)

Si el jurado pide simular múltiples dispositivos publicando simultáneamente (un escenario realista y fácil de pedir), existe una probabilidad real y no despreciable de presenciar un falso "Cadena Rota" en la pantalla de Trazabilidad, debido al hallazgo B-01 (condición de carrera). **Mitigación antes de sustentar**: corregir el `SELECT ... FOR UPDATE` o aislamiento serializable, y probar deliberadamente con concurrencia antes de la demo.

## Riesgo 2 — Pregunta directa sobre la cifra ">96%" y su cronología

Clasificación: **afirmación actualmente respaldable, pero históricamente no sustentada en la versión original de la presentación**. La presentación es de mayo de 2026; el entrenamiento/evaluación propio (`training_metrics.json`) es de julio de 2026 — la cifra de mayo no pudo estar sustentada en ese momento, sin importar que hoy exista un resultado real cercano. Si el jurado revisa fechas, puede preguntar cómo se obtuvo esa cifra si el entrenamiento real es posterior. **Respuesta preparada recomendada**: reconocer explícitamente que la cifra de mayo no tenía sustento propio en ese momento, presentar el entrenamiento real de julio como la evidencia vigente, y aclarar que la métrica correcta exigida por RNF-04 es F1 ponderado (0.9659), no accuracy (96.56%) — evitar usar "exactitud de la IA" sin especificar de qué métrica se habla. La diapositiva 25 debe corregirse antes de sustentar: fecha real del entrenamiento, dataset (sintético, 8000 muestras), partición train/test, y métrica exacta reportada.

## Riesgo 3 — Preguntas sobre funcionalidades ausentes (checklist BPA, PDF)

Ambas son nombradas explícitamente en el documento de tesis (TI menciona PDF; el backlog detalla el checklist con gran nivel de especificidad). Si el jurado pide ver cualquiera de las dos en funcionamiento, no existen. **Mitigación**: implementarlas antes de sustentar, o presentar un plan concreto y honesto de por qué quedaron fuera del alcance de esta iteración (priorización de sprints), sin minimizar la brecha.

## Riesgo 4 — Origen sintético del dataset de IA

Si se pregunta "¿con qué datos reales entrenaron el modelo?", la respuesta honesta es que el dataset es sintético. Esto es metodológicamente defendible (documentado, con inyección de ruido realista), pero **debe presentarse como una decisión consciente**, no descubrirse como una sorpresa incómoda.

## Riesgo 5 — Duplicación/no verificabilidad de cifras en la matriz bibliográfica (MD_)

Si el jurado audita el Anexo MD_ en detalle línea por línea (posible en una revisión exhaustiva), encontrará que dos entradas comparten cifras idénticas no verificables en los PDFs fuente. **Mitigación**: corregir el documento MD_ antes de la entrega final, localizando o retirando esas cifras específicas.

## Riesgo 6 — Historia de backlog corrupta (HU-43)

Bajo riesgo de que el jurado revise el Excel del Product Backlog en ese nivel de detalle, pero si lo hace, encontrará una historia con contenido copiado y sin planificación real. **Mitigación**: corregir o eliminar la fila antes de la entrega.

## Riesgo 7 — Ejecutabilidad no demostrada de las pruebas del backend

Esta auditoría no pudo ejecutar las 18 pruebas del backend por incompatibilidad de entorno Python. Si el equipo tampoco las ha ejecutado recientemente en su propio entorno, existe el riesgo de que no todas pasen realmente. **Mitigación**: ejecutar `pytest` en un entorno con Python 3.12 real antes de sustentar, y tener el resultado (verde) listo como evidencia.

## Riesgo 8 — Confusión sobre roles ("Auditor")

Si el jurado leyó el backlog y pregunta por el rol "Auditor" mencionado en varias historias, y luego ve que el sistema solo tiene 3 roles, puede percibirse como una inconsistencia. **Mitigación**: aclarar proactivamente que "Auditor" es un término descriptivo de función (quien realiza auditoría), cumplido por el rol "administrador", no un cuarto rol de sistema.

## Riesgo 9 — Margen de benchmarking no declarado

Bajo riesgo, pero si el jurado pide ver el detalle del Anexo2 componente por componente, notará que 2 de 13 comparaciones tuvieron márgenes de solo 2%, contradiciendo la afirmación textual de TI de que "ninguna selección quedó definida por un margen estrecho". **Mitigación**: ajustar la redacción de esa conclusión en TI si aún hay tiempo de corrección editorial, o estar preparados para explicarlo con los números reales.

## Riesgo 10 — RF-18 visible en backend pero no en frontend

Bajo riesgo, pero si el jurado pregunta específicamente por "estado de conectividad del dispositivo" (mencionado explícitamente como RF-18 en TI) y se les muestra el dashboard, no lo verán. **Mitigación**: agregar un indicador visual antes de sustentar — es una corrección de bajo esfuerzo relativo al impacto de la pregunta.
