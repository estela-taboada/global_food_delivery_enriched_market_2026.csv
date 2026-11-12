# Optimización de Márgenes Netos y Reducción de Fricción Logística en Delivery Gastronómico
### Planteamiento del Caso de Negocio y Auditoría Inicial
Una plataforma/cadena de restaurantes multi-ciudad opera bajo diversos canales de venta (order_channel), niveles de precio (restaurant_price_tier) y estrategias promocionales (discount_percentage). Aunque el volumen de pedidos crece, la rentabilidad neta por restaurante (estimated_restaurant_net_margin_usd) presenta alta volatilidad y las calificaciones (customer_rating) sufren caídas críticas en ciertos escenarios operativos.
###  Mandato Ejecutivo:
* Detener la erosión de margen: Identificar qué promociones, canales y rangos de distancia están generando pérdida neta por pedido ("vender más para perder dinero").
* Optimizar el cuello de botella cocina–reparto: Cuantificar cómo el tiempo de preparación (estimated_prep_time_min) y la fricción logística (logistics_friction_index) destruyen la calificación del cliente en horas pico (is_peak_hour) y mal clima (weather_condition).
* Focalizar la inversión comercial: Determinar qué segmento de clientes (customer_spending_tier) y categorías gastronómicas (cuisine_category) aportan rentabilidad sostenible sin depender de subsidios o descuentos excesivos.

### 1. Auditar la salud financiera y operativa del dataset.
```python
import pandas as pd
import numpy as np

# 1. Carga de datos
df = pd.read_csv('/content/global_food_delivery_enriched_market_2026.csv') 

print(f"Total de órdenes auditadas: {len(df):,}")
print(f"Columnas disponibles: {df.shape[1]}")

# 2. Auditoría de calidad y valores nulos (Riesgo de datos incompletos)
nulos = df.isnull().sum()
print("\n--- Auditoría de Nulos ---")
print(nulos[nulos > 0] if nulos.sum() > 0 else "Cero valores nulos: Dataset limpio para análisis.")

# 3. Radiografía Financiera Rápida (KPIs de Negocio)
ventas_brutas = df['order_value_usd'].sum()
ventas_netas = df['net_effective_order_price'].sum()
margen_total = df['estimated_restaurant_net_margin_usd'].sum()
ordenes_perdida = (df['estimated_restaurant_net_margin_usd'] <= 0).sum()
pct_perdida = (ordenes_perdida / len(df)) * 100

print("\n--- Radiografía Financiera de la Operación ---")
print(f"Ventas Brutas Totales: ${ventas_brutas:,.2f} USD")
print(f"Ventas Netas Efectivas (post-descuentos): ${ventas_netas:,.2f} USD")
print(f"Margen Neto Total Estimado: ${margen_total:,.2f} USD")
print(f"Órdenes con Margen Cero o Negativo: {ordenes_perdida:,} ({pct_perdida:.2f}% del total)")

# 4. Radiografía Operativa y de Cliente
rating_promedio = df['customer_rating'].mean()
tiempo_total_prom = df['total_fulfillment_time_min'].mean()
print("\n--- Radiografía Operativa ---")
print(f"Calificación Promedio del Cliente: {rating_promedio:.2f} / 5.0")
print(f"Tiempo Total Promedio (Cocina + Entrega): {tiempo_total_prom:.1f} min")
```
<img width="414" height="238" alt="image" src="https://github.com/user-attachments/assets/f064de15-199e-473c-936a-7e76b95eafa4" />

* Volumen y Limpieza: 8,000 órdenes auditadas a través de 26 variables, con 0% de valores nulos.   
* Salud Financiera:
  ### Ventas Brutas: $241,782.71 USD   
  ### Ventas Netas Efectivas (post-descuentos): $238,854.63 USD (solo se cedieron $2,928.08 USD en descuentos globales, un 1.21%).   
* Margen Neto Total: $86,318.38 USD (36.14% de margen sobre venta neta).   
* Foco Rojo: Existen 91 órdenes (1.14%) con margen cero o negativo.
  Aunque parecen pocas a nivel global, como analista debemos aislarlas: ¿se concentran en un solo canal, en restaurantes Budget o en distancias largas?   
* Salud Operativa:
  ### Calificación Promedio: 4.28 / 5.0
  ### Tiempo Total Promedio: 29.3 min   
