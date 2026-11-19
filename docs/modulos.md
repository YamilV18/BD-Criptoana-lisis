# Módulos y funcionalidades

| Módulo | Funcionalidad | Relación con el problema |
|---|---|---|
| Ingesta | WebSocket → Kafka con key = `symbol` | Alimenta el tiempo real |
| Batch (U1) | Calidad, capas Medallion, modelos por dimensión | Mide qué tan predecible es cada variable |
| Streaming (U2) | Ventanas, watermark, deduplicación, checkpoint, triggers | Convierte eventos en barras de 10 s |
| Serie de tiempo (U2) | Barras de 10 s, 8 639 ejemplos supervisados, comparación con persistencia | Responde «qué pasará en el próximo intervalo» |
| Observabilidad (U2/U3) | Métricas `cripto_*`, alertas y tablero | Muestra si el sistema merece confianza |
| Integración (U3) | 9 compuertas, manifiesto, dos corridas, hardening | Hace el sistema comprobable |

## Modelos ganadores de U1

| Integrante | Modelo | Métrica |
|---|---|---|
| 1 · Monto | LinearRegression (reg 0,0) | RMSE 38,5 · R² 0,9998 |
| 2 · Ballenas | RandomForest 15 ponderado | AUC-PR 0,9946 · recall 0,9940 |
| 3 · Rol | GBT (10 iteraciones, prof. 5) | AUC-ROC 0,5548 |
| 4 · Cantidad | LinearRegression (reg 0,0) | MAE 0,000073 · R² 0,9998 |

!!! note "Sobre el R² de monto y cantidad"
    `quote_qty = price × qty`: los modelos de los integrantes 1 y 4 recuperan una identidad algebraica, no una capacidad predictiva general.
