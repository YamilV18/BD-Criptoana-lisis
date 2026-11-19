# Datos

| Fuente | Descripción |
|---|---|
| Batch | `BTCUSDT-trades-2026-01-05.csv`, 5 485 489 operaciones de Binance |
| Streaming | WebSocket `wss://stream.binance.com:9443/ws/btcusdt@trade` → topic `binance-btc-trades` (≈ 3 eventos/s) |

## Calidad (U1)

0 nulos, 0 duplicados, 0 valores ≤ 0.

## Capas

| Capa | Contenido |
|---|---|
| Bronze | CSV con esquema explícito |
| Silver | 5 485 489 filas limpias |
| Gold ML | Particionado por `is_buyer_maker` (false 2 978 348 / true 2 507 141) |
| Gold KPI | 1 440 minutos |
| Eventos U2 | Parquet de eventos en vivo (`stream/eventos_obs`) |
| Barras | 8 640 barras históricas de 10 s, 0 huecos |

## Esquema del mensaje en vivo

Campos del evento `trade` de Binance: precio, cantidad, `tradeId`, tiempo y `isBuyerMaker` (se normaliza a `is_buyer_maker`); `quote_qty` = precio × cantidad.
