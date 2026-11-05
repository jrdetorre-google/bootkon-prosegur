## Lab 3: La Arquitectura Medallion — Dataform asistido por `agy`

<walkthrough-tutorial-duration duration="40"></walkthrough-tutorial-duration>
{{ author('Fabian Hirschmann', 'https://linkedin.com/in/fhirschmann') }}
<walkthrough-tutorial-difficulty difficulty="3"></walkthrough-tutorial-difficulty>
<bootkon-cloud-shell-note/>

Tu capa *bronze* es una réplica exacta del sistema operacional — lo que significa que también hereda todos los problemas de calidad del origen: clientes duplicados, el error tipográfico `shiped`, códigos de divisa en minúsculas, líneas de pedido huérfanas y pagos negativos. En este laboratorio construirás las capas **silver** (datos limpios, normalizados y validados) y **gold** (*data marts* de negocio listos para analítica e IA) utilizando **Dataform**.

La clave de este laboratorio: **`agy` escribirá todo el código SQLX por ti**. Tú defines el encargo, revisas los cambios, ejecutas el pipeline y verificas el resultado. Aquí es donde la ingeniería de datos asistida por agentes pasa de la teoría a tu terminal.

En este laboratorio utilizaremos la **Dataform CLI** open-source: orientada a código, versionable en Git e ideal para trabajar con un agente en bucle cerrado (si hay algún error de compilación, `agy` lo lee directamente en la terminal y lo corrige al instante). En producción puedes ejecutar exactamente el mismo proyecto de forma planificada en **Managed Dataform** dentro de Google Cloud.

### Acerca de Dataform y el patrón Medallion

**Dataform** gestiona las transformaciones SQL como *código*: cada tabla o vista se define en un fichero **SQLX** (una cabecera de configuración más la consulta SQL), las dependencias entre tablas se declaran con `${ref(...)}` en lugar de escribir nombres físicos de tablas, y a partir de esas referencias Dataform construye automáticamente el grafo de dependencias (DAG) y lo ejecuta en el orden correcto sobre **BigQuery**. Las **assertions** son tests de calidad de datos que se ejecutan en cada ejecución — unicidad de clave primaria, columnas no nulas o condiciones de negocio arbitrarias — haciendo fallar la ejecución si los datos incumplen alguna regla; y los **tags** permiten ejecutar subconjuntos del grafo (hoy lanzaremos `silver` y `gold` por separado).

El **patrón Medallion** que vas a construir es el estándar de arquitectura para estructurar un *lakehouse* analítico:
- **Bronze** conserva los datos brutos tal cual llegan del origen (tu réplica CDC).
- **Silver** limpia, deduplica, tipa y valida los datos con *assertions*.
- **Gold** expone las tablas agregadas y modelos dimensionales de negocio que consumen los usuarios — y, a partir del Lab 5, los agentes de IA.

