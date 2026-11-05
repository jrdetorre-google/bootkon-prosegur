## Lab 4: Gobierno del Dato con Knowledge Catalog

<walkthrough-tutorial-duration duration="30"></walkthrough-tutorial-duration>
{{ author('Fabian Hirschmann', 'https://linkedin.com/in/fhirschmann') }}
<walkthrough-tutorial-difficulty difficulty="2"></walkthrough-tutorial-difficulty>
<bootkon-cloud-shell-note/>

Ya tienes tu arquitectura Medallion funcionando — ahora toca hacerla *confiable y gobernada*. Antes de permitir que un agente de IA consulte tus datos en el Lab 5, toda plataforma empresarial necesita responder a tres preguntas clave: **¿Puedo encontrar el dato? ¿Puedo fiarme de su calidad? ¿Quién tiene permiso para verlo?** En este laboratorio responderás a las tres con **Knowledge Catalog** (anteriormente *Dataplex*), comprobando además que el catálogo ya ha hecho de forma automática gran parte del trabajo por ti.

### Acerca de Knowledge Catalog

**Knowledge Catalog** es la capa unificada de metadatos y gobierno de **Google Cloud**, y se **alimenta automáticamente**: cada dataset, tabla y columna de **BigQuery** que has creado en los Labs 2 y 3 ya está registrado en el catálogo, junto con las descripciones técnicas definidas en tu código de **Dataform**. Sobre esa base automática, tú añades los dos controles que la plataforma no puede adivinar por sí sola:
1. **Data Quality Rules**, que miden objetivamente la calidad de los datos almacenados en una tabla.
2. **Policy Tags**, que restringen quién puede leer columnas sensibles (*PII*).

Tanto los analistas humanos como los agentes de IA se apoyan en estos metadatos para descubrir y comprender tus datos con seguridad.

