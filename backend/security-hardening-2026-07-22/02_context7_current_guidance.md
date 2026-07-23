# Referencia de versiones actuales

Se consultó Context7 para CPython y React antes de endurecer el proyecto. El runtime de backend es Python 3.12.13; se mantuvo la compatibilidad del código con Python 3.12 y se aplicaron prácticas actuales: secretos por entorno, operaciones de archivo/deserialización bajo confianza explícita y validación de entradas.

En frontend se verificó React 19 y se actualizó el lockfile a React y React DOM 19.2.8. React escapa texto interpolado; por ello los estados IA/SSE renderizados como JSX no constituyen un sumidero XSS. Se mantiene la prohibición de `dangerouslySetInnerHTML`, `innerHTML`, `eval` y HTML no confiable.

También se actualizó ECharts a 6.1.0 para corregir GHSA-fgmj-fm8m-jvvx. `npm audit --omit=dev` terminó con 0 vulnerabilidades.
