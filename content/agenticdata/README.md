# Agentic Data Bootkon para Prosegur

<p align="center">
  <img src="../common/img/prosegur_logo.jpg" width="200" alt="Prosegur"/>
</p>

¡Bienvenido a **Agentic Data Bootkon para Prosegur**!

**Agentic Data Bootkon para Prosegur** es un hackathon práctico e intensivo diseñado a medida para los equipos de ingeniería de datos, arquitectura e inteligencia artificial de las Unidades de Negocio y áreas corporativas de **Prosegur** (*Prosegur Security, Prosegur Cash, Movistar Prosegur Alarmas, AVOS Tech, Cipher y Corporativo / ITaaS*).

A lo largo de esta jornada construirás de principio a fin una **agentic data platform** completa sobre **Google Cloud**: desde una base de datos operacional transaccional en vivo hasta un ecosistema de agentes de IA que colaboran entre sí sobre tus datos gobernados utilizando protocolos abiertos. Durante todo el evento, **Antigravity CLI (`agy`)** trabajará contigo como tu co-ingeniero de IA: tú operas la infraestructura desde la terminal, `agy` escribe el código y las configuraciones bajo tu dirección, y **Google Cloud Console** es donde verificas todo lo que ocurre.

## Caso de Uso del Laboratorio

Para que cada participante disponga de un entorno aislado, reproducible y con millones de registros listos para operar desde el minuto uno, en los laboratorios asumirás el rol de ingeniero de plataforma de datos en **Cymbal**, una compañía cuyos sistemas transaccionales de pedidos corren sobre **PostgreSQL**. Al igual que ocurre en entornos empresariales complejos multi-negocio, el equipo de analítica sufre con exportaciones manuales en CSV, existen discrepancias en las cifras, y la dirección quiere activar **"IA sobre nuestros datos"** de forma segura y gobernada.

A lo largo de la sesión construirás los cuatro pilares de la plataforma:

1. **Replicación continua en tiempo real en lugar de exportaciones batch.** **Datastream** captura cada `INSERT`, `UPDATE` y `DELETE` desde **Cloud SQL for PostgreSQL** mediante *Change Data Capture (CDC)* y lo fusiona en **BigQuery** en cuestión de segundos — íntegramente a través de **Private Service Connect (PSC)**, sin exponer IPs públicas en ningún punto.
2. **Datos fiables mediante una arquitectura Medallion asistida por IA.** Construirás las capas *bronze → silver → gold* con **Dataform** (SQLX), donde **Antigravity CLI (`agy`)** redactará las transformaciones y los tests de calidad (*assertions*) bajo tu supervisión. Los datos en bruto contienen errores deliberados de calidad del mundo real (clientes duplicados, errores tipográficos de estado, divisas inconsistentes, registros huérfanos e importes negativos) que limpiarás de forma automatizada.
3. **Gobierno universal antes de desplegar agentes.** Con **Knowledge Catalog** harás que cada tabla sea descubrible semánticamente, medirás la calidad del dato mediante *Data Quality Scans* automatizados y protegerás la información personal (*PII*) con seguridad a nivel de columna (*Policy Tags*), estableciendo la base de confianza sobre la que operarán los agentes.
4. **Activación agéntica y sistema multi-agente abierto.** Primero crearás y publicarás un **BigQuery Data Agent** (*Conversational Analytics*) conectado a tu capa *gold* gobernada. Después, construirás un sistema multi-agente con **Agent Development Kit (ADK)**: un agente *concierge* capaz de consultar pedidos operacionales en tiempo real en PostgreSQL y delegar preguntas analíticas complejas al agente de BigQuery a través del protocolo abierto **A2A (*Agent2Agent*)**.

## Arquitectura

