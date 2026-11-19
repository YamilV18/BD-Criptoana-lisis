# Limitaciones y mejoras

## Limitaciones

- Un solo día de entrenamiento y un solo par.
- El R² de monto y cantidad mide una identidad (`quote_qty = price × qty`).
- El rol del comprador es casi impredecible (AUC ≈ 0,55).
- Las ballenas son muy raras (≈ 0,05 %): el recall en vivo no se pudo validar.
- La serie de tiempo no supera siempre a la persistencia.
- Kafka sin autenticación; Pushgateway guarda el último valor.

## Mejoras priorizadas

1. Entrenar con varios días y escala logarítmica para el monto. *Métrica:* R² y MAE en vivo.
2. Ventanas más largas para ballenas. *Métrica:* recall en vivo.
3. Alertas con percentiles, no con el promedio. *Métrica:* falsos avisos de latencia.
4. Autenticación y TLS en Kafka.
5. CI de la documentación y de las compuertas.
