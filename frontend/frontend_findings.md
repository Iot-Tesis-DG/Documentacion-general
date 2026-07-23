# Hallazgos — Frontend (Fase 2)

## F-01 [ALTO] Checklist BPA no persiste en backend — contradice HU-37 explícitamente

- **Archivo**: `src/presentation/pages/ChecklistBPAPage.tsx`, líneas 1-30 (función `cargarEstado`, `useEffect` de persistencia).
- **Evidencia**: `localStorage.getItem(CLAVE_STORAGE)` / `localStorage.setItem(CLAVE_STORAGE, ...)`. Ninguna importación de `apiClient` en todo el archivo.
- **Descripción**: HU-37 del backlog (Anexo5) especifica literalmente: *"el documento se envía al backend (FastAPI) firmando la transacción con la identidad extraída de mi token JWT y la fecha exacta del registro en PostgreSQL"*. La implementación real guarda el estado únicamente en el `localStorage` del navegador del usuario, sin llamada de red alguna.
- **Impacto**: El checklist BPA, pensado como evidencia digital verificable ante fiscalización sanitaria (RN-03 en TI), **no genera ningún registro auditable ni trazable**. Si el usuario limpia el navegador o usa otro dispositivo, el progreso desaparece sin dejar rastro — contradice directamente el propósito de "soporte documental exportable para auditoría" (TI, propuesta de solución).
- **Requisito afectado**: RF-13 (parcial), HU-37 (Épica EP05).
- **Recomendación**: Conectar a un endpoint real (`POST /api/checklist-bpa` o similar) que persista en PostgreSQL con firma JWT y, opcionalmente, registro en `traceability_records`, tal como describe la propia historia de usuario.
- **Estado epistemológico**: Verificado en código.

## F-02 [ALTO] Exportación de reportes no incluye PDF — contradice HU-38 y TI

- **Archivo**: `src/application/hooks/useReportesBPA.ts`, funciones `descargarJson`/`descargarCsv`. `package.json` sin librería PDF.
- **Evidencia**: `grep` exhaustivo de "pdf" en todo `package.json`+`src/` → cero coincidencias.
- **Descripción**: TI (línea 647) describe explícitamente exportación "en JSON o PDF"; HU-38 del backlog se titula "Exportación de Reporte PDF (Evidencia Sanitaria)" y su primer escenario de aceptación dice: *"el navegador descarga un documento formal que incluye el resumen gráfico de ECharts, la tabla de telemetría y las firmas criptográficas SHA-256... para respaldar su inmutabilidad algorítmica"*. La implementación real solo ofrece CSV y JSON.
- **Impacto**: El formato PDF es, según TI, el que un auditor/inspector (DIRIS/DIGEMID) usaría como evidencia formal imprimible con gráficos incluidos — CSV/JSON no cumplen ese propósito de presentación ante un inspector no técnico.
- **Requisito afectado**: RF-13, HU-38.
- **Recomendación**: Agregar generación de PDF (client-side con librería tipo `jsPDF`/`pdfmake`, o solicitar el PDF ya renderizado desde el backend).
- **Estado epistemológico**: Verificado en código.

## F-03 [ALTO] RNF-04 (F1≥0.85) sin ninguna superficie de UI — el frontend no expone métricas del modelo

- **Archivo**: N/A (ausencia transversal — revisado `DashboardPage.tsx`, `HistorialPage.tsx`, `AlertasPage.tsx` y todo `presentation/`).
- **Evidencia**: `grep` de "f1\|precision\|recall\|accuracy\|confianza\|métrica" en `presentation/` no arroja ningún panel de métricas del modelo (solo se ve `probabilidad`/`confianza` mencionada en Anexo5 HU-18 a nivel de backlog, pero no implementada visualmente en ninguna página revisada).
- **Descripción**: Coincide con el gap de planificación ya detectado en Fase 1 (backlog sin historia de entrenamiento/medición). El frontend refuerza esta ausencia: no hay ningún lugar donde el sistema, aunque tuviera las métricas, las mostraría al usuario o al jurado.
- **Impacto**: Sin evidencia de UI ni de backlog, es más difícil sostener en sustentación que RNF-04 fue verificado con datos reales — refuerza el riesgo académico ya anticipado en Fase 1.
- **Requisito afectado**: RNF-04.
- **Recomendación**: Agregar, aunque sea en una vista de administración, un indicador de la última evaluación del modelo (F1, fecha de entrenamiento, tamaño del dataset).
- **Estado epistemológico**: Verificado en código (ausencia).

