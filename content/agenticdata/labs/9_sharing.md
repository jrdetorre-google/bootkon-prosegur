## Lab 9: Compartición Segura Zero-Copy con BigQuery Sharing (Analytics Hub)

<walkthrough-tutorial-duration duration="25"></walkthrough-tutorial-duration>
{{ author('Fabian Hirschmann', 'https://linkedin.com/in/fhirschmann') }}
<walkthrough-tutorial-difficulty difficulty="1"></walkthrough-tutorial-difficulty>
<bootkon-cloud-shell-note/>
<!-- Optional module: not wired into TUTORIAL.md. Renumber the "Lab N" title to match the event agenda when including. Not yet smoke-tested — smoke-test item 1: subscribe to your OWN listing in one sandbox project (the "Subscribe like a partner" step). If the dialog refuses same-project subscriptions, restructure the default path around neighbor pairing and pre-verify the org policy allows cross-account Analytics Hub Subscriber grants. When wiring into TUTORIAL.md: merge src/optional/sharing/prompts.md into src/prompts.md and repoint the prompts.md link in "Draft the data contract with agy". -->

Recuerda cómo empezaba el caso de uso de Cymbal: equipos ahogados en exportaciones manuales de ficheros CSV. Con los laboratorios anteriores has resuelto el problema *dentro* de tu proyecto, pero ¿qué ocurre cuando tienes que compartir productos de datos certificados entre distintas Unidades de Negocio, filiales o partners externos? En este laboratorio opcional cerrarás ese círculo con **BigQuery Sharing (Analytics Hub)**: publicarás tu capa `cymbal_gold` como un **listing** dentro de tu propio **data exchange**, redactarás su contrato de datos con `agy` y te suscribirás a él como consumidor, obteniendo acceso en vivo y *zero-copy* sin duplicar almacenamiento ni enviar ficheros por correo.

Este laboratorio asume que has completado los **Labs 1 a 3** (el dataset `cymbal_gold` ya existe) y que el simulador de pedidos (pestaña 2 de la terminal) del Lab 2 sigue ejecutándose.

### Acerca de BigQuery Sharing (Analytics Hub)

**BigQuery Sharing** (conocido también como **Analytics Hub**) es el canal nativo de publicación e intercambio de datos de **BigQuery**:
- Un **data exchange** es el escaparate o catálogo de compartición que creas en tu proyecto.
- Un **listing** representa un activo compartido (por ejemplo, el dataset `cymbal_gold`) acompañado de su documentación, contrato de uso y políticas de control de exfiltración (*egress controls*).
- Cuando otra unidad de negocio o partner se **suscribe** al *listing*, aparece en su propio proyecto un **linked dataset**: un *puntero* de solo lectura hacia tus tablas. Como BigQuery separa el almacenamiento del cómputo, tener diez o mil suscriptores no añade ni un solo byte de coste extra de almacenamiento, los cambios se ven al instante (*zero-copy*) y el publicador conserva en todo momento el control de acceso y las métricas de uso.

