## Lab 2: Replicación en Tiempo Real con Datastream

<walkthrough-tutorial-duration duration="40"></walkthrough-tutorial-duration>
{{ author('Fabian Hirschmann', 'https://linkedin.com/in/fhirschmann') }}
<walkthrough-tutorial-difficulty difficulty="2"></walkthrough-tutorial-difficulty>
<bootkon-cloud-shell-note/>

En este laboratorio darás vida a la base de datos operacional de Cymbal y la replicarás de forma continua hacia **BigQuery** utilizando *Change Data Capture (CDC)* con **Datastream**. Al terminar este bloque, cada `INSERT`, `UPDATE` y `DELETE` que ocurra en **PostgreSQL** aterrizará en tu dataset `cymbal_bronze` en cuestión de segundos.

*(Los comandos siguientes ya incluyen las contraseñas de base de datos generadas para tu entorno e inyectadas en el tutorial. Si en algún comando vieras comillas vacías `""`, ejecuta el comando `. bk && bk-start` del final del Lab 1 para refrescarlas).*

### Acerca de Datastream

**Datastream** es el servicio *serverless* de **Change Data Capture (CDC)** de Google Cloud: en lugar de exportar tablas completas periódicamente por lotes, se conecta directamente al log de transacciones de la base de datos origen y replica cada `INSERT`, `UPDATE` y `DELETE` a medida que sucede. En **PostgreSQL**, ese registro es el **write-ahead log (WAL)**: Postgres emite cada cambio confirmado en formato *lógico*, y Datastream consume ese flujo mediante un *replication slot*. Un stream de Datastream combina dos fases que operan en paralelo — un **backfill** inicial (instantánea de las filas existentes) y **CDC streaming** continuo — escribiendo directamente en **BigQuery** sin que tengas que mantener código de ingesta ni gestionar clústeres.

