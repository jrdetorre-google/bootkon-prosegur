## Lab 8: Predicción de Ingresos con `AI.FORECAST` y BigQuery Data Canvas

<walkthrough-tutorial-duration duration="30"></walkthrough-tutorial-duration>
{{ author('Fabian Hirschmann', 'https://linkedin.com/in/fhirschmann') }}
<walkthrough-tutorial-difficulty difficulty="2"></walkthrough-tutorial-difficulty>
<bootkon-cloud-shell-note/>
<!-- Optional module: not wired into TUTORIAL.md. Renumber the "Lab N" title to match the event agenda when including. When wiring into TUTORIAL.md: merge src/optional/forecast/prompts.md into src/prompts.md and repoint the prompts.md link in "Materialize the forecast with Dataform". Not yet smoke-tested. -->

Hasta ahora tu plataforma explica lo que ya ha ocurrido. En este laboratorio opcional empezará a predecir el futuro: pronosticarás los ingresos diarios de Cymbal con **una sola función SQL** — `AI.FORECAST`, impulsada por el modelo fundacional de series temporales **TimesFM** integrado en **BigQuery** —, pedirás a `agy` que materialice esa predicción como una tabla *gold* dentro de **Dataform**, visualizarás la serie histórica junto a la predicción en **BigQuery Data Canvas** usando lenguaje natural y, finalmente, conectarás esa tabla al **BigQuery Data Agent** del Lab 5.

Antes de empezar: este laboratorio asume que has completado los **Labs 1 a 3** (`cymbal_gold.fct_daily_revenue` ya existe). La última sección utiliza además el agente `cymbal-data-agent` creado en el Lab 5.

### Acerca de TimesFM y `AI.FORECAST`

**TimesFM** es el modelo fundacional para series temporales desarrollado por **Google Research**, pre-entrenado sobre miles de millones de puntos temporales: permite predecir nuevas series en modo *zero-shot*, sin necesidad de entrenar un modelo específico sobre tus datos históricos. **BigQuery ML** lo incorpora de serie a través de la función SQL en disponibilidad general (**GA**) **`AI.FORECAST`**: sin sentencias `CREATE MODEL`, sin tiempos de entrenamiento y sin endpoints que mantener — le pasas una consulta SQL con tu serie temporal y te devuelve directamente las filas pronosticadas con sus intervalos de confianza.

