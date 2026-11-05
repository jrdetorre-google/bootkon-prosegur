## Lab 5: El Agente Conversacional de Datos en BigQuery (BigQuery Data Agent)

<walkthrough-tutorial-duration duration="20"></walkthrough-tutorial-duration>
{{ author('Fabian Hirschmann', 'https://linkedin.com/in/fhirschmann') }}
<walkthrough-tutorial-difficulty difficulty="2"></walkthrough-tutorial-difficulty>
<bootkon-cloud-shell-note/>

Ha llegado el momento de crear nuestro primer agente de negocio. Los **Conversational Analytics Data Agents** viven de forma nativa dentro de **BigQuery**: los conectas a tus tablas gobernadas de la capa *gold*, les proporcionas contexto de negocio y los usuarios pueden conversar con los datos en lenguaje natural mientras el agente genera, ejecuta y explica el SQL por debajo. Además, en el Lab 6 este mismo agente se convertirá en un *servicio* al que otros agentes invocarán mediante el protocolo **A2A**.

Observa cómo encaja todo el trabajo que has realizado en los laboratorios anteriores: el agente solo tendrá acceso a la capa **gold** — cuyos datos ya han sido limpiados y validados por tus modelos *silver*, auditados en calidad, protegidos frente a fugas de *PII* y documentados en el catálogo. Los agentes fiables empiezan siempre por datos gobernados.

### Acerca de los BigQuery Data Agents

Un **Data Agent** de *Conversational Analytics* combina tres elementos fundamentales:
1. **Knowledge sources**: las tablas o vistas que está autorizado a consultar (nuestras tres tablas de `cymbal_gold`).
2. **Context (System instructions)**: las instrucciones semánticas que le enseñan el lenguaje de tu negocio (sinónimos, métricas oficiales, agrupaciones por defecto y reglas de formato).
3. **Verified queries**: pares de *pregunta en lenguaje natural + consulta SQL certificada* que el agente prioriza como referencia exacta en lugar de improvisar.

Cuando un usuario formula una pregunta, **Gemini** la traduce a SQL sobre esas fuentes de conocimiento, ejecuta la consulta como un trabajo estándar de **BigQuery** — respetando íntegramente los permisos de IAM y el *Column-Level Security* configurado en el Lab 4 — y resume la respuesta mostrando la consulta SQL empleada. Y dado que la **Conversational Analytics API** está en disponibilidad general (*GA*), cuando publicas el agente no se queda encerrado en la consola web: se convierte en un recurso invocable por API con su propio control de acceso IAM, que es exactamente lo que aprovecharemos en el Lab 6.

