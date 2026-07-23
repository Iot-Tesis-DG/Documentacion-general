# Arquitectura real del frontend

## Stack confirmado (vs benchmarking/TI)

| Elemento | Declarado en TI/Anexo2 | Real en código | Coincidencia |
|---|---|---|---|
| React | React 19 | `react@^19.0.0`, `react-dom@^19.0.0` | ✅ |
| Vite | Vite | `vite@^6.1.0` | ✅ (TI no especifica versión exacta) |
| TypeScript | TypeScript | `typescript@^5.7.3`, modo `strict: true` | ✅ y mejor de lo mínimo esperado |
| shadcn/ui | shadcn/ui | Patrón manual (Radix+cva+tailwind-merge), sin `components.json`, solo 6 componentes | ⚠️ Parcial |
| Apache ECharts v5 | Apache ECharts v5 | `echarts@^5.6.0`, wrapper propio (no `echarts-for-react`, decisión documentada en comentario) | ✅ |
| SSE | Server-Sent Events | `EventSource` real con ticket de autenticación efímero | ✅ |
| Gestión de estado | (no especificado en TI, pero backlog HU-40 sugiere Zustand/Context) | `zustand@^5.0.3` | ✅ |
| Cliente HTTP | (no especificado) | `axios@^1.7.9` con interceptores | ✅ |
| Routing | (no especificado) | `react-router@^7.1.5` | ✅ |
| i18n | No mencionado en TI | `i18next`+`react-i18next`, ES/EN completo | Extra no documentado (positivo) |

## Punto de entrada y proveedores globales

`index.html` → `main.tsx` (StrictMode + i18n init) → `App.tsx` (BrowserRouter + Routes). No hay Context Providers adicionales aparte del store Zustand (que no usa Context, es un store externo por hook). No hay ErrorBoundary global — un error de renderizado no capturado colapsaría toda la app (hallazgo, ver findings).

## Árbol arquitectónico real

```
src/
├── domain/                    ← capa de dominio (DDD), sin dependencias externas
│   ├── entities/               (LecturaTermica, AlertaTermica, RegistroTrazabilidad, Usuario — solo interfaces TS)
│   └── value-objects/          (Rol con tienePermiso(), NivelRiesgo con estaEnRango())
│
├── application/                ← casos de uso / hooks, dependen de domain + infrastructure
│   ├── hooks/                  (7 hooks, uno por caso de uso: alertas, auditoría, historial, monitoreo, reportes, trazabilidad, usuarios)
│   └── stores/authStore.ts     (Zustand — único store global, solo sesión)
│
├── infrastructure/              ← adaptadores a servicios externos
│   ├── api/apiClient.ts         (instancia Axios única, interceptors JWT + 401)
│   ├── auth/                    (authService: login+decode JWT; avisoSesion: flags de UX en sessionStorage)
│   ├── charts/EChartWrapper.tsx (wrapper propio sobre echarts/core)
│   ├── demo/                    (modoDemo flag + demoAdapter Axios + datosDemo generador determinista)
│   ├── i18n/                    (i18next config + locales es/en)
│   └── sse/sseClient.ts         (EventSource real + simulador de demo)
│
├── presentation/                ← UI, depende de application+domain, nunca de infrastructure directamente (excepto MODO_DEMO flag importado en 2 puntos de UI para mostrar badge/panel demo)
│   ├── components/              (RouteGuards, RiskBadge, PageHeader, LanguageSwitcher)
│   ├── components/ui/           (6 primitivos estilo shadcn: badge, button, card, dialog, input, table)
│   ├── layouts/AppLayout.tsx    (sidebar+drawer responsive, nav filtrado por rol)
│   └── pages/                   (9 páginas, una por ruta)
│
├── lib/utils.ts                 (helper `cn` de clases Tailwind)
├── App.tsx                      (router + listener de sesión expirada)
└── main.tsx                     (bootstrap)
```

Esta es una arquitectura DDD limpia y bien respetada: la dirección de dependencias es correcta (`domain` no importa nada; `application` importa `domain`+`infrastructure`; `presentation` importa `application`+`domain`, e importa `infrastructure` únicamente para el flag `MODO_DEMO` en 2 puntos de UI, que es una filtración menor pero justificable — ver hallazgos). Coincide con el patrón de capas que TI describe para el **backend** (Interface/Application/Domain/Infrastructure); el frontend replica el mismo vocabulario de capas de forma consistente, aunque TI no exige esto explícitamente para el frontend (es una decisión de calidad adicional del equipo, no un requisito).

## Comunicación con backend

- **HTTP**: Axios (`apiClient`), baseURL desde `VITE_API_BASE_URL`, header `Authorization: Bearer <token>` inyectado por interceptor si hay token en memoria.
- **Tiempo real**: SSE vía `EventSource` nativo, con esquema de "ticket efímero" (POST `/api/auth/sse-ticket` → query param `?ticket=`) para sortear la limitación de `EventSource` de no poder enviar headers `Authorization`. Diseño correcto y más seguro que pasar el JWT completo en la URL.
- **Persistencia local**: solo `sessionStorage` (flags de UX, nunca el JWT) y, en modo demo, `sessionStorage` para el token falso + `localStorage` para el estado del checklist BPA (éste último es el único uso de `localStorage` real en toda la app, y es fuera de alcance de RF-14/hash — ver hallazgos).

## Modo demo — arquitectura de sustitución

`MODO_DEMO` (booleano desde `VITE_MODO_DEMO`) activa, en tiempo de build:
1. `apiClient` usa un `adapter` Axios custom (`demoAdapter`) que intercepta CADA request y responde desde datos en memoria (`estadoDemo`), sin tocar la red.
2. `sseClient` usa `simularStream()` en vez de `EventSource` real, emitiendo lecturas sintéticas cada 8s.
3. El resto de la aplicación (hooks, páginas, componentes) es **exactamente el mismo código** — no hay ramas condicionales en la lógica de negocio, solo en la capa de infraestructura. Esto es una arquitectura de sustitución (adapter pattern) correctamente aislada, no un "modo simulado" contaminando el resto del código.

Esto confirma lo indicado en memoria de sesiones previas: Vercel despliega vía `build:demo`, por lo que **la URL pública en producción NUNCA se conecta al backend real** — todo lo que un visitante externo ve es 100% sintético, aunque construido con honestidad técnica (comentarios explícitos, badge visual "DEMO" en la UI cuando `MODO_DEMO=true`).
