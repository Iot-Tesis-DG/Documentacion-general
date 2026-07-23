# Revisión de PF_Soto_Diego_Gamio_Brenda.pptx (28 diapositivas)

Método: extracción completa de texto de todas las shapes (incluyendo grupos anidados) + notas del orador de las 28 diapositivas + inspección de tipos de shape para determinar si existen capturas reales (raster) del prototipo. Confirmado: **0 shapes de tipo PICTURE** en todo el documento — todos los diagramas son AutoShape/Freeform/Group nativos de PowerPoint (vectores dibujados a mano en el editor), no capturas de pantalla de un sistema en ejecución.

## Inventario por diapositiva

| # | Título | Mensaje principal | Datos/métricas declaradas |
|---|---|---|---|
| 1-3 | (portada/separadores sin texto) | — | — |
| 4 | Portada | Título completo, integrantes, fecha 16-may-2026, TDP | Códigos: Diego U202214477, Brenda U202120344 |
| 5 | Agenda | 9 secciones (Antecedentes→Conclusión) | — |
| 6 | Separador Antecedentes | — | — |
| 7 | 2 antecedentes (Ferraz et al.; Adilah et al.) | Evidencia de sonda fija vs medicamento real; ESP32+SHT31 validado | 12.5min/89min/17.5min (Ferraz); error<1% (Adilah) |
| 8 | Separador Problema | — | — |
| 9 | Planteamiento del problema | Control manual/discontinuo compromete calidad/eficacia/seguridad | — |
| 10 | "Caso: Boticas & Salud" | Nombre ficticio de caso de estudio | — |
| 11 | Formulación del problema | Pregunta de investigación completa (idéntica a TI) | — |
| 12 | Objetivo General | Idéntico a TI/PT/ACP | — |
| 13 | Objetivos Específicos OE1-OE4 | Idénticos a TI | — |
| 14 | Modelo Conceptual | Diagrama vectorial (sin texto extraíble adicional) | — |
| 15 | Arquitectura Lógica | Diagrama vectorial | — |
| 16 | Arquitectura Física | Diagrama vectorial | — |
| 17 | Separador Marco Teórico | — | — |
| 18 | Definiciones | 5 conceptos clave (termolábil, cadena frío, excursión, trazabilidad, Random Forest) | — |
| 19 | Estándares/Frameworks | Scrum Guide 2020 + Expansion Pack 2026, PMBOK v6 | — |
| 20 | Normativa Legal | Ley 31814 (IA), Ley 29733 (datos personales), RM 810-2024/MINSA | — |
| 21 | Separador Estado del Arte | — | — |
| 22 | (solo diagrama vectorial, sin texto) | — | — |
| 23 | Separador Comparativa de Soluciones | — | — |
| 24 | Hardware Edge y Sensado | Benchmarking vs Bhatti/Pires/Ferraz | Puntaje 4.80-5.00 |
| 25 | Inteligencia Artificial | Benchmarking vs Harrabi (deep learning/Isolated Forest) | **"Exactitud >96% para 3 estados"**, puntaje 4.80 |
| 26 | Seguridad y Arquitectura Backend/Web | Benchmarking vs apps móviles/backends monolíticos | Puntaje 4.70 |
| 27 | Conclusión | Polarización estado del arte (pasivo vs sobredimensionado); Random Forest balance óptimo; blockchain inviable para farmacias independientes | — |
| 28 | Cierre "Gracias" | Repite portada | Códigos: **"U202211477"** (Diego), U202120344 (Brenda) |

## Hallazgos

### [ALTO] Métrica de exactitud sin sustento verificable en la propia tesis
Diapositiva 25 afirma **"Exactitud >96% para 3 estados"** como resultado atribuido a "Nuestra propuesta" (el modelo Random Forest del propio prototipo). Esta cifra:
- No aparece en TI como resultado medido (TI solo declara F1≥0.85 como **criterio de validación a cumplir**, no como resultado ya obtenido).
- El Product Backlog (Anexo5) muestra las 46 tareas en estado "No se ha iniciado" — es decir, a la fecha de los documentos analizados, el modelo ni siquiera se había entrenado.
- La cifra 96% coincide con el resultado del artículo #9 del corpus (Quispe-Astorga et al. 2025, detección de fallas en unidades de enfriamiento con Random Forest, verificado como cifra real de ESE paper en `article_evidence_matrix.md`), un sistema distinto (fallas mecánicas de refrigeración industrial, no clasificación de riesgo térmico farmacéutico).

Esto constituye una posible sobre-afirmación académica: presentar como resultado propio una métrica que en realidad pertenece a un antecedente del estado del arte. Riesgo directo de ser señalado por el jurado si se pide evidencia del entrenamiento real del modelo.

### [MEDIO] Inconsistencia de identidad en diapositiva de cierre
Diapositiva 4 (portada): código de Diego = `U202214477` (coincide con TI). Diapositiva 28 (cierre, "Gracias"): código de Diego = `U202211477` — dígito transpuesto/incorrecto — y el nombre aparece como "Soto Quispe, Diego Soto" en vez de "Diego Ulises". Error tipográfico menor pero visible en la última diapositiva que verá el jurado.

### [BAJO] Nombre ficticio del caso de estudio no declarado en TI
Diapositiva 10 introduce el nombre "Boticas & Salud" como el caso de estudio. TI es explícito en aclarar que el escenario es "representativo" y "no corresponde a diagnóstico, auditoría ni levantamiento de infraestructura de ninguna empresa real" (TI línea 563) — pero no menciona este nombre ficticio en ningún pasaje leído de TI. Verificar si "Boticas & Salud" aparece en TI en alguna sección no revisada en profundidad (posible en el Caso de Estudio 1.2.3, sección no citada literalmente en esta auditoría) para confirmar consistencia de nomenclatura entre PPTX y TI.

### Ausencia de evidencia visual del prototipo
No hay una sola captura de pantalla del dashboard, del backend en ejecución, ni de hardware físico armado en las 28 diapositivas. Todos los "diagramas de arquitectura" son ilustraciones conceptuales (formas vectoriales), consistente con que, a la fecha de esta presentación (16-may-2026, anterior a TI del 22-jun-2026), el desarrollo real aún no había comenzado según el Product Backlog.

### Slides normativas — cobertura distinta a TI
Diapositiva 20 (Normativa Legal) no menciona ISO/IEC 30141:2024 ni OWASP IoT/Web STG, que TI sí cita como marco de alineamiento de seguridad — la diapositiva se limita a normativa peruana (leyes 31814/29733, RM 810-2024). No es necesariamente un error (la diapositiva puede estar enfocada solo en "normativa legal" separándola de "estándares técnicos", que aparecen en la diapositiva 19 sin nombrar ISO/OWASP tampoco). Verificar en frontend/backend si ISO/IEC 30141 y OWASP se mencionan en documentación técnica del código, para no depender solo de TI.
