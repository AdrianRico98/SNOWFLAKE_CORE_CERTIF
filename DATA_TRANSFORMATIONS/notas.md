# Data Transformations

## Snowflake Functions

Snowflake quiere que entiendas broad classes de functions disponibles. Grupos comparables: scalar, aggregate, window, table, system (+ user-defined y external ya cubiertos).

### Scalar Functions

Retornan **one value per call**. Divididas en 15 grupos.

**Example - Data Generation category:** `UUID_STRING()` function produce single universally unique identifier. Uso: tag unique transaction en table, cualquier cosa needing unique value con very low probability de repeating.

### Aggregate Functions

A diferencia de scalar, toman **multiple rows y produce single output**. Más comúnmente para mathematical calculations (sum, average, counting). También: statistical calculations (linear regression), non-mathematical calculations (array aggregation).

**Example:** `MAX()` function. Input: multiple rows. Output: single value (maximum value from column).

### Window Functions

**Subset de aggregate functions**. Permite perform same type de operations pero en **small groups de rows** dentro total number de rows provided como input. Small group = **window**.

**Syntax:** `OVER` y `PARTITION BY` para select qué window de rows perform calculation.

**Example:** Calculate max amount per account_ID usando `MAX(amount) OVER (PARTITION BY account_id)`. Calcula max amount per window (per account_ID) y apply a every row de ese window.

### Table Functions

**Opposite de lo visto:** Para each input row, table function puede output **multiple rows** (técnicamente zero, one o multiple). Siete categorías.

**Example - Data Generation:** Combinar scalar + tabular functions para produce table de synthetic test data. `GENERATOR()` function creates rows based en specified number de rows O number de seconds to run (generating as many rows as possible). Wrap en `TABLE()` literal para query results.

### System Functions

Tres broad classes:

**1. Execute actions en system:** Ej: cancel running query (accepts query ID como input)

**2. Provide information about system:** Ej: `SYSTEM$PIPE_STATUS()` retorna JSON object detailing execution status de pipe object (running/paused, good para Snowpipe troubleshooting)

**3. Return information about queries:** Ej: Generate explain plan para given SQL statement (output: JSON)

## Estimation Functions

Cuatro grupos de aggregate functions para perform estimation. **Estimation:** Perform calculation **quicker pero less accuracy**.

### Cardinality Estimation

Estimate number de distinct values en set de values.

**¿Por qué usar estimation vs DISTINCT?** Executing `COUNT DISTINCT` requiere memory proportional a cardinality (costly para very large tables).

**HyperLogLog Algorithm:** Snowflake implementó este algorithm que retorna approximation de distinct number de values. **Main benefit:** Consumes significantly less memory, suitable cuando input quite large y approximate result acceptable.

**Average relative error:** ~1.6%. Si `COUNT DISTINCT` retorna 1 million, HyperLogLog typically retorna result en range ±1.6% de 1 million.

**Functions disponibles:** 6 functions, main one = `HLL()` (alias: `APPROX_COUNT_DISTINCT()`).

**Example:** `APPROX_COUNT_DISTINCT(order_key)` en ~160 GB table completó en 44 segundos vs `COUNT(DISTINCT order_key)` tomó significantly longer pero dio exact result (1.5 billion rows). Si margin de error acceptable, great para rough idea de distinct values.

### Similarity Estimation

Estimate similarity de two o more sets de values.

**Jaccard Similarity Coefficient:** Method para find similarity/difference entre two sets. Computa ratio de intersection y union de two sets. Computationally expensive operation.

**Snowflake's two-step process:**

**Step 1 - MINHASH function:** Run en two sets de input rows. Output = **MINHASH state** (array de hash values derived from set de input rows, forms basis para comparison). Parameters:
- **K:** Cuántos hash values calculate (higher = more accurate estimation, pero increases computation time. Max: 1024)
- **Expression:** Define qué input rows pass a function (ej: column de values)

