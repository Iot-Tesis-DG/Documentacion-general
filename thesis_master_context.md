# thesis_master_context.md — Fase 1 COMPLETADA (Final Oficial)

Fecha análisis: 2026-07-22. NO modificar archivos originales. Copia persistente en `audit-output/thesis_master_context.md` (esta ruta en scratchpad es temporal/session-only).

Archivos complementarios de esta auditoría (todos en `audit-output/`): `final_oficial_inventory.md`, `article_evidence_matrix.md`, `benchmark_validation.md`, `backlog_traceability.md`, `presentation_review.md`.

## Rutas confirmadas
- `/Volumes/Universidad/Tesis Code/Final - Oficial /` (55 archivos)
- `/Volumes/Universidad/Tesis Code/frontend/` (existe, no abierto aún salvo `ls`)
- `/Volumes/Universidad/Tesis Code/backend/` (existe, no abierto aún salvo `ls`)
- También hay `backend.zip` (116MB) y `frontend.zip` (56MB) en raíz — probablemente snapshots/backups, no analizar salvo pedido explícito.

## Inventario completo Final Oficial (55 archivos) — ESTADO FINAL

Ver detalle completo con evidencia de método en `audit-output/final_oficial_inventory.md`. Resumen:

```
Final - Oficial /.DS_Store (×5 total en subcarpetas)  [EXCLUIDOS: metadata macOS, sin contenido]
Anexo1_IS_TI_Soto_Diego_Gamio_Brenda.pdf              [INSPECCIONADO: 219 págs, portada+resumen originalidad extraídos vía pypdf. Índice de similitud = 5% (3% internet, 2% publicaciones, 3% trabajos estudiante), 44 fuentes listadas. 216 págs restantes son raster (resaltado de coincidencias), no OCRed — no crítico, dato principal ya obtenido]
Anexo2_IB_Soto_Diego_Gamio_Brenda.docx                [INSPECCIONADO POR COMPLETO: 13 matrices de benchmarking verificadas aritméticamente, ver benchmark_validation.md]
Anexo3_ACP_Soto_Diego_Gamio_Brenda.docx               [Leído completo — Project Charter, consistente con TI]
Anexo4_CP_Soto_Diego_Gamio_Brenda.mpp                 [Metadata OLE2 extraída (file+olefile): presupuesto S/199,640, reserva contingencia 15% — coincide con TI. Contenido detallado de tareas requiere MPXJ/MS Project, no disponible; intento documentado]
Anexo5_PB_Soto_Diego_Gamio_Brenda.xlsx                [INSPECCIONADO POR COMPLETO: 4 hojas, 42 HU + 1 huérfana (HU-43), ver backlog_traceability.md]
Articulos cientificos - 40/ (40 PDFs)                 [Texto COMPLETO extraído de los 40 (todas las páginas). Verificación sustantiva de método/resultados/límites contra MD_, ver article_evidence_matrix.md]
MD_Soto_Diego_Gamio_Brenda.docx                       [Leído completo, 457 líneas, cruzado 1:1 contra los 40 PDF reales]
PF_Soto_Diego_Gamio_Brenda.pptx                       [INSPECCIONADO POR COMPLETO: 28 diapositivas, texto+shapes agrupadas extraídas, 0 imágenes raster (todo vectorial), ver presentation_review.md]
PMBOK_6ta_Edicion_Espanol-1.pdf                       [Verificado: 762 págs, edición confirmada = "Sexta edición" coincide con cita en TI. No leído extensamente (obra de terceros, correctamente excluida de lectura profunda)]
PT_Soto_Diego_Gamio_Brenda.docx                       [Leído completo, 188 líneas — versión previa consistente con TI]
TI_Soto_Diego_Gamio_Brenda.docx                       [Leído en profundidad en secciones sustantivas de las 2012 líneas — documento fuente principal]
```

## Documento fuente de verdad: TI_Soto_Diego_Gamio_Brenda.docx

