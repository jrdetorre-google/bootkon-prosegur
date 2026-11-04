<walkthrough-metadata>
  <meta name="title" content="Agentic Data Bootkon para Prosegur" />
  <meta name="description" content="Construye una plataforma de datos agéntica de extremo a extremo en Google Cloud: replicación CDC de Cloud SQL a BigQuery con Datastream, pipeline Medallion bronze/silver/gold en Dataform asistido por agy, gobierno con Knowledge Catalog, un Data Agent en BigQuery y un sistema multi-agente ADK sobre el protocolo A2A." />
  <meta name="keywords" content="data, ai, agents, bigquery, datastream, cloudsql, dataform, dataplex, adk, a2a, prosegur" />
  <meta name="component_id" content="1734803" />
</walkthrough-metadata>


# Agentic Data Bootkon para Prosegur

<walkthrough-tutorial-duration duration="15"></walkthrough-tutorial-duration>
Author: <a href="https://linkedin.com/in/fhirschmann">Fabian Hirschmann</a> · Adaptado para Prosegur
<walkthrough-tutorial-difficulty difficulty="2"></walkthrough-tutorial-difficulty>


## Introducción

{% if MY_NAME %}¡Hola **{{ MY_NAME }}! 👋** {% endif %}Te damos la bienvenida a **Agentic Data Bootkon para Prosegur**. ¡Estamos encantados de contar contigo hoy! En este panel lateral encontrarás la guía interactiva paso a paso de todos los laboratorios que vamos a realizar. Antes de empezar, verifiquemos un par de detalles rápidos:

### Cómo trabajar con los laboratorios

Puedes insertar cualquier bloque de comandos directamente en tu terminal pulsando el icono situado en la esquina superior derecha de cada bloque de código:
<walkthrough-cloud-shell-icon></walkthrough-cloud-shell-icon>. Ese botón pegará el comando automáticamente en la terminal activa.
Asegúrate siempre de estar utilizando la terminal integrada del IDE.

Vamos a probarlo:

```bash
echo "Lets go, Cymbal"
```

Pulsa `Enter` en la terminal de la parte inferior de tu pantalla para ejecutarlo.

### Comprueba tu entorno

El comando de inicialización que acabas de pegar ya ha configurado tu entorno automáticamente. Comprueba que ambos valores son correctos:

* Tu `PROJECT_ID` es `{% if PROJECT_ID == "" %}None{% else %}{{ PROJECT_ID }}{% endif %}`

* Tu `GCP_USERNAME` es `{% if GCP_USERNAME == "" %}None{% else %}{{ GCP_USERNAME }}{% endif %}`

Si ambos valores son correctos, ¡ya está todo listo! Pulsa el botón **`START`** (abajo a la derecha) para comenzar con el Lab 1.
*(Consejo: si en algún momento cierras por error este panel de tutorial o el editor, escribe `bk-start` en la terminal para volver a abrirlos).*

Si alguno de los valores es incorrecto o muestra `None`: abre el archivo `vars.local.sh`
<walkthrough-editor-open-file filePath="vars.local.sh">haciendo clic aquí</walkthrough-editor-open-file>,
corrige el valor (por ejemplo, asignando a `PROJECT_ID` el ID de tu proyecto del evento) y guarda el archivo.

Después, recarga el entorno y actualiza los valores mostrados en esta página ejecutando:

```bash
. bk && bk-start
```

{% include 'agenticdata/labs/1_environment_setup.md' %}

{% include 'agenticdata/labs/2_replication.md' %}

{% include 'agenticdata/labs/3_dataform.md' %}

{% include 'agenticdata/labs/4_governance.md' %}

{% include 'agenticdata/labs/5_data_agent.md' %}

{% include 'agenticdata/labs/6_a2a.md' %}


## ¡Final completado!

¡Lo has conseguido! Enhorabuena y muchas gracias por participar en el **Agentic Data Bootkon para Prosegur**.

### Créditos

Edición adaptada para **Prosegur** por el equipo de **Google Cloud Iberia**, basada en el stream **Agentic Data Bootkon** creado por:
- [Fabian Hirschmann](https://www.linkedin.com/in/fhirschmann/) (main author)
- [Florian Baumert](https://www.linkedin.com/in/florian-baumert/)
- [Cary Edwards](https://www.linkedin.com/in/cary-edwards-a3a557a6/)