Para saber más:
- [Create and use data agents in BigQuery](https://docs.cloud.google.com/bigquery/docs/create-data-agents)
- [Conversational Analytics API](https://docs.cloud.google.com/gemini/data-agents/conversational-analytics-api/overview)

### Redacta las instrucciones del agente con `agy`

Un *Data Agent* es tan bueno como el contexto semántico que le des. Como ese contexto se deriva directamente de los esquemas de tus tablas en `cymbal_gold`, vamos a pedirle a `agy` que inspeccione los esquemas en BigQuery y redacte las instrucciones óptimas. Ejecuta este comando en tu terminal (en modo no interactivo `-p`; el prompt también está disponible en <walkthrough-editor-open-file filePath="content/agenticdata/src/prompts.md">prompts.md</walkthrough-editor-open-file>):

```bash
agy -p "Read the table schemas: bq show --schema {{ PROJECT_ID }}:cymbal_gold.fct_daily_revenue ; bq show --schema {{ PROJECT_ID }}:cymbal_gold.dim_customer_360 ; bq show --schema {{ PROJECT_ID }}:cymbal_gold.fct_product_performance. Then draft system instructions for a BigQuery conversational data agent over these three tables: synonyms business users might use (revenue, sales, LTV, best sellers), which table answers which kind of question, default groupings (order_date, currency, category), and columns to exclude from summaries (email). Output only the instructions text." --dangerously-skip-permissions
```

Revisa el texto generado por `agy`. Si prefieres utilizar directamente una plantilla base probada (o compararla con la propuesta de `agy`), aquí tienes la referencia:

```
Tables and their roles:
- fct_daily_revenue: revenue, sales, turnover questions. Group by order_date and currency by default. gross_revenue is SUM(qty * unit_price) excluding cancelled orders.
- dim_customer_360: customer questions. "LTV", "lifetime value", "best customers" -> lifetime_value. One row per deduplicated customer.
- fct_product_performance: product questions. "best sellers" -> units_sold; "most profitable" -> gross_margin. Group by category for overviews.

Rules:
- Amounts are per currency; never sum across currencies without saying so.
- Exclude the email column from any output or summary.
- "Orders" means non-cancelled orders unless the user asks otherwise.
- Data freshness: near real time via CDC + Dataform runs; mention the latest order_date when answering trend questions.
```

### Crear el Data Agent en la consola de BigQuery

1. Abre la [consola de BigQuery](https://console.cloud.google.com/bigquery).
2. En el panel izquierdo, despliega tu proyecto <walkthrough-spotlight-pointer locator="semantic({treeitem 'Toggle node {{ PROJECT_ID }}'} {button 'Toggle node'})">{{ PROJECT_ID }}</walkthrough-spotlight-pointer> y haz clic en <walkthrough-spotlight-pointer locator="text('Agents')">Agents</walkthrough-spotlight-pointer>.
3. Haz clic en <walkthrough-spotlight-pointer locator="semantic({button 'Create agent'})">Create agent</walkthrough-spotlight-pointer>.
4. Asígnale exactamente el nombre **`cymbal-data-agent`** — es fundamental respetar este nombre porque el código del Lab 6 lo invoca por ese identificador — y añade una descripción como *"Analytics over Cymbal's gold layer"*.
5. Selecciona la región **GLOBAL** (o ``global``) — la *Conversational Analytics API* que utilizaremos en el Lab 6 enruta las peticiones a `locations/global`.
6. Haz clic en **Add Source** y selecciona las tres tablas del dataset `cymbal_gold`: ``fct_daily_revenue``, ``dim_customer_360`` y ``fct_product_performance``.
7. Pega las instrucciones del sistema (el borrador de `agy` o el bloque de referencia anterior) en el cuadro de instrucciones del agente.

### Añade una consulta verificada (Verified Query)

Las *verified queries* son consultas SQL certificadas que el agente toma como patrón oficial ante preguntas recurrentes de negocio. Añade una para la pregunta más habitual (los ingresos diarios por divisa):

```sql
SELECT order_date, currency, SUM(gross_revenue) AS revenue
FROM `{{ PROJECT_ID }}.cymbal_gold.fct_daily_revenue`
GROUP BY order_date, currency
ORDER BY order_date DESC
```

asociándola a la pregunta en lenguaje natural: ``What was our revenue by day?``

### Publica el agente y conversa con tus datos

En la barra superior del agente:
1. Haz clic en **`Save`**.
2. A continuación, haz clic en **`Publish`** — publicar el agente es lo que lo expone a través de la **Conversational Analytics API**, requisito imprescindible para el Lab 6.

Ahora haz clic en **`Create conversation`** y pon a prueba tu plataforma. Puedes probar estas preguntas (en inglés o en español) e improvisar las tuyas propias:

- `What was our total revenue in EUR in the last 30 days?`
- `Who are our top 10 customers by lifetime value, and which countries are they from?`
- `Which product category has the best gross margin?`
- `Are there any days with unusually high revenue? What might explain them?`

Observa las respuestas: el agente muestra la consulta SQL exacta que ha generado y ejecutado contra tus tablas `cymbal_gold`. Todo el trabajo de los tres laboratorios anteriores para limpiar y estructurar el modelo de datos da aquí su fruto. *(Prueba también a pedirle los correos electrónicos de los clientes: comprobarás cómo las reglas del agente y las políticas de gobierno impiden exponer el campo `email`).*

### ¡Completado!

🎉 ¡Excelente{% if MY_NAME %}, {{ MY_NAME }}{% endif %}! Cymbal cuenta ya con un **BigQuery Data Agent** publicado: recibe preguntas en lenguaje natural y devuelve respuestas analíticas precisas basadas en SQL gobernado sobre la capa *gold*. Ya tenemos nuestro primer agente; en el laboratorio final le daremos un compañero operacional para trabajar en equipo mediante **A2A**. 🤖
