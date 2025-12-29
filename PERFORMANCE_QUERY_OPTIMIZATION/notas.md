# Performance Concepts: Query Optimization

## Query Performance Analysis Tools

A pesar de que Snowflake es managed service con la mayoría de low-level tasks (data partitioning, compression, encryption) handled automáticamente, hay áreas que necesitan monitoring cercano para ensure optimal query performance. Ejemplo: escribir well-formed robust SQL es igual de relevante en Snowflake que en Oracle. Poorly optimized SQL manteniendo warehouse busy por hora adicional (preventing suspended state) puede contribute significantly a monthly bill.

**Importante:** Necesitamos métodos para identify y analyze cuando algo no está running optimally.

### Tres Herramientas Principales

**1. Query History:** Encontrado en classic console y Snowsight UI.

**2. Query History Views y Table Functions:** Para programmatic analysis de query performance.

**3. Query Profile:** Breakdown de steps involved en ejecutar query para pinpoint issues.

### Query History Page

**Características:**
- Solo almacena query history para **last 14 days**
- Users pueden ver other users' query text, pero **NO pueden ver query results** de otros users
- Available para all users independientemente de active role durante session
- Interactive tabular view de queries executed en last 14 days
- Puedes ver other users' queries si tienes OPERATE o MONITOR privilege en warehouse usado para execute query

**Filtros:** Puede filtrarse por diferentes criteria para isolate particular query (ej: all failed queries, identify long-running queries).

**Duration column:** High-level breakdown de query steps y cuánto tardó cada uno.
- **Compilation time:** Muy rápido, query engine performing operations (cost-based optimization, micro-partition pruning)
- **Execution time:** Steps para actually process data (processing time en CPU, reading/writing a local/remote disk)

**Columnas adicionales:** Bytes scanned, rows (sense de size de data being computed).

### Query Details Page

Click en query → expanded version de information en table view. Query results disponibles at bottom si eres el user que ejecutó query.

**Export results locally:** Límite **100 MB**, solo CSV o TSV format.

### Query Profile

**Herramienta más importante** para analyzing query performance. Breaks down single query en steps que query engine took para execute. Útil para understanding mechanics de query, helping diagnose issues y areas de improvement.

**Similar a EXPLAIN command:** EXPLAIN retorna execution plan de query **not yet executed**, detailing operations que Snowflake **would perform**. Query profile details operations que Snowflake **actually did perform** (graphically represented).

**Componentes:**
- **Operator nodes:** Cada selectable box es operator node. Línea representing relationship entre nodes con número de records passed between them (shown over arrow). Juntos forman **operator tree**
- **Most expensive nodes:** Box showing cuál operator node fue most expensive en terms de duration to complete
- **Profile overview:** Más granular breakdown de execution time

**Execution time breakdown:**
- **Processing:** % de total execution time spent en data processing por CPU de virtual warehouse
- **Local Disk I/O:** % de time processing was blocked por local disk access (cuando VW tuvo que read/write data desde local SSD storage)
- **Remote Disk I/O:** Cuando VW tuvo que read/write data desde remote storage layer

**Statistics:** Input/output, operations performed, información sobre pruning de micro-partitions.

**Operator information:**
- **Operator type:** Ej: table scan (operation to read from single table)
- **Operator ID:** En square brackets
- **% de query time:** Cada node took
- Click en node → additional information con relevant statistics/attributes para ese operator type

### Programmatic Query History Analysis

**Account Usage View (QUERY_HISTORY):**
- En ACCOUNT_USAGE schema de SNOWFLAKE database
- Investiga programmatically query history **within last year**
- Default: solo ACCOUNTADMIN tiene permissions to read
- **Latency:** ~45 minutos (query ejecutada no guaranteed aparecer hasta al menos 45 min después)
- Sheer number de columns disponibles para análisis

**Use case:** Get 10 longest running queries (expresado en segundos).

**Information Schema Table Function (QUERY_HISTORY):**
- Disponible en cada database's INFORMATION_SCHEMA
- Data solo **going back 7 days**
- **Benefit:** Practically **zero latency**
- Misma funcionalidad (ej: 10 longest running queries)

**Nota:** Puedes ver different results entre information schema table function y account usage view por difference en latency y data retention.

## SQL Query Optimization Best Practices

No es difícil crear SQL query que tarde considerablemente más de lo esperado en ejecutar, especialmente al join many tables, usar subqueries o crear pipeline steps. Importante optimizar SQL para evitar performance limitations por poorly implemented SQL.

### Order of Execution