<table align="center">
  <tr>
    <td align="center" width="130">
      <img src="https://icon.icepanel.io/GCP/svg/Cloud-SQL.svg" width="48" alt="Cloud SQL"/><br>
      <b>Cloud SQL</b><br>
      <sub>PostgreSQL, PSC-only<br>BBDD operacional en vivo</sub>
    </td>
    <td align="center"><b>— CDC ▶</b></td>
    <td align="center" width="130">
      <img src="https://icon.icepanel.io/GCP/svg/Datastream.svg" width="48" alt="Datastream"/><br>
      <b>Datastream</b><br>
      <sub>backfill + streaming<br>merge mode</sub>
    </td>
    <td align="center"><b>—▶</b></td>
    <td align="center" width="180">
      <img src="https://icon.icepanel.io/GCP/svg/BigQuery.svg" width="48" alt="BigQuery"/><br>
      <b>BigQuery</b><br>
      <sub>bronze → silver → gold<br>Dataform SQLX, creado con agy</sub>
    </td>
    <td align="center"><b>—▶</b></td>
    <td align="center" width="150">
      <img src="https://icon.icepanel.io/GCP/svg/AI-Platform.svg" width="48" alt="BigQuery data agent"/><br>
      <b>Data Agent</b><br>
      <sub>conversational analytics<br>anclado sobre gold</sub>
    </td>
  </tr>
  <tr>
    <td></td><td></td><td></td><td></td>
    <td align="center">
      <b>▲</b><br>
      <img src="https://icon.icepanel.io/GCP/svg/Dataplex.svg" width="40" alt="Knowledge Catalog"/><br>
      <b>Knowledge Catalog</b><br>
      <sub>búsqueda · data quality<br>PII policy tags</sub>
    </td>
    <td></td>
    <td align="center"><b>▲</b><br><b>A2A</b></td>
  </tr>
  <tr>
    <td align="center">
      <b>▲</b><br>
      <sub>consultas de<br>pedidos en vivo</sub>
    </td>
    <td colspan="5" align="center"><b>◀————————————————</b></td>
    <td align="center">
      <img src="https://icon.icepanel.io/GCP/svg/Vertex-AI.svg" width="48" alt="ADK concierge agent"/><br>
      <b>cymbal_concierge</b><br>
      <sub>ADK agent, adk web</sub>
    </td>
  </tr>
  <tr>
    <td colspan="7" align="center">
      <img src="https://icon.icepanel.io/GCP/svg/Cloud-Shell.svg" width="40" alt="Cloud Shell"/><br>
      <b>Cloud Shell + Antigravity CLI (agy)</b><br>
      <sub>tú operas la terminal · agy escribe el código · la consola verifica</sub>
    </td>
  </tr>
</table>

## Sobre el conjunto de datos

No es necesario descargar nada manualmente: el dataset transaccional (**Cymbal orders**) es 100% sintético y ha sido generado de forma determinista *dentro de tu propio proyecto de Google Cloud*, de modo que todos los participantes trabajan exactamente con los mismos datos (~2,3 millones de filas, menos de 2 GB). Incluye problemas de calidad de datos introducidos deliberadamente — clientes duplicados, emails inválidos, un error tipográfico en el estado `shiped`, códigos de divisa en minúsculas, líneas de pedido huérfanas, pagos negativos y marcas de tiempo en el futuro — además de columnas con datos personales (*PII*: nombres, emails, teléfonos y direcciones) para el laboratorio de gobierno. Durante el evento lanzarás un simulador de actividad que insertará y actualizará filas en continuo para ver fluir el pipeline *CDC* en tiempo real.

## Inicio de sesión en Google Cloud

> [!CAUTION]
> Por favor, sigue los pasos siguientes exactamente como se indican. Saltarse alguno puede impedir que el entorno interactivo cargue correctamente.

Vamos a configurar tu acceso a **Google Cloud Console**:

1. Abre una ventana nueva de tu navegador en modo <picture><img src="../common/img/incognito_upper.svg" align="texttop" alt="Incógnito"></picture> (`Ctrl+Shift+N` en Windows/Linux o `Cmd+Shift+N` en Mac).
2. Abre esta página del repositorio en la ventana de <picture><img src="../common/img/incognito_lower.svg" align="texttop" alt="incógnito"></picture> que acabas de abrir para seguir leyendo desde ahí.
3. Abre <a href="https://console.cloud.google.com" target="_blank">Google Cloud Console</a> e inicia sesión con las credenciales (`@gcplab.me`) que se te han asignado para el evento.
4. Acepta los Términos del Servicio (*Terms of Service*).

    ![](../common/img/termsofservice.png)

