## Lab 1: Configuración del Entorno y Presentación de tu Co-Ingeniero

<walkthrough-tutorial-duration duration="20"></walkthrough-tutorial-duration>
{{ author('Fabian Hirschmann', 'https://linkedin.com/in/fhirschmann') }}
<walkthrough-tutorial-difficulty difficulty="1"></walkthrough-tutorial-difficulty>
<bootkon-cloud-shell-note/>

¡Bienvenido al laboratorio de Cymbal{% if MY_NAME %}, {{ MY_NAME }}{% endif %}! Al finalizar la sesión habrás construido una **agentic data platform** completa: una base de datos operacional en vivo, replicación *change-data-capture (CDC)* hacia **BigQuery**, una arquitectura *medallion* (*bronze → silver → gold*) gobernada, y dos agentes de IA conversando entre sí mediante **A2A** — el protocolo abierto *agent-to-agent* que permite a los agentes de IA invocarse mutuamente del mismo modo que HTTP conecta microservicios. Si es la primera vez que trabajas con estos componentes, no te preocupes: los construiremos paso a paso.

En este primer laboratorio verificarás la activación de servicios, ejecutarás el script de *bootstrap*, tomarás posesión de tu base de datos de producción pre-aprovisionada y conocerás a **Antigravity CLI (`agy`)** — tu co-ingeniero de IA durante toda la jornada.

La regla de oro para hoy es sencilla: **tú** ejecutas los comandos de infraestructura, **`agy`** escribe el código y las configuraciones, y **Google Cloud Console** es donde verificas lo que ha ocurrido.

A lo largo de algunos pasos verás notas opcionales para realizar acciones desde la consola gráfica cuando prefieras hacer clic en lugar de usar la terminal: ambas rutas llevan exactamente al mismo resultado.

### Habilitar servicios

<walkthrough-enable-apis apis=
  "sqladmin.googleapis.com,
  datastream.googleapis.com,
  compute.googleapis.com,
  servicenetworking.googleapis.com,
  iap.googleapis.com,
  bigquery.googleapis.com,
  bigqueryconnection.googleapis.com,
  dataplex.googleapis.com,
  datacatalog.googleapis.com,
  datalineage.googleapis.com,
  dlp.googleapis.com,
  geminidataanalytics.googleapis.com,
  cloudaicompanion.googleapis.com,
  aiplatform.googleapis.com,
  storage-component.googleapis.com,
  serviceusage.googleapis.com,
  cloudresourcemanager.googleapis.com,
  iam.googleapis.com,
  artifactregistry.googleapis.com">
</walkthrough-enable-apis>

{% if ON_ARGOLIS %}
### Prepara tu proyecto (Argolis)

En un evento guiado, los organizadores pre-aprovisionan todos los proyectos con antelación; si estás ejecutando este laboratorio por tu cuenta en **Argolis** (entorno interno de Google), actúa como tu propio organizador ejecutando primero el script de preparación. Habilita las APIs del stream, asigna los roles de IAM necesarios, configura el *service agent* de **Datastream** y su *private connection*, crea la *service account* de calidad de datos y despliega la red y la instancia de **Cloud SQL** (es idempotente y seguro de re-ejecutar; tarda entre 15 y 25 minutos):

```bash
content/agenticdata/provisioning/bk-prep-project
```

❗ Algunas políticas de organización de Argolis (por ejemplo `constraints/compute.requireOsLogin`, `constraints/compute.requireShieldedVm` o restricciones de Cloud SQL) pueden bloquear la creación de la VM o de la instancia. Si falla por un error de política, desactiva la restricción en *IAM & Admin → Organization Policies* y vuelve a ejecutar el comando.

Una vez finalizado, recarga el entorno una vez — tu *jump VM* ya existe y esto levantará el túnel hacia la base de datos que utilizan los laboratorios:

```bash
. bk
```
{% endif %}

### Ejecutar el bootstrap

