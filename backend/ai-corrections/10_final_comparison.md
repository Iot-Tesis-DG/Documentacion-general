# 10 — Comparación final de modelo

| Modelo | Dataset hash | F1 macro | F1 weighted | Accuracy | Recall crítica | CV media ± std |
|---|---|---:|---:|---:|---:|---:|
| RF v1 | histórico | 0.9535 | 0.9659 | 0.9656 | 0.9730 | 0.9615 ± 0.0031 |
| RF v2 | histórico | 0.9613 | 0.9748 | 0.9744 | 0.9762 | 0.9722 ± 0.0039 |
| RF v3 final | `36fef345...:c94fc260...` | 0.9572 | 0.9671 | 0.9668 | 0.9778 | 0.9727 ± 0.0023 |

Resultado: v3 cumple F1 weighted >= 0.85, usa partición por escenario sin
contaminación, tiene hash externo válido y conserva interpretación Escenario B.
Queda oficial. No se forzó igualdad con v2.
