# Bitácora de integración y revisión técnica - Integrante 4

**Proyecto Sello · BTC/USDT (Binance) · CriptoAnalytics** - Unidad III (S13 integración y DataOps, S14 revisión técnica y hardening)

| | |
|---|---|
| **Dimensión** | Cantidad operada (`qty`) · Regresión |
| **Pregunta de mi dimensión** | ¿Cuál es la relación entre el volumen en BTC (qty) y el valor nominal en USDT (quote_qty), ajustada por el comportamiento del creador de mercado? |
| **Modelo de U1 desplegado** | `LinearRegression_reg0.0` (ganador de U1) |
| **Indicador** | R²: U1 (offline) 0.9998 · U2 (eventos en vivo) 0.9656 |
| **Corridas comparadas** | `20261113T192321Z` → `20261113T194933Z` (manifiestos en `artifacts/integrante4/u3/`) |

> **Nota sobre las cifras:** la fuente de streaming es el WebSocket de Binance en vivo, por lo que los valores de U2 y de U3 cambian de una ejecución a otra. Las cifras de U2 de esta bitácora vienen de la ejecución del notebook previa a la exposición; las de U3, de las dos corridas indicadas. Cuando un valor se explica con una causa que no se verificó, se marca como **hipótesis**.

## 1. Bitácora de integración (S13)

Hallazgos propios de la dimensión del integrante 4:

| # | Hallazgo | Causa | Corrección | Prueba de corrección |
|---|---|---|---|---|
| 1 | El R² del modelo baja al pasar a los eventos en vivo. | Offline 0.9998; en vivo 0.9656 con 1 231 eventos (RMSE 0.0048, MAE 0.00128 BTC) y 0.932 en la ventana de streaming. La relación `quote_qty = price × qty` se conserva; **hipótesis:** una pocas cantidades grandes dominan el error en una ventana corta. | Se registran los dos valores en la matriz de trazabilidad y se muestran juntos en el tablero. | Sección 3 del notebook U3: `r2: U1 = 0.99984 | U2 = 0.96559`; métricas confirmadas en Prometheus. |
| 2 | El resultado de la serie de tiempo depende del tamaño de la muestra. | Histórico (U2): Random Forest, RMSE 0.005222 frente a 0.006900 (24.3 % menos). En vivo (U2, 138 barras): MAE 0.00330 frente a 0.00347 (el modelo es mejor por 5 %). En U3, con 7 a 8 barras: peor en las dos corridas (0.00293 frente a 0.00225, y 0.00254 frente a 0.00161). | Con 7 a 8 barras de 10 s no se puede concluir; se reporta el tamaño de la muestra junto a cada MAE y se da más peso a la medición de U2. | Métrica `cripto_int_stream_barras_cerradas` (7 y 8) en el tablero, junto al panel de MAE. |
| 3 | El notebook de U2 de este integrante se ejecutó en otra fecha que los demás. | Las marcas de tiempo de las barras son del 2026-10-30; las de los otros tres integrantes, del 2026-11-06. El precio y el volumen del flujo en vivo no son comparables entre fechas. | Antes de la sustentación conviene re-ejecutar U2 (ver nota de la bitácora) o declarar la fecha en el informe. | Salida final del notebook U2 (inferencia en vivo): `bar_start 2026-10-30 21:35:50 … 21:37:00`. |

## 2. Bitácora de la revisión técnica y de la integración común (S13 + S14)

Hallazgos que comparten los cuatro integrantes (el sistema es uno solo):