## F-04 [MEDIO] RF-18 (estado de conectividad del dispositivo) no se renderiza en ninguna pantalla

- **Archivo**: campo `LecturaTermica.estado_conectividad` (`domain/entities/LecturaTermica.ts`), nunca referenciado en `presentation/`.
- **Evidencia**: `grep -rn "estado_conectividad" presentation/` → sin resultados.
- **Descripción**: TI RF-18 exige explícitamente "El dashboard muestra el estado de conectividad de cada dispositivo registrado." El tipo de datos incluye el campo (indicando que el contrato con el backend lo contempla), pero ninguna página lo consume visualmente.
- **Impacto**: Requisito funcional formalmente declarado en TI, no implementado en la interfaz, pese a que el dato ya viaja en el modelo.
- **Requisito afectado**: RF-18.
- **Recomendación**: Agregar un indicador visual (ej. badge "en línea"/"desconectado") en Dashboard o en una futura pantalla de gestión de dispositivos.
- **Estado epistemológico**: Verificado en código.

## F-05 [MEDIO] shadcn/ui declarado pero implementado de forma manual/parcial

- **Archivo**: ausencia de `components.json`; `presentation/components/ui/` con solo 6 componentes.
- **Evidencia**: `find` confirma no existe `components.json`; benchmarking (Anexo2, componente "Frontend y dashboard") declara "React + Vite + TypeScript + shadcn/ui" como tecnología ganadora con puntaje 5.00/5.00.
- **Descripción**: El proyecto usa las mismas dependencias base que shadcn/ui (Radix primitives, `class-variance-authority`, `clsx`, `tailwind-merge`) pero los componentes fueron escritos a mano, no generados/instalados vía la CLI oficial de shadcn. Es una diferencia de proceso, no necesariamente de resultado visual, pero afecta la trazabilidad de "qué librería se usó realmente" si el jurado pregunta específicamente por shadcn/ui.
- **Impacto**: Bajo riesgo funcional, riesgo académico menor (posible pregunta de jurado sobre por qué no hay `components.json`).
- **Requisito afectado**: Ninguno de TI directamente (es una decisión de benchmarking/tecnología, no un RF/RNF).
- **Recomendación**: Aclarar en la sustentación que se adoptó el *patrón* shadcn/ui manualmente, no la herramienta CLI.
- **Estado epistemológico**: Verificado en configuración.

## F-06 [MEDIO] Cero pruebas automatizadas

- **Archivo**: N/A (ausencia transversal — `package.json` sin `test` script, sin `vitest`/`jest`/`@testing-library` en dependencias, `find` sin archivos `*.test.*`/`*.spec.*`).
- **Descripción**: No existe ningún tipo de prueba (unitaria, de componente, de integración, e2e, accesibilidad) en el frontend.
- **Impacto**: No hay red de seguridad ante regresiones; la calidad actual del código (alta, ver hallazgos positivos) depende enteramente de disciplina manual, no de verificación automatizada.
- **Requisito afectado**: Validación técnica general (OE4 de TI menciona pruebas controladas, pero se refiere a la validación del prototipo completo, no necesariamente pruebas unitarias de frontend — de cualquier modo, es una brecha de calidad).
- **Recomendación**: Agregar al menos pruebas de humo con Vitest + Testing Library para los hooks críticos (auth, RBAC) antes de sustentación.
- **Estado epistemológico**: Verificado en configuración.

## F-07 [MEDIO] Sin ESLint configurado

