# Resultados de ejecución (validaciones seguras)

Entorno: `node_modules` ya instalado previamente (no se ejecutó `npm install`, no se alteró el lockfile).

| Comando | Descripción | Exit code | Resultado |
|---|---|---|---|
| `npm run typecheck` (`tsc -b --noEmit`) | Verificación de tipos estricta | **0** | Sin errores. Confirma `strict: true`, `noUnusedLocals`, `noUnusedParameters` pasan limpio en todo el proyecto. |
| `npm run build` (`tsc -b && vite build`) | Build de producción real (sin demo) | **0** | Compila en 7.13s. 2318 módulos transformados. Advertencia (no error): chunk de `echarts` (568KB) supera 500KB — ver hallazgo F-11. |
| `npm run build:demo` (`tsc -b && vite build --mode demo`) | Build de demostración (el que despliega Vercel) | **0** | Compila en 1.94s (caché de tsc). Mismo tamaño de bundle esperado. |

No se ejecutó `npm run lint` (script inexistente, no hay ESLint configurado — ver hallazgo F-07) ni `npm test` (script inexistente, cero pruebas — ver hallazgo F-06).

No se instalaron dependencias adicionales. No se modificó `package-lock.json` ni ningún archivo fuente.

## Interpretación

El frontend **compila y tipa sin errores en ambos modos** (real y demo). Esto confirma que el código es sintácticamente correcto y que los tipos declarados son internamente consistentes. **Esto NO confirma que el frontend funcione correctamente contra un backend real** — el build no ejecuta la aplicación ni prueba ninguna petición HTTP/SSE real; solo demuestra ausencia de errores de compilación/tipos. La funcionalidad real end-to-end contra el backend queda pendiente de Fase 3 (backend) y, eventualmente, de una prueba manual con ambos servicios corriendo.
