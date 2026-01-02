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


## Cloning

### Concepto General

**Cloning:** Process de creating copy de existing object dentro Snowflake account.

**Syntax básico:** 
```sql
CREATE TABLE clone_name CLONE source_table;
```

Provide identifier para clone + keyword `CLONE` + identifier de source object to clone.

### Objects que Pueden Clonarse

**Direct cloning (usando CLONE keyword):**
- Tables
- Streams
- Stages
- File formats
- Sequences
- Tasks
- Pipes (limitación: solo pipes referencing **external stage** pueden clonarse, not internal stage)

**Recursive cloning (databases y schemas):**

Cloning containers es **recursive** y could include objects que **can't be directly cloned** usando CLONE keyword. Example: clone schema con view en él → cloned schema también contains view, despite fact you can't directly clone view.

**Never cloned:** External tables, internal named stages.

### Zero-Copy Cloning

**Metadata-only operation:** Copies properties y structure de source object. Simpler objects (ej: file formats) just named wrappers para properties.

**Objects managing micro-partitions (ej: tables):**

Execute CLONE command en table → **NOT actually copying data files** de source table. Making identical **metadata version** de object que **points to source table's micro-partitions**. Really not duplicating any data → **cloning does NOT contribute to storage costs** hasta data modified o new data added to clone.

**Funcionamiento visualizado:**

1. **Initial state:** Clone initially just points a existing micro-partitions de source table
2. **Query clone:** Returns results pulled from source table's micro-partitions (identical en structure/data)
3. **Changes to clone:** Inserting new rows after cloning → start creating **additional micro-partitions** under clone (estos incur additional storage cost)
4. **Result:** Clone table ends up con mixture de **shared micro-partitions** (original) + **independent micro-partitions** (new)

**Independence:** Changes made to source o clone **NOT reflected** entre each other. Completely independent objects.

**Cloning cloned objects:** Puede clone already cloned object con nearly no limits.

**Speed:** Process not instant (depends on qué cloning) pero very quick. Allows rapidly create clone (particularly good para testing purposes - ej: quickly get copy from live environment para perform integration tests).

### Caveats al Usar Cloning

❌ **Privileges:** Cloned object **does NOT retain granted privileges** from source object sin specify additional command `COPY GRANTS`. **Exception:** Tables. SYSADMIN o owner de cloned object puede manually grant privileges to clone after created.

❌ **Recursive cloning rules:** Cloning database clones all schemas/tables + other objects (views, pipes, stages, etc.). Cada tiene own specific rules. Example: pipe cloned como part de clone database command solo copies pipes referencing **external stage** (not internal).

❌ **Load history:** Clone table **does NOT contain load history** de source table (no memory de qué files uploaded para make source table). Could lead to situation donde duplicate data ingested en clone.

❌ **Table type restrictions:** 
- Temporary/transient tables solo pueden cloned como temporary/transient tables (not permanent)
- Permanent tables pueden cloned como temporary, transient, O permanent

### Cloning con Time Travel

Possible combine time travel + cloning para create table/schema/database/stream clone from **point en past** dentro object's retention period.

**Syntax:** Use `AT` clause (o `BEFORE` clause):
```sql
CREATE TABLE clone_name CLONE source_table AT (TIMESTAMP => '2024-01-01 12:00:00');
```

**Error condition:** Si source object didn't exist at time specified en AT/BEFORE parameter → error thrown.

## Replication vs Cloning

### Replication

**Definition:** Feature enabling replicating databases **between Snowflake accounts** dentro single organization.

**Typical use:** Ensure business continuity y disaster recovery si entire cloud provider region goes down (outage).

### Key Differences

| Aspecto | Replication | Cloning |
|---------|-------------|---------|
| **Scope** | Between accounts dentro organization | Within one account |
| **Data movement** | Data **physically moved** | Cloned object solo **references underlying storage vía metadata** |

### Setup de Replication

**Prerequisites:**
- Organization feature enabled para account
- User con ORGADMIN role

**Steps:**

**1. Enable replication:** Set account parameter `ENABLE_ACCOUNT_DATABASE_REPLICATION = TRUE` para each source y target accounts. Si no tienes organization feature enabled → contact Snowflake Support.

**2. Select primary database:** En source account.

**3. Create secondary database:** En target account como replica de primary database. Read-only replica. Running command kicks off initial process para transfer database objects + data a secondary database.

**4. Periodic refresh:** Secondary databases pueden refreshed periodically con snapshot de primary database, replicating all data + DDL operations en database objects.

### Additional Information

❌ **Not replicated:** Algunos database objects currently no replicated (includes external tables).

✅ **Automation:** Command para perform incremental refresh de secondary database puede automated configurando task para run refresh command en schedule.

❌ **Refresh operation fails:** Si primary database contains event o external table type.

❌ **Privileges:** Granted privileges a database objects **NOT replicated** to secondary database.

**Billing:** Based en:
- **Data transfer:** Initial database replication + subsequent synchronization operations transfer data between regions (cloud providers charge for this)
- **Compute resources:** Snowflake resources required para copy data between accounts

## Storage Billing

### Qué se Billa

Data storage cost calculated **monthly** based en average number de **on-disk bytes** para all data stored each day en Snowflake account.

**"All data" refers to dos areas:**

**1. Database Table Data:** Comprises data from each step en data lifecycle:
- Most current up-to-date micro-partitions
- Older versions de micro-partitions (enabling Time Travel)
- Micro-partitions Snowflake maintained para enable Fail-Safe

**All contribute to cost.** 90-day retention period en all tables = powerful data recovery capabilities pero potentially greatly increase billing amount. Weighing pros/cons when toggling retention period.

**2. Internal Stage:** Anything stored en internal stages contributes to storage costs. Unlike table data, we decide how files look en stages (ej: upload very large uncompressed file y forget delete → contributes to storage billing).

### Cómo se Calcula

**Problema:** Can add/remove data at any point durante billing window.

**Snowflake's calculation:**
1. **Daily average:** Calculate every 24 hours average amount de data stored en dos areas
2. **Monthly average:** At end de month, take another average across all days en month
3. **Billing:** Figure billed based en **flat rate per terabyte**

**Rate variation:** Amount charged per TB varies based en:
- Region
- Purchasing plan (Capacity vs On-Demand)

**Example rate:** On-Demand plan, AWS London Region = $42 USD per TB.

### Examples de Calculation

Using Snowflake's Credit Consumption Table document (contains subset de cloud providers' regions + respective TB/month cost). Amount (USD) affected por:
- Cloud provider (AWS, Azure, GCP)
- Region
- Pricing plan (On-Demand vs Capacity)

**Example 1:** Average 15 TB stored durante month, account deployed en AWS London Region = **$630** (as of November 2021).

**Example 2:** Average 14 TB stored durante billing month, account deployed en AWS AP Mumbai Region = **$644**. Billed more money para less data purely por being en different region.
