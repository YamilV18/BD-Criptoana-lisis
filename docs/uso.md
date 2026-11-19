# Uso del sistema

El flujo se ejecuta **en orden**: U3 depende de los artefactos de U1 y U2.

| Orden | Notebook | Qué hace | Salida |
|---|---|---|---|
| 1 | `U1_Integrante{N}_*.ipynb` | Batch: calidad, Silver/Gold, entrenamiento y modelo ganador | Parquet Silver/Gold, modelo guardado y métricas JSON |
| 2 | `U2_Integrante{N}_*.ipynb` | Streaming desde Kafka, ventanas, serie de tiempo y métricas | Parquet de eventos, modelo de serie y tablero U2 |
| 3 | `U3_Integrante{N}_*.ipynb` | Compuertas previas, ejecución integrada, Prometheus y manifiesto | `manifiesto_<runId>.json` y métricas `cripto_int_*` |

## Pasos

1. Abrir Jupyter y la carpeta `notebooks/`.
2. Ejecutar el notebook de U1 de la dimensión (todas las celdas).
3. Con el puente publicando en Kafka (`docker compose ps`), ejecutar el notebook de U2.
4. Ejecutar el notebook de U3. Las **nueve compuertas previas** deben dar OK; si una falla, el mensaje nombra qué artefacto o servicio falta.
5. Abrir Grafana (dashboard *CriptoAnalytics U3*) y confirmar los paneles de latencia, antigüedad e indicador U1 frente a U2.
6. Revisar el manifiesto de la corrida en `artifacts/integranteN/u3/`.

!!! tip "Dos corridas"
    Ejecutar U3 dos veces permite la prueba de no regresión: las compuertas deben seguir en 9/9 y las filas del stream no deben ser cero. Que cambien las filas es normal, la fuente es en vivo.
