# Instalación y ejecución

## Requisitos

| Requisito | Detalle |
|---|---|
| Docker y Docker Compose v2 | Compatible con `docker compose up -d --wait` |
| Memoria | Al menos 8 GB libres para Spark, Kafka y Grafana |
| Red | Salida a `stream.binance.com:9443` (WebSocket) |
| Archivo de datos | `BTCUSDT-trades-2026-01-05.csv` (5 485 489 filas) en la carpeta de datos que monta el `compose.yml` |

## Pasos

```bash
# 1. Clonar
git clone https://github.com/USUARIO/REPOSITORIO.git && cd REPOSITORIO

# 2. Configuración por variables (no se versiona el .env)
cp .env.example .env        # cambiar contraseñas si no es un entorno de clase

# 3. Levantar y esperar a que los servicios estén healthy
docker compose up -d --wait

# 4. Comprobar
docker compose ps           # todos en (healthy)
```

## Puertos (solo en 127.0.0.1)

| Servicio | URL |
|---|---|
| Jupyter (PySpark) | <http://127.0.0.1:8888> (token en `JUPYTER_TOKEN`) |
| Kafka UI | <http://127.0.0.1:48085> |
| Prometheus | <http://127.0.0.1:49090> |
| Pushgateway | <http://127.0.0.1:49091> |
| Grafana | <http://127.0.0.1:43000> (usuario y contraseña del `.env`) |
| Kafka (externo) | `127.0.0.1:49092` |

!!! warning "Valores por defecto"
    Si falta el `.env`, se usan `admin/admin` y el token `criptoanalytics`. Es un riesgo aceptado para clase; ver [Hardening y riesgos](hardening.md).

## Detener

```bash
docker compose down        # conserva volúmenes
docker compose down -v     # borra también los datos
```