Para saber más:
- [Datastream overview](https://docs.cloud.google.com/datastream/docs/overview)
- [PostgreSQL as a source](https://docs.cloud.google.com/datastream/docs/sources-postgresql)

### De vuelta a la terminal

En el Lab 1 dejamos a `agy` abierto en la terminal, pero los primeros comandos de este laboratorio son **comandos de shell**, así que sal primero de `agy` escribiendo `/quit` (volverás a invocar a tu co-ingeniero un poco más adelante cuando toque redactar los ficheros de configuración).

### El endpoint de Private Service Connect

Repasemos brevemente la conectividad privada que tienes desplegada hoy: tu instancia de **Cloud SQL** expone un **service attachment** — un conector privado al que pueden enchufarse redes autorizadas — y tu VPC cuenta con el **endpoint** correspondiente en la IP interna `10.10.0.5` (`cymbal-endpoint`, reservada durante el aprovisionamiento). Esa dirección IP **es** tu base de datos tanto para **Datastream** (cuya conexión privada `cymbal-psc` ya está conectada a tu VPC) como para la *jump VM*.

### La Jump VM

Como **Cloud Shell** se ejecuta fuera de tu VPC y una instancia de Cloud SQL *PSC-only* no tiene IP pública, tu proyecto incluye un pequeño puente: `cymbal-jump`, una máquina virtual `e2-micro` (¡también sin IP externa!) que reenvía el puerto `5432` hacia el endpoint `10.10.0.5` de la base de datos. Su <walkthrough-editor-open-file filePath="content/agenticdata/src/jumpvm-startup.sh">startup script</walkthrough-editor-open-file> consta únicamente de cuatro líneas de `iptables`. Te conectas a ella mediante un **túnel IAP** — basado en identidad de Google Cloud y sin exponer IPs en internet.

### El túnel IAP

El puerto `localhost:5432` dentro de tu **Cloud Shell** apunta directamente a la base de datos de Cymbal sin que tengas que configurar nada: al iniciar Bootkon se abrió en segundo plano un túnel **IAP** hacia la *jump VM*, y esta redirige el tráfico al endpoint PSC en `10.10.0.5`.

El reenvío TCP de **Identity-Aware Proxy (IAP)** hace que este acceso sea completamente seguro: el túnel se autoriza mediante tu identidad de Google y un rol de IAM (`iap.tunnelResourceAccessor`), no por ubicación de red — sin VPN, sin bastiones con IP pública y con auditoría completa de cada conexión ([IAP TCP forwarding](https://docs.cloud.google.com/iap/docs/using-tcp-forwarding)).

❗ Si en algún momento alguno de los comandos `psql` siguientes devuelve *connection refused*, significa que el túnel en segundo plano se ha cerrado (por ejemplo, tras una pausa larga en la que Cloud Shell se haya reciclado). Ejecuta `. bk` para levantarlo de nuevo, o lánzalo manualmente(registra su salida en `~/.bootkon-tunnel.log`):

```bash
~/bootkon/content/agenticdata/bk-tunnel
```

### El histórico de Cymbal, ya cargado en la base de datos

La base de datos `cymbal`, su esquema y el medio millón de pedidos históricos ya se cargaron en tu instancia durante el aprovisionamiento mediante una **importación server-side desde Cloud Storage**, tal y como se realiza una carga inicial masiva en producción: la propia instancia de Cloud SQL descarga los CSV usando su identidad gestionada de Google, sin pasar por el portátil de nadie ni por ningún túnel.

Abre <walkthrough-editor-open-file filePath="content/agenticdata/src/datagen/schema.sql">schema.sql</walkthrough-editor-open-file> en el editor: observa que todas las tablas tienen clave primaria (*primary key*, necesaria para el *merge mode* de Datastream) y que deliberadamente no se han definido claves foráneas (*foreign keys*) para permitir que entren datos sucios que limpiaremos en la capa *silver*.

### Verifica los datos en la consola con Cloud SQL Studio

Vamos a consultar la base de datos desde la consola web: abre [Cloud SQL Studio](https://console.cloud.google.com/sql/instances/cymbal-oltp/studio) e inicia sesión con el usuario `postgres` y la contraseña `{{ BK_DB_PASSWORD }}` — **muy importante**: en el desplegable **Database**, cambia el valor preseleccionado `postgres` por `cymbal` (si entras en la base de datos `postgres` por defecto, luego verás el error *"relation cymbal.orders does not exist"*). Una vez dentro, ejecuta:

```sql
SELECT COUNT(*) AS total_orders FROM cymbal.orders;
```

Verás `500000` pedidos: el histórico operacional de Cymbal viviendo en **tu** base de datos PostgreSQL. Puedes desplegar el árbol de la izquierda para explorar las seis tablas del esquema `cymbal`.

### Preparar la replicación lógica en PostgreSQL

**Datastream** lee el *write-ahead log (WAL)* de PostgreSQL a través de una **publication** y un **replication slot**, utilizando un usuario de base de datos dedicado a la replicación. La *publication* define *qué* tablas exponen sus cambios (`FOR ALL TABLES` en nuestro caso), mientras que el *replication slot* registra *hasta qué punto* ha leído el consumidor: PostgreSQL conserva los segmentos de WAL hasta que el slot confirma su lectura, garantizando que Datastream no pierda ningún cambio ni siquiera ante desconexiones o reinicios ([Configure a Cloud SQL for PostgreSQL source](https://docs.cloud.google.com/datastream/docs/configure-cloudsql-psql)).

Inspecciona el archivo <walkthrough-editor-open-file filePath="content/agenticdata/src/datastream/replication_setup.sql">replication_setup.sql</walkthrough-editor-open-file> y ejecútalo desde tu terminal:

```bash
PGPASSWORD="{{ BK_DB_PASSWORD }}" psql -h localhost -p 5432 -U postgres -d cymbal \
    -v ds_password="{{ BK_DS_PASSWORD }}" \
    -f content/agenticdata/src/datastream/replication_setup.sql
```

### Aprovisionar el stream en Datastream

Un stream de **Datastream** requiere dos **connection profiles**: uno para el origen y otro para el destino. El perfil de destino ya está creado en tu proyecto (`cymbal-bq-profile`), al igual que el dataset vacío `cymbal_bronze` en **BigQuery** donde aterrizarán las tablas. Compruébalo ahora mientras sigue vacío: abre [BigQuery](https://console.cloud.google.com/bigquery), despliega tu proyecto <walkthrough-spotlight-pointer locator="semantic({treeitem 'Toggle node {{ PROJECT_ID }}'} {button 'Toggle node'})">{{ PROJECT_ID }}</walkthrough-spotlight-pointer> y revisa su sección **Datasets**.

Ahora vas a crear tú el **connection profile de origen** hacia PostgreSQL con la contraseña de replicación generada para tu entorno. Fíjate en el `--postgresql-hostname`: pasamos directamente la IP `10.10.0.5` del *PSC endpoint*, ya que las conexiones privadas de Datastream se conectan por IP directa:

```bash
gcloud datastream connection-profiles create cymbal-postgres-profile --location={{ REGION }} \
    --type=postgresql --display-name=cymbal-postgres-profile \
    --postgresql-hostname=10.10.0.5 --postgresql-port=5432 \
    --postgresql-username=datastream_user --postgresql-password="{{ BK_DS_PASSWORD }}" \
    --postgresql-database=cymbal --private-connection=cymbal-psc
```

El comportamiento del stream se define mediante dos ficheros JSON incluidos en el repositorio dentro de `content/agenticdata/src/datastream/` (el fichero de destino incluye todavía un *placeholder*). Redactar configuraciones JSON conformes al esquema exacto de la API es tarea de autoría para tu co-ingeniero de IA, así que abre **`agy`** en la terminal:

```bash
cd ~/bootkon && agy
```

Y pásale este encargo con `/goal`:

```bash
/goal Rewrite content/agenticdata/src/datastream/source_config.json and content/agenticdata/src/datastream/destination_config.json for gcloud datastream streams create. The PostgreSQL source uses publication "cymbal_pub", replication slot "cymbal_slot", and should include all tables of the "cymbal" schema. The BigQuery destination writes every table into the single dataset "{{ PROJECT_ID }}:cymbal_bronze" with a data freshness of 0 seconds. Use the exact JSON field names of the Datastream v1 API (publication, replicationSlot, includeObjects/postgresqlSchemas, singleTargetDataset/datasetId, dataFreshness).
```

**Revisa lo que ha escrito `agy`**: abre <walkthrough-editor-open-file filePath="content/agenticdata/src/datastream/source_config.json">source_config.json</walkthrough-editor-open-file> y <walkthrough-editor-open-file filePath="content/agenticdata/src/datastream/destination_config.json">destination_config.json</walkthrough-editor-open-file>. Si por cualquier motivo `agy` se hubiera desviado del formato esperado, siempre tienes el respaldo de `git` para restaurar los ficheros originales y sustituir el *placeholder* directamente:

```bash
git -C ~/bootkon restore content/agenticdata/src/datastream/
sed -i "s/PROJECT_ID_PLACEHOLDER/{{ PROJECT_ID }}/" content/agenticdata/src/datastream/destination_config.json
```

Los siguientes comandos se ejecutan **en la shell normal, no dentro de `agy`** — sal primero de `agy` escribiendo `/quit`. A continuación, crea el stream `cymbal-cdc-stream` activando el *backfill* completo de los datos semilla:

```bash
gcloud datastream streams create cymbal-cdc-stream --location={{ REGION }} \
    --display-name=cymbal-cdc-stream \
    --source=cymbal-postgres-profile --postgresql-source-config=content/agenticdata/src/datastream/source_config.json \
    --destination=cymbal-bq-profile --bigquery-destination-config=content/agenticdata/src/datastream/destination_config.json \
    --backfill-all
```

La creación tarda un par de minutos; puedes observar el progreso en la [consola de Datastream](https://console.cloud.google.com/datastream/streams), donde el stream aparecerá primero como **Creating** y luego como **Not started**. Antes de arrancarlo, ejecuta este script auxiliar que espera hasta que la operación interna de creación haya finalizado por completo en la API:

```bash
content/agenticdata/bk-wait-stream
```

Ahora cambia el estado del stream a `RUNNING` para iniciar la replicación:

```bash
gcloud datastream streams update cymbal-cdc-stream --location={{ REGION }} \
    --state=RUNNING --update-mask=state
```

En este instante **Datastream** valida de extremo a extremo la configuración (*logical decoding*, *replication slot*, *publication*, permisos y conectividad PSC) y lanza el *backfill*. El stream pasará a estado **Running** en unos dos minutos, y en aproximadamente un minuto más las seis tablas con los 2,3 millones de registros habrán aterrizado en **BigQuery**.

Conviene destacar dos decisiones de arquitectura de tu `destination_config.json`:
1. El stream opera en **merge mode**, lo que significa que cada evento de cambio se fusiona (*upsert*) en la tabla de **BigQuery** (utilizando el soporte nativo de CDC de la *BigQuery Storage Write API*), de forma que `cymbal_bronze` refleja siempre el *estado actual* de PostgreSQL (la alternativa, *append-only*, conservaría cada evento como una fila histórica inmutable).
2. `dataFreshness: "0s"` indica a **BigQuery** que aplique los cambios pendientes en el momento de consultar (*query-time merge*) en lugar de esperar a ventanas periódicas de compactación, logrando que la demostración en tiempo real que haremos a continuación sea instantánea.

Para saber más:
- [BigQuery as a destination (merge vs. append-only)](https://docs.cloud.google.com/datastream/docs/destination-bigquery)
- [BigQuery change data capture](https://docs.cloud.google.com/bigquery/docs/change-data-capture)

### Observa el flujo de datos en tiempo real

Abre [Datastream](https://console.cloud.google.com/datastream/streams) y haz clic en <walkthrough-spotlight-pointer locator="text('cymbal-cdc-stream')">cymbal-cdc-stream</walkthrough-spotlight-pointer>. Entra en la pestaña <walkthrough-spotlight-pointer locator="text('Objects')">Objects</walkthrough-spotlight-pointer> para ver el progreso del *backfill* tabla por tabla. En un par de minutos el stream mostrará **Running** y las seis tablas `cymbal_*` aparecerán en BigQuery.

Ahora vamos a poner el sistema **en vivo**. Abre una **segunda pestaña de terminal** pulsando el icono `+` en la barra de la terminal, y lanza el simulador de actividad transaccional de clientes de Cymbal:

```bash
cd ~/bootkon
python3 content/agenticdata/src/datagen/simulate.py
```

**Déjalo ejecutándose en esa segunda pestaña**. Ve a [BigQuery](https://console.cloud.google.com/bigquery), despliega <walkthrough-spotlight-pointer locator="semantic({treeitem 'Toggle node {{ PROJECT_ID }}'} {button 'Toggle node'})">{{ PROJECT_ID }}</walkthrough-spotlight-pointer> → **Datasets** → `cymbal_bronze`, y ejecuta esta consulta varias veces en el editor SQL (usa siempre el editor de consultas y **no** la pestaña *Preview*, ya que la vista previa no ejecuta el merge en tiempo real de CDC):

```sql
SELECT MAX(order_id) AS latest_order, COUNT(*) AS total_orders
FROM `{{ PROJECT_ID }}.cymbal_bronze.cymbal_orders`
```

Verás cómo `latest_order` y `total_orders` suben cada pocos segundos a medida que el simulador inserta nuevos pedidos en PostgreSQL.

Por último, comprobemos que las actualizaciones (`UPDATE`) también se replican de extremo a extremo. Vuelve a tu **pestaña 1 de la terminal** y ejecuta un `UPDATE` directo en PostgreSQL:

```bash
PGPASSWORD="{{ BK_DB_PASSWORD }}" psql -h localhost -p 5432 -U postgres -d cymbal \
    -c "UPDATE cymbal.customers SET country = 'Iceland', updated_at = now() WHERE customer_id = 42;"
```

*(También puedes lanzar ese mismo `UPDATE` desde Cloud SQL Studio si lo sigues teniendo abierto).*

Ahora ve a **BigQuery** (dale entre 30 y 60 segundos para que el evento cruce el stream) y ejecuta:

```sql
SELECT customer_id, full_name, country
FROM `{{ PROJECT_ID }}.cymbal_bronze.cymbal_customers`
WHERE customer_id = 42
```

El cliente `42` acaba de mudarse a `Iceland` — desde un `UPDATE` transaccional en PostgreSQL hasta la fila actualizada en BigQuery, sin procesos batch ni intervención manual. Así funciona el *merge-mode CDC*.

### ¡Completado!

🎉 ¡Excelente trabajo{% if MY_NAME %}, {{ MY_NAME }}{% endif %}! Has puesto en marcha un pipeline CDC empresarial de extremo a extremo: una instancia PostgreSQL 100% privada por PSC, un túnel basado en identidad con IAP, replicación lógica sobre el WAL y un stream de **Datastream** que refleja cada cambio en **BigQuery** en tiempo prácticamente real. Tu capa *bronze* ya está viva; en el siguiente laboratorio vamos a refinarla. 🥉→🥈