| # | Hallazgo | Causa | Corrección | Prueba de corrección |
|---|---|---|---|---|
| 1 | Ningún servicio declaraba si estaba listo; las dependencias esperaban solo a que arrancara el contenedor. | El `compose.yml` se escribió con la configuración mínima de clase, sin `healthcheck` en `kafka`, `prometheus`, `pushgateway` ni `grafana`. | `healthcheck` en esos cuatro servicios (y en `pyspark-jupyter`) y `depends_on` con `condition: service_healthy` (Kafka → resto; Pushgateway → Prometheus → Grafana). | Revisión de hardening (sección 6 del notebook U3): «Salud (healthcheck)» pasó de **Falla** a **Cumple** y el total de **3/5 a 5/5**. |
| 2 | Usuario y contraseña de Grafana y token de Jupyter escritos en el `compose.yml` versionado. | Valores fijos en el archivo (`JUPYTER_TOKEN`, `GF_SECURITY_ADMIN_USER`, `GF_SECURITY_ADMIN_PASSWORD`). | Variables `${VAR:-valor}` con los valores reales en un `.env` fuera del repositorio y un `.env.example` versionado. | «Secretos fuera del archivo» pasó de **Falla** a **Cumple** (misma sección). *Límite:* los valores por defecto siguen siendo de desarrollo; en un despliegue real se cambian en el `.env`. |
| 3 | Una copia clonada del repositorio no puede ejecutar la integración: faltan los artefactos de U1 y U2. | `artifacts/` no se versiona, así que no viaja con `git clone`. | Nueve compuertas previas que nombran el artefacto o servicio que falta, y una compuerta que se hace fallar a propósito (puerto equivocado). | Checklist U3: «Al menos 4 compuertas previas, todas OK» y «Una compuerta demostrada FALLANDO (detuvo la ejecución)» en `[OK]` en las dos corridas. |
| 4 | Pushgateway conserva el último valor publicado: un panel puede parecer vivo cuando el pipeline lleva minutos detenido. | Pushgateway no expira las métricas; solo guarda la última recibida. | Métrica `cripto_int_ultima_ejecucion_timestamp` y panel «Antigüedad de la última ejecución» (`time() - cripto_int_ultima_ejecucion_timestamp`). | En el tablero el panel marca en rojo entre 3.4 y 11.1 min tras la corrida, y se reinicia al volver a ejecutar (capturas de la segunda corrida). |
| 5 | La confirmación en Prometheus podía dar por buena una corrida anterior. | Comprobar que la métrica «tenga algún valor» no distingue el valor nuevo del viejo. | Cada métrica devuelta por Prometheus se compara con el valor recién publicado en la misma corrida. | La sección 5 del notebook U3 imprime `métrica: valor (publicado: valor)` para las 16 a 23 métricas `cripto_int_*`; coinciden en las dos corridas. |
| 6 | El promedio de latencia no describe a los eventos: oculta la cola. | El promedio mezcla la mayoría rápida con unos pocos eventos lentos; la alerta de U2 (`cripto_spark_latencia_prom_ms`) se basaba en el promedio. | En U3 se publican p50, p95 y p99. La compuerta de plausibilidad detiene con el **p50** (umbral 30 s); el **p95** solo advierte (umbral 60 s). | Fracción de eventos sobre el umbral = 0 y 9/9 compuertas superadas; los tres percentiles se ven juntos en el primer panel del tablero. |
| 7 | En U2 se escribió una alerta sobre `avgLatencyMs`, una métrica que no existe en Prometheus. | Se usó el nombre de una columna de Spark como si fuera una métrica publicada. | Alerta reescrita con la métrica real `cripto_spark_latencia_prom_ms` y una alerta «pipeline detenido» con `time() - push_time_seconds`. | Tabla de alertas de la sección S9 de U2, con la expresión PromQL de cada alerta. |
| 8 | Los resultados de U2 cambian de una ejecución a otra. | La fuente es el WebSocket de Binance en vivo; además, al re-ejecutar U2 para la exposición se reinicia `stream/eventos_obs`, que es el Parquet que U3 lee como batch. | Cada corrida de U3 escribe su manifiesto con `runId` y no sobrescribe los anteriores; la comparación indica qué diferencias son esperables. | Dos manifiestos por integrante en `artifacts/integranteN/u3/` y checklist «Dos corridas comparadas» en `[OK]`. |

## 3. Prueba de no regresión: antes y después del cambio en el `compose.yml`

