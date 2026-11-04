## Lab 10: Operaciones Día 2 (Day-2 Ops) — Dashboards, Alertas y Respuesta a Incidentes con `agy`

<walkthrough-tutorial-duration duration="45"></walkthrough-tutorial-duration>
{{ author('Fabian Hirschmann', 'https://linkedin.com/in/fhirschmann') }}
<walkthrough-tutorial-difficulty difficulty="3"></walkthrough-tutorial-difficulty>
<bootkon-cloud-shell-note/>
<!-- Optional module: not wired into TUTORIAL.md. Renumber the "Lab N" title to match the event agenda when including. Not yet smoke-tested. -->

Ya has construido tu plataforma de datos. Pero ¿qué ocurre si esta noche a las 3 de la madrugada se interrumpe la replicación y nadie se entera hasta la reunión del lunes? En este laboratorio opcional asumirás el rol de ingeniero de guardia (*on-call*) de Cymbal: construirás un **dashboard de Cloud Monitoring como código** (generado por `agy`), configurarás una **política de alertas de SLO sobre la frescura del dato**, y a continuación — deliberadamente — **romperás tu propia replicación CDC** ejecutando un script de caos sin leer antes su código fuente. Observarás el incidente en tu cuadro de mando, diagnosticarás la causa raíz junto a `agy` actuando como co-respondedor en modo de solo lectura, aplicarás el arreglo en una sola línea de SQL y demostrarás que no se ha perdido ni una sola fila durante la caída.

Este laboratorio asume que has completado los **Labs 1 a 3** (`cymbal-cdc-stream` replicando en `cymbal_bronze` y el Medallion construido) y que **el simulador de pedidos (pestaña 2 de la terminal)** sigue ejecutándose en vivo.

### Acerca de las Operaciones de Día 2 (Day-2 Ops)

El Día 1 consiste en construir el pipeline; el **Día 2** consiste en observarlo, alertar cuando se degrada y recuperarlo sin pérdida de datos. **Cloud Monitoring** proporciona las tres piezas clave:
- **Metrics**: series temporales emitidas automáticamente por cada servicio gestionado — **Datastream** reporta cuántos segundos de retraso lleva respecto al origen (**data freshness**), **Cloud SQL** reporta conexiones y uso de CPU, y la *BigQuery Storage Write API* reporta las filas ingeridas.
- **Dashboards as code**: como un dashboard en Cloud Monitoring es un documento JSON desplegable por API, puedes generarlo con `agy` y versionarlo en Git igual que el resto de tu plataforma.
- **Alerting policies**: vigilan una métrica frente a un umbral de SLO y abren un **incidente** automáticamente cuando se vulnera.