Query optimizer aplica operations en orden específico:

1. **Row operations** (FROM, JOIN, WHERE): Prune micro-partition files
2. **Grouping** (GROUP BY, HAVING)
3. **Result operations** (SELECT, DISTINCT, ORDER BY, LIMIT): Afectan result que user ve

**Row operations primero:** Decrease total amount de data used para more expensive operations (ej: GROUP BY). Less data = faster operations.

**Recomendación:** Prioritize including WHERE clause, focus en dónde exactamente usarlo. Si hay subquery, filtrar en subquery o outer query? Ensure usar WHERE clause **as early as possible**.

### Join Best Practices (Evitar Join Explosions)

**Problema común:** Join que genera más rows que expected. Ocurre cuando column usado en join predicate **no es unique**.

**Example:** Orders table (unique order per row) joined con Products table. Si Products table tiene duplicate product names (ej: current price + outdated price), output genera additional rows (join explosion). Problema compounded con millions de rows, considerably increasing query time.

**Identificación en Query Profile:** Above join operator, número de rows output muestra additional rows añadidas.

**Soluciones:**
- **Join on unique columns:** Row de left table joins a exactly one row en right table
- **Understand table relationships:** Tables pueden no estar perfectly normalized. Entender cómo fields en múltiples tables relate y impact de joining on them. Example: si Products table contiene pricing history, implementar way para get solo latest price o usar different table con one row per product (most recent date)

### ORDER BY Optimization

**Problema:** ORDER BY ordering lot de records es expensive operation.

**Identificación:** Query profile muestra SORT operator taking significant % de total duration. Cuando calculating sort, intermediate results need storage. Si virtual warehouse **in-memory storage runs out → spilling to local SSD storage** (in-memory mucho más rápido que local storage). Peor caso: **local storage fills up → spilling to remote storage** (aún más lento, shown en query profile overview como "bytes spill to remote storage").

**Soluciones:**
1. **Include LIMIT clause:** Produce SORT WITH LIMIT operator en query profile, finishes en fraction de time
2. **Process less data:** Add WHERE clause para reduce número de micro-partitions scanned
3. **Increase virtual warehouse size:** Larger size = más memory/CPU en underlying servers = less likely spill to local/remote disks

**Position de ORDER BY:** Si hay two ORDER BYs (uno en subquery, otro en outer query), ORDER BY en subquery es **redundant** y waste de compute resources (reordering result nuevamente en outer query). **Recomendación:** Just one ORDER BY en top-level SELECT.

### GROUP BY Optimization

Permite calculate aggregates based on rows sharing same value. Importante considerar **cardinality** de column grouping (drastically impacts query performance).

**Cardinality:** Measure de how unique column is.

**Low/Medium cardinality** (ej: country - limited number de countries):
- Query profile: aggregate took small % de total duration (ej: 3.5%)
- Processing small % de time (ej: 6%)

**High cardinality** (ej: unique ID, timestamp, full name - many distinct values):
- GROUP BY operation becomes very expensive
- Must perform aggregate (ej: COUNT) en many small groups
- Query profile indicators: spilling to local storage, aggregate operator took significant % (ej: 32.6%), proportion de processing time over 50%

**Recomendación:** Si aggregate operation taking disproportionate amount de time, look at cardinality de column grouping. Puede ser easy way para squeeze out más performance.

## Caching en Snowflake

Tres storage mechanisms behind scenes con caching-like behavior:

### 1. Metadata Cache (Cloud Services Layer)

**También llamado:** Metadata store, cloud services layer, services layer (nombres usados interchangeably en exam).

**Qué es:** Highly available service en cloud services layer manteniendo metadata sobre object information y statistics.

**Qué almacena:**
- Metadata importante para various entities en Snowflake
- Understanding de tables creadas, databases donde están
- Micro-partitions que componen tables
- Clustering information y más

**Beneficio:** Habilita queries ejecutadas **sin necesidad de virtual warehouse**.

**Examples de uso:**
- Row count de cada table (ej: `SELECT COUNT(*) FROM table` no usa VW)
- System functions
- Context functions
- DESCRIBE commands
- SHOW commands

### 2. Results Cache (Cloud Services Layer)

**También llamado:** Result set cache, 24-hour cache, query result cache (keywords en exam: result, 24-hour, query).

**Qué es:** Cache de results de queries por **non-configurable duration de 24 horas**. Cada vez que persisted result se reusa, 24-hour retention period se extiende hasta **maximum de 31 días**, después result purged from cache.