Ejecuta el siguiente script. Instalará las dependencias de Python y la **Dataform CLI** (que utilizarás en el Lab 3) y dejará pre-configurado tu co-ingeniero de IA. (Todo lo vinculado al proyecto — APIs, roles de IAM y *service accounts* — {% if ON_ARGOLIS %}ya ha quedado cubierto en el paso anterior{% else %}ya ha sido pre-aprovisionado en tu proyecto del evento antes de empezar; por eso el paso de activación de APIs superior se ha puesto en verde al instante{% endif %}). Tarda aproximadamente dos minutos; mientras se ejecuta puedes echar un vistazo a <walkthrough-editor-open-file filePath="content/agenticdata/bk-bootstrap">bk-bootstrap</walkthrough-editor-open-file> en el editor:

```bash
content/agenticdata/bk-bootstrap
```

Las contraseñas de tu base de datos ya se han generado automáticamente durante la inicialización y están guardadas en `vars.local.sh` — los comandos siguientes las utilizan mediante la variable `$BK_DB_PASSWORD`, y **cualquier pestaña de terminal que abras las carga automáticamente**.

### La arquitectura de red privada

Nuestra base de datos operacional **no tiene IP pública**. En su lugar, **Datastream** se conectará a ella a través de **Private Service Connect (PSC)** — y toda esa topología de red ya está desplegada en tu proyecto{% if ON_ARGOLIS %} (creada por el paso de preparación anterior){% endif %}: la VPC `cymbal-vpc` con la subred `cymbal-subnet` (`10.10.0.0/24`), la dirección IP interna `10.10.0.5` — el *PSC endpoint* que actúa como puerta de entrada privada a tu base de datos —, el *network attachment* `cymbal-attachment` con la conexión privada de Datastream `cymbal-psc` ya conectada a él, y una pequeña *jump VM* (`cymbal-jump`) que utilizaremos en el Lab 2. Puedes abrir [VPC networks](https://console.cloud.google.com/networking/networks/list) para inspeccionar estos componentes en la consola.

¿En qué consiste exactamente este diseño? **Private Service Connect (PSC)** es el mecanismo de **Google Cloud** para exponer un servicio a través de distintas redes VPC sin necesidad de hacer *VPC peering* completo ni exponer IPs públicas: el productor del servicio (en este caso, tu instancia de **Cloud SQL**) publica un *service attachment*, y los consumidores conectan un *endpoint* a él — todo el tráfico permanece dentro del *backbone* de Google y cada extremo solo ve el socket específico autorizado. El **network attachment** cumple la función simétrica: permite a un servicio gestionado de Google (**Datastream**) colocar una interfaz de red privada *dentro* de tu VPC. En el Lab 2 uniremos ambas mitades.

Para saber más:
- [Private Service Connect](https://docs.cloud.google.com/vpc/docs/private-service-connect)
- [Network attachments](https://docs.cloud.google.com/vpc/docs/about-network-attachments)
- [Cloud SQL and Private Service Connect](https://docs.cloud.google.com/sql/docs/postgres/about-private-service-connect)

### Toma el control de tu base de datos

La base de datos de producción de pedidos de Cymbal ya está creada: se llama `cymbal-oltp`, corre sobre **Cloud SQL for PostgreSQL** y cuenta con **logical decoding** habilitado desde su creación (PostgreSQL emite cada transacción confirmada en formato lógico, requisito indispensable para *CDC*), conectividad exclusiva por **Private Service Connect** (su *endpoint* es la IP `10.10.0.5` anterior) y un tamaño de máquina contenido (`db-custom-1-3840`), ya que la lectura del log de *CDC* apenas añade carga a la base de datos.

La instancia permanecía aparcada en estado **STOPPED** hasta el inicio del evento, y el script de arranque de Bootkon ya ha enviado la orden de encendido en segundo plano, por lo que ya debería estar activa (`RUNNABLE`). Como se aprovisionó con una contraseña temporal desechable, asígnale ahora al usuario `postgres` tu contraseña generada `$BK_DB_PASSWORD`:

```bash
gcloud sql users set-password postgres --instance=cymbal-oltp --password=$BK_DB_PASSWORD
```

❗ Solo si este comando devuelve un error indicando *"instance is not running"*: significa que el encendido en segundo plano aún no ha terminado. Enciende la instancia manualmente (tarda un par de minutos) y después vuelve a ejecutar el comando `set-password` anterior:

```bash
gcloud sql instances patch cymbal-oltp --activation-policy=ALWAYS
```

Para saber más:
- [Set up logical replication on Cloud SQL](https://docs.cloud.google.com/sql/docs/postgres/replication/configure-logical-replication)

### Los datos semilla (Seed Data)

El histórico de pedidos de Cymbal es **sintético y determinista**: un generador con semilla fija lo produce para que todos los asistentes trabajen exactamente con las mismas filas — incluyendo errores de calidad introducidos a propósito que limpiaremos en el Lab 3. Abre <walkthrough-editor-open-file filePath="content/agenticdata/src/datagen/generate.py">generate.py</walkthrough-editor-open-file> y observa la sección *planted flaws* al principio del archivo.

Los seis archivos CSV se depositaron en tu bucket de **Cloud Storage** durante el aprovisionamiento del proyecto (y se importaron desde ahí a la base de datos). Comprueba que están en tu bucket:

```bash
gcloud storage ls -l gs://{{ PROJECT_ID }}-bucket/seed/
```

Verás seis ficheros (~111 MB en total): medio millón de pedidos junto con sus clientes, productos, líneas de pedido, pagos y reseñas.

### Conoce a tu co-ingeniero: Antigravity CLI (`agy`)

`agy` (**Antigravity CLI**) ya viene instalado en **Cloud Shell**, y `bk-bootstrap` ha pre-configurado sus ajustes básicos (proyecto, región y carpetas de confianza del taller). Lo único que debes hacer una sola vez es iniciar sesión. **Maximiza primero el panel de la terminal** para que el cuadro de diálogo de inicio de sesión se vea completo en pantalla y lanza `agy` desde la raíz del repositorio (donde ya te encuentras):

```bash
agy
```

En su primera ejecución, `agy` te preguntará cómo quieres autenticarte: elige la opción **Google Cloud project**. `agy` te informará de que tu cuenta **no tiene licencia de Gemini Enterprise** — es totalmente normal, hoy utilizamos directamente tu proyecto de GCP: si te pide el ID del proyecto, introduce el `PROJECT_ID` de tu evento (puede que ya aparezca pre-rellenado):

```bash
{{ PROJECT_ID }}
```

Si aparece una URL de autenticación, ábrela con **Ctrl+Clic** (o **Cmd+Clic** en Mac), inicia sesión con tu cuenta `{{ GCP_USERNAME }}` y pega el código de confirmación de vuelta en el campo de token de la terminal. Acepta los *Terms of Service* si se solicita. A partir de este momento, cada vez que ejecutes `agy` entrarás directamente al prompt interactivo — ya puedes devolver el panel de la terminal a su tamaño habitual.

Los comandos cotidianos que `agy` necesita hoy (explorar ficheros, ejecutar `dataform compile`, validaciones con Python) están **pre-aprobados**, para que la generación de código fluya sin interrupciones; en cambio, cualquier comando que toque recursos reales de tu nube te pedirá confirmación antes de ejecutarse. Ese reparto es deliberado: `agy` escribe el código, tú lo revisas (con `/diff`), y las operaciones en la nube siempre están bajo tu mando.

### Pide a `agy` que te explique qué hace el bootstrap

Vamos a enviar tu primer prompt. Copia y pega este texto dentro de `agy`:

```bash
What does content/agenticdata/bk-bootstrap do? Explain how the seed data generation stays deterministic and why identical data matters for this workshop.
```

Lee su respuesta — esta será la dinámica durante toda la mañana: tú diriges la arquitectura, `agy` analiza y escribe el código, y tú verificas el resultado.

### Verifica en la consola

Echemos un vistazo a tu base de datos en la interfaz gráfica. Abre [Cloud SQL instances](https://console.cloud.google.com/sql/instances) y localiza <walkthrough-spotlight-pointer locator="text('cymbal-oltp')">cymbal-oltp</walkthrough-spotlight-pointer>: verás que está en ejecución, con **Private service connect** como método de conectividad y sin ninguna IP pública expuesta.

### ¡Completado!

🎉 ¡Enhorabuena{% if MY_NAME %}, {{ MY_NAME }}{% endif %}! Tu proyecto ya tiene los permisos de IAM listos, **Datastream** conectado de forma privada a tu VPC, una base de datos de producción respondiendo con **tu** contraseña, medio millón de pedidos sintéticos preparados en **Cloud Storage**, y un co-ingeniero de IA esperando órdenes en tu terminal. ¡Vamos a por los datos en tiempo real! 🚀