**Step 2 - APPROXIMATE_SIMILARITY function:** Pass two MINHASH states. Retorna floating point number (0-1):
- 1 = sets identical
- 0 = sets no overlap

### Frequency Estimation

**APPROX_TOP_K function:** Implementa **space saving algorithm** para produce approximation de values y frequencies.

**Input parameters:**
1. Column para calculate frequency de values
2. Number de values to approximate
3. Max number de distinct values tracked at time durante estimation (increasing makes estimation more accurate, theoretically uses more memory)

**Output:** Values con most frequencies.

**Comparison con exact result:** Puede achieve exact result con GROUP BY query, pero approximation often accurate.

### Percentile Estimation

Estimate percentile de values usando **T-digest algorithm**.

**APPROX_PERCENTILE function:** Main function (otros son helper commands o intermediate steps).

**Percentile definition:** Statistical method expresando qué percentage de set de values está below certain value.

**Example:** Test scores de 10 students. `APPROX_PERCENTILE(score, 0.8)` retorna score needed para achieve as good/better que 80% de other students (80th percentile). Input: score column + percentile value (0-1, donde 0.1=10th percentile, 0.2=20th percentile).

## Table Sampling

Process de reading **random subset de rows** from table. `LIMIT` restricts rows pero sampling es helpful way para get more representative group de rows.

### Dos Métodos de Sampling

#### 1. Fraction-Based

**Syntax:** `SAMPLE` o `TABLESAMPLE` keyword + sampling method + probability (percentage) en brackets.

**Sampling methods:**

**ROW / BERNOULLI (default si no specified):** Probability applied a **individual rows**. Number (0-100) representa probability expressed como percentage que specific row aparecerá en result. Example: 50 = 50-50 chance individual row presente. Larger number = generally more rows. **Resulting sample size:** approximately `(probability/100) × total rows`.

**BLOCK / SYSTEM:** Probability applied a **larger blocks de rows** (not individual rows). May produce less random results, particularly para smaller tables.

**SEED / REPEATABLE:** Include random integer (positive, specific range) para generate same results each time sampling query repeated (making it deterministic). Ordinarily, rerun produce different set de rows. Using seed puede produce different results entre executions si table data changed o sampling copy de table.

#### 2. Fixed-Size

Más straightforward. Determine **exactly** cuántos rows produce incluyendo integer (0 - 1 million) + keyword `ROWS`.

**Example:** `SAMPLE (3 ROWS)` samples 3 rows from table.

**Limitaciones:** BLOCK sampling method y SEED **not supported** con fixed-size (result en error si included).

## Unstructured Data Functions

Snowflake tiene 6 file functions en scalar function category. Data no arranged según predefined data model/schema: multimedia (images, audio, video), documents (PDFs, spreadsheets, Word docs). No fit neatly en table data structure.

### High-Level Process

1. Upload unstructured data file (ej: image) a internal/external named stage
2. Leverage one de 3 functions para generate URL from file (cada function con different requirements around authorization/validity)
3. Use generated URL para access staged unstructured file

### BUILD_SCOPED_FILE_URL

**Input parameters:** External/internal named stage identifier + relative path para file en stage.

**Output:** Encoded URL granting access a file por **24 horas** al user que requested.

**Example output:** Snowflake-hosted URL donde stage/file names encoded. User que called function puede click URL en Snowsight para download file. También retrieve image via API request.

**Privileges:** When used directly en query, currently active role must have permissions en stage. **Exception:** Puede give other users/roles ability generate file URL **without requiring privileges en stage** incluyendo function call en UDF, stored procedure o view definition. Roles solo need privileges en view, not underlying stage.

**Use case:** Designate view como secure y share con data consumers vía secure data sharing.

### BUILD_STAGE_FILE_URL

Similar a anterior pero para **permanent access** (no validity period).

**Input parameters:** Same two (stage identifier + relative path).

**Output structure:** Different - no longer encoded. Includes database, schema, stage identifier, relative path de file location. URL clickable en Snowsight (not classic console) para download file.