**Beneficio principal:** Evitar regenerate costly results cuando nothing has changed.

**Reglas para uso de Results Cache:**

✅ **New query debe syntactically match** previously executed query (even añadir LIMIT causa que result cache no se use)

✅ **Table data contributing al query result NO debe haber cambiado**

✅ **Available across virtual warehouses** y across users (siempre que tengan role con required privileges)

❌ **NO se usa si:** Time context functions usadas (ej: CURRENT_TIME)

**Control:** Por default, result reuse enabled. Puede overridden a account/user/session level usando parameter `USE_CACHED_RESULT`.

### 3. Local Disk Cache / Warehouse Cache (Query Processing Layer)

**También llamado:** SSD cache, data cache, raw data cache (porque data no stored en aggregated form - son raw micro-partition files themselves).

**Qué es:** Mientras virtual warehouse running, mantiene micro-partition files en local SSD conforme queries procesadas.

**Beneficio:** Improved performance para subsequent queries (pueden read from cache en lugar de slower remote blob storage).

**Size:** Increase warehouse size = increase local cache size. Cache es **finite en size** incluso para very large warehouses.

**Cache management:**
- **Least Recently Used (LRU) replacement policy:** Para purge cache contents, keeping most useful para users
- **Purged cuando:** Virtual warehouse resized, suspended, o dropped

**Impact de purge:** Puede resultar en slower initial performance para some queries after warehouse resumed. Conforme resumed warehouse runs queries, cache rebuilt y queries que pueden take advantage de cache experience improved performance. **Factor esto al decidir value de warehouse's AUTO_SUSPEND property**.

**Diferencia con Results Cache:** Warehouse local storage puede **partially used** (retrieving remaining data necessary para complete query from remote storage).

### Jerarquía de Cache Usage

Cuando query no puede satisfacerse por:
1. Metadata Cache →
2. Results Cache →
3. Local Disk Cache →
4. **Retrieve required data from remote disk en storage layer** (most costly operation)

## Clustering

### Concepto de Clustering

**Definición:** Way de describir distribution de column's values.

**Optimal clustering:** Data ordenada, sin overlap entre groups. Usando summarized information, puedes exclude groups y focus search en specific group.

**Poor clustering:** Values no ordered. Para find instances de value, cada group debe checked.

### Clustering en Snowflake

**Natural clustering:** Table data partitioned y stored en **order it was loaded**. Data ordered prior to ingestion será naturally clustered (stored en micro-partitions next to each other).

**Example:** CSV file con ORDER_ID, PRODUCT_ID, ORDER_DATE cargado produciendo 3 micro-partitions:
- **Sequential numerical types** (ORDER_ID) arriving ordered → evenly distributed entre micro-partitions. ORDER_ID unlikely appear en multiple micro-partitions (little overlap)
- **ORDER_DATE:** También closely grouped, single date no span all micro-partitions (just subset)
- **PRODUCT_ID:** Likely appear across different micro-partitions, no ordered prior to ingestion (like jumbled letters)

**Importancia:** Tables well-clustered aumentan odds de **micro-partition pruning** (process de ignorar micro-partitions not required para compute query result). One de surest ways para improve query performance.

**Example:** Looking for order 7, usando min/max metadata associated con micro-partition, puedes ignore micro-partitions 1 y 2, solo retrieve data from micro-partition 3. Query performance better filtering en ORDER_ID que en PRODUCT_ID.

### Clustering Metadata

Snowflake mantiene clustering metadata para micro-partitions en all tables:

**Metrics:**
1. **Total number de micro-partitions** comprising table
2. **Number de micro-partitions** con values que overlap con each other (para table column)
3. **Depth** de overlapping micro-partitions: Cuántas micro-partitions debes look into para find one value (correlated con average overlap)

**System Functions para acceder metadata:**

**1. SYSTEM$CLUSTERING_INFORMATION():** Retorna total micro-partition count, constant partition count, average overlaps, average depth, average depth histogram para table.

**2. SYSTEM$CLUSTERING_DEPTH():** Subset de clustering information function, solo average depth para una o más columns en table.

### Understanding Overlap vs Depth

**Scenario 1 (Worst case):** All micro-partitions tienen same range de values. Finding letter J requiere look through all micro-partitions. 3 micro-partitions overlap, depth=3 (depth y overlap same porque must look en all 3).

**Scenario 2 (Slightly better):** All 3 micro-partitions overlap pero solo **partially** (no entire range). Finding individual letter requiere at most look en 2 micro-partitions. Clustering depth=2 (no longer 3).

