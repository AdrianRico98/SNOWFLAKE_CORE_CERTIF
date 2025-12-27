# Performance Concepts: Virtual Warehouses

## Overview

**Virtual warehouse:** Named abstraction para massively parallel processing (MPP) compute cluster. Clusters compuestos de worker nodes en cloud platform donde está deployed tu Snowflake account (ej: AWS account usa múltiples EC2 instances trabajando juntas de manera distribuida para computation requerida en queries).

**Función:** Colectivamente forman **compute layer** de Snowflake architecture. Responsables de processing tasks para completar SELECT commands, DML operations, data loading operations. Algunos commands NO requieren virtual warehouse.

**Interacción:** User solo interactúa con named warehouse object vía SQL, no con underlying nodes del cluster. No necesitas conocimiento ni management overhead de complex distributed system para ejecutar data-intensive queries.

**Escalabilidad:** Puedes spin up/shut down virtualmente unlimited number de warehouses en account (ej: diferentes departamentos, diferentes workloads como ingestion vs analysis). Snowflake es ACID-compliant database, todo con consistent reads/writes a centralized storage layer.

**Configuración:** Puede cambiarse on the fly, incluyendo size.

**Warehouse cache:** Contienen local fast SSD storage donde almacenan raw table data retrieved desde storage layer para procesar query results. Puede hacer subsequent queries más rápidas de compute.

**Uso:** Crear warehouse (UI o SQL) → set warehouse en current context → subsequent queries ejecutadas por ese virtual warehouse (no necesitas especificar warehouse cada vez).

## Propiedades Únicas

- Pueden crearse/destruirse near instantly
- Varios sizes (corresponden a relative power)
- Pueden pausarse/resumirse anytime (manually o automatically setting parameters). **Suspended = no cuesta nada**
- Pueden manually resizarse anytime
- Pueden dynamically scale in/out usando virtual warehouse clustering

Gran flexibilidad para fitting workload type a virtual warehouse.

## State y Properties

### Estados del Virtual Warehouse

**1. Started:** Warehouse activo, ready para procesar queries. **Consuming credits**.

**2. Suspended:** Warehouse shut down, **NO consuming credits**.

**3. Resizing:** Warehouse puede resizearse up/down anytime (incluso mientras ejecuta queries). Este status indica proceso en progress.

### Manual Control

**Default:** Warehouse creado automáticamente en **started state**.

**Suspend:** `ALTER WAREHOUSE ... SUSPEND` command. Remueve todos los compute nodes, pone warehouse en suspended state después de completar queries actualmente ejecutando.

**Resume:** `ALTER WAREHOUSE ... RESUME` command. Pone warehouse back en started state.

**Benefit:** Evitar pagar por compute resources cuando idle (ej: solo querying 9-5 business hours → suspend al final del día, evitar billing overnight).

### Parámetros Automatizados

**1. AUTO_SUSPEND:**
- Especifica segundos de inactividad (no queries usando warehouse) tras los cuales warehouse se suspende automáticamente
- Setting value <60s puede no tener expected result (background process suspends warehouse runs ~every 60s, no guaranteed shutdown en <60s)
- Setting 0 o NULL = warehouse nunca se suspende
- **Default:** Enabled, 600 segundos (10 min inactividad)

**Caveats de setting auto_suspend muy bajo:**
- Warehouse cache cleared cuando instance stops → queries pueden tardar más al resumir
- Provisioning de warehouse puede tardar couple de segundos (añadido a execution time next query)
- Billing: si gaps esperados de 5 min entre workloads y auto_suspend=3 min → constantemente suspending/resuming → billed minimum 60s repetidamente

**2. AUTO_RESUME:**
- Especifica si automáticamente resume suspended warehouse cuando query se submite a él
- TRUE = warehouse "wakes up" si query lo usa mientras suspended
- **Default:** Turned ON
- Puede turnarse OFF para controlar costs (manually resume warehouse solo cuando requerido)

**3. INITIALLY_SUSPENDED:**
- Indica si warehouse se crea inicialmente en suspended state
- **Default behavior:** Crear warehouse en started state
- Usar property si quieres warehouse que NO immediately start billing (ej: solo crear tables inicialmente - actividad que no requiere running virtual warehouse)

## Sizing y Billing

### T-Shirt Sizing

10 sizes disponibles al crear/resize virtual warehouse. Snowflake usa **T-shirt sizing** (relative estimation technique) indicando relative power de underlying compute clusters y servers. Users no necesitan conocer actual cloud platform resources deployed (pueden cambiar over time).

**Sizes:** Extra Small (XS) → Small (S) → Medium (M) → Large (L) → X-Large (XL) → 2X-Large (2XL) → 3XL → 4XL → 5XL → 6X-Large (6XL)

