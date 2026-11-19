# Bitácora de integración y revisión técnica - Integrante 3

**Proyecto Sello · BTC/USDT (Binance) · CriptoAnalytics** - Unidad III (S13 integración y DataOps, S14 revisión técnica y hardening)

| | |
|---|---|
| **Dimensión** | Rol del comprador (`is_buyer_maker`) · Clasificación |
| **Pregunta de mi dimensión** | ¿Es posible predecir el rol de la orden (Maker Comprador vs. Maker Vendedor) según el tamaño del volumen y la fluctuación del precio? |
| **Modelo de U1 desplegado** | `GBT_it10_prof5` (ganador de U1) |
| **Indicador** | AUC-ROC: U1 (offline) 0.5548 · U2 (eventos en vivo) 0.5160 |
| **Corridas comparadas** | `20261113T192318Z` → `20261113T194921Z` (manifiestos en `artifacts/integrante3/u3/`) |

> **Nota sobre las cifras:** la fuente de streaming es el WebSocket de Binance en vivo, por lo que los valores de U2 y de U3 cambian de una ejecución a otra. Las cifras de U2 de esta bitácora vienen de la ejecución del notebook previa a la exposición; las de U3, de las dos corridas indicadas. Cuando un valor se explica con una causa que no se verificó, se marca como **hipótesis**.

## 1. Bitácora de integración (S13)

Hallazgos propios de la dimensión del integrante 3:

| # | Hallazgo | Causa | Corrección | Prueba de corrección |
|---|---|---|---|---|
| 1 | El modelo apenas discrimina el rol y la ventana en vivo cambia de signo. | AUC 0.5548 en el CSV y 0.5160 en vivo (casi el azar, coherente con U1). La clase positiva pasó de 22.7 % a 82.9 % entre corridas; en la segunda, exactitud 0.176 frente a un baseline de 0.829, recall 0.0100 y precisión 0.731. **Causa probable:** la fracción de operaciones `buyer maker` oscila con la dirección del mercado (en U2 las barras de 10 s van de 0.02 a 0.87). | Se evalúa con AUC y con recall/precisión junto a la proporción de clases, no con la exactitud sola. El resultado se presenta como un límite del problema, no como un fallo del pipeline. | Sección 3 del notebook U3 y métricas `ventana_positivos`, `ventana_tp/fp/fn`, `ventana_recall_pos` en Prometheus. |
| 2 | En la serie de tiempo el modelo sí supera a la persistencia. | Histórico (U2): Random Forest, RMSE 0.350 frente a 0.437 (19.8 % menos). En vivo (U2, 1 141 barras): MAE 0.305 frente a 0.331. En U3: 0.441 frente a 0.552 y 0.392 frente a 0.411. | Es la dimensión donde el modelo supera a la persistencia en todas las mediciones (histórico, vivo y las dos de U3); se presenta con sus números y con el tamaño de muestra (7 a 8 barras en U3). | Panel «Serie de tiempo: MAE modelo vs persistencia»: la línea del modelo queda por debajo en las dos corridas. |
| 3 | El promedio de latencia de U2 (3 336 ms) y el p95 (4 914 ms) están cerca, pero el p99 de U3 es 12 050 ms. | La cola extrema existe aunque el p95 sea moderado. | Se reportan los tres percentiles; solo el p50 detiene y el p95 advierte. | Primer panel del tablero: p50 ≈ 3 s, p95 ≈ 5 s, p99 ≈ 12 s. |

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

| Medida | Corrida 1 (`20261113T192318Z`) | Corrida 2 (`20261113T194921Z`) | ¿Esperable? |
|---|---|---|---|
| Hardening (puntos que cumplen) | 3/5 | 5/5 | Esperable: es lo que se corrigió en el `compose.yml`. |
| Compuertas previas superadas | 9/9 | 9/9 | Sin cambio. |
| Eventos en el batch (Parquet de U2) | 1 823 | 1 823 | Sin cambio. |
| Filas procesadas por el stream | 1 658 | 2 300 | Esperable: fuente en vivo. |
| Proporción de la clase positiva en la ventana | 22.7 % (377/1 658) | 82.9 % (1 907/2 300) | Esperable con fuente en vivo, pero es un cambio grande (ver hallazgo 1). |
| Exactitud / baseline en la ventana | 0.761 / 0.773 | 0.176 / 0.829 | Depende de la proporción de clases. |
| MAE serie de tiempo: modelo / persistencia | 0.441 / 0.552 | 0.392 / 0.411 | El modelo supera a la persistencia en las dos. |

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
- Métricas `cripto_int_*` publicadas y confirmadas en Prometheus con el valor recién publicado (sección 5); tablero «CriptoAnalytics - Integración U3», instancia `integrante3`.
- Revisión de hardening del `compose.yml`: **3/5 → 5/5** (sección 6). El punto «Superficie expuesta (puertos)» queda en «Revisar» y se registra como riesgo aceptado (sección 4).
- Matriz de trazabilidad, manifiestos de dos corridas y comparación (sección 7): `matriz_trazabilidad_integrante3.md`, `manifiesto_20261113T192318Z.json`, `manifiesto_20261113T194921Z.json`.

## 6. Pendientes antes de la sustentación

- Preparar la explicación del cambio de la clase positiva entre corridas (22.7 % → 82.9 %).
- Ejecutar U3 justo antes de la demostración para que el panel de antigüedad no aparezca en rojo.
