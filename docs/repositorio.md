# Repositorio y estándares

## Estructura

```text
.
├── compose.yml            # orquestación del sistema
├── .env.example           # variables (el .env no se versiona)
├── bridge/                # WebSocket de Binance → Kafka
├── consumer/              # validación de mensajes
├── pyspark/               # imagen de Spark y Jupyter
├── obs/                   # Prometheus y aprovisionamiento de Grafana
├── notebooks/             # U1, U2 y U3 por integrante
├── docs/ + mkdocs.yml     # esta documentación
└── .github/workflows/     # publicación automática de la documentación
```

## Topics del repositorio

`big-data` · `apache-spark` · `apache-kafka` · `structured-streaming` · `pyspark` · `docker-compose` · `grafana` · `prometheus` · `lambda-architecture` · `binance` · `upeu` <!-- COMPLETAR/ajustar -->

## Criterios de reproducibilidad

- Un solo comando levanta y espera: `docker compose up -d --wait`.
- Configuración por variables; sin secretos en el repositorio.
- Compuertas previas que nombran lo que falta y manifiesto por corrida.
- Documentación publicada con MkDocs desde `main`.
