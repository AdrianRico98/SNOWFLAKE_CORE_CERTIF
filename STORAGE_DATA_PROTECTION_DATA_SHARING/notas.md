# Storage, Data Protection y Data Sharing

## Introducción

Sección final covering: secure data sharing, data marketplace, data exchange, deep dive en storage layer (including micro-partitions), data protection features (time travel, fail-safe, cloning, replication), y storage billing.

## Micro-Partitions Deep Dive

### Loading Process y Physical Storage

**Example scenario:** Ingest simple CSV file (3 fields) en Snowflake table → storage layer.

**COPY INTO table command execution kicks off:**
1. **Reorganization process:** Transforms input file en Snowflake's own optimized file format
2. **Transparent partitioning:** Input file rows partitioned, forming discrete files llamados **micro-partitions** (containing subset de data)

**Key difference vs traditional systems:** User **NO define key** por el cual table es partitioned durante loading. Done **automatically por Snowflake**.

### Automatic Partitioning Logic

**¿Qué usa Snowflake para partition input data?**

Snowflake automatically partitions based en **natural ordering de input data as it's loaded**. NO reorder input data, simply divide based en **order en cual arrives**.

**Physical files:** Stored together en cloud blob storage. Range en size: **50-500 MB uncompressed data**. Llamados "micro-partitions" porque large table podría potentially tener millions de estos relatively small partition files.

**Example nota:** Minimum size para micro-partition = 50 MB. Very small CSV file only formaría one micro-partition (no two como en example ilustrativo).

### Columnar Storage Format

**Reorganization process:** Column values en micro-partition grouped together. All data para one attribute/column stored **contiguously on disk** (opposed a row store donde all data de row stored together).

**Columnar store benefits:**

**1. Optimal compression:** Apply compression a single data type para column → optimal compression scheme para ese data type → keeping storage down.

**2. Column pruning:** When querying, solo columns en projection need retrieved from file → level de column pruning en micro-partition.

### Immutability y Versioning

**Immutable:** Once partition written, **cannot be altered**. Write once, read many.

**DML operations effect:** Execute UPDATE operation → completely **new partition written**, forming latest version de table.

**Versioning functionality:** Underpins lot de features around data protection (ej: time travel feature allowing restore previous version de table data = restoring old micro-partition).

**Metadata Service responsibility:** En global services layer de Snowflake architecture, responsible keeping track de micro-partition versioning metadata.

### Metadata Management

**Table-level metadata:** Structure de table, row count, table size, much more.

**Micro-partition-level metadata stored:**

**1. MIN y MAX values:** Para each column en micro-partition. Example: MIN order_ID para micro-partition 1 = 1, MAX order_ID = 3.

**Key benefit:** Si run MIN/MAX function en column de table, **won't require running virtual warehouse**. Pull result from metadata store.

**2. Number de distinct values** en micro-partition.

**3. Number de micro-partitions** que make up table.

**Summary:** System maintains understanding de **distribution de data en micro-partitions**.

### Micro-Partition Pruning

**Concept:** MIN/MAX metadata key para understanding important concept.

**Process:** Conforme keep loading files, new micro-partitions created. Snowflake can **optimize queries** incorporating constraint en column:
1. First checking MIN/MAX values en metadata store
2. **Discarding micro-partitions** from query plan

**Key benefit:** Limit reading micro-partitions from storage layer a **only those needed** para compute query result.

**Example:** Query con predicate `WHERE order_id > 360 AND order_id < 460`:
- Puede safely **prune/ignore** micro-partitions que no contienen values en ese range
- Metadata typically quite smaller que actual data → querying metadata quicker que scanning values de entire micro-partition

## Time Travel y Fail-Safe

Snowflake es fully updateable relational database con complete set de DML SQL capabilities (update rows, delete rows, drop objects). Risk de data loss. Para address: implemented **two unique measures** to safeguard against accidental errors, system failures, malicious acts.

### Lifecycle de Table Data (Tres Steps)

**1. Active Data:** Makes up table. Comprises most up-to-date micro-partition versions.

**2. Time Travel:** Once micro-partitions out-of-date (updated/deleted), pueden sit en storage por **configurable retention period**. Allows go back in time para view data as it was en past.

**3. Fail-Safe:** After time travel retention period elapsed. Snowflake maintains copy de data por **7 días**, pero solo recovered **on request**.

### Time Travel

**Three main functions:**

**1. Restore objects:** Tables, schemas, databases deleted usando `UNDROP` command.

**2. Analyze table data** at point en past: Query usando special SQL extensions. Syntax: `SELECT ... FROM table AT ...` (using AT keyword choosing specific point en past).

**3. Combine con cloning:** Create copies de objects from point en past.

### Time Travel Retention Period

**Definition:** Configurable number de días Snowflake mantiene out-of-date micro-partitions para table. Effectively defines period en past we can go back into para perform time travel data restoration tasks.

**Configuration:** Parameter `DATA_RETENTION_TIME_IN_DAYS`.

**Levels donde puede setarse:**
- Account level (requiere ACCOUNTADMIN role)
- Database level
- Schema level  
- Individual table level

**Precedence:** Si set en all levels, **child object takes precedence**. Example: database set a 90 días, pero table set a 10 días → table solo puede use time travel features up to 10 días en past.

**Default retention period:** **1 día** para all editions en account/database/schema/table level. Si never play con parameter, always tienes 1 día para undrop tables y look at table data up to 24 horas en past.

**Edition differences:**

| Edition | Min Days | Max Days (Permanent) | Max Days (Temporary/Transient) |
|---------|----------|---------------------|-------------------------------|
| **Standard** | 0 | 1 | 0 o 1 |
| **Enterprise+** | 0 | 90 | 0 o 1 |

**Setting to zero:** Effectively turns off time travel feature para specific object. Si set a zero en account level, all objects que no tienen retention period specifically defined would effectively have time travel disabled.

### Time Travel SQL Keywords

**1. AT Clause:**

Añadir a end de SELECT query permite view table as it was at time de running statement. **Includes changes made por statement** usado como input.

**Input parameters:**
- **Statement ID:** Query executed (inclusive de changes - ej: si statement inserted 10 rows, result contiene those)
- **OFFSET:** Seconds en past to look back
- **TIMESTAMP:** Exact time en past

**2. BEFORE Keyword:**

Similar a AT pero **NOT inclusive** de statement usado como input. Si use statement ID para query que inserted 10 rows, result **would NOT include** those 10 records.

**Input parameter:** Solo statement ID.

**3. UNDROP Command:**

Cuando table/schema/database dropped, kept por duration de configured retention period. Durante time, UNDROP restores **most recent version** de dropped object.

**Error condition:** Si table/schema/database ya exists con name de dropped object being restored → error returned.

**Check dropped objects:** `SHOW <OBJECT> HISTORY` command.

### Fail-Safe

**Definition:** **Non-configurable period de 7 días** después de time travel retention period. Historical data puede recovered contactando **Snowflake Support**.

**Intended use:** Ensure historical data protected en event de system failure o catastrophic event (ej: hardware failure, security breach). **Reserved para last resort data recovery**, not intended frequently accessed como time travel.

**Recovery time:** Could take several hours o several days para Snowflake complete recovery.

**Applies to:** **Permanent objects only** (not transient/temporary).

**Storage costs:** Data en fail-safe Y time travel both **contribute to data storage costs**.