Se repitió la ejecución integrada después de aplicar los `healthcheck`, `depends_on` con `service_healthy`, variables de entorno y puertos en `127.0.0.1`.

| Medida | Corrida 1 (`20261113T192321Z`) | Corrida 2 (`20261113T194933Z`) | ¿Esperable? |
|---|---|---|---|
| Hardening (puntos que cumplen) | 3/5 | 5/5 | Esperable: es lo que se corrigió en el `compose.yml`. |
| Compuertas previas superadas | 9/9 | 9/9 | Sin cambio. |
| Eventos en el batch (Parquet de U2) | 1 231 | 1 231 | Sin cambio. |
| Filas procesadas por el stream | 1 644 | 2 387 | Esperable: fuente en vivo. |
| R² de la ventana de streaming | 0.9328 | 0.9323 | Estable. |
| MAE serie de tiempo: modelo / persistencia | 0.00293 / 0.00225 | 0.00254 / 0.00161 | El modelo queda peor en las dos, con solo 7 a 8 barras. |
| Latencia p50 / p95 / p99 (ms) | 3 045 / 5 125 / 9 697 | 3 045 / 5 125 / 9 697 | Sin cambio: mismo Parquet batch. |

**Regresión sería:** una compuerta que antes pasaba y ahora falla, o un stream que procesa cero filas. No ocurrió ninguna: las 9 compuertas pasan en las dos corridas y el checklist de U3 queda completo.

## 4. Riesgos aceptados

| Riesgo | Justificación | Qué cambiaría en producción |
|---|---|---|
| Puertos publicados en la máquina anfitriona (Kafka, Prometheus, Pushgateway, Grafana, Jupyter, exportador). | Entorno de clase en una sola máquina; se publican solo en `127.0.0.1`. | Quitar la publicación de puertos y dejar un proxy inverso con TLS y autenticación. |
| Kafka en `PLAINTEXT` y sin autenticación. | Broker único, red interna de Docker, sin datos personales (solo precios públicos de Binance). | SASL/SCRAM y TLS entre cliente y broker, y ACL por topic. |
| Un solo broker y una sola partición (`key = symbol`, un único par). | Preserva el orden de un único par y simplifica la demo; el ritmo medido (≈ 2.4–3 eventos/s) no la satura. | Más particiones y réplicas; varias instancias del consumidor. |
| Valores por defecto de credenciales (`admin`/`admin`) si falta el `.env`. | Facilita que otra persona levante el sistema sin configurar nada. | Sin valor por defecto: que el arranque falle si falta la variable. |

## 5. Evidencia de aprendizaje

- Nueve compuertas previas y una compuerta demostrada fallando (sección 1 del notebook U3).
- Calidad del batch con p50 / p95 / p99 y compuerta de plausibilidad sobre el p50 (sección 2).
- Indicador U1 (offline) frente a U2 (en vivo) con el mismo modelo (sección 3).
- Stream de ventana fija con el modelo de U1 y la serie de tiempo; error en producción frente a la persistencia (sección 4).
- Métricas `cripto_int_*` publicadas y confirmadas en Prometheus con el valor recién publicado (sección 5); tablero «CriptoAnalytics - Integración U3», instancia `integrante4`.
- Revisión de hardening del `compose.yml`: **3/5 → 5/5** (sección 6). El punto «Superficie expuesta (puertos)» queda en «Revisar» y se registra como riesgo aceptado (sección 4).
- Matriz de trazabilidad, manifiestos de dos corridas y comparación (sección 7): `matriz_trazabilidad_integrante4.md`, `manifiesto_20261113T192321Z.json`, `manifiesto_20261113T194933Z.json`.

## 6. Pendientes antes de la sustentación

- Re-ejecutar el notebook de U2 de este integrante (la evidencia actual es del 2026-10-30) o declarar la fecha en el informe.
- Ejecutar U3 justo antes de la demostración para que el panel de antigüedad no aparezca en rojo.
