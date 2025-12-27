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