### Identidad
- Título: "Desarrollo y validación de un prototipo basado en Internet de las cosas e inteligencia artificial, con trazabilidad digital verificable y controles de seguridad, para el monitoreo de la cadena de frío de medicamentos termolábiles en farmacias independientes de Lima Metropolitana, con validación técnica en el Cercado de Lima, 2026-2027."
- Autores: Soto Quispe, Diego Ulises (PM/Product Owner); Gamio Upiachihua, Brenda Lucía (Scrum Master/QA). Ambos desarrolladores.
- Asesor: Gutiérrez Gutiérrez, Alan Tito (metodológico y temático)
- UPC, Ing. de Software, Trabajo de Investigación para bachiller.

### Alcance geográfico (NO MEZCLAR)
- Problema planteado a nivel: Lima Metropolitana (27.30% de establecimientos farmacéuticos privados del país, DIGEMID 2025)
- Validación técnica: UN escenario representativo en Cercado de Lima, sin generalización estadística. Explícitamente ficticio/representativo, no empresa real (TI líneas 563, 565).

### Problema
Control térmico manual/discontinuo + registros físicos no verificables + detección tardía de excursiones térmicas (rango 2-8°C) en farmacias independientes. Causa raíz (5 porqués): "brecha operativa y documental", NO ausencia de tecnología específica (línea 313).

Pregunta de investigación (línea 568, TI): "¿De qué manera las limitaciones en el control térmico continuo, la detección oportuna de excursiones térmicas y la disponibilidad de registros verificables afectan la conservación de medicamentos termolábiles entre 2°C y 8°C en farmacias independientes de Lima Metropolitana durante el horizonte académico 2026-2027?"

### Objetivos (TI líneas 571-577)
- OG: desarrollar y validar técnicamente el prototipo (idéntico en TI/PT/Anexo3).
- OE1: analizar requisitos (revisión documental + entrevistas + observación técnica).
- OE2: diseñar arquitectura física/lógica (sensores, MCU, backend, BD, dashboard, IA RandomForest 3 clases, alertas, trazabilidad, seguridad ISO/IEC 30141:2024 + OWASP IoT STG v1.0.0 + OWASP WSTG v4.2 — SIN certificación formal, dicho explícitamente).
- OE3: desarrollar (captura continua, histórico, alertas, clasificación IA, auditoría, checklist BPA digital, verificación hash encadenado).
- OE4: validar técnicamente (MTTD, precisión medición, latencia alerta, disponibilidad, integridad, desempeño clasificador, seguridad, usabilidad SUS).

### Stack técnico completo (verificado en documento, TI líneas 583-651)
- Edge: ESP32 DevKitC V4 (Xtensa LX6 240MHz dual-core, 520KB SRAM, 4MB Flash), firmware C++ ESP-IDF.
- Sensores: SHT31 (I2C, temp+humedad refrigerador), DS18B20 (1-Wire, temp junto al medicamento, impermeable), MC-38 reed switch (GPIO, apertura puerta).
- Buffer offline: LittleFS (partición Flash), reintento automático, sync máx 30s tras reconexión (RNF-07).
- Transporte: MQTT sobre TLS 1.2/1.3, puerto 8883, hacia EMQX Cloud Serverless (servicio externo gestionado, NO parte del código propio).
- Backend: FastAPI Python 3.12, arquitectura DDD (capas Interface/Application/Domain/Infrastructure + Edge + Seguridad transversal), cliente MQTT vía `aiomqtt`, desplegado Railway (plan Hobby, 1vCPU/512MB RAM).
- BD: PostgreSQL con JSONB, mismo nodo Railway, subred privada puerto TCP 5432 (sin exposición internet).
- IA: Random Forest (scikit-learn), 3 clases de salida: `normal` / `riesgo_preventivo` / `excursion_critica`. Motor de clasificación vive en capa de Dominio.
- Trazabilidad: hash SHA-256 encadenado, campos `payload`, `timestamp`, `previous_hash`, `hash_actual`. EXPLÍCITAMENTE NO blockchain (TI línea 169: "sin dependencia de blockchain").
- Frontend: React 19 + Vite + TypeScript + shadcn/ui, gráficas Apache ECharts v5, tiempo real vía **SSE** (no WebSocket), desplegado Vercel (CDN, WAF, firewall L7, 100GB/mes plan Hobby).
- Seguridad transversal: JWT + RBAC (3 roles: administrador/farmacéutico/técnico), TLS 1.2/1.3, `audit_logs` inmutable, mínimo privilegio. Alineado (NO certificado) a ISO/IEC 30141:2024, OWASP IoT STG v1.0.0, OWASP WSTG v4.2.