- **Archivo**: N/A — ausencia de `eslint.config.*`/`.eslintrc*` en el proyecto.
- **Descripción**: Pese a que el código observado es de alta calidad (TypeScript estricto, cero `any`, cero código muerto detectado), no hay una herramienta de lint que lo garantice de forma automatizada y continua.
- **Impacto**: Bajo actualmente (el código es limpio), pero es un riesgo de mantenibilidad futura.
- **Recomendación**: Agregar ESLint con el plugin oficial de `typescript-eslint` + `eslint-plugin-react-hooks`.
- **Estado epistemológico**: Verificado en configuración.

## F-08 [BAJO] `MODO_DEMO` — riesgo teórico si se activara accidentalmente contra un backend real

- **Archivo**: `infrastructure/demo/demoAdapter.ts`, función de login demo (no verifica contraseña, cualquier password es aceptado).
- **Descripción**: `MODO_DEMO` se define solo en tiempo de build (`VITE_MODO_DEMO`), por lo que el riesgo de activación accidental en producción real es bajo (requeriría un error de configuración de build, no de runtime). Se documenta como medida de higiene, no como vulnerabilidad explotable en el estado actual.
- **Impacto**: Bajo, dado el diseño de build-time flag.
- **Recomendación**: Ninguna acción urgente; considerar un aviso adicional en CI/CD que impida desplegar `build:demo` a un dominio que no sea el de demostración pública.
- **Estado epistemológico**: Verificado en código.

## F-09 [BAJO] Sin `ErrorBoundary` global

- **Archivo**: `src/App.tsx`, `src/main.tsx` — ningún componente de tipo `ErrorBoundary` en todo el árbol.
- **Descripción**: Un error de renderizado no capturado en cualquier página colapsaría el árbol de React completo (pantalla en blanco), sin mensaje de recuperación para el usuario.
- **Impacto**: UX pobre ante errores inesperados, no es un defecto funcional de los RF/RNF.
- **Recomendación**: Agregar un `ErrorBoundary` de nivel raíz con mensaje de recuperación.
- **Estado epistemológico**: Verificado en código.

## F-10 [BAJO] Sin manejo granular de 403/404/422/500 en la mayoría de hooks

- **Archivo**: todos los hooks en `application/hooks/`, salvo el manejo de 409 en `useUsuarios.ts`.
- **Descripción**: Ver detalle en `frontend_api_map.md`. La mayoría de hooks no distinguen entre tipos de error HTTP más allá del interceptor global de 401.
- **Impacto**: Mensajes de error genéricos para el usuario en escenarios de fallo de validación (422) o error de servidor (500).
- **Recomendación**: Uniformizar manejo de errores con un wrapper común que traduzca códigos HTTP a mensajes i18n específicos.
- **Estado epistemológico**: Verificado en código.

## F-11 [BAJO] Bundle de ECharts pesado, sin code-splitting por ruta

- **Archivo**: `vite.config.ts` (manualChunks), resultado de build: `echarts` chunk 568KB (189KB gzip).
- **Descripción**: ECharts se carga en el bundle inicial aunque solo `DashboardPage` lo usa; no hay `React.lazy()` para diferir la carga de páginas que no lo necesitan.
- **Impacto**: Afecta tiempo de carga inicial (relevante para RNF-10, ≤3s, no medido directamente pero el peso del bundle es un factor).
- **Recomendación**: Code-splitting por ruta con `React.lazy` + `Suspense`.
- **Estado epistemológico**: Verificado en configuración/build real (no inferido).

## Hallazgos positivos (para contexto, no defectos)

- JWT en memoria: implementación real y correcta, no solo declarada.
- RBAC de rutas: bloqueo real de renderizado, no solo ocultamiento visual.
- SSE con ticket de autenticación: diseño sofisticado y correcto.
- Cero `any`, cero código muerto detectado, cero `dangerouslySetInnerHTML`, cero secretos expuestos.
- Paridad i18n ES/EN 100% (186/186 claves).
- Build y typecheck limpios (exit code 0 en ambos).
- Ninguna clasificación de riesgo calculada falsamente en el cliente — el frontend siempre consume `nivel_riesgo` del backend, nunca lo inventa (ni siquiera en el modo demo, donde el cálculo está claramente aislado y documentado como réplica simplificada para fines de simulación).
- No se repite el overclaim "Exactitud >96%" de la presentación PPTX en ningún lugar del código.