**Scenario 3 (Ideal):** All 26 letters evenly distributed entre 3 micro-partitions. **No overlap** entre micro-partition values. Must look en at least 1 micro-partition para find single value. Column values evenly distributed = **constant state** (ideal para clustering, pero unlikely en real world).

**Main takeaway:** **Higher overlapping number de micro-partitions y clustering depth = worse column clustering.**

### Clustering Keys (Automatic Clustering)

**Problema:** Natural clustering puede degrade over time, especialmente si table es large con lot de DML statements rearranging micro-partitions.

**Solución manual:** Sort data y reinsert en table (costly y time-consuming).

**Automatic clustering:** Designate una o más table columns/expressions como **clustering keys** para table (también pueden definirse en materialized views). Una vez applied, table considerada **clustered**. Background process periodically reorder data aiming to co-locate data en same micro-partitions.

**Effect:** Determina how quieres data ordered en micro-partitions para facilitate improved query performance. Example: cluster en PRODUCT_ID → micro-partitions recreated con same value co-located.

### Cuándo Usar Clustering Keys

✅ **Good candidates:**
- Tables en **terabyte range**
- Tables suffering de increasingly poor performance
- Queries frequently **filter/sort** en cluster keys
- Columns usados en: joins, WHERE conditions, GROUP BY, ORDER BY (más benefit de WHERE conditions y join operations)
- Tables **frequently queried** y **change infrequently**

❌ **Not suitable:**
- Tables queried infrequently
- Not using many filter/sort operations
- Small tables (incluso si poorly clustered, performance still good)

**Cost consideration:** Hay cost associated con clustering. Debe compararse benefit gained vs cost. Larger tables benefit más. More frequently table changes = higher cost para maintain clustering (cost might outweigh performance benefit).

### Choosing Clustering Keys

**Best practices:**
- **Maximum 3-4 columns/expressions** per key (más de 3-4 increases costs more than benefits)
- Select columns usando **common queries que perform filtering/sorting** operations
- Consider **cardinality** de clustering keys

**Cardinality guidelines:**

❌ **Too LOW cardinality** (ej: country, gender): Won't allow effective pruning (values exist en many micro-partitions). Example: finding F requiere go through all micro-partitions.

❌ **Too HIGH cardinality** (ej: unique ID, timestamp to millisecond): Negatively impacts pruning (values too unique para group en micro-partitions).

✅ **Multiple columns:** Lowest cardinality selected first, followed by higher cardinality.

### Comandos

**Specify at table creation:** `CREATE TABLE ... CLUSTER BY (column1, column2)`

**Use expressions:** `CLUSTER BY (TO_DATE(date_column), SUBSTRING(other_column, 1, 5))`

**Add to existing table:** `ALTER TABLE ... CLUSTER BY (...)`

### Re-Clustering

**Problema:** Conforme DML operations (INSERT, UPDATE, DELETE, MERGE, COPY) performed en clustered table, data puede become less clustered. Example: insert 3 values en clustered table existentes en all pre-existing micro-partitions → effectively doubles número de micro-partitions to search through.

**Re-clustering:** Background process reorganizing data en micro-partitions para restore clustering along columns/expressions specified en key. Similar a reinsert sorted data en table para enforce natural clustering.

### Costs de Automatic Clustering

**Serverless feature** con costs:

**Compute:**
- Snowflake-managed compute: 2 credits/hora (at recording time)
- Cloud services compute: 1 credit/hora

**Storage:** Re-clustering results en storage costs. Rows physically grouped based en clustering key → Snowflake generates new micro-partitions. Adding even small number de rows puede cause all micro-partitions containing those values ser recreated.

**Más expensive en:**
- Large tables con large amount de DML operations
- Tables que change frequently

**More cost-effective para:**
- Tables **queried frequently** y **don't change frequently**

**Formula:** More frequently table queried = more benefit clustering provides. More frequently table changes = more expensive keep it clustered.

## Clustering - Interpretación y Uso Práctico

### Verificar si Table está Clustered

**Comando:** `SHOW TABLES`

**Output:** Column `CLUSTER_BY` populated para clustered tables. Si table no clustered, field estará empty.

### SYSTEM$CLUSTERING_INFORMATION() Function

**Input parameter:** Table name (si table tiene clustering key). Si no hay clustering key specified, incluir solo table name resulta en error.

**JSON Output components:**

**1. Clustering keys:** List de columns/expressions usados para sort data en micro-partitions.

