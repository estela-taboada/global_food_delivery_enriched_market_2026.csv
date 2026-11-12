# Optimización de Márgenes Netos y Reducción de Fricción Logística en Delivery Gastronómico
### Planteamiento del Caso de Negocio y Auditoría Inicial
Una plataforma/cadena de restaurantes multi-ciudad opera bajo diversos canales de venta (order_channel), niveles de precio (restaurant_price_tier) y estrategias promocionales (discount_percentage). Aunque el volumen de pedidos crece, la rentabilidad neta por restaurante (estimated_restaurant_net_margin_usd) presenta alta volatilidad y las calificaciones (customer_rating) sufren caídas críticas en ciertos escenarios operativos.
###  Mandato Ejecutivo:
- **Detener la erosión de margen:** Identificar qué promociones, canales y rangos de distancia están generando pérdida neta por pedido ("vender más para perder dinero").
- **Optimizar el cuello de botella cocina–reparto:** Cuantificar cómo el tiempo de preparación (estimated_prep_time_min) y la fricción logística (logistics_friction_index) destruyen la calificación del cliente en horas pico (is_peak_hour) y mal clima (weather_condition).
- **Focalizar la inversión comercial:** Determinar qué segmento de clientes (customer_spending_tier) y categorías gastronómicas (cuisine_category) aportan rentabilidad sostenible sin depender de subsidios o descuentos excesivos.

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

## Hallazgo
- **Volumen y Limpieza:** 8,000 órdenes auditadas a través de 26 variables, con 0% de valores nulos.
- **Salud Financiera:**
  - Ventas Brutas: $241,782.71 USD
  - Ventas Netas Efectivas (post-descuentos): $238,854.63 USD (solo se cedieron $2,928.08 USD en descuentos globales, un 1.21%).
- **Margen Neto Total:** $86,318.38 USD (36.14% de margen sobre venta neta).   
- **Foco Rojo:** Existen 91 órdenes (1.14%) con margen cero o negativo.
  Aunque parecen pocas a nivel global, como analista debemos aislarlas: ¿se concentran en un solo canal, en restaurantes Budget o en distancias largas?   
- **Salud Operativa:**
  - Calificación Promedio: 4.28 / 5.0
  - Tiempo Total Promedio: 29.3 min

## Diagnóstico de Fugas y Fricción Operativa (SQL)
- Query 1 (Rentabilidad por Nivel de Precio y Canal): ¿Dónde están escondidas esas 91 órdenes en pérdida y qué canal deja mejor margen neto?
- Query 2 (Impacto de Hora Pico y Clima): ¿Cuánto cae la calificación (4.28 base) y cuánto sube el tiempo (29.3 min base) cuando hay hora pico o mal clima?
- Query 3 (Top y Bottom Categorías de Cocina): ¿Qué tipo de comida es la más rentable y cuál tiene mayor fricción logística?.

