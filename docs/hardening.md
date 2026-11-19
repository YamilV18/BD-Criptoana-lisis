# Hardening y riesgos aceptados

## Lista de comprobación (5 puntos)

Pasó de **3/5 a 5/5** tras corregir `compose.yml`:

- [x] Imágenes con versión exacta
- [x] Salud (`healthcheck`) en Kafka, Prometheus, Pushgateway, Grafana y Jupyter
- [x] `depends_on` con `service_healthy`
- [x] Secretos fuera del archivo (`${VAR:-valor}` y `.env`)
- [x] Puertos solo en `127.0.0.1` (revisión marcada «Revisar»: riesgo aceptado)

## Prueba de no regresión

Se repitió la ejecución integrada y se comparó con la anterior: las compuertas siguen en 9/9 y los eventos del batch no cambian. Que cambien las filas del stream no es regresión; que una compuerta falle o haya cero filas sí lo sería.

## Riesgos aceptados

| Riesgo | Por qué se acepta | En producción |
|---|---|---|
| Puertos publicados en el anfitrión | Entorno de clase, solo en 127.0.0.1 | Proxy inverso con TLS y autenticación |
| Kafka en PLAINTEXT, sin autenticación | Broker único, red interna, precios públicos | SASL/SCRAM, TLS y ACL por topic |
| 1 broker y 1 partición | Orden de un único par; ≈ 3 eventos/s no la satura | Más particiones y réplicas |
| Credenciales por defecto si falta `.env` | Permite levantar sin configurar | Sin valor por defecto: fallar si falta |
