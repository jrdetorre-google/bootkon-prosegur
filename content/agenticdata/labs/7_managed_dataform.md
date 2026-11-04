## Lab 7: De la Terminal a la Planificación en Producción — El Medallion en Managed Dataform

<walkthrough-tutorial-duration duration="40"></walkthrough-tutorial-duration>
{{ author('Fabian Hirschmann', 'https://linkedin.com/in/fhirschmann') }}
<walkthrough-tutorial-difficulty difficulty="2"></walkthrough-tutorial-difficulty>
<bootkon-cloud-shell-note/>
<!-- Optional module: not wired into TUTORIAL.md. Not yet smoke-tested.
     When including: renumber the "Lab N" title to match the event agenda, merge
     src/optional/managed_dataform/prompts.md into src/prompts.md, and add the
     40 min to the README agenda table. Placement: slot this module AFTER Lab 4
     (governance) — once the 15-minute schedule is live, every rebuild removes
     console-attached column metadata such as Lab 4's hand-attached policy tags. -->

El Lab 3 terminó con una premisa importante: *"En producción ejecutarás exactamente este mismo proyecto de forma planificada en Managed Dataform"*. En este módulo opcional llevarás ese paso a la práctica: subirás el proyecto exacto que escribió `agy` — sin reescribir ni una línea, los mismos archivos SQLX — a un **repositorio gestionado de Dataform**, explorarás su grafo interactivo de dependencias (DAG) y configurarás una ejecución automatizada cada 15 minutos bajo una *service account* dedicada. Al salir de este laboratorio, tu arquitectura Medallion se refrescará por sí sola.

Antes de empezar: este laboratorio asume que has completado los **Labs 1 a 3** (el proyecto de Dataform en `content/agenticdata/src/dataform` compila y se ha ejecutado al menos una vez) y que el **simulador de datos (pestaña 2 de la terminal)** del Lab 2 sigue en ejecución.

### Acerca de Managed Dataform

**Managed Dataform** es el plano gestionado en la nube de la herramienta que utilizaste en el Lab 3 — mismo motor de Dataform, mismo código SQLX y mismo `workflow_settings.yaml`, pero operado íntegramente por Google Cloud:
- Un **repository** es un repositorio Git alojado por Google (o conectado a GitHub/GitLab).
- Los **development workspaces** son ramas de edición individuales con compilación en vivo en el navegador.
- Una **release configuration** compila el código de una rama Git produciendo un **compilation result** inmutable.
- Y una **workflow configuration** ejecuta esos resultados de compilación según un calendario cron utilizando una *service account* sin depender del portátil ni de la sesión de ningún desarrollador.