```Python
import sqlite3

# Creamos una base de datos SQL en memoria y cargamos tu tabla limpia
conn = sqlite3.connect(':memory:')
df.to_sql('delivery_orders', conn, index=False, if_exists='replace')

# QUERY 1: Auditoría de Rentabilidad y Órdenes en Pérdida por Segmento de Precio y Canal
query_1 = """
SELECT 
    restaurant_price_tier AS segmento_precio,
    order_channel AS canal,
    COUNT(order_id) AS total_ordenes,
    ROUND(AVG(net_effective_order_price), 2) AS ticket_neto_prom,
    ROUND(AVG(estimated_restaurant_net_margin_usd), 2) AS margen_neto_prom,
    SUM(CASE WHEN estimated_restaurant_net_margin_usd <= 0 THEN 1 ELSE 0 END) AS ordenes_en_perdida,
    ROUND(100.0 * SUM(CASE WHEN estimated_restaurant_net_margin_usd <= 0 THEN 1 ELSE 0 END) / COUNT(order_id), 2) AS pct_perdida
FROM delivery_orders
GROUP BY restaurant_price_tier, order_channel
ORDER BY ordenes_en_perdida DESC, margen_neto_prom ASC;
"""

# QUERY 2: Impacto Operativo de Hora Pico y Clima en Tiempos y Satisfacción
query_2 = """
SELECT 
    is_peak_hour AS hora_pico,
    weather_condition AS clima,
    COUNT(order_id) AS ordenes,
    ROUND(AVG(estimated_prep_time_min), 1) AS prep_cocina_min,
    ROUND(AVG(actual_delivery_time_min), 1) AS reparto_min,
    ROUND(AVG(total_fulfillment_time_min), 1) AS tiempo_total_min,
    ROUND(AVG(logistics_friction_index), 2) AS indice_friccion,
    ROUND(AVG(customer_rating), 2) AS rating_prom
FROM delivery_orders
GROUP BY is_peak_hour, weather_condition
ORDER BY rating_prom ASC;
"""

# QUERY 3: Desempeño Comercial y Logístico por Categoría de Cocina
query_3 = """
SELECT 
    cuisine_category AS categoria_cocina,
    COUNT(order_id) AS ordenes,
    ROUND(SUM(estimated_restaurant_net_margin_usd), 2) AS margen_total_usd,
    ROUND(AVG(estimated_restaurant_net_margin_usd), 2) AS margen_por_orden,
    ROUND(AVG(discount_percentage), 1) AS descuento_prom_pct,
    ROUND(AVG(total_fulfillment_time_min), 1) AS tiempo_total_min,
    ROUND(AVG(customer_rating), 2) AS rating_prom
FROM delivery_orders
GROUP BY cuisine_category
ORDER BY margen_total_usd DESC;
"""

print("=== SQL 1: RENTABILIDAD POR SEGMENTO Y CANAL ===")
display(pd.read_sql_query(query_1, conn))

print("\n=== SQL 2: IMPACTO DE HORA PICO Y CLIMA ===")
display(pd.read_sql_query(query_2, conn))

print("\n=== SQL 3: DESEMPEÑO POR CATEGORÍA DE COCINA ===")
display(pd.read_sql_query(query_3, conn))

```

<img width="907" height="315" alt="image" src="https://github.com/user-attachments/assets/1e587ae3-3355-4942-8107-af3fa32c8667" />

<img width="831" height="276" alt="image" src="https://github.com/user-attachments/assets/0203d1e7-1cd4-4eb4-89dd-8aeddd753d8b" />

<img width="884" height="250" alt="image" src="https://github.com/user-attachments/assets/e8279281-7265-4d42-b13c-68eff9c86167" />

**1. Diagnóstico de Rentabilidad (SQL 1)**
- El culpable único de las pérdidas: El 96.7% de todas las órdenes en pérdida (88 de las 91 totales) ocurren en una sola combinación: Restaurantes $ Budget vendiendo a través de Online Delivery App.
- Cero fugas en otros canales: Ni Takeaway / Pickup ni Dine-In POS tienen una sola orden negativa en ningún nivel de precio.
- En $ Budget, vender por app deja apenas $3.93 USD de margen neto promedio frente a $6.58 USD en salón (Dine-In POS) —una erosión del 40% en rentabilidad por comisiones, tarifas de entrega y descuentos sobre tickets bajos ($17.38 USD).
  
**2. Diagnóstico Operativo (SQL 2)**
- La tormenta perfecta: Cuando coinciden Hora Pico (1) + Tormenta (Heavy Rain / Storm), el índice de fricción se dispara casi 9 veces respecto a un día despejado fuera de pico (6.11 vs 0.68).
- Cuello de botella doble (Cocina + Reparto): En ese escenario crítico, el tiempo total sube de 25.2 min a 39.7 min (+14.5 minutos): cocina sube +4.8 min (15.9 a 20.7 min) y reparto sube +9.7 min (9.3 a 19.0 min), hundiendo la calificación promedio por debajo del umbral crítico de 4 estrellas (3.90).
- En cualquier clima, la Hora Pico agrega invariablemente ~5 minutos extra en preparación de cocina (pasa de ~16 min a ~21 min).

**3. Diagnóstico por Categoría de Cocina (SQL 3)**
  - Motor de volumen vs. Motor de margen: Burgers & Fast Food lidera en ganancia total ($18,740.67 USD) por volumen (1,788 órdenes) y tiene la mejor calificación (4.31), pero Japanese & Sushi es la categoría más rentable por pedido ($11.54 USD de margen neto).
  -  Fuga por subsidio innecesario: Categorías como Healthy & Bowls e Indian & Curries están recibiendo el descuento promedio más alto (12.5%), aunque tienen tickets altos y buena aceptación.   