Para saber más:
- [The AI.FORECAST function](https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-forecast)
- [The built-in TimesFM model](https://docs.cloud.google.com/bigquery/docs/timesfm-model)
- [The AI.EVALUATE function](https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-evaluate)
- [The AI.DETECT_ANOMALIES function](https://docs.cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-ai-detect-anomalies)
- [BigQuery data canvas](https://docs.cloud.google.com/bigquery/docs/data-canvas)

### Genera tu primera predicción en SQL

Los datos semilla de Cymbal abarcan **540 días** de histórico (del 1 de enero de 2025 al 24 de junio de 2026), y `fct_daily_revenue` contiene una fila por día y divisa (`5 × 540 = 2.700` filas completas, más las filas del día en curso generadas por el simulador).

Abre [BigQuery](https://console.cloud.google.com/bigquery) y ejecuta tu primera predicción a 14 días vista para los ingresos en euros (`EUR`):

```sql
SELECT *
FROM AI.FORECAST(
  (
    SELECT order_date, gross_revenue
    FROM `{{ PROJECT_ID }}.cymbal_gold.fct_daily_revenue`
    WHERE currency = 'EUR' AND order_date < CURRENT_DATE()
  ),
  data_col => 'gross_revenue',
  timestamp_col => 'order_date',
  horizon => 14)
ORDER BY forecast_timestamp
```

Fíjate en los argumentos: el primero es la subconsulta con la serie temporal; `data_col` y `timestamp_col` indican la métrica y la fecha; y `horizon => 14` solicita 14 días hacia el futuro. Filtramos por una única divisa (`currency = 'EUR'`) y excluimos el día actual incompleto (`order_date < CURRENT_DATE()`) para que un día en curso a medio poblar por el simulador no distorsione la tendencia.

En pocos segundos obtendrás 14 filas con `forecast_value` (la predicción mediana) acompañada de los límites inferior y superior del intervalo de confianza al 95% (`prediction_interval_lower_bound` y `prediction_interval_upper_bound`).

### Materializa el pronóstico multimoneda con Dataform y `agy`

Vamos a convertir este pronóstico en una tabla oficial de la capa `cymbal_gold` dentro de tu pipeline de **Dataform**, pronosticando todas las divisas en una sola llamada mediante el parámetro `id_cols => ['currency']` (que indica a **TimesFM** que trate cada divisa como una serie independiente).

Ve al directorio de Dataform y arranca `agy`:

```bash
cd ~/bootkon/content/agenticdata/src/dataform
agy
```

Pásale el siguiente encargo (disponible también en <walkthrough-editor-open-file filePath="content/agenticdata/src/optional/forecast/prompts.md">prompts.md</walkthrough-editor-open-file>):

```bash
/goal Create definitions/fct_revenue_forecast.sqlx: a Dataform table in schema "cymbal_gold" carrying exactly one tag, "forecast" (not silver, not gold). It materializes a 14-day daily revenue forecast per currency from ${ref("fct_daily_revenue")} (columns order_date DATE, currency STRING, gross_revenue NUMERIC), excluding the current incomplete day (order_date < CURRENT_DATE()). Use BigQuery's GA table-valued function AI.FORECAST with its exact named arguments: the SELECT as first argument, then data_col => 'gross_revenue', timestamp_col => 'order_date', id_cols => ['currency'], horizon => 14, confidence_level => 0.95. From its output keep currency, forecast_timestamp cast to DATE as forecast_date, and forecast_value, prediction_interval_lower_bound, prediction_interval_upper_bound rounded to 2 decimals as forecast_revenue, lower_bound, upper_bound. Add a table description and column descriptions, then run dataform compile and fix any errors until it compiles cleanly.
```

**Revisa el archivo creado por `agy`**: abre <walkthrough-editor-open-file filePath="content/agenticdata/src/dataform/definitions/fct_revenue_forecast.sqlx">fct_revenue_forecast.sqlx</walkthrough-editor-open-file> y compáralo con la referencia incluida en <walkthrough-editor-open-file filePath="content/agenticdata/src/optional/forecast/fct_revenue_forecast.sqlx">src/optional/forecast</walkthrough-editor-open-file>. Si necesitas usar la referencia directamente, ejecuta:

```bash
cp ~/bootkon/content/agenticdata/src/optional/forecast/fct_revenue_forecast.sqlx ~/bootkon/content/agenticdata/src/dataform/definitions/
```

Sal de `agy` con `/quit` y ejecuta el nuevo tag `forecast` en Dataform:

```bash
dataform run --tags forecast
```

Comprueba en el [editor de BigQuery](https://console.cloud.google.com/bigquery) que la tabla `fct_revenue_forecast` se ha materializado con las 5 divisas × 14 días = 70 filas:

```sql
SELECT currency, COUNT(*) AS days, MIN(forecast_date) AS first_day, MAX(forecast_date) AS last_day
FROM `{{ PROJECT_ID }}.cymbal_gold.fct_revenue_forecast`
GROUP BY currency
ORDER BY currency
```

### Visualiza el histórico más la predicción en BigQuery Data Canvas

**Data Canvas** es el lienzo visual asistido por **Gemini** dentro de **BigQuery** para combinar tablas, consultas en lenguaje natural y gráficos interactivos.

En [BigQuery](https://console.cloud.google.com/bigquery), junto al botón **SQL query** de la barra superior, abre el desplegable **Create new** → **AI and knowledge** → <walkthrough-spotlight-pointer locator="text('Data canvas')">Data canvas</walkthrough-spotlight-pointer>. Pulsa **Search for data** y busca:

```
cymbal revenue
```

Añade al lienzo tanto `fct_daily_revenue` como `fct_revenue_forecast`. Después, sobre el nodo `fct_daily_revenue`, haz clic en **Query** y pide en lenguaje natural:

```
One row per day and series: the last 60 complete days of EUR actuals from fct_daily_revenue (order_date as day, gross_revenue as revenue, series = 'actual', excluding the current date), combined with all EUR rows of cymbal_gold.fct_revenue_forecast (forecast_date as day, forecast_revenue as revenue, series = 'forecast').
```

Revisa el SQL generado por Gemini (o pega directamente esta consulta equivalente):

```sql
WITH actuals AS (
  SELECT order_date, gross_revenue
  FROM `{{ PROJECT_ID }}.cymbal_gold.fct_daily_revenue`
  WHERE currency = 'EUR' AND order_date < CURRENT_DATE()
)
SELECT 'actual' AS series, order_date AS day, gross_revenue AS revenue
FROM actuals
WHERE order_date >= (SELECT DATE_SUB(MAX(order_date), INTERVAL 59 DAY) FROM actuals)
UNION ALL
SELECT 'forecast', forecast_date, forecast_revenue
FROM `{{ PROJECT_ID }}.cymbal_gold.fct_revenue_forecast`
WHERE currency = 'EUR'
ORDER BY day
```

Ejecuta el nodo, pulsa **Visualize** y elige un gráfico de líneas (*Line chart*):

```
Line chart over day, one line per series, forecast in a different color
```

Verás cómo la curva de predicción de **TimesFM** recoge el nivel y la estacionalidad semanal de los últimos 60 días reales y los proyecta 14 días hacia delante sin haber tenido que entrenar ningún modelo.

❗ **Data Canvas** es una funcionalidad de **Gemini in BigQuery**; si en algún entorno estuviera restringido por política de organización, puedes ejecutar directamente la consulta SQL anterior en el editor estándar de BigQuery y abrir la pestaña <walkthrough-spotlight-pointer locator="semantic({tab 'Visualization'})">Visualization</walkthrough-spotlight-pointer> en el panel de resultados para visualizar el mismo gráfico de líneas.

### Enseña a tu BigQuery Data Agent a responder sobre el futuro

**Si completaste el Lab 5**, puedes ampliar tu agente `cymbal-data-agent` para que también responda preguntas predictivas.

Primero, comprueba qué ocurre antes de conectarle la tabla de pronóstico. Abre [BigQuery Agents](https://console.cloud.google.com/bigquery/agents_hub), localiza la tarjeta de ``cymbal-data-agent``, abre una conversación y pregúntale algo sobre el futuro:

```
What will revenue look like next week?
```

Como aún no tiene ninguna fuente predictiva conectada, te indicará honestamente hasta qué fecha llega su histórico. Ahora conéctale tu nueva tabla de pronóstico:

1. De vuelta en la pestaña **Agent Catalog**, abre el agente para editarlo desde el menú de tres puntos <walkthrough-spotlight-pointer locator="semantic({button 'Open actions'})">Open actions</walkthrough-spotlight-pointer> → **Edit**.
2. En *Knowledge sources*, haz clic en <walkthrough-spotlight-pointer locator="semantic({button 'Add source'})">Add source</walkthrough-spotlight-pointer>, busca ``fct_revenue_forecast`` y añádela como cuarta tabla del agente.
3. Añade esta regla al final de las instrucciones del agente:

```
- fct_revenue_forecast: use for forward-looking questions ("what will", "next week", "forecast", "projection"). forecast_revenue is the median TimesFM forecast per currency and forecast_date; always report lower_bound and upper_bound alongside it, and never sum forecasts across currencies. For dates beyond the last forecast_date, say the forecast horizon ends there instead of guessing.
```

Pulsa **Save** y luego **Publish**. Inicia una nueva conversación con el agente y pregúntale:

```
What does our revenue forecast look like, per currency?
```

Observa cómo el agente consulta `cymbal_gold.fct_revenue_forecast` y te presenta la predicción por divisa junto con sus intervalos de confianza inferior y superior.

### ¡Completado!

🎉 ¡Visionario{% if MY_NAME %}, {{ MY_NAME }}{% endif %}! Con una sola función nativa (`AI.FORECAST`) has transformado 540 días de histórico gobernado en una previsión multimoneda a dos semanas vista, materializada en **Dataform**, visualizada en **Data Canvas** y conectada a tu **BigQuery Data Agent**. 🔮
