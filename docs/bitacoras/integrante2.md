# Bitácora de integración y revisión técnica - Integrante 2

**Proyecto Sello · BTC/USDT (Binance) · CriptoAnalytics** - Unidad III (S13 integración y DataOps, S14 revisión técnica y hardening)

| | |
|---|---|
| **Dimensión** | Ballenas (`es_ballena`) · Clasificación |
| **Pregunta de mi dimensión** | ¿Se puede clasificar probabilísticamente una operación institucional/ballena (> 50 000 USDT) según la liquidez y la dirección del ejecutor? |
| **Modelo de U1 desplegado** | `RandomForest_15_ponderado` (ganador de U1) |
| **Indicador** | Recall de ballenas: U1 (offline) 0.9940 · U2 (eventos en vivo) n/d (sin ballenas en la ventana) |
| **Corridas comparadas** | `20261113T192059Z` → `20261113T195359Z` (manifiestos en `artifacts/integrante2/u3/`) |

> **Nota sobre las cifras:** la fuente de streaming es el WebSocket de Binance en vivo, por lo que los valores de U2 y de U3 cambian de una ejecución a otra. Las cifras de U2 de esta bitácora vienen de la ejecución del notebook previa a la exposición; las de U3, de las dos corridas indicadas. Cuando un valor se explica con una causa que no se verificó, se marca como **hipótesis**.

## 1. Bitácora de integración (S13)

Hallazgos propios de la dimensión del integrante 2:

| # | Hallazgo | Causa | Corrección | Prueba de corrección |
|---|---|---|---|---|
| 1 | El recall de ballenas en vivo no se pudo calcular (n/d). | Las ballenas son ≈ 0.05 % de las operaciones (referencia batch de U1). El Parquet usado como batch no contiene ballenas en ninguna de las dos corridas (0 positivos); en la ventana de streaming de la primera hubo 1 (acertada: TP 1, recall 1.0) y en la segunda ninguna, con 4 falsos positivos: la precisión es 0 y la exactitud 0.9973 frente a un baseline de 1.0. | El recall del batch (0.9940) queda como el valor estable y el vivo se presenta como «n/d» con su causa. La métrica `cripto_int_ventana_positivos` se publica para que se vea cuántos positivos hubo. | Sección 3 del notebook U3: indicador U2 = n/d; el tablero muestra solo la línea U1 en «Indicador U1 vs U2»; métricas `ventana_tp`, `ventana_fp` y `ventana_positivos` en Prometheus. |
| 2 | En la serie de tiempo la persistencia llega a error 0. | Sin ballenas en la ventana, el valor siguiente es igual al actual, y la persistencia no falla. En U2: histórico, RMSE 1.149 frente a 1.530 (24.9 % menos); en vivo (1 143 barras), MAE 0.394 frente a 0.355 (el modelo queda peor). | No se compara el modelo con un baseline perfecto sin explicarlo. Se declara que la serie `n_ballenas` es casi siempre 0 y que el baseline es difícil de superar. | Panel «Serie de tiempo: MAE modelo vs persistencia» (se ve el salto a 0 de la persistencia tras la segunda corrida). |
| 3 | El ritmo de ingesta medido en U2 (127 eventos/s) no es representativo. | En los otros integrantes se midió entre 2.4 y 3.0 eventos/s. **Causa no diagnosticada** (posible ráfaga de mercado o de reproducción del topic). La tabla de costos de U2 se calcula con ese ritmo. | Para el informe se usa el ritmo típico (≈ 3 eventos/s) y se aclara el valor atípico. | Sección S9 del notebook U2 (`Medido: 127.0 eventos/s`) frente a la de los demás integrantes. |
| 4 | Entre las dos corridas de U3 cambió el tamaño del batch (1 870 → 743 eventos). | Al re-ejecutar U2 para la exposición se reinició `stream/eventos_obs`, el Parquet que U3 lee. | Se registra el `runId` y se advierte que las dos corridas no usan el mismo batch; la comparación solo evalúa regresión (compuertas y hardening), no igualdad de valores. | Manifiestos 20261113T192059Z y 20261113T195359Z con `batch.eventos` distinto. |

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

| Medida | Corrida 1 (`20261113T192059Z`) | Corrida 2 (`20261113T195359Z`) | ¿Esperable? |
|---|---|---|---|
| Hardening (puntos que cumplen) | 3/5 | 5/5 | Esperable: es lo que se corrigió en el `compose.yml`. |
| Compuertas previas superadas | 9/9 | 9/9 | Sin cambio. |
| Eventos en el batch (Parquet de U2) | 1 870 | 743 | Esperable: se re-ejecutó U2 entre corridas y reinició el Parquet. |
| Ballenas en la ventana de streaming | 1 | 0 | Esperable: son eventos raros. |
| Recall / precisión en la ventana | 1.0 / 1.0 | n/d / 0.0 (4 falsos positivos) | Esperable por lo anterior; el recall no es calculable sin positivos. |
| MAE serie de tiempo: modelo / persistencia | 0.327 / 0.333 | 0.266 / 0.000 | Esperable: sin ballenas, predecir «igual que antes» acierta siempre. |
| Latencia p50 / p95 / p99 (ms) | 3 361 / 8 526 / 13 857 | 2 673 / 5 025 / 10 759 | Esperable: otra ventana en vivo. |

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
- Métricas `cripto_int_*` publicadas y confirmadas en Prometheus con el valor recién publicado (sección 5); tablero «CriptoAnalytics - Integración U3», instancia `integrante2`.
- Revisión de hardening del `compose.yml`: **3/5 → 5/5** (sección 6). El punto «Superficie expuesta (puertos)» queda en «Revisar» y se registra como riesgo aceptado (sección 4).
- Matriz de trazabilidad, manifiestos de dos corridas y comparación (sección 7): `matriz_trazabilidad_integrante2.md`, `manifiesto_20261113T192059Z.json`, `manifiesto_20261113T195359Z.json`.

## 6. Pendientes antes de la sustentación

- Aclarar en la exposición por qué el recall en vivo es n/d y que el valor estable es el del batch (0.9940).
- Ejecutar U3 justo antes de la demostración para que el panel de antigüedad no aparezca en rojo.