Para saber más:
- [Monitor a Datastream stream](https://docs.cloud.google.com/datastream/docs/monitor-a-stream)
- [Dashboards via the API](https://docs.cloud.google.com/monitoring/dashboards/api-dashboard)
- [Alerting overview](https://docs.cloud.google.com/monitoring/alerts)

### Prepara el entorno de observabilidad

Sitúa primero tu pestaña 1 de la terminal en la raíz del repositorio `~/bootkon`:

```bash
cd ~/bootkon
```

Habilita las APIs de **Cloud Monitoring** y **Cloud Logging** y asegúrate de contar con los roles correspondientes:

```bash
gcloud services enable monitoring.googleapis.com logging.googleapis.com
gcloud projects add-iam-policy-binding {{ PROJECT_ID }} \
    --member="user:{{ GCP_USERNAME }}" --role="roles/monitoring.editor" --condition=None >/dev/null
gcloud projects add-iam-policy-binding {{ PROJECT_ID }} \
    --member="user:{{ GCP_USERNAME }}" --role="roles/logging.viewer" --condition=None >/dev/null
```

Comprueba antes de empezar que tu stream `cymbal-cdc-stream` devuelve estado `RUNNING`:

```bash
gcloud datastream streams describe cymbal-cdc-stream --location={{ REGION }} --format='value(state)'
```

### Genera el Dashboard como Código con `agy`

La especificación completa del cuadro de mando — qué seis gráficas debe incluir, los nombres exactos de las métricas y el umbral de SLO de 120 segundos — está documentada en <walkthrough-editor-open-file filePath="content/agenticdata/src/optional/day2_ops/AGENTS.md">AGENTS.md</walkthrough-editor-open-file>.

Elimina primero el archivo de referencia para que `agy` lo construya desde cero:

```bash
rm content/agenticdata/src/optional/day2_ops/dashboard.json
```

Inicia `agy`:

```bash
cd ~/bootkon && agy
```

Y pásale el encargo:

```bash
/goal Read content/agenticdata/src/optional/day2_ops/AGENTS.md and create content/agenticdata/src/optional/day2_ops/dashboard.json exactly as it specifies: valid Cloud Monitoring Dashboard API JSON, six widgets in the given order, the exact metric types from the brief, and the 120-second threshold line on the freshness chart. No comments, no trailing commas.
```

**Revisa el JSON generado**: abre <walkthrough-editor-open-file filePath="content/agenticdata/src/optional/day2_ops/dashboard.json">dashboard.json</walkthrough-editor-open-file>. Si prefieres restaurar el archivo de referencia probado en cualquier momento, tienes el respaldo de `git`:

```bash
git -C ~/bootkon restore content/agenticdata/src/optional/day2_ops/dashboard.json
```

### Despliega el Dashboard en Cloud Monitoring

Sal de `agy` con `/quit` y despliega el cuadro de mando con un único comando de `gcloud`:

```bash
gcloud monitoring dashboards create \
    --config-from-file=content/agenticdata/src/optional/day2_ops/dashboard.json
```

Abre [Monitoring → Dashboards](https://console.cloud.google.com/monitoring/dashboards) y haz clic en <walkthrough-spotlight-pointer locator="text('Cymbal Day-2 Ops')">Cymbal Day-2 Ops</walkthrough-spotlight-pointer>. Ajusta la ventana temporal a **1 hour** y observa las constantes vitales de tu pipeline en tiempo real:
- **Datastream — data freshness** se mantiene pegada a cero, muy por debajo de la línea roja del SLO de 120 segundos.
- **Datastream — throughput** muestra el goteo continuo de eventos generados por el simulador.
- **Cloud SQL — connections/CPU** reflejan una carga estable.
- **BigQuery Storage Write API — ingested rows** muestra las filas que van aterrizando en `cymbal_bronze`.

### Crea la política de alerta de SLO (120 segundos)

Para que el sistema avise automáticamente si `cymbal_bronze` se queda más de dos minutos por detrás de PostgreSQL:

1. Abre [Datastream](https://console.cloud.google.com/datastream/streams) y entra en <walkthrough-spotlight-pointer locator="text('cymbal-cdc-stream')">cymbal-cdc-stream</walkthrough-spotlight-pointer>.
2. Busca la gráfica **Data freshness** y haz clic en **Create alerting policy** (o ve directamente a [Monitoring → Alerting](https://console.cloud.google.com/monitoring/alerting) → **Create policy** → métrica **Datastream Stream → Stream freshness**).
3. Configura la condición: tipo **Threshold**, **Above threshold**, valor `120` (segundos).
4. En el paso de notificaciones, omite los canales de correo y asigna a la política el nombre `cymbal-freshness-slo`. Pulsa **Create**.

### Provoca el incidente (Chaos Engineering)

El repositorio incluye un script de caos. La regla del ejercicio: **ejecútalo ahora sin abrir antes su código fuente** — en un incidente real de producción no recibes de antemano el script que causó el fallo:

```bash
content/agenticdata/src/optional/day2_ops/chaos.sh
```

Anota la hora que imprime por pantalla; tu línea temporal del incidente acaba de comenzar.

### Primeras señales en el Dashboard

Vuelve a tu dashboard **Cymbal Day-2 Ops** durante los siguientes dos minutos y observa los síntomas:
- **Data freshness** deja de estar cerca de cero y empieza a subir en línea recta hacia el umbral rojo de 120 s.
- **Throughput** cae a cero: ya no entran eventos desde el origen.
- Sin embargo, **las conexiones y CPU de Cloud SQL siguen activas** y en la pestaña 2 de tu terminal el simulador sigue insertando pedidos sin ningún error. Es decir: **PostgreSQL de producción está sano; solo se ha roto el carril de replicación**.

Si abres [Datastream](https://console.cloud.google.com/datastream/streams) → <walkthrough-spotlight-pointer locator="text('cymbal-cdc-stream')">cymbal-cdc-stream</walkthrough-spotlight-pointer>, verás cómo el estado del stream pasa de **Running** a **Failed**.

### Diagnostica la causa raíz con tu co-respondedor (`agy`)

Vamos a investigar el incidente junto a `agy` en modo **co-respondedor de solo lectura**: le pediremos que diagnostique el fallo usando únicamente comandos de lectura (`gcloud datastream streams describe`, `gcloud logging read`) y que nos pida permiso antes de lanzar cada comando. Inicia `agy`:

```bash
cd ~/bootkon && agy
```

Y pásale el síntoma del incidente:

```bash
Our BigQuery bronze dataset stopped receiving new rows a few minutes ago, and data freshness on the Datastream stream cymbal-cdc-stream (location us-central1) keeps climbing. The source Postgres itself is healthy and still taking writes. Investigate the Datastream side with read-only commands only (gcloud datastream streams describe, gcloud logging read) and tell me your diagnosis. Ask me before each command, and do not change anything.
```

Aprueba los comandos de lectura que te proponga `agy` (o si prefieres ejecutarlos tú directamente tras salir con `/quit`, son estos dos):

```bash
gcloud datastream streams describe cymbal-cdc-stream --location={{ REGION }} \
    --format='yaml(state, errors)'
```

```bash
gcloud logging read 'resource.type="datastream.googleapis.com/Stream" severity>=WARNING' \
    --freshness=15m --limit=5 --format='yaml(timestamp, severity, jsonPayload)'
```

En los detalles del error verás la causa exacta devuelta por PostgreSQL: `role "datastream_user" is not permitted to log in`. El usuario de replicación que utiliza Datastream desde el Lab 2 ha perdido el permiso de inicio de sesión (`NOLOGIN`), mientras que el tráfico operacional del usuario `postgres` no se vio afectado.

Ahora que ya tienes el diagnóstico, abre <walkthrough-editor-open-file filePath="content/agenticdata/src/optional/day2_ops/chaos.sh">chaos.sh</walkthrough-editor-open-file> y pídele a `agy` que te explique línea por línea qué hizo el script:

```bash
cd ~/bootkon && agy
```

```bash
Read content/agenticdata/src/optional/day2_ops/chaos.sh and explain, statement by statement, how it broke replication -- and why the simulator kept writing happily the whole time.
```

Observa lo que el script de caos **no** tocó: ni la publicación `cymbal_pub` ni el slot de replicación `cymbal_slot`. Gracias a eso, PostgreSQL ha seguido guardando en el *WAL* todos los pedidos ocurridos durante la caída.

### Resuelve el incidente en una línea y verifica Pérdida de Datos Cero

Abre una pestaña de terminal y restaura el permiso `LOGIN` al usuario `datastream_user`:

```bash
PGPASSWORD="{{ BK_DB_PASSWORD }}" psql -h localhost -p 5432 -U postgres -d cymbal \
    -c "ALTER ROLE datastream_user LOGIN;"
```

❗ **Datastream** reintenta las conexiones fallidas automáticamente con *backoff*, por lo que en un par de minutos verás en la [página del stream](https://console.cloud.google.com/datastream/streams) que vuelve por sí solo a estado **Running**, y en tu dashboard la curva de **Data freshness** caerá en picado de vuelta a cero.

Por último, respondamos a la pregunta más importante tras cualquier incidente: **¿hemos perdido algún pedido emitido durante la caída?** Comprueba el último `order_id` en PostgreSQL:

```bash
PGPASSWORD="{{ BK_DB_PASSWORD }}" psql -h localhost -p 5432 -U postgres -d cymbal \
    -c "SELECT MAX(order_id) FROM cymbal.orders;"
```

Y compáralo con el último `order_id` replicado en **BigQuery**:

```sql
SELECT MAX(order_id) AS latest_order, COUNT(*) AS total_orders
FROM `{{ PROJECT_ID }}.cymbal_bronze.cymbal_orders`
```

En cuanto el stream recupera el retraso, ambos valores de `MAX(order_id)` coinciden: **cero filas perdidas**. El *replication slot* `cymbal_slot` retuvo el *WAL* durante toda la interrupción y **Datastream** retomó la lectura exactamente en el punto donde se había detenido. Además, en [Monitoring → Alerting](https://console.cloud.google.com/monitoring/alerting) verás cómo el incidente de `cymbal-freshness-slo` se cierra automáticamente en cuanto el SLO vuelve a cumplirse.

### ¡Completado!

🎉 ¡Fenomenal{% if MY_NAME %}, {{ MY_NAME }}{% endif %}! Has dotado a tu plataforma de observabilidad *as-code*, alertas de SLO sobre la frescura del dato y un flujo de respuesta a incidentes asistido por `agy` con pérdida de datos cero. 📟✅
