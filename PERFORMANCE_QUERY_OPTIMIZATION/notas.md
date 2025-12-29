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