Para saber más:
- [Dataform overview](https://docs.cloud.google.com/dataform/docs/overview)
- [Dataform core concepts](https://docs.cloud.google.com/dataform/docs/dataform-core)
- [The Dataform CLI](https://docs.cloud.google.com/dataform/docs/use-dataform-cli)
- [Assertions](https://docs.cloud.google.com/dataform/docs/assertions)

### Preparar el proyecto de Dataform

La **Dataform CLI** ya está instalada y conectada a tu proyecto gracias al `bk-bootstrap` del Lab 1. El esqueleto inicial del proyecto se encuentra en `content/agenticdata/src/dataform` (contiene únicamente la configuración y las declaraciones de las tablas origen de *bronze*; crear los modelos *silver* y *gold* será el trabajo de `agy`). **Esta carpeta será tu directorio de trabajo durante todo el Lab 3**, así que sitúate en ella:

```bash
cd ~/bootkon/content/agenticdata/src/dataform
```

Antes de pedir nada a `agy`, merece la pena inspeccionar dos archivos:
1. <walkthrough-editor-open-file filePath="content/agenticdata/src/dataform/workflow_settings.yaml">workflow_settings.yaml</walkthrough-editor-open-file> (generado a partir de la plantilla durante el bootstrap): comprueba que `defaultProject` contiene `{{ PROJECT_ID }}`. Junto a él, el archivo `.df-credentials.json` (ignorado por Git) indica a la CLI que utilice tus *Application Default Credentials (ADC)* contra BigQuery sin necesidad de claves JSON.
2. <walkthrough-editor-open-file filePath="content/agenticdata/src/dataform/definitions/sources.js">sources.js</walkthrough-editor-open-file>: declara las seis tablas de `cymbal_bronze` para que cualquier modelo pueda referenciarlas mediante `${ref(...)}` y construir el linaje automáticamente.

### Encarga la construcción del pipeline a tu co-ingeniero (`agy`)

La especificación completa de lo que queremos construir — esquemas de origen, problemas de calidad conocidos que hay que corregir, definición exacta de las tablas *silver* y *gold*, y reglas de las *assertions* — está documentada en el archivo <walkthrough-editor-open-file filePath="content/agenticdata/src/dataform/AGENTS.md">AGENTS.md</walkthrough-editor-open-file> dentro de la propia carpeta de Dataform. Ábrelo y léelo: mantener la especificación técnica en un fichero `AGENTS.md` versionado en el repositorio (en lugar de improvisar un prompt gigante) es una de las mejores prácticas para que el trabajo de los agentes de código sea repetible y auditable.

Inicia `agy` (asegúrate de estar ya dentro de `~/bootkon/content/agenticdata/src/dataform`):

```bash
agy
```

Y entrégale el encargo:

```bash
/goal Read AGENTS.md and build the complete silver and gold layers it specifies. Run dataform compile yourself and fix any errors until the project compiles cleanly.
```

Observa cómo trabaja `agy`: leerá `AGENTS.md` y `sources.js`, creará los ficheros `.sqlx` de las capas *silver* y *gold*, ejecutará `dataform compile`, analizará cualquier error de compilación y lo corregirá por sí mismo hasta que el grafo compile limpiamente. Cuando termine, puedes revisar los cambios propuestos con `/diff`.

> ⚠️ **Solo en caso de emergencia (Opcional — NO ejecutar si `agy` ya compiló sin errores):**
> Únicamente si `agy` se hubiera atascado y necesitaras sobrescribir tu carpeta `definitions/` con la solución de referencia oficial de `src/dataform_reference/` (por ejemplo <walkthrough-editor-open-file filePath="content/agenticdata/src/dataform_reference/definitions/stg_customers.sqlx">stg_customers.sqlx</walkthrough-editor-open-file> y <walkthrough-editor-open-file filePath="content/agenticdata/src/dataform_reference/definitions/fct_daily_revenue.sqlx">fct_daily_revenue.sqlx</walkthrough-editor-open-file>), ejecuta:
> ```bash
> cp ~/bootkon/content/agenticdata/src/dataform_reference/definitions/*.sqlx ~/bootkon/content/agenticdata/src/dataform/definitions/
> ```

### Compilar y ejecutar en BigQuery

Sal de `agy` escribiendo `/quit` y verifica tú mismo que el proyecto compila sin errores — *confía, pero verifica*:

```bash
dataform compile
```

Verás el resumen con las tablas, vistas y *assertions* compiladas. Ahora ejecuta en **BigQuery** primero la capa `silver` y a continuación la capa `gold` (cada ejecución tarda menos de un minuto sobre los datos semilla):

```bash
dataform run --tags silver
```

```bash
dataform run --tags gold
```

Cada ejecución lanza también automáticamente las **assertions** — los tests de calidad que hacen fallar el pipeline si alguna regla de limpieza no se cumple. Si alguna tabla o *assertion* fallara en ejecución contra BigQuery, copia el mensaje de error, vuelve a abrir `agy` y pídele que lo repare: ese ciclo de retroalimentación es precisamente la esencia del flujo agéntico.

### Verifica el resultado en la consola de BigQuery

Abre [BigQuery](https://console.cloud.google.com/bigquery) y despliega tu proyecto en el panel izquierdo:

1. Verás que ahora existen los tres datasets de la arquitectura Medallion: <walkthrough-spotlight-pointer locator="text('cymbal_bronze')">cymbal_bronze</walkthrough-spotlight-pointer>, <walkthrough-spotlight-pointer locator="text('cymbal_silver')">cymbal_silver</walkthrough-spotlight-pointer> y <walkthrough-spotlight-pointer locator="text('cymbal_gold')">cymbal_gold</walkthrough-spotlight-pointer> (además de <walkthrough-spotlight-pointer locator="text('cymbal_assertions')">cymbal_assertions</walkthrough-spotlight-pointer> con las vistas de validación).
2. Abre <walkthrough-spotlight-pointer locator="text('cymbal_silver')">cymbal_silver</walkthrough-spotlight-pointer> → `stg_customers` y comprueba el número de filas en la pestaña <walkthrough-spotlight-pointer locator="semantic({tab 'Details'})">Details</walkthrough-spotlight-pointer>: verás que hay menos filas que en `cymbal_bronze` → `cymbal_customers`, porque se han eliminado los duplicados y los registros con emails inválidos.
3. Comprueba con una consulta SQL cómo se ha corregido el error tipográfico `shiped` entre *bronze* y *silver*:

```sql
SELECT 'bronze' AS layer, COUNT(*) AS shiped_typos
FROM `{{ PROJECT_ID }}.cymbal_bronze.cymbal_orders` WHERE status = 'shiped'
UNION ALL
SELECT 'silver', COUNT(*)
FROM `{{ PROJECT_ID }}.cymbal_silver.stg_orders` WHERE status = 'shiped'
```

Un detalle más: recuerda que tu simulador (en la pestaña 2 de la terminal) sigue insertando pedidos en PostgreSQL y **Datastream** sigue actualizando `cymbal_bronze` en vivo. Puedes volver a lanzar `dataform run` en cualquier momento para refrescar toda tu arquitectura Medallion con los últimos datos.

### ¡Completado!

🎉 ¡Bravo{% if MY_NAME %}, {{ MY_NAME }}{% endif %}! Has dirigido a un agente de IA para que escriba, compile y valide un pipeline Medallion completo en **Dataform** — y tú lo has revisado, ejecutado y verificado en **BigQuery** con *assertions* incluidas. Los duplicados han desaparecido, los estados corruptos se han corregido y las líneas huérfanas han sido filtradas. Tu capa *gold* ya brilla; en el siguiente laboratorio vamos a gobernarla y protegerla. 🥇