**Privileges:** Requiere privileges en underlying stage (USAGE para external stages, READ para internal stages). Applies si using directly en query O si parte de view/UDF/stored procedure.

**Issue con internal stages:** Si downloaded files tienen issues (corruption), puede ser porque client-side encryption set en stage. **Fix:** Set server-side encryption durante stage creation o después con ALTER command.

### GET_PRESIGNED_URL

Similar a lo que obtienes directly from cloud provider blob storage. URL dando access a file **without needing authenticate con Snowflake**.

**Additional input parameter:** Expiration time (expresado en seconds) after cual access a file/files revoked.

**Example:** Set expiration time a 600 seconds (10 min). Con URL, puedes paste directly en browser address field para download file O usar en BI tool para display unstructured data.

**Privileges:** Role calling function must have USAGE privilege en external stage, READ privilege en internal stage.

### Practical Example

**Objetivo:** Present en one view link a PDF document alongside relevant metadata (author, publication date).

**Steps:**
1. Create internal named stage `documents_stage` storing: unstructured PDF file + JSON file describing document contents (including relative path de file en stage)
2. Create table para store document metadata
3. COPY INTO statement para parse metadata JSON file from stage y load en table
4. Create view combining metadata + access a unstructured data file (generating URL from stage name + relative path stored en metadata table)

**Result:** SELECT from view → access a PDF document sitting alongside relevant descriptive information.

## Directory Tables

**No son separate database object.** Pueden enable en external/internal named stages durante stage creation o after fact con `ALTER STAGE` command.

**Purpose:** Interface queryable para retrieve **metadata** about files residing en stage. Output somewhat similar a `LIST` function pero includes additional field: **FILE_URL** (Snowflake-hosted file URL identical a URL de `BUILD_STAGE_FILE_URL` function). También: file size, file hash, last modified time.

**En resumen:** Queryable dataset que automáticamente comes con permanent file URLs a stage files.

### Syntax
```sql
SELECT * FROM DIRECTORY(@stage_name)
```

### Refresh Requirement

Directory tables need refreshed = synchronizing metadata con latest files en stage. Si upload file a stage y query directory table **without first refreshing**, new file no aparecerá en output.

**Dos opciones de refresh:**

**1. Manual Refresh (both stage types):** Execute command específico.

**2. Automated Refresh (external stages only):** Set up event notifications en cloud provider donde stage hosted.

**AWS example steps:**
1. Enable directory table en external stage
2. Execute `DESCRIBE` command en stage para retrieve Amazon Resource Name (ARN) para Snowflake-managed SQS Queue (allows stage receive notifications about new files)
3. Configure event notifications para S3 bucket underpinning external stage. Notifications notify SQS Queue about new/updated files → refreshing directory table metadata without manually executing stage refresh statement

## File Support REST API

Otro method para access URLs generated en lectures previas. Snowflake exposed many different endpoints para interact con accounts (ej: Snowpipe REST API, SQL REST API).

**File Support REST API:** Permite download data files from internal/external stage **programmatically**. Ideal para building custom applications (ej: custom dashboard rendering product images).

### Endpoint

**Single GET method:** `GET /api/files`

**Takes:** Scoped URL o file URL (not pre-signed URLs).

**Authorization depends on URL type:**
- **Scoped URLs:** Solo user who generated scoped URL puede download stage file
- **File URLs:** Any role con privileges en underlying stage puede access file (USAGE para external, READ para internal)

### Python Client Example

Simple Python client making HTTP GET request a REST API.

**Steps:**
1. Import requests Python module (includes methods para send HTTP requests easily). Alternative: use curl command en terminal
2. First argument a GET method: URL (hardcoded o usar SQL REST API para call stored procedure que generates URL)
3. Headers con request: providing OAuth token para authenticate con Snowflake

**Response:** Stored en object from cual contents pueden accessed (image, PDF file, etc.)