Para saber más:
- [Knowledge Catalog overview](https://docs.cloud.google.com/knowledge-catalog/docs/introduction)
- [Auto data quality](https://docs.cloud.google.com/knowledge-catalog/docs/auto-data-quality-overview)
- [Column-level security with policy tags](https://docs.cloud.google.com/bigquery/docs/column-level-security-intro)

### Descubre tus datos con búsqueda semántica

Aunque nadie ha registrado manualmente ninguna tabla en el catálogo, vamos a buscarla en lenguaje natural. Abre [Knowledge Catalog search](https://console.cloud.google.com/dataplex/dp-search) y escribe esta pregunta en el buscador:

```
Which tables contain revenue by day?
```

Verás aparecer `fct_daily_revenue` en los resultados (si prefieres buscar por palabra clave, `revenue` también la localiza al instante). El catálogo la ha indexado automáticamente desde **BigQuery**, sin necesidad de configurar *crawlers* ni procesos manuales de registro.

Comprueba ahora qué metadatos conoce el catálogo sobre ella. En [BigQuery](https://console.cloud.google.com/bigquery), abre `cymbal_gold` → `fct_daily_revenue` → <walkthrough-spotlight-pointer locator="semantic({tab 'Details'})">Details</walkthrough-spotlight-pointer>. Si `agy` incluyó el campo `description` en la cabecera SQLX del modelo de Dataform, aparecerá aquí publicado: la documentación escrita como código (*documentation-as-code*) viaja directamente hasta el catálogo corporativo.

### Audita la calidad de la capa Bronze con Auto Data Quality

En el Lab 3 tus *assertions* validaron lo que **Dataform** construía en tiempo de compilación/ejecución. Un **Data Quality Scan** de **Knowledge Catalog**, en cambio, audita una tabla de forma independiente a cualquier pipeline — bajo demanda o de manera programada — y publica la puntuación oficial de calidad directamente en la ficha de la tabla en el catálogo. Vamos a lanzar un escaneo sobre `cymbal_bronze.cymbal_orders`, donde sabemos que residen los errores del sistema origen.

Las reglas de calidad están definidas en un fichero YAML declarativo: <walkthrough-editor-open-file filePath="content/agenticdata/src/governance/orders_quality.yaml">orders_quality.yaml</walkthrough-editor-open-file>. Contiene dos reglas equivalentes a las que exige la capa *silver*: que `status` pertenezca al conjunto de seis estados válidos y que `order_ts` no tenga fechas en el futuro.

El escaneo se ejecuta con la cuenta de servicio `dataquality-service-account`, ya pre-aprovisionada en tu proyecto con permisos de lectura en BigQuery. Primero, autoriza al *service agent* de **Knowledge Catalog** para actuar en nombre de esa cuenta de servicio:

```bash
gcloud iam service-accounts add-iam-policy-binding dataquality-service-account@{{ PROJECT_ID }}.iam.gserviceaccount.com \
    --member=serviceAccount:service-{{ PROJECT_NUMBER }}@gcp-sa-dataplex.iam.gserviceaccount.com \
    --role=roles/iam.serviceAccountTokenCreator
```

Los cambios de IAM tardan alrededor de un minuto en propagarse. Si el comando de creación del escaneo que aparece justo debajo devolviera un error de permisos sobre la *service account*, espera unos segundos y vuelve a lanzarlo.

A continuación, crea el *Data Quality Scan* `bronzeorders`:

```bash
cd ~/bootkon
gcloud dataplex datascans create data-quality bronzeorders \
    --location={{ REGION }} \
    --data-source-resource=//bigquery.googleapis.com/projects/{{ PROJECT_ID }}/datasets/cymbal_bronze/tables/cymbal_orders \
    --data-quality-spec-file=content/agenticdata/src/governance/orders_quality.yaml \
    --service-account=dataquality-service-account@{{ PROJECT_ID }}.iam.gserviceaccount.com
```

Un detalle sobre los parámetros: tus datasets están en la multi-región `us`, por lo que el escaneo puede residir en cualquier región dentro de ese ámbito (`{{ REGION }}`). Al no especificar `--schedule`, el escaneo se ejecuta bajo demanda. Además, la propiedad `catalogPublishingEnabled: true` del fichero YAML publica automáticamente los resultados en la pestaña *Data quality* de la tabla en **BigQuery** y **Knowledge Catalog**.

Lanza ahora la ejecución del escaneo:

```bash
gcloud dataplex datascans run bronzeorders --location={{ REGION }}
```

El trabajo tarda entre dos y tres minutos en completarse. Pasado ese tiempo, ve a [Knowledge Catalog search](https://console.cloud.google.com/dataplex/dp-search) y busca `cymbal_orders` y haz clic en la pestaña <walkthrough-spotlight-pointer locator="semantic({tab 'Data quality'})">Data quality</walkthrough-spotlight-pointer> (refresca la página del navegador si aún aparece vacía).

**Verás que el escaneo falla — y ese es exactamente el objetivo.** Un porcentaje de las filas incumple la regla de `status` (por el error tipográfico `shiped`) y otro grupo incumple la regla de `order_ts` con fechas futuras. Tu pipeline de **Dataform** ya limpia estos defectos al pasar a *silver*, pero gracias a este escaneo el catálogo deja constancia visible de que `cymbal_bronze` contiene datos en bruto no aptos para consumo directo por usuarios o agentes.

### Protege los datos personales (PII) a nivel de columna

La columna `stg_customers.email` contiene información personal identificable (*PII*). Vamos a protegerla aplicando seguridad a nivel de columna (*Column-Level Security*) mediante una **Policy Tag**:

1. Abre [BigQuery policy tags](https://console.cloud.google.com/bigquery/policy-tags) y haz clic en <walkthrough-spotlight-pointer locator="text('Create taxonomy')">Create Taxonomy</walkthrough-spotlight-pointer>:
    - **Taxonomy name**: `cymbal`, **Location**: `us`
    - **Policy tag**: `pii`, **Description**: `Personal data — restricted`
2. Pulsa **Create** y, una vez creada la taxonomía, activa el interruptor **`Enforce access control`**.
3. Ve a [BigQuery](https://console.cloud.google.com/bigquery), abre `cymbal_silver` → `stg_customers` → haz clic en <walkthrough-spotlight-pointer locator="semantic({button 'Edit schema'})">Edit schema</walkthrough-spotlight-pointer>, selecciona la casilla de la columna `email`, pulsa **Add policy tag**, elige `cymbal > pii` y pulsa **Save**.

Ahora comprobemos que el bloqueo es efectivo. Ejecuta esta consulta en el editor de BigQuery — **debe fallar** con un error de acceso denegado (*Access Denied*) sobre la columna `email`:

```sql
SELECT email FROM `{{ PROJECT_ID }}.cymbal_silver.stg_customers` LIMIT 5
```

En cambio, esta segunda consulta excluyendo `email` funciona sin ningún problema:

```sql
SELECT * EXCEPT (email) FROM `{{ PROJECT_ID }}.cymbal_silver.stg_customers` LIMIT 5
```

Fíjate en la implicación de seguridad: incluso siendo propietario/administrador del proyecto en BigQuery, no puedes leer esa columna a menos que tengas explícitamente concedido el rol **Fine-Grained Reader** sobre el *policy tag*. Esa es la garantía de gobierno que necesitas antes de conectar agentes de IA a tu plataforma de datos.

❗ *Nota*: Si más adelante vuelves a ejecutar `dataform run --tags silver`, Dataform recreará la tabla `stg_customers` desde cero y eliminará la etiqueta asignada manualmente desde la consola; si quieres conservarla tras re-ejecutar Dataform, basta con volver a asociarla en *Edit schema* (o declararla directamente en el bloque `config` del fichero `.sqlx`).

### Reto opcional: demuestra que la capa Silver pasa el control de calidad al 100%

**\[RETO\]** Si dispones de 5 minutos extra:

1. Crea un segundo escaneo apuntando a `cymbal_silver.stg_orders` reutilizando el mismo fichero `orders_quality.yaml` y llamándolo `silverorders` (solo cambian el nombre del scan y el parámetro `--data-source-resource`). Ejecútalo y compara la puntuación del 100% en la pestaña *Data quality* de `stg_orders` frente al fallo en `cymbal_orders`.
2. Comenta con tu mesa: si `silver` pasa siempre al 100%, ¿por qué sigue siendo buena práctica mantener un escaneo de calidad sobre `bronze` en producción?

### ¡Completado!

🎉 ¡Magnífico{% if MY_NAME %}, {{ MY_NAME }}{% endif %}! Tu plataforma ahora se explica y se protege por sí sola: todas las tablas son localizables sin registro manual, la capa *bronze* exhibe su indicador real de calidad y los correos electrónicos de los clientes están blindados a nivel de columna incluso frente a administradores. Con el gobierno listo, ¡es la hora de los agentes de IA! 🛡️
