# 08 — AIV-05 frontend

Cambios mínimos en `frontend/src`:

- `LecturaTermica` tipa origen, estado de inferencia, motivo y estado de sensores.
- SSE mantiene mismo contrato `LecturaTermica`; no duplica eventos en cliente.
- Dashboard muestra estado no completado junto con origen. Por tanto no muestra
  `0 %` ni etiqueta "IA normal" cuando hubo salvaguarda, dato insuficiente,
  fallo de sensor o modelo no disponible.
- Confianza y versión sólo aparecen cuando hay inferencia completada y confianza
  distinta de `null`.

Evidencia textual: `npm run typecheck` y `npm run build` completaron sin error.
Vite produjo bundle; única advertencia: chunk ECharts de 568.05 kB minificado.

Estado AIV-05: **RESUELTO**.