5. Selecciona tu **Project ID**. Haz clic en el selector de proyectos (arriba a la izquierda) y elige el proyecto asignado a tu cuenta (como en las capturas siguientes):
    ![](../common/img/selectproject.png)


    ![](../common/img/selectproject2.png)


    ![](../common/img/selectproject3.png)

6. Entra en los [ajustes de idioma de la consola](https://console.cloud.google.com/user-preferences/languages) y asegúrate de que el idioma sea **`English (US)`**. Esto es imprescindible para que los punteros interactivos del tutorial reconozcan los botones en pantalla y los *Table Captains* puedan asistirte rápidamente.

    ![](../common/img/select_language.png)

## Ejecución de los laboratorios en Cloud Shell

Durante el evento realizarás todos los laboratorios desde **Google Cloud Shell**.

**Cloud Shell** es un entorno interactivo basado en navegador que incluye la CLI de Google Cloud preinstalada, utilidades de desarrollo y un editor de código integrado basado en Visual Studio Code, con el panel interactivo del tutorial a la derecha:

![](../common/img/cloud_shell_window.png)

En el panel derecho verás las instrucciones paso a paso del laboratorio. En cada bloque de comandos encontrarás dos iconos en la esquina superior derecha: uno para copiar el comando al portapapeles y otro para insertarlo directamente en la terminal de Cloud Shell.

## Recomendaciones importantes al trabajar con los laboratorios

> [!CAUTION]
> Revisa estos puntos antes de lanzar el comando de inicio de la siguiente sección.

Si en algún momento ves dos terminales distintas en pantalla, utiliza únicamente la terminal integrada del IDE (fondo claro) y cierra la terminal inferior independiente (fondo negro) pulsando su botón `X`:

![](../common/img/code_terminals.png)

Asimismo, <font color="red">no pulses los botones *Open Terminal* ni *Open in new window*</font>, ya que sacarán el entorno de la ventana integrada con el tutorial:

![](../common/img/code_newwindow.png)

Comprueba siempre que la URL de tu navegador empiece por `console.cloud.google.com` y <font color="red">no por `shell.cloud.google.com`</font>:

![](../common/img/wrong_url.png)

Si en algún momento cierras por accidente el tutorial o el editor, basta con ejecutar este comando en la terminal para volver a abrirlos:

```bash
bk-start
```

> [!NOTE]
> A lo largo de los laboratorios utilizarás varias pestañas de terminal simultáneamente (por ejemplo, para mantener en segundo plano el túnel IAP y el simulador de pedidos en tiempo real). No cierres esas pestañas; cada nueva pestaña que abras con el botón `+` cargará automáticamente tu configuración desde `~/.bashrc`.

## Comenzar el laboratorio

En tu ventana de **Google Cloud Console**, haz clic en el icono **Activate Cloud Shell** (arriba a la derecha):

![](../common/img/activate_cloud_shell.png)

Haz clic dentro de la terminal que se abre en la parte inferior de la pantalla:

![](../common/img/cloud_shell_terminal.png)

Copia y pega el siguiente comando y pulsa `Enter`:

```bash
BK_STREAM=agenticdata BK_REPO=jrdetorre-google/bootkon-prosegur; . <(curl -fsSL https://raw.githubusercontent.com/${BK_REPO}/main/.scripts/bk)
```

Cuando te pregunte tu nombre en la terminal (`What should I call you?`), escribe tu nombre (o pulsa `Enter`) y automáticamente se abrirá el editor y el panel interactivo del tutorial a la derecha de tu pantalla. ¡Ya puedes pulsar **START** en el panel derecho para comenzar!

## Créditos y Autores

Edición adaptada para **Prosegur** por [Juan Ramon De Torre](https://es.linkedin.com/in/juanramon-detorre), basada en el stream oficial **Agentic Data Bootkon** creado por:
- [Fabian Hirschmann](https://www.linkedin.com/in/fhirschmann/) (maintainer; main author)
- [Juan Ramon De Torre](https://es.linkedin.com/in/juanramon-detorre)
- [Florian Baumert](https://www.linkedin.com/in/florian-baumert/)
- [Cary Edwards](https://www.linkedin.com/in/cary-edwards-a3a557a6/)