**2. Total micro-partition count:** Total número de micro-partitions en table (ej: ~600,000).

**3. Total constant partition count:** Cuántas partitions tienen exclusively la combination de columns defined en key, **no shared** con other micro-partition. 
- **Higher = better**
- No realista expect todas las partitions en constant state

**4. Average overlaps:** Número de micro-partitions containing values de subset de table columns/expressions (defined en key) que overlap con each other. Example: cada date appears on average en 2 micro-partitions.
- **Lower = better**

**5. Average depth:** Cuántas micro-partitions debes look into para find values de subset de table columns/expressions defined en key. Example: look on average en 3 micro-partitions para find particular combination de two columns en key.
- **Lower = better**

**6. Partition depth histogram:** Distribution de values among table's micro-partitions. 
- **Further to bottom = worse clustering** (value spread among 9 micro-partitions vs 3 = 3x as many to search through)

### Interpretación: Well-Clustered vs Poorly-Clustered

**WELL-CLUSTERED example:**
- Low average overlap y average depth
- Good amount de constant partitions
- Histogram shows values concentrated at top

**POORLY-CLUSTERED example:**
- Total constant partition count = **0**
- Average overlaps = average depth = **total micro-partition count** (worst case)
- Histogram confirms: must search **every single micro-partition** para get single value de column

### Natural Clustering Check

**Function con second parameter:** `SYSTEM$CLUSTERING_INFORMATION('table_name', ('column_name'))`

Acepta one o more columns para check natural clustering. Good way para:
- Identify si column es good para filter on
- Determinar si quieres specify key en column para improve clustering

### Query Performance Impact

**Example:** 9.2 TB table con ~145 billion rows, result en <2 segundos.

**Query Profile - Pruning section:** Solo needed data from **4 micro-partitions** para compute/return result (vs full table scan). Becomes increasingly important conforme table gets larger (más table data = longer full table scan).

**Importante:** Si performance problems vienen de operator downstream (ej: join), implementing clustering probablemente **won't help with that**.

### Crear Clustering Key

**At table creation:** `CREATE TABLE ... CLUSTER BY (column1, column2)`

**Existing table:** `ALTER TABLE ... CLUSTER BY (...)`

### Monitoring Clustering Costs

**Account Usage View:** `AUTOMATIC_CLUSTERING_HISTORY`

**Información provided:**
- Amount de credits consumed
- Cuánto data fue re-clustered (expressed como rows o bytes)

**Nota:** Si no has set up clustering keys, won't return results.

**Remember:** Automatic clustering es serverless feature que consumes compute y storage resources durante:
1. Initial clustering
2. Background process que re-clusters table periodically.

## Search Optimization Service

Feature aimed at improving performance de **selective point lookup queries** (typically return single row o small group de rows) en very large tables (terabytes size). Operation puede ser costly y time-consuming sin optimization.

### Tipos de Queries Soportadas

Speeds up **equality searches** en dos formas:

**1. Equal sign:** Find specific row para value
```sql
WHERE column = 'value'
```

**2. IN operator:** Test si column value es member de explicit list de values
```sql
WHERE column IN ('value1', 'value2', 'value3')
```

### Data Types Soportados

- Fixed-point numbers (INTEGER, NUMERIC)
- DATE, TIME, TIMESTAMP
- VARCHAR
- BINARY

**Edition:** Enterprise+ feature.

### Funcionamiento

**Search Access Path:** Background process creates/maintains metadata structure recording metadata about entire table para understand donde reside all data en underlying micro-partitions. Selective point lookup queries usan metadata para find relevant micro-partitions **faster que usual pruning mechanism**.

### Comandos

**Add search optimization:** `ALTER TABLE ... ADD SEARCH OPTIMIZATION`

**Remove search optimization:** `ALTER TABLE ... DROP SEARCH OPTIMIZATION`

**Check status/progress:** `SHOW TABLES` muestra percentage de table optimized so far.

**Privileges requeridos:**
- OWNERSHIP privilege en table
- ADD SEARCH OPTIMIZATION privilege en schema containing table

### Costs

**Serverless feature** con unique credit cost:

**Storage:** Access path data structure requiere space (depends on número de distinct values en table). Normally around **25% de original table size**. Big table con lot de columns con distinct values puede end up costing quite a bit.

**Compute:** Creating/maintaining consumes compute resources (increases con más DML en table):
- Snowflake-managed compute: 10 credits/hora
- Cloud services compute: 5 credits/hora

**Exam note:** Unlikely aparecer en exam pero good idea tener high-level understanding just in case.