- **XS:** Least underlying resources (CPU, memory, storage)
- **6XL:** Most resources
- **Cada size increase:** Aproximadamente **doubles** underlying compute resources

### Performance vs Size

**Regla general:** Bigger T-shirt size = más compute power = faster queries. Pero solo very general rule of thumb.

**Simple query en small table:** No likely verás performance benefit al increase size.

**Very complex query en very large dataset:** Podrías ver considerable performance benefit sizing up.

**¿Cómo decidir size?** Snowflake recomienda **experimentar** con diferentes workloads. Más complex query (más predicates, projections, joins) y mayor size de table → generalmente larger warehouse needed.

**Proceso recomendado:** Run representative queries de typical workload en XS warehouse. Si run en acceptable timeframe → great. Si no → size up, subsequent queries run en larger size.

**Data loading operations:** Smaller virtual warehouses generalmente sufficient. Snowflake recomienda stick con Small, Medium o Large, a menos que planees ingest hundreds/thousands de files concurrently. Data performance determinado por **número de files** a ingest, **no strongly correlated con compute power** como query performance.

### Billing

**Cost doubles** con cada T-shirt size (como compute power).

**Per-hour credit cost examples:**
- XS = 1 credit/hour
- S = 2 credits/hour
- M = 4 credits/hour
- L = 8 credits/hour
- XL = 16 credits/hour
- 2XL = 32 credits/hour
- etc.

**Per-second billing:** Snowflake usa billing per-second, dando flexibilidad para start/stop warehouses y charged en increments.

**IMPORTANTE para exam:** Virtual warehouses billed cuando están en **started state**, NO basado en cuántas queries se ejecutan usando ellos.

**First 60 seconds:** Siempre charged (minimum billable period). Después, per-second basis. No hay benefit stopping warehouse 30 segundos después de provisioned (60s minimum siempre se cobra).

**Visual example XS warehouse:**
- 0-60 segundos = same billing (1/60 de credit)
- 1 hora = 1 credit
- Minute increments = fractions de credit

### Credit to Dollar Value

Cada Snowflake credit representa **monetary value** basado en:
1. Cloud provider donde account está deployed
2. Snowflake edition del account

**Cost de credit puede cambiar over time.**

**Examples (aproximados, basado en Snowflake service consumption table):**
- XS warehouse, 1 hora, Standard edition, AWS Europe London = 1 credit ≈ $2
- Large warehouse, 3 horas, Enterprise edition, AWS Tokyo = 24 credits ≈ $103

Gran diferencia en costs dependiendo de configuración.


## Scaling de Virtual Warehouses

Conforme workload cambia over time (ej: ingest 100x más data, onboard 20+ analysts), necesitas métodos para scale compute para meet new demands. Virtual warehouses scale de dos formas primarias: **scale up** y **scale out**.

### Scale Up

Proceso de alter size de **individual virtual warehouse** con objetivo de improve query performance. `ALTER WAREHOUSE ... SET WAREHOUSE_SIZE = ...` command puede cambiar size de running warehouse (ej: XS a Large).

**Cuándo scale up:** Query taking unusually long to complete (no stuck en queue, sino single query running very long). Quizás dataset muy large o query sufficiently complex para burden current size warehouse.

**Para exam:** Scaling up = improve query performance.

**Verificación:** Query profile tool en Snowflake UI para verificar si query está exceeding resources de warehouse (indicaciones: spilling to disk, spilling to remote disk).

**No automatizado:** Requiere direct user interaction. Puede hacerse vía UI también.

**Queries existentes:** Resizing running warehouse **no detiene queries** que ya lo usan. Extra compute resources de larger size solo used para **queued y new queries**.

**Decreasing size:** Remueve compute resources de warehouse. Cuando resources removed, **cache associated dropped** (como suspending warehouse). Puede negative impact query performance (ya no tiene raw table data en cache, debe hacer call a remote long-term storage).

### Scale Out (Multi-Cluster Warehouses)

Habilitado por feature **multi-cluster warehouses**: named pools de virtual warehouses que automáticamente add/remove individual warehouses dynamically en response a número de concurrent users o queries. Intención: improve **query throughput** cuando many users issuing queries simultaneously.

**Parámetros:**
- **MIN_CLUSTER_COUNT:** Minimum número de warehouses en multi-cluster warehouse
- **MAX_CLUSTER_COUNT:** Maximum número de warehouses

#### Modos de Multi-Cluster Warehouse

**1. Maximized Mode:** Setting MIN y MAX al **mismo número**. Mantiene static number de virtual warehouses running. Usado cuando concurrent user sessions/queries **no fluctúan significantly**.

Example: MIN=4, MAX=4 → multi-cluster warehouse siempre hecho de 4 virtual warehouses (a menos que suspended/dropped).