Para saber más:
- [Introduction to BigQuery sharing](https://docs.cloud.google.com/bigquery/docs/analytics-hub-introduction)
- [Manage listings](https://docs.cloud.google.com/bigquery/docs/analytics-hub-manage-listings)
- [View and subscribe to listings](https://docs.cloud.google.com/bigquery/docs/analytics-hub-view-subscribe-listings)

### Habilita la API y asigna los roles de publicador y suscriptor

Hoy actuarás tanto de publicador (**Analytics Hub Admin**) como de consumidor (**Analytics Hub Subscriber**). Habilita la API y concédete ambos roles en tu proyecto:

```bash
gcloud services enable analyticshub.googleapis.com
gcloud projects add-iam-policy-binding {{ PROJECT_ID }} \
    --member=user:{{ GCP_USERNAME }} --role=roles/analyticshub.admin --condition=None
gcloud projects add-iam-policy-binding {{ PROJECT_ID }} \
    --member=user:{{ GCP_USERNAME }} --role=roles/analyticshub.subscriber --condition=None
```

❗ **¿Aparece un mensaje de permiso denegado al abrir la página de Sharing?** Los permisos de IAM recién asignados pueden tardar alrededor de un minuto en propagarse; espera unos segundos y recarga la página.

### Crea el Data Exchange

1. Abre [Sharing (Analytics Hub)](https://console.cloud.google.com/bigquery/analytics-hub) en la consola y haz clic en <walkthrough-spotlight-pointer locator="semantic({button 'Create exchange'})">Create exchange</walkthrough-spotlight-pointer>.
2. Rellena los campos:
    - **Display name**: `cymbal-exchange`
    - **Region**: **US** (la multi-región `US`, ya que el *exchange* debe estar en la misma ubicación que el dataset `cymbal_gold` que va a compartir).
    - **Description**: `Cymbal's curated data products`
3. Haz clic en **Create exchange**. Cuando aparezca el paso *Exchange permissions*, pulsa **Skip**.

### Redacta el contrato de datos (Data Contract) con `agy`

Un producto de datos compartido solo genera confianza si va acompañado de un contrato de datos claro que explique qué contiene cada tabla, su frecuencia de actualización y las reglas de negocio de sus métricas. Pide a `agy` que lo redacte en Markdown a partir de los esquemas reales de `cymbal_gold` (el prompt también está en <walkthrough-editor-open-file filePath="content/agenticdata/src/optional/sharing/prompts.md">prompts.md</walkthrough-editor-open-file>):

```bash
agy -p "Read the table schemas: bq show --schema {{ PROJECT_ID }}:cymbal_gold.fct_daily_revenue ; bq show --schema {{ PROJECT_ID }}:cymbal_gold.dim_customer_360 ; bq show --schema {{ PROJECT_ID }}:cymbal_gold.fct_product_performance. Then write a listing description for sharing these three tables with Cymbal's partners — a data contract as prose, in markdown: one paragraph on what the dataset is, a short section per table (grain, key columns, what questions it answers), a freshness section (the tables are fed by live Datastream CDC from the production Postgres and rebuilt by Dataform runs), and a definitions section (gross revenue is SUM(qty * unit_price) excluding cancelled orders; lifetime value is the gross revenue of a customer's non-cancelled orders; in fct_daily_revenue amounts are per currency and must never be summed across currencies; lifetime_value and the fct_product_performance amounts sum across order currencies without conversion, so present them as activity indicators, not accounting figures). Output only the markdown." --dangerously-skip-permissions
```

Revisa el borrador generado (o abre directamente la plantilla de referencia incluida en <walkthrough-editor-open-file filePath="content/agenticdata/src/optional/sharing/listing_description.md">listing_description.md</walkthrough-editor-open-file>):

```bash
cat content/agenticdata/src/optional/sharing/listing_description.md
```

### Publica el Listing

1. De vuelta en la página de Sharing, abre <walkthrough-spotlight-pointer locator="text('cymbal-exchange')">cymbal-exchange</walkthrough-spotlight-pointer> y haz clic en <walkthrough-spotlight-pointer locator="semantic({button 'Create listing'})">Create listing</walkthrough-spotlight-pointer>.
2. **Resource type**: selecciona **BigQuery dataset** → elige `cymbal_gold`.
3. **Listing details**:
    - **Display name**: `Cymbal Gold — Commerce Analytics`
    - **Description**: `Analysis-ready commerce marts: daily revenue, customer 360, product performance. Live CDC + Dataform.`
    - Pega el contrato de datos en Markdown dentro del campo **Documentation**.
4. Localiza la sección **data egress controls** y marca la casilla *Disable copy and export of shared data* — de este modo los suscriptores pueden consultar las tablas `gold` pero no copiarlas ni exportarlas masivamente fuera del entorno gobernado.
5. Haz clic en <walkthrough-spotlight-pointer locator="semantic({button 'Publish'})">Publish</walkthrough-spotlight-pointer>.

### Suscríbete al Listing como consumidor

Ahora ponte en el lugar de otra unidad de negocio o consumidor. Abre el *listing* que acabas de publicar dentro de `cymbal-exchange`:

1. Haz clic en <walkthrough-spotlight-pointer locator="semantic({button 'Subscribe'})">Subscribe</walkthrough-spotlight-pointer> para abrir el diálogo *Create linked dataset*.
2. Completa los datos:
    - **Project**: `{{ PROJECT_ID }}` (un publicador puede suscribirse a su propio *listing* para validar la experiencia del consumidor).
    - **Linked dataset name**: `cymbal_gold_linked`
    - **Primary region**: **US**
3. Haz clic en <walkthrough-spotlight-pointer locator="semantic({button 'Save'})">Save</walkthrough-spotlight-pointer>.

Abre ahora [BigQuery](https://console.cloud.google.com/bigquery), despliega tu proyecto y localiza <walkthrough-spotlight-pointer locator="text('cymbal_gold_linked')">cymbal_gold_linked</walkthrough-spotlight-pointer>: verás que lleva un icono especial de dataset enlazado (*linked*). Si entras en `fct_daily_revenue`, verás exactamente el mismo esquema y metadatos que en el original, pero en modo solo lectura y sin haber movido ni duplicado un solo byte.

### Demuestra en vivo que es un puntero Zero-Copy y no una copia estática

En el editor de consultas de **BigQuery**, ejecuta esta consulta comparando lado a lado el dataset origen y el dataset suscrito:

```sql
SELECT 'source' AS via, SUM(orders) AS orders,
       ROUND(SUM(IF(currency = 'EUR', gross_revenue, 0)), 2) AS eur_revenue
FROM `{{ PROJECT_ID }}.cymbal_gold.fct_daily_revenue`
UNION ALL
SELECT 'linked', SUM(orders),
       ROUND(SUM(IF(currency = 'EUR', gross_revenue, 0)), 2)
FROM `{{ PROJECT_ID }}.cymbal_gold_linked.fct_daily_revenue`
```

Ambas filas devuelven exactamente las mismas cifras porque leen del mismo almacenamiento físico. Ahora actualiza las tablas origen ejecutando **Dataform** para incorporar los nuevos pedidos que el simulador y **Datastream** han seguido inyectando en `cymbal_bronze`:

```bash
cd ~/bootkon/content/agenticdata/src/dataform
dataform run --tags silver && dataform run --tags gold
```

Vuelve a ejecutar la consulta SQL anterior en BigQuery: comprobarás que ambas filas (`source` y `linked`) han subido **al unísono y al instante**, sin re-exportaciones ni pipelines de sincronización en el lado del consumidor.

### Revisa las métricas de uso y el bloqueo de exfiltración

De vuelta en [Sharing (Analytics Hub)](https://console.cloud.google.com/bigquery/analytics-hub), entra en `cymbal-exchange`, abre la pestaña <walkthrough-spotlight-pointer locator="text('Usage metrics')">Usage metrics</walkthrough-spotlight-pointer> y selecciona tu *listing*: como publicador tienes visibilidad completa de las suscripciones activas, los trabajos ejecutados y las tablas más consultadas (disponibles también por SQL en `` `region-us`.INFORMATION_SCHEMA.SHARED_DATASET_USAGE ``).

Y si intentas hacer una copia de una tabla dentro de `cymbal_gold_linked` (menú de tres puntos junto a la tabla → *Copy*), comprobarás que la operación queda bloqueada gracias a la política de *egress control* que activaste al publicar el *listing*.

### ¡Completado!

🎉 ¡Soberbio{% if MY_NAME %}, {{ MY_NAME }}{% endif %}! La era de compartir datos mediante exportaciones CSV ha terminado: tu capa *gold* es ahora un producto de datos gobernado, con contrato de datos, control de exfiltración, auditoría de uso y distribución instantánea *zero-copy*. 📬→📡