Para saber más:
- [Create a Dataform repository](https://docs.cloud.google.com/dataform/docs/create-repository)
- [Release configurations](https://docs.cloud.google.com/dataform/docs/release-configurations)
- [Workflow configurations](https://docs.cloud.google.com/dataform/docs/workflow-configurations)
- [The Dataform REST API](https://docs.cloud.google.com/dataform/reference/rest)

### Configura la identidad de ejecución (Execution Identity)

¿Quién ejecuta el pipeline a las 3 de la madrugada? No un usuario humano, sino una identidad de servicio. En las ejecuciones planificadas de Dataform intervienen dos identidades: el **Dataform service agent** gestionado por Google (que compila el código y orquesta las corridas) y una **custom service account** con la que se lanzan los trabajos sobre **BigQuery**. Ya dispones de una candidata ideal en tu proyecto: `dataquality-service-account`, pre-aprovisionada para los escaneos del Lab 4 con permisos de lectura en BigQuery; solo necesitamos darle permisos de escritura y autorizar la suplantación (*impersonation*).

Habilita primero la API de Dataform y materializa su *service agent*:

```bash
gcloud services enable dataform.googleapis.com
gcloud beta services identity create --service=dataform.googleapis.com --project={{ PROJECT_ID }}
```

Asígnate el rol de administrador de Dataform:

```bash
gcloud projects add-iam-policy-binding {{ PROJECT_ID }} \
    --member=user:{{ GCP_USERNAME }} --role=roles/dataform.admin --condition=None
```

Amplía los permisos de `dataquality-service-account` a editor de datos de BigQuery para que pueda reconstruir las capas *silver* y *gold*:

```bash
gcloud projects add-iam-policy-binding {{ PROJECT_ID }} \
    --member=serviceAccount:dataquality-service-account@{{ PROJECT_ID }}.iam.gserviceaccount.com \
    --role=roles/bigquery.dataEditor --condition=None
```

Por último, autoriza al *service agent* de Dataform a actuar como esa cuenta de servicio:

```bash
for role in roles/iam.serviceAccountTokenCreator roles/iam.serviceAccountUser; do gcloud iam service-accounts add-iam-policy-binding dataquality-service-account@{{ PROJECT_ID }}.iam.gserviceaccount.com --member=serviceAccount:service-{{ PROJECT_NUMBER }}@gcp-sa-dataform.iam.gserviceaccount.com --role=$role; done
```

Esta cadena de delegación (*service agent → token efímero → service account de ejecución → job de BigQuery*) permite que el pipeline se ejecute en producción sin utilizar credenciales personales ([Control access with IAM](https://docs.cloud.google.com/dataform/docs/access-control)).

### Crea el repositorio en Dataform

Abre [Dataform](https://console.cloud.google.com/bigquery/dataform) en la consola (dentro de BigQuery) y haz clic en <walkthrough-spotlight-pointer locator="semantic({button 'Create repository'})">Create repository</walkthrough-spotlight-pointer>:

1. **Repository ID**: `cymbal-dataform`
2. **Region**: `{{ REGION }}`
3. **Service account**: selecciona `dataquality-service-account@{{ PROJECT_ID }}.iam.gserviceaccount.com`.
4. Deja el resto de opciones por defecto y pulsa **Create**.

### Crea un espacio de trabajo de desarrollo (Development Workspace)

Abre <walkthrough-spotlight-pointer locator="text('cymbal-dataform')">cymbal-dataform</walkthrough-spotlight-pointer> y haz clic en <walkthrough-spotlight-pointer locator="semantic({button 'Create development workspace'})">Create development workspace</walkthrough-spotlight-pointer>. Asigna como **Workspace ID** `cymbal-dev` y pulsa **Create**.

Entra en el workspace recién creado: verás que está vacío y ofrece un botón para inicializarlo con código de plantilla. **No pulses *Initialize workspace***: tu proyecto ya existe (lo escribió `agy` en el Lab 3) y vamos a subirlo directamente a través de la API.

### Sube el proyecto mediante la REST API de Dataform

Managed Dataform expone toda su gestión de repositorios y ficheros a través de su **REST API**, que es como los sistemas de CI/CD sincronizan código. El repositorio del Bootkon incluye un script listo para subir tu proyecto del Lab 3 al workspace `cymbal-dev`; ejecútalo desde tu terminal:

```bash
cd ~/bootkon
content/agenticdata/src/optional/managed_dataform/push_to_dataform.sh
```

*(Por defecto apunta a `cymbal-dataform` / `cymbal-dev`; si usaste otros nombres puedes pasarlos como argumentos).*

Verás una línea `writeFile` por cada archivo del proyecto, seguida de un `commit` y un `push` a la rama `main`. Puedes abrir <walkthrough-editor-open-file filePath="content/agenticdata/src/optional/managed_dataform/push_to_dataform.sh">push_to_dataform.sh</walkthrough-editor-open-file> para ver las seis llamadas REST que realiza (`GET`, `:makeDirectory`, `:writeFile`, `:commit`, `:push` e `:installNpmPackages`) autenticándose con `gcloud auth print-access-token` y omitiendo deliberadamente `.df-credentials.json`. Refresca ahora la pestaña de `cymbal-dev` en la consola y verás allí `workflow_settings.yaml` y todos los modelos SQLX de `definitions/`.

### Explora el grafo compilado (Compiled Graph)

El workspace compila tu proyecto automáticamente. Haz clic en la pestaña <walkthrough-spotlight-pointer locator="semantic({tab 'Compiled graph'})">Compiled graph</walkthrough-spotlight-pointer>: verás tu arquitectura Medallion representada como un DAG interactivo — las seis fuentes de *bronze* a la izquierda, los cinco modelos de *silver*, los tres *data marts* de *gold* y todas las *assertions* que creó `agy`.

❗ Si apareciera un aviso de compilación mencionando `Failed to resolve workflow_settings.yaml` o paquetes pendientes, abre `workflow_settings.yaml` dentro del workspace, pulsa **Install packages** y vuelve a abrir el grafo.

Haz clic sobre `fct_daily_revenue`: el panel lateral te mostrará sus dependencias, los objetos dependientes y el SQL compilado con todas las referencias `${ref(...)}` resueltas a nombres físicos de tablas en BigQuery.

### Compila una Release Configuration

Un workspace es para desarrollo interactivo; las ejecuciones productivas se alimentan desde la rama `main` de Git mediante una **Release Configuration**. En la página del repositorio `cymbal-dataform`, abre la pestaña <walkthrough-spotlight-pointer locator="semantic({tab 'Releases & scheduling'})">Releases & scheduling</walkthrough-spotlight-pointer> y, bajo <walkthrough-spotlight-pointer locator="text('Release configurations')">Release configurations</walkthrough-spotlight-pointer>, haz clic en **Create**:

1. **Release ID**: `cymbal-release`
2. **Git commitish**: `main`
3. Deja vacíos la frecuencia y los *compilation overrides* (tu `workflow_settings.yaml` ya define el proyecto y los datasets).

Créala, entra en <walkthrough-spotlight-pointer locator="text('cymbal-release')">cymbal-release</walkthrough-spotlight-pointer> y pulsa <walkthrough-spotlight-pointer locator="semantic({button 'New compilation'})">New compilation</walkthrough-spotlight-pointer>. En pocos segundos aparecerá un **compilation result** en la tabla inferior: una instantánea inmutable de tu código SQL listo para ejecutarse una y otra vez sobre los nuevos datos que vayan llegando a *bronze*.

### Configura la ejecución programada cada 15 minutos (Workflow Configuration)

Vuelve a la pestaña *Releases & scheduling* y, bajo <walkthrough-spotlight-pointer locator="text('Workflow configurations')">Workflow configurations</walkthrough-spotlight-pointer>, haz clic en **Create**:

1. **Configuration ID**: `cymbal-15min`
2. **Release configuration**: `cymbal-release`
3. **Authentication**: elige **Execute with selected service account** y selecciona `dataquality-service-account`.
4. **Schedule frequency**: `*/15 * * * *` (cada cuarto de hora).
5. En **Workflow actions**, selecciona **Selection of tags** y marca `silver` y `gold` (incluyendo sus *assertions*).
6. Haz clic en **Create**.

Mientras el planificador queda armado para saltar en el próximo cuarto de hora, puedes pedirle a `agy` que profundice en el modelo operativo de Managed Dataform:

```bash
cd ~/bootkon && agy
```

```bash
Explain the managed Dataform model like I run data platforms for a living: what exactly is frozen in a compilation result, why does re-executing the same compilation result still pick up new bronze rows, and when would I schedule release compilations more often than daily?
```

### Dispara la primera ejecución por API

Sal de `agy` con `/quit` y dispara inmediatamente una ejecución del workflow realizando la misma llamada a la API REST que hace el scheduler cada 15 minutos:

```bash
curl -s -X POST \
    -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    -H "Content-Type: application/json" \
    -d '{"workflowConfig": "projects/{{ PROJECT_ID }}/locations/{{ REGION }}/repositories/cymbal-dataform/workflowConfigs/cymbal-15min"}' \
    https://dataform.googleapis.com/v1/projects/{{ PROJECT_ID }}/locations/{{ REGION }}/repositories/cymbal-dataform/workflowInvocations
```

Recibirás como respuesta un objeto **workflowInvocation** en estado `RUNNING`.

❗ Si devolviera `PERMISSION_DENIED` mencionando la *service account* o el *service agent* de Dataform, espera un par de minutos a que termine de propagarse el binding de IAM del inicio del laboratorio y vuelve a lanzar el `curl`.

### Observa cómo se actualiza la capa Gold automáticamente

En la página del repositorio en la consola, abre la pestaña <walkthrough-spotlight-pointer locator="semantic({tab 'Workflow execution logs'})">Workflow execution logs</walkthrough-spotlight-pointer>. Verás tu ejecución en estado **Running** y, tras un minuto, en **Succeeded**. Al hacer clic en ella podrás inspeccionar el estado y tiempo de ejecución de cada tabla y cada *assertion*.

Si abres [BigQuery](https://console.cloud.google.com/bigquery) y ejecutas esta consulta en el editor:

```sql
SELECT order_date, currency, orders, gross_revenue
FROM `{{ PROJECT_ID }}.cymbal_gold.fct_daily_revenue`
ORDER BY order_date DESC, currency
LIMIT 8
```

Comprobarás que las filas del día de hoy reflejan ya los nuevos pedidos generados por el simulador e ingeridos por **Datastream**, procesados íntegramente por `dataquality-service-account` desde **Managed Dataform**.

### ¡Completado!

🎉 ¡Magnífico{% if MY_NAME %}, {{ MY_NAME }}{% endif %}! Tu arquitectura Medallion ya no depende de tu terminal: el código generado por `agy` vive en un repositorio gestionado, se compila desde `main` y se ejecuta automáticamente cada 15 minutos bajo una cuenta de servicio dedicada. ⏰