**Nota:** All virtual warehouses en Enterprise+ editions configuradas como multi-cluster pero en maximized mode con MAX=1, MIN=1.

**2. Auto-Scale Mode:** Setting MIN y MAX **differently**. Snowflake add/remove warehouses dynamically based on usage curve.

**SCALING_POLICY property:** Controla cómo multi-cluster warehouse scales en auto-scale mode. Acepta dos valores:

**Standard Policy:** Objetivo = minimize queuing, más aggressively scaling out warehouses.
- Cluster inicia con MIN_CLUSTER_COUNT warehouses
- Additional warehouse añadido **immediately** cuando query is queued
- **Scale down:** Check every 1 minute si load en least busy warehouse puede redistributed a otros warehouses con spare capacity. Si sí, después **2-3 minutes** warehouse marked for shutdown, subsequent queries rebalanced

**Economy Policy:** Objetivo = conserve credits. Warehouses run fully loaded for longer.
- Cluster inicia con MIN_CLUSTER_COUNT warehouses
- Additional warehouse solo añadido si system estimates hay **enough query load** para keep new warehouse busy **at least 6 minutes** (puede lead a queries queuing longer que standard mode, pero longer period determina si increase en usage es anomalous o genuinely increasing enough para warrant adding warehouse)
- **Scale down:** Check every 1 minute, pero warehouse marked for shutdown después **5-6 minutes** (vs 2-3 en standard)

### Billing Multi-Cluster Warehouses

**Total credit cost** = sum de all running individual warehouses en cluster.

**Maximum credits consumption:** MAX_CLUSTER_COUNT × hourly credit rate de warehouse size.

Example: Medium-sized multi-cluster warehouse con 3 warehouses = 12 credits/hour (si all 3 running durante hour completa).

**Realidad:** Multi-cluster warehouses scale in/out based on demand → typical obtener fraction de maximum credit consumption (un warehouse run 2 horas, otro shutdown después 1 hora → billed solo cuando active = total 3 horas).

### Warehouse Properties para Queuing y Concurrency

**1. MAX_CONCURRENCY_LEVEL:**
- Determina cuántos concurrent SQL statements pueden submitted a warehouse antes de queued o additional compute provided
- Multi-cluster maximized mode: additional compute = manual scaling up
- Auto-scale mode: automatic addition de warehouses
- **Default:** 8 queries

**2. STATEMENT_QUEUED_TIMEOUT_IN_SECONDS:**
- Time en segundos que SQL statement puede estar queued en warehouse antes de aborted
- Aplicable a warehouse específico o globalmente como session parameter
- **Default:** No timeout (query puede wait indefinidamente para compute resources)

**3. STATEMENT_TIMEOUT_IN_SECONDS:**
- Similar a anterior pero no solo para queued queries
- Time en segundos después del cual any running SQL statement es aborted
- Usado para stop queries running por unreasonable amount de time

## Resource Monitors

Standalone objects que permiten set **credit consumption limits** en user-managed virtual warehouses. Pueden aplicarse a account level (monitoring múltiples warehouses en account) o individual warehouses.

**Propósito:** Track credit consumption a high level para prevent runaway costs.

**Límites:** Pueden set para specified interval o date range (daily, weekly, monthly). Cuando límites reached o approaching, resource monitor puede trigger actions: send alert notifications o suspend warehouse.

**Creación:** Solo **ACCOUNTADMIN** role puede crear resource monitors. Account admins pueden allow otros roles view/modify resource monitors.

### Properties de Resource Monitor

**CREDIT_QUOTA:** Número de credits (any size) usado como upper limit para determinar cuándo certain action es triggered.

**FREQUENCY:** Cuándo credit quota será reset (restarting monitoring). Accepted values: DAILY, WEEKLY, MONTHLY, YEARLY, NEVER. Example: DAILY con 100 credits = 100 credits/día base para monitoring notifications. NEVER = credit quota never resets, assigned warehouse continúa usando credits hasta reach quota.

**START_TIMESTAMP:** (Opcional) Default = resource monitor starts monitoring immediately. Puede cambiar para start later. Frequency value sería relative a newly specified start timestamp.

**TRIGGERS:** Compuesto de dos partes:
1. **Condition:** Expresado como percentage de credit quota
2. **Action:** Qué hacer al reach condition

**Actions disponibles:**
- **NOTIFY:** Send alert notification a all account admins con notifications enabled
- **SUSPEND:** Send alert notification + suspend virtual warehouse(s) attached a resource monitor, pero solo **after** currently running queries completed
- **SUSPEND_IMMEDIATE:** Similar a SUSPEND pero **no wait** para queries complete. Hard stop, abort running queries y suspend virtual warehouses

**Aplicación:** Una vez creado, puede aplicarse a **single virtual warehouse** o a **account level**.

## Query Acceleration Service (QAS)