## Aislamiento de Fugas en $ Budget + Exportación 
Limpia para Power BI Antes de pasar a Power BI, se última consulta SQL quirúrgica para descubrir qué provoca exactamente que esas 88 órdenes de $ Budget en Online Delivery App den pérdida (¿es el porcentaje de descuento o la distancia de entrega?), y de una vez exportamos el CSV limpio listo para Power BI.   

```Python
# QUERY 4: Autopsia de las órdenes en pérdida ($ Budget + Online Delivery App)
query_4 = """
SELECT 
    CASE WHEN estimated_restaurant_net_margin_usd <= 0 THEN 'En Pérdida (<= $0)' ELSE 'Rentable (> $0)' END AS estado_margen,
    COUNT(order_id) AS ordenes,
    ROUND(AVG(order_value_usd), 2) AS venta_bruta_prom,
    ROUND(AVG(discount_percentage), 1) AS descuento_prom_pct,
    ROUND(AVG(delivery_distance_km), 2) AS distancia_prom_km,
    ROUND(AVG(delivery_fee_ratio), 2) AS ratio_tarifa_entrega,
    ROUND(AVG(estimated_restaurant_net_margin_usd), 2) AS margen_neto_prom
FROM delivery_orders
WHERE restaurant_price_tier = '$ Budget' 
  AND order_channel = 'Online Delivery App'
GROUP BY estado_margen;
"""

print("=== SQL 4: AUTOPSIA DE ÓRDENES EN PÉRDIDA ($ BUDGET + APP) ===")
display(pd.read_sql_query(query_4, conn))

# --- PREPARACIÓN Y EXPORTACIÓN DEL DATASET LIMPIO PARA POWER BI ---
df['is_profitable'] = df['estimated_restaurant_net_margin_usd'].apply(lambda x: 'Rentable' if x > 0 else 'En Pérdida')
df['peak_hour_label'] = df['is_peak_hour'].map({1: 'Hora Pico', 0: 'Hora Regular'})

# Guardamos el archivo limpio para descargarlo y cargarlo en Power BI
df.to_csv('/content/delivery_clean_powerbi.csv', index=False)
print("\nArchivo 'delivery_clean_powerbi.csv' listo para descargar desde el panel izquierdo de Colab.")
```
<img width="884" height="136" alt="image" src="https://github.com/user-attachments/assets/030787ab-530c-409f-aa10-7ab54ff81dbb" />

## Hallazgo 
Las 88 órdenes en pérdida dentro de $ Budget + Online Delivery App no son casualidad; son una trampa de micro-tickets subsidiados:   
- Ticket demasiado bajo: Su venta bruta promedio es de apenas $3.60 USD (frente a $15.13 USD en las órdenes rentables del mismo canal).  
- Doble golpe (Descuento + Logística): A ese ticket diminuto de $3.60 USD se le aplicó un 19.8% de descuento promedio (vs. 12.8% en rentables) y tuvo un ratio de tarifa de entrega de 1.31 (es decir, el costo de envío superó en un 31% el valor de la comida) para recorrer 5.13 km.
- Resultado: El restaurante terminó pagando $0.17 USD de su bolsillo por cada una de esas entregas.
- Regla de negocio accionable (tu recomendación ejecutiva): Establecer un pedido mínimo de $8.00 – $10.00 USD en Online Delivery App para restaurantes $ Budget y bloquear descuentos mayores al 15% en tickets inferiores a $12.00 USD. Con esa sola regla eliminas el 96.7% de las pérdidas operativas sin afectar el volumen rentable.

**Resumen**
**1. Línea Base Auditada (Python):**  
8,000 órdenes limpias, $238,854.63 USD en ventas netas y $86,318.38 USD de margen neto (36.14%).
**2. Diagnóstico Quirúrgico (4 Queries SQL):** 
- Financiero: Aislamiento de las 91 órdenes en pérdida (88 concentradas en micro-tickets de $ Budget por app).
- Operativo: Cuantificación del cuello de botella en Hora Pico + Tormenta (índice de fricción sube 9x, tiempo total pasa de 25.2 a 39.7 min y calificación cae a 3.90).
- Comercial: Comparativa de rentabilidad y descuentos por categoría de cocina (Japanese & Sushi líder en margen por orden con $11.54 USD; Burgers & Fast Food líder en volumen y rating con 4.31).
** 3. Dataset Limpio Exportado:** delivery_clean_powerbi.csv listo para descargar y cargar directo en Power BI.   
