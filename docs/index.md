# CriptoAnalytics · Proyecto Sello

**Curso:** Big Data · UPeU Juliaca · Ciclo IX · Docente: Abel Ángel Sullón Macalupu  
**Repositorio:** [USUARIO/REPOSITORIO](https://github.com/USUARIO/REPOSITORIO) <!-- COMPLETAR -->

## Problema

Un mercado cripto produce decenas de operaciones por segundo. Para decidir hay que combinar el histórico con lo que ocurre ahora y saber **cuánto confiar en cada modelo**.

> ¿Cómo se comporta el volumen operado en BTC/USDT y qué tan predecible es el monto de una operación?

## Solución

Una arquitectura **Lambda** sobre operaciones BTC/USDT de Binance:

- **Batch (U1):** PySpark sobre 5 485 489 operaciones del 2026-01-05, capas Bronze/Silver/Gold en Parquet y un modelo ganador por integrante.
- **Streaming (U2):** WebSocket de Binance → Kafka (`binance-btc-trades`) → Spark Structured Streaming (ventanas de 10 s, watermark de 5 s, deduplicación, checkpoint) con inferencia del modelo de U1.
- **Observabilidad e integración (U3):** Pushgateway, Prometheus y Grafana; compuertas previas, manifiesto de ejecución, hardening y prueba de no regresión.

## Integrantes y dimensiones

| # | Integrante | Dimensión |
|---|---|---|
| 1 | Yamil Valente Quispe Cuellar | Monto operado (regresión) |
| 2 | Condori Quispe Brayan Raúl | Ballenas > 50 000 USDT (clasificación) |
| 3 | Juli Machaca Renzo Rodi | Rol del comprador (clasificación) |
| 4 | Coaquira Puma Raquel Kantu | Cantidad operada (regresión) |

## Cómo leer esta documentación

1. [Instalación y ejecución](instalacion.md) para levantar el sistema.
2. [Uso del sistema](uso.md) para correr U1, U2 y U3.
3. [Pruebas y evidencias](evidencias.md) para verificar los resultados.