Feature que se habilita en virtual warehouse para dynamically add **serverless compute power** solo cuando sufficiently complex query lo necesita. Contraste con multi-cluster warehouses (created/defined por users): cómo additional compute resources allocated con QAS está completamente **controlled by Snowflake** (classified como serverless feature).

### Funcionamiento

Cuando QAS enabled en warehouse:
1. Snowflake analiza query plan de submitted query
2. Offload fragments que pueden run in parallel a dynamically requisitioned serverless compute
3. Example: joining 3 tables → offload large table scan operation a serverless compute → join result con otras table scans en virtual warehouse's compute
4. Query completa → serverless compute relinquished → solo pagas por task performed helping complete complex query

**Beneficio:** Speeds up query execution → improves overall utility de warehouse (free para work en otras queries).

### Use Case Example

**Scenario:** Team de data analysts usando Large virtual warehouse para query transform table. Plot utilization across typical day:
- **Most of time:** Run similar queries (simple/medium complexity), never fully utilizing warehouse
- **Occasionally:** Analyst runs ad hoc complex query reading large data → fully utilizes warehouse → other analysts' queries start getting queued

**Options sin QAS:**
1. **Scale up:** Increase warehouse size → pagar por larger warehouse incluso cuando utilization low
2. **Multi-cluster warehouse:** Works pero requiere setup, puede lead a paying more que expected (depends on scaling policy)

**QAS aim:** Provide burst de compute behind scenes → query causing spike finishes quicker → improves concurrency.

### Queries Elegibles para Acceleration

**Importante:** Turning ON QAS NO necessarily boost every query que tarda too long. Solo **certain queries** pueden accelerated.

**Dos factores primarios dictating elegibilidad:**
1. **Parte del query debe poder run in parallel** (ej: scan con aggregation)
2. **Número de partitions a escanear** (size de data being queried)

**Verificar elegibilidad:**
- `QUERY_ACCELERATION_ELIGIBLE` view
- `SYSTEM$ESTIMATE_QUERY_ACCELERATION()` function

### SYSTEM$ESTIMATE_QUERY_ACCELERATION Function

**Input:** Query ID de previously executed query.

**Output:** JSON object con dos formas:

**1. Query INELIGIBLE:**
```json
{
  "estimatedQueryTimes": {},  // Empty
  "originalQueryTime": <seconds>,
  "queryId": "<id>",
  "eligible": false,
  "upperLimitScaleFactor": null
}
```

**2. Query ELIGIBLE:**
```json
{
  "estimatedQueryTimes": {
    "1": <seconds>,
    "2": <seconds>,
    "3": <seconds>
  },
  "originalQueryTime": <seconds>,
  "queryId": "<id>",
  "eligible": true,
  "upperLimitScaleFactor": <max_scale_factor>
}
```

**estimatedQueryTimes:** Object asociando estimated execution time (segundos) con **scale factor**.

**Scale factor:** Option que set en warehouse determinando cuántas veces el size de current warehouse quieres que query tenga available para accelerate. Usar original query time con estimated improvements para determinar dónde tiene sentido set scale factor (weighing cost vs performance).

**upperLimitScaleFactor:** Highest scale factor en estimatedQueryTimes object. Indica punto donde recibes maximum benefit scaling up.

**Nota:** Function es solo **estimate** de si query podría potentially usar QAS. Para ver si works in practice, necesitas enable y monitor effects.

**Monitoring:** Query profile tool → statistic "scan selected for acceleration" + total query runtime.

### Habilitación y Cost

**Enable QAS:** `CREATE WAREHOUSE ... ENABLE_QUERY_ACCELERATION = TRUE` o `ALTER WAREHOUSE ... SET ENABLE_QUERY_ACCELERATION = TRUE`

**MAX_SCALE_FACTOR:** Controls cuántos compute resources QAS usará, expresado como **multiplier de warehouse size**.

**Setting:** `QUERY_ACCELERATION_MAX_SCALE_FACTOR = <value>`
- **Default:** 8
- **Range:** 0-100 (0 = no limits en cuánto compute será leased para accelerate eligible queries)
- **Upper limit reason:** Si query puede accelerated/completed con less compute, lo hará. Si Snowflake can't provision enough resources in time (not available en shared pool), otra razón para no reach upper scale factor

**Credit Consumption:**

Turning ON QAS y upping scale factor puede **potentially increase credit consumption**.

**Example:** MAX_SCALE_FACTOR=3, Large warehouse (8 credits/hora)
- **Maximum additional credits QAS:** Si accelerating queries por 1 hora continuously = 24 credits (3 × 8)

**Billing:** QAS es serverless feature, billed **every second** QAS está accelerating query. Example: single large query causing queuing, solo needs ON por 1 minuto → solo 60 segundos billed.