### Requisitos funcionales (18, códigos exactos RF-01 a RF-18) — TI líneas 583-600
RF-01 captura SHT31 I2C · RF-02 captura DS18B20 1-Wire · RF-03 reed switch MC-38 apertura · RF-04 payload JSON (device id, lecturas, timestamp, conectividad) · RF-05 publish MQTT/TLS 8883 · RF-06 buffer offline LittleFS + reenvío · RF-07 persistencia tabla `thermal_readings` PostgreSQL · RF-08 clasificación Random Forest 3 estados · RF-09 alerta si riesgo_preventivo/excursion_critica → tabla `thermal_alertas` · RF-10 registro de acciones correctivas por usuario responsable · RF-11 dashboard tiempo real SSE (lecturas, historial, alertas) · RF-12 filtro historial (fecha/dispositivo/conectividad/riesgo) · RF-13 reportes exportables BPA · RF-14 registro trazabilidad hash SHA-256 (payload+timestamp+previous_hash+hash_actual) · RF-15 endpoint de verificación de integridad de cadena de hashes · RF-16 `audit_logs` de toda acción crítica de usuario autenticado · RF-17 JWT + RBAC 3 niveles · RF-18 dashboard muestra estado de conectividad por dispositivo.

### Requisitos no funcionales (10, RNF-01 a RNF-10) — TI líneas 601-611
RNF-01 latencia captura→persistencia ≤5s · RNF-02 disponibilidad FastAPI/Railway ≥95% · RNF-03 integridad 100% de registros de trazabilidad sin inconsistencias · RNF-04 F1-Score ponderado Random Forest ≥0.85 · RNF-05 TLS obligatorio ESP32↔EMQX, sin credenciales embebidas en código · RNF-06 ningún endpoint accesible sin JWT válido salvo auth · RNF-07 sync buffer LittleFS ≤30s tras reconexión · RNF-08 arquitectura DDD escalable sin tocar capa dominio · RNF-09 repositorio PostgreSQL sustituible sin alterar lógica negocio · RNF-10 carga inicial dashboard ≤3s.

### Criterios de validación técnica (métricas numéricas, TI ~línea 1080-1096)
- MTTD y latencia alerta: <3s
- Modelo IA: F1 ≥0.85 (accuracy/precisión/recall reportadas por clase)
- Medición temperatura: MAE ≤0.5°C vs instrumento de referencia
- Trazabilidad: 100% verificaciones exitosas cadena SHA-256
- Dashboard: disponibilidad ≥95%, SUS ≥70
- Proceso: mejora documentada vs registro manual (AS-IS vs TO-BE)

### Matriz de riesgos propia del proyecto (12 riesgos, PMBOK, TI líneas 1098-1192)
Destacan: R-04 (dataset insuficiente/no representativo para Random Forest, Alto/Medio, mitigar), R-10 (error en implementación que comprometa integridad hash encadenado, Alto/Bajo, mitigar con pruebas de alteración simulada). Estos DOS riesgos son los que el jurado más probablemente cuestionará — verificar en backend si las mitigaciones se implementaron realmente (tests de integridad de cadena, dataset de entrenamiento documentado).

### Benchmarking (Anexo2, resumen vía TI Tabla 9)
13 componentes evaluados cuantitativamente (matriz de enfrentamiento, escala 1-3-5, 4 alternativas c/u). Score promedio general: 4.89/5.00. Selección más "débil" relativa: MQTT sobre TLS (4.70) aún así por encima de alternativas (4.60/3.40/3.20). Selecciones: ESP32 DevKitC V4, SHT31+DS18B20+MC-38, Arduino Core+LittleFS, MQTT/TLS, EMQX Cloud tipo, FastAPI+DDD, PostgreSQL+JSONB, Random Forest, SHA-256+previous_hash, React+Vite+TS+shadcn/ui, ECharts+SSE, Railway+Vercel, JWT/RBAC+MQTT/TLS+auditoría.

