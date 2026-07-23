# Inventario recursivo — frontend/

Fuente: `find . -type f` excluyendo generados. Total: **67 archivos** (incluyendo `node_modules`/`dist` detectados y excluidos de lectura).

## Exclusiones (detectadas, no leídas línea por línea)

| Carpeta/patrón | Motivo |
|---|---|
| `node_modules/` | Dependencias de terceros, no código propio |
| `dist/` | Artefacto de build generado |
| `.vercel/` (excepto `project.json`/`README.txt`, sí leídos) | Metadata de despliegue Vercel autogenerada |
| `.DS_Store` (×2: raíz y `src/`) | Metadata macOS sin contenido |

## Archivos propios inventariados (55 archivos de código + configuración)

```
frontend/
├── .env.demo                          [config demo build]
├── .env.example                       [plantilla variables de entorno]
├── .gitignore
├── .vercel/project.json               [leído: config proyecto Vercel]
├── .vercel/README.txt                 [leído]
├── index.html
├── package.json                       [leído completo]
├── package-lock.json                  [presente, npm — no leído línea por línea, solo confirmado su existencia y gestor]
├── tsconfig.json / tsconfig.app.json / tsconfig.node.json  [leídos completos]
├── vercel.json                        [leído completo — build:demo + CSP headers]
├── vite.config.ts                     [leído completo]
├── public/favicon.svg
└── src/
    ├── App.tsx                        [leído — router raíz]
    ├── main.tsx                       [leído — entry point]
    ├── index.css                      [no leído línea por línea — solo tema/tokens Tailwind v4, no lógica]
    ├── vite-env.d.ts
    ├── application/
    │   ├── hooks/ (7 archivos: useAlertas, useAuditoria, useHistorial, useMonitoreoTermico, useReportesBPA, useTrazabilidad, useUsuarios) [TODOS leídos completos]
    │   └── stores/authStore.ts         [leído completo]
    ├── domain/
    │   ├── entities/ (4: AlertaTermica, LecturaTermica, RegistroTrazabilidad, Usuario) [TODOS leídos]
    │   └── value-objects/ (2: NivelRiesgo, Rol) [TODOS leídos]
    ├── infrastructure/
    │   ├── api/apiClient.ts            [leído completo]
    │   ├── auth/ (authService.ts, avisoSesion.ts) [leídos completos]
    │   ├── charts/EChartWrapper.tsx    [leído completo]
    │   ├── demo/ (modoDemo.ts, demoAdapter.ts, datosDemo.ts) [TODOS leídos completos — capa de simulación para build Vercel]
    │   ├── i18n/ (index.ts + locales/es.json + locales/en.json) [leídos completos, paridad de claves verificada 186/186]
    │   └── sse/sseClient.ts            [leído completo]
    ├── lib/utils.ts                    [leído completo]
    └── presentation/
        ├── components/ (LanguageSwitcher, PageHeader, RiskBadge, RouteGuards) [TODOS leídos completos]
        ├── components/ui/ (badge, button, card, dialog, input, table — 6 componentes shadcn-style) [existencia confirmada, lectura parcial de badge/button/card/dialog/input; table no crítico]
        ├── layouts/AppLayout.tsx        [leído completo]
        └── pages/ (9 páginas: AlertasPage, AuditoriaPage, ChecklistBPAPage, DashboardPage, HistorialPage, LoginPage, ReportesPage, TrazabilidadPage, UsuariosPage) [TODAS leídas completas]
```

## Hallazgo de inventario: NO existe `components.json`

No hay archivo `components.json` (configuración estándar de la CLI de shadcn/ui). Los 6 componentes en `presentation/components/ui/` fueron creados manualmente replicando el patrón shadcn (Radix + `cva` + `clsx`/`tailwind-merge`), no instalados vía `npx shadcn add`. Esto es una discrepancia menor frente a la declaración de TI/benchmarking ("React + Vite + TypeScript + **shadcn/ui**") — el uso es real pero parcial/manual, no la biblioteca completa de componentes shadcn.

## Sin ESLint, sin Tailwind config dedicado, sin pruebas

- No existe `eslint.config.*` ni `.eslintrc*` en ninguna parte del proyecto — **no hay linting configurado**.
- No existe `tailwind.config.*` — Tailwind v4 se configura vía el plugin `@tailwindcss/vite` (enfoque "CSS-first" válido en v4, no es un defecto).
- No existe ningún archivo de prueba (`*.test.*`, `*.spec.*`), ni `vitest`/`jest`/`@testing-library` en `package.json`. **Cobertura de pruebas: 0%.**

## Variables de entorno

`.env.example`: `VITE_API_BASE_URL`, `VITE_SSE_URL` — sin secretos reales expuestos (solo URLs de plantilla `localhost`).
`.env.demo`: solo `VITE_MODO_DEMO=true` — sin secretos.
No se encontró ningún archivo `.env` con credenciales reales en el repositorio.