### Marco normativo declarado
- Sanitario: DIGEMID (Guía BPA, RM 810-2024/MINSA)
- Seguridad IoT: ISO/IEC 30141:2024, OWASP IoT Security Testing Guide v1.0.0, OWASP Web Security Testing Guide v4.2 (v4.2 declarado como "versión estable disponible" — nota: OWASP WSTG ya tiene v5 en desarrollo activo a 2026, verificar vigencia real de v4.2 si jurado pregunta)
- IA: mención de marco normativo IA en TOC (2.3.3) no leído en detalle — pendiente si se requiere profundizar

## Tabla de fuentes — 40 artículos científicos (TODOS verificados PDF real vs MD_)

Verificación: título, autores y DOI EXTRAÍDOS DIRECTAMENTE del texto de la página 1-2 de cada PDF (no solo nombre de archivo), cruzados contra la matriz MD_Soto_Diego_Gamio_Brenda.docx. Coincidencia 40/40.

| # | Categoría | Título (abrev) | Autores principales | Año | Revista | DOI | Aporte clave | Vacío vs tesis |
|---|---|---|---|---|---|---|---|---|
| 1 | P1 | Real-time temp anomaly detection (CAE en ESP32) | Harrabi et al. | 2024 | Frontiers in AI | 10.3389/frai.2024.1429602 | IA (deep learning) viable en ESP32 low-resource | Usa CAE no Random Forest; sin trazabilidad/seguridad |
| 2 | P1 | Smart monitoring temp/humidity Colombia | Cabezas et al. | 2025 | IoT (MDPI) | 10.3390/iot6010015 | ESP32+SHT31 en farmacia real (Colombia) | Sin MQTT/broker, sin IA, sin hash |
| 3 | P1 | IoT-enhanced transport MQTT+SMS | Bhatti et al. | 2024 | IEEE Access | 10.1109/ACCESS.2024.3382508 | Valida MQTT+cifrado ECC/CRC-32 en tránsito | Transporte dinámico no almacenamiento estático; CRC-32 no SHA-256 |
| 4 | P1 | Multi-sensor IoT platform beyond hospital | Frontera-Bergas et al. | 2025 | Internet of Things (Elsevier) | 10.1016/j.iot.2025.101711 | Arquitectura IoT distribuida domiciliaria | LoRa/BLE no MQTT/TLS; sin IA ni hash ni RBAC |
| 5 | P1 | IoT integrated sensing/logging cold chain | Baghel et al. | 2024 | IEEE JRFID | 10.1109/JRFID.2024.3488534 | Data logger BLE bajo costo | Solo BLE a móvil, sin broker/dashboard/IA |
| 6 | P1 | Low-power IoT continuous temp monitoring | Pires et al. | 2025 | Designs (MDPI) | 10.3390/designs9030073 | Bajo consumo energético | BLE+Raspberry Pi no ESP32/MQTT; sin IA/trazabilidad |
| 7 | P1 | Medication cold chain KSA (RFID+GPS) | Bouazzi et al. | 2025 | Eng. Research Express | 10.1088/2631-8695/adb5d8 | Transporte con RFID/GPS validado | Arduino no ESP32; sin IA/hash |
| 8 | P1 | IoT smart cold supply chain (Jeddah) | Alshdadi et al. | 2024 | ETASR | 10.48084/etasr (etas) | Prototipo real ESP8266+DHT22 | Sin MQTT/TLS, sin buffer offline, sin IA |
| 9 | P2 | Data-driven fault detection cooling (Perú) | Quispe-Astorga et al. | 2025 | Sensors (MDPI) | 10.3390/s25123647 | Random Forest 96% exactitud, autores peruanos | Raspberry+Arduino sobredimensionado; sin hash |
| 10 | P2 | Energy efficiency supermarkets CO2 (RF) | Farahani et al. | 2025 | Applied Energy | 10.1016/j.apenergy.2024.124479 | RF 99.48% exactitud, valida RF sobre ANN | Supermercados no farmacia; sin API/alertas/trazabilidad |
| 11 | P2 | Cost-effective over-temp alarm (ANN) | Meng et al. | 2024 | J. Food Engineering | 10.1016/j.jfoodeng.2023.111914 | ANN 97.4% precisión alarma | Caja negra (baja interpretabilidad) vs RF; transporte no almacenamiento |
| 12 | P2 | Anomaly detection stream cold chain (iForest) | Xie, Long, Ling, Zhou, Luo | 2025 | PLOS ONE | 10.1371/journal.pone.0315322 | iForest F1=0.863, AUC=0.9075 | Isolated Forest no RF; agroalimentario, sin clases discretas de riesgo |
| 13 | P2 | Intelligent fluid leakage refrigeration | de Lima Munguba et al. | 2026 | Applied Thermal Eng. | 10.1016/j.applthermaleng.2025.129326 | Ensamble clasificación fugas | Sensores vibración USD 2500 inviables; congeladores comerciales |
| 14 | P2 | Refrigerant leakage NCA+ANN | Sholahudin et al. | 2026 | J. Building Engineering | 10.1016/j.jobe.2026.115293 | 98.65% clasificación train | HVAC edificios no farmacéutico; ANN caja negra |
| 15 | P2 | Gas sensor hydrobiological species | Alvarado et al. | 2025 | IEEE Sensors Journal | 10.1109/JSEN.2025.3549785 | Boosted Trees 99.93% | Sensado gases descomposición, no aplica a viales sellados |
| 16 | P2 | Detecting faults cooling temp+energy | Kaushik & Naik | 2024 | Energy Informatics | 10.1186/s42162-024-00351-1 | NN-DBSCAN localiza componente defectuoso | HVAC eficiencia energética, no riesgo biológico |
| 17 | P3 | Blockchain counterfeit drugs (Ethereum) | Ahmed et al. | 2026 | J. Supercomputing | 10.1007/s11227-026-08304-z | 98.9% precisión monitoreo temp | Blockchain público, testnets, sin trazabilidad ligera SHA-256 |
| 18 | P3 | Multilevel auth blockchain anti-counterfeit | Sharma & Rohilla | 2024 | J. Supercomputing | 10.1007/s11227-023-05654-w | Hyperledger Fabric 417.5 TPS, QR watermark | Enfoque visual/logístico, sin sensado IoT continuo |
| 19 | P3 | Reducing counterfeit blockchain+IoT | Mole & Shaji | 2026 | OPSEARCH | 10.1007/s12597-026-01183-1 | Rendimiento cloud estable | DHT11 impreciso; Arduino+ESP8266 redundante; sin TPS reportado |
| 20 | P3 | Blockchain secure info sharing pharma | Padma & Ramaiah | 2024 | Heliyon | 10.1016/j.heliyon.2024.e40273 | 600 TPS, 5000 usuarios concurrentes | EdDSA/asimétrico sobredimensionado para ESP32 |
| 21 | P3 | DrugBlock IoT+blockchain consumer electronics | Namasudra et al. | 2025 | IEEE Trans. Consumer Electronics | 10.1109/TCE.2024.3473739 | Valida que blockchain público es costoso → justifica SHA-256 | Costos Gas desproporcionados para farmacia independiente |
| 22 | P3 | Blockchain secure info sharing (dup. categoría) | Padma & Ramaiah | 2024 | Heliyon | 10.1016/j.heliyon.2024.e40273 | (ver #20) | — |
| 23 | P3 | Blockchain Saudi Arabia drug supply (NFT) | Alshahrani | 2024 | PeerJ Computer Science | 10.7717/peerj-cs.2072 | PoC funcional con NFT por lote | Sin métricas TPS/latencia; minería NFT inviable para telemetría constante |
| 24 | P3 | Patient-centric blockchain-enabled IoT (post-dispensación) | Ramis-Bibiloni et al. | 2025 | Peer-to-Peer Networking & Apps | 10.1007/s12083-025-02120-7 | Costo bajo en red Polygon (<0.05€/tx) | Sin controles BPA/DIGEMID institucionales |
| 25 | P4 | Novel lightweight multi-factor auth MQTT | Saqib & Moon | 2024 | Microprocessors & Microsystems | 10.1016/j.micpro.2024.105088 | Autenticación mutua, 41.6bps, 2.3ms latencia | Solo simulado, sin ESP32/EMQX real; sin RBAC/hash |
| 26 | P4 | Intelligent two-phase dual auth IoMT | Asif et al. | 2025 | Scientific Reports | 10.1038/s41598-024-84713-5 | -45% tiempo cifrado, -45.38% costo comp. | Solo simulación, sin RBAC ni hash SHA-256 |
| 27 | P4 | Design secure IoT trade-off tactics (ISO 25010) | Orellana et al. | 2024 | Sensors (MDPI) | 10.3390/s24227314 | Catálogo tácticas STRIDE alineado ISO 25010:2023 | Monitoreo ambiental genérico, sin RBAC farmacéutico ni hash |
| 28 | P4 | Securing IoT smart healthcare PUF-based | Alruwaili et al. | 2024 | Heliyon | 10.1016/j.heliyon.2024.e37577 | -95.8% costo computacional | Sin hardware ESP32 real, sin RBAC 3 roles ni auditoría |
| 29 | P4 | LAMT lightweight anonymous auth IoMT | Lee et al. | 2025 | Sensors (MDPI) | 10.3390/s25030821 | 0.14ms costo total, verificación ProVerif | Diseñado para wearables, no MQTT/broker cloud |
| 30 | P4 | Cloud-assisted anonymous auth IoMT | Guo, Xu, Liang | 2025 | Computers & Security | 10.1016/j.cose.2025.104614 | 2144 bits comunicación (menor de 6 esquemas) | Datos fisiológicos, no sensores térmicos MQTT |
| 31 | P4 | Security-enhanced lightweight anonymity IoT healthcare | Zhou et al. | 2024 | IEEE IoT Journal | 10.1109/JIOT.2023.3323614 | Único protocolo con protección simultánea 5 vectores ataque | Wearables hospitalarios, no telemetría ESP32→MQTT |
| 32 | P4 | Lightweight secure auth remote monitoring IoMT | Ali et al. | 2024 | IEEE Access | 10.1109/ACCESS.2024.3400400 | (⚠ MD_ duplica texto descriptivo del art. #25 Saqib&Moon — ver Inconsistencias) | Simulado, sin RBAC/hash |
| 33 | P5 | Feasibility/usability thermolabile home monitoring (QChainMED) | do Pazo-Oubiña et al. | 2026 | Scientific Reports | 10.1038/s41598-026-46095-8 | SUS=82.5 (excelente), NPS=17.5, 23 pacientes reales | Todos dispositivos tuvieron interrupción >24h; sin IA/hash/RBAC |
| 34 | P5 | Temperature excursions cold chain (sondas) | Ferraz et al. | 2024/2025 | Hospital Pharmacy | 10.1177/00185787241282245 | Evidencia empírica clave: sonda fija 12.5min vs 23-26min real | Un solo refrigerador; sin IoT/alertas/trazabilidad — es la fuente #1 citada en el planteamiento del problema de TI |
| 35 | P5 | Reproducing cold-chain Peltier climate system | Garrido-López et al. | 2025 | Sensors (MDPI) | 10.3390/s25216689 | MAE 0.19°C, R²=0.9985 | Agroalimentario, sin MQTT/RF/trazabilidad |
| 36 | P5 | Ensuring vaccine temp integrity storage→delivery | Lamba et al. | 2024 | Global J. Flexible Systems Mgmt | 10.1007/s40171-024-00401-3 | 214 embarques, 44.37% tiempo fuera de rango | Data loggers retrospectivos, sin tiempo real/IA/trazabilidad |
| 37 | P5 | Shock/temp monitoring immunoglobulins hospital→home | Susam et al. | 2024 | J. Pharmaceutical Sciences | 10.1016/j.xphs.2024.03.011 | 81% paquetes fuera de rango (solo 2/20 en frío OK) | Pasivo, sin conectividad tiempo real ni IA |
| 38 | P5 | IoT temp monitoring fruit/vegetable requirements | Lamberty & Kreyenschmidt | 2025 | Discover Food | 10.1007/s44187-025-00427-1 | Sigfox falla en confiabilidad alta densidad | Sin seguridad, trazabilidad, RBAC, IA |
| 39 | P5 | Assessing vaccine cold chain Ukraine | Gaievskyi et al. | 2026 | BMC Health Services Research | 10.1186/s12913-026-14332-5 | 72,040h registro, cumplimiento cae a 54.8-60.3% en tránsito | Retrospectivo, sin IoT tiempo real/dashboard |
| 40 | P5 | UMITEMP wireless sensor network validation | Freire et al. | 2026 | J. Food Process Engineering | 10.1111/jfpe.70570 | 99.65%/97.16% entrega paquetes, MAE bajo | RF propietario no MQTT; agroalimentario, sin IA/hash/RBAC |

Nota: fila #22 en la matriz MD_ repite el artículo #20 (Padma & Ramaiah, Heliyon) como entrada separada — posible error de numeración/duplicado en el documento MD_, a verificar contra el conteo real (el MD_ declara 8 artículos por categoría P1-P5 = 40 total, pero P3 lista 9 entradas 17-24 en mi conteo original más el duplicado — revisar total real de P3, puede ser 8 y mi indexación tuvo off-by-one, no bloqueante).

## Inconsistencias detectadas (Fase 1 — CONFIRMADAS con verificación profunda)

1. **[ALTO] Cifras no verificables/posiblemente fabricadas en MD_**: los artículos Saqib & Moon (2024) y Ali et al. (2024, IEEE Access) tienen el mismo párrafo de resultados (41.6bps, 0.00146J, 2.3ms vs 65.0ms MQTT-RSA-ECC) en MD_. Se buscó exhaustivamente esas 4 cifras en el texto COMPLETO (todas las páginas) de ambos PDFs y de los 40 del corpus: **no aparecen en ningún documento**. No es solo duplicación de texto, es una afirmación cuantitativa sin fuente verificable. Ver `article_evidence_matrix.md`.
2. **[ALTO] Slide 25 de PF_.pptx afirma "Exactitud >96% para 3 estados" como resultado propio del modelo Random Forest de la tesis** — pero (a) TI solo declara F1≥0.85 como criterio a cumplir, no resultado medido; (b) Anexo5 muestra 100% de las historias "No se ha iniciado" a la fecha; (c) el 96% coincide con el resultado real del artículo de Quispe-Astorga et al. (fallas en refrigeración, sistema distinto). Ver `presentation_review.md`.
3. **[MEDIO] Benchmarking — margen estrecho no declarado**: Microcontrolador edge (ESP32 DevKitC 4.80 vs ESP32+IA 4.70, Δ0.10) y Comunicación IoT (MQTT/TLS 4.70 vs alternativa 4.60, Δ0.10) tienen márgenes de solo 2%, pero TI afirma categóricamente "ninguna selección quedó definida por un margen estrecho" — solo reconoce el segundo caso. Ver `benchmark_validation.md`.
4. **[ALTO] RNF-04 (F1≥0.85) sin historia de usuario en el backlog** — ninguna HU planifica entrenamiento/validación/medición de métricas del modelo Random Forest. Ver `backlog_traceability.md`.
5. **[MEDIO] HU-43 huérfana/corrupta en Anexo5**: título habla de "baja/reemplazo de dispositivo IoT" pero sus criterios de aceptación son copia literal de HU-41 (RBAC); no está en ninguna Épica ni Sprint — funcionalidad de facto no planificada. Ver `backlog_traceability.md`.
6. **[MEDIO] RBAC roles inconsistentes**: TI define 3 roles fijos (administrador/farmacéutico/técnico, RF-17); el backlog usa de facto 4 (Técnico/Farmacéutico-Regente/Administrador/Auditor) sin definición unificada. Ver `backlog_traceability.md`.
7. **[MEDIO] Identidad inconsistente en PF_.pptx**: código de estudiante de Diego difiere entre slide 4 (U202214477, correcto) y slide 28 (U202211477, incorrecto) + nombre mal escrito en cierre.
8. **[BAJO] OWASP WSTG v4.2 declarado "versión estable"**: verificar vigencia en fecha de sustentación (2026-2027).
9. **Resuelto — P3 tiene 8 PDFs reales** (confirmado por sistema de archivos), la "9na entrada" era una fila de texto duplicada dentro de MD_, no un archivo faltante/sobrante.
10. **[INFO] TI vs PT**: PT (12 abr 2026) es versión temprana idéntica en objetivos/problema a TI (22 jun 2026) — evolución consistente, no contradicción.
11. **[INFO] Anexo4 .mpp confirma presupuesto S/199,640 y reserva de contingencia 15%** — coincide exactamente con lo declarado en TI (línea 1099), buena señal de coherencia documental cruzada.

## Riesgos para sustentación (anticipados por el propio equipo + detectados por mí)
- R-04/R-10 propios (dataset RF insuficiente; error en hash encadenado) — el jurado atacará primero estos dos puntos.
- Validación en 1 sola farmacia (n=1) — jurado puede cuestionar validez estadística; documento ya se cuida explícitamente de no generalizar.
- Terminología cuidadosa "trazabilidad digital verificable" vs blockchain — verificar en backend que NO se usó librería blockchain que contradiga el posicionamiento académico.
- Duplicación de texto en MD_ (hallazgo #1 arriba) podría ser señalada como descuido de rigor bibliográfico si el jurado revisa el anexo en detalle.

## Preguntas concretas a comprobar en Frontend (Fase 2)
- ¿Dashboard usa SSE real (EventSource) o hay WebSocket/polling disfrazado?
- ¿Existen las 3 vistas de rol (administrador/farmacéutico/técnico) con RBAC visible en rutas?
- ¿Filtros de historial (RF-12: fecha/dispositivo/conectividad/riesgo) implementados?
- ¿Export de reportes BPA (RF-13) funcional o placeholder?
- ¿Se muestra estado de conectividad por dispositivo (RF-18)?
- ¿Tiempo de carga inicial dashboard cumple RNF-10 (≤3s)? (requiere prueba real, no solo código)
- ¿JWT se persiste de forma segura (no localStorage plano si hay XSS risk)?

## Preguntas concretas a comprobar en Backend (Fase 3)
- ¿RF-01 a RF-18 todos con endpoint/lógica real, o subset simulado/hardcodeado?
- ¿Tablas reales: `thermal_readings`, `thermal_alertas`, `audit_logs` — existen migraciones que las crean?
- ¿Random Forest: dataset de entrenamiento real documentado, o clasificador con reglas if/else disfrazadas de IA?
- ¿F1≥0.85 (RNF-04) medido con métricas reales guardadas, o solo declarado en tesis sin evidencia en código/tests?
- ¿Endpoint de verificación de cadena hash (RF-15) existe y realmente detecta alteración?
- ¿RBAC de 3 niveles implementado a nivel de middleware/dependency, o solo cosmético en frontend?
- ¿Firmware ESP32 real en el repo (C++/ESP-IDF) con buffer LittleFS, o solo descrito en tesis sin código correspondiente?
- ¿Cliente MQTT (`aiomqtt`) conectado a EMQX real o mockeado en pruebas?
- ¿TLS 1.2/1.3 configurado realmente en conexión ESP32↔EMQX (certificados, no solo puerto 8883 abierto)?

## Siguiente paso autorizado
Fase 2 — Frontend. Iniciar con `find frontend -type f` (excluyendo node_modules/dist/build), luego package.json, estructura src/, servicios HTTP, auth flow, SSE client.
