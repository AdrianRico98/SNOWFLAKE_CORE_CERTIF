# Data Loading y Unloading

## Simple Data Loading Methods

### INSERT Statement

Método más simple para get data en table. Permite append records a table.

**Variaciones:**
- **INSERT con SELECT:** Crear values en command mismo (no selecting from table)
- **Subset de columns:** Deja columns omitidas como NULL values (error si omitted column tiene NOT NULL constraint)
- **VALUES keyword:** Specify values de cada row enclosed en brackets. Múltiples rows separated por commas
- **INSERT con SELECT FROM table:** Populate table desde otra table
- **INSERT OVERWRITE:** Truncate table y insert (clearing table down y repopulating). Works con VALUES o SELECT statement

### Upload vía UI

**Proceso:**
1. Databases tab → select database/table → "Load Table" button
2. Select warehouse (hace processing task para load data en storage layer)
3. Select CSV file desde local file system
4. **File Format:** Object conteniendo información sobre qué data expected (CSV, JSON, etc.). Ayuda Snowflake parse data files. UI tiene handy defaults
5. Edit schema: Option para change mapping de columns en CSV file a table
6. File format options: Más granular control sobre cómo loading process parsea CSV

**Límite:** Currently 250 MB por file uploaded vía Snowsight.

## Stages

Area para temporarily hold raw data files before being loaded en table o after being unloaded from table. Crucial step en data movement process.

### Tipos de Stages

**Internal Stages:** Provisioned/managed por Snowflake. Data files stored internally within Snowflake usando underlying blob storage de cloud platform (pero almacenan raw files uploaded, NO optimized micro-partitions de tables).

**External Stages:** Cloud storage areas managed por users outside Snowflake (ej: S3 bucket, Google Cloud Storage, Azure containers).

### Internal Stages (3 tipos)

**1. User Stage:**
- Automáticamente allocated a cada user
- Solo accessible al user
- No require CREATE statement
- Referencia: `@~` (@ character + tilde)
- Cannot be altered o dropped
- Upload files: PUT command
- No appropriate si múltiples users need access

**2. Table Stage:**
- Automáticamente allocated a cada table
- Multiple users pueden stage files, pero solo loaded en esa table
- Referencia: `@%` (@ character + percentage)
- Cannot be altered o dropped
- Access requiere ownership privileges en table
- Upload files: PUT command

**3. Named Internal Stage:**
- Database object created/named por user (CREATE STAGE command)
- Más flexible en options, fit greater number de use cases
- No restricted a one user/table
- Referencia: `@stage_name` (@ character + stage name)
- **Securable objects:** Privileges pueden grantarse a roles para manage access
- Upload files: PUT command

**Características All Internal Stages:**
- Uncompressed files automáticamente compressed con GZIP when loaded (unless explicitly set not to)
- Stage files automáticamente encrypted con 128-bit keys

### External Stages

Reference data files stored en location outside Snowflake (cloud storage service managed por users).

**Características:**
- User-created con CREATE STAGE DDL
- Files uploaded usando cloud utilities de cloud provider (ej: AWS UI, AWS CLI para S3)
- Referencia: `@stage_name` (como named internal stage)
- Storage location: private o public (affects cómo configure secure access en Snowflake side)
- **Copy options:** Pueden setarse en stages, dictating behavior around loading data (ej: behavior cuando errors encountered, delete files after successful load)

**CREATE Statement components:**
- Stage identifier
- Cloud storage service URL (ej: S3 bucket URL)
- **Access credentials** (si bucket protected): Para que Snowflake tenga permissions read contents. Puede hard-code o attach **Storage Integration object**

**Storage Integration:** Object encapsulating required información para authenticate/gain access a private external storage service (read, write, delete). Typically usa generated identity entity (ej: AWS role). Puede aplicarse across stages. **Recomendado** para avoid explicitly setting sensitive info para cada stage definition.

### Comandos para Interactuar con Stages

**1. LIST:** Lista contents de stage. Output: path de stage file, size, hash, timestamp last updated. Puede append path para return files dentro de specific directory. All stages except user stages pueden include database/schema global pointer.

**2. SELECT FROM stage:** Query contents de stage files directly usando standard SQL (internal y external stages). Útil para inspect stage's files prior to loading/unloading. Snowflake expone metadata columns (filename, row number).

**3. REMOVE:** Remueve files desde external o internal stage. Puede specify path para specific folders/files. Puede include database/schema global pointer.

### PUT Command

Upload data files desde local directory en client machine a any de 3 tipos de internal stages.

**Syntax examples:**
- Named internal stage: `PUT file:///path/to/file.csv @stage_name`
- User stage: `PUT file:///path/to/file.csv @~`
- Table stage: `PUT file:///path/to/file.csv @%table_name`

**Características:**
- **No puede ejecutarse desde worksheets en UI**
- Duplicate files uploaded ignored
- Uploaded files automáticamente encrypted con AES-128 o 256-bit keys (account parameter `CLIENT_ENCRYPTION_KEY_SIZE` determina size, default=128)

## Bulk Loading (COPY INTO <table>)

**Método más importante** para understand para exam concerning data loading. Cuando manually executed por user = bulk loading.

**Definición:** Copy contents de stage (internal o external) en table. "Into table" = process de copy stage data a storage layer, storing en columnar format, querying vía logical structure de table.

**Características:**
- Requiere **user-created virtual warehouse** para execute
- Load history stored en metadata de target table por **64 days** (usado para deduplicate files - mismo name/contents no loaded again)
- Natively supports loading variety de data formats: delimited (CSV), semi-structured (JSON, Avro, ORC, Parquet, XML)

### Basic Syntax

**Más básico:** `COPY INTO my_table FROM @my_int_stage`

Intenta copy all contents de stage en table.

**Con path:** Puede provide path de folder o specific file. Folder = attempt load everything en folder.

### Options para Controlar Behavior

**FILE option:** List de one o more file names separated por commas para upload from stage.

**PATTERN option:** Regular expression pattern extracting files to load from stage.

### Transformations Durante Loading

Perform simple transformations en data as it's loaded. Make data en raw input files conform a structure de target table.

**Transformations disponibles:**
- **Column reordering**
- **Column omission:** Not bring through every column available
- **Casting values:** Cast to other data types (ej: DOUBLE, TIMESTAMP)
- **Truncate text strings:** Exceeding target column length usando `ENFORCE_LENGTH` o `TRUNCATE_COLUMN` options

**Syntax con subquery:**
```sql
COPY INTO table 
FROM (
  SELECT $1, $2::DOUBLE, $3::TIMESTAMP 
  FROM @stage_name
)
```

`$1, $2, $3` = dollar sign + column numbers (relevant para CSV files). Number de columns en SELECT must match number de columns en target table.

### External Stages Loading

Files loaded from external stages much same way como internal stages. External security/authentication settings encapsulated en integration object applied a stage.

**Nota:** Data transfer billing charges pueden apply cuando loading data desde files en cloud storage service en different region o cloud platform from Snowflake account.

**Direct S3 loading:** Método para load directly from S3 usando COPY INTO, bypassing stage. Requiere same info specified en stage. **Snowflake recomienda** instead crear reusable/securable separate stage object.

### Copy Options

Properties para alter behavior de COPY INTO command. Pueden setarse en stage o COPY INTO statement.

**ON_ERROR:** Error handling para load operation.
- `ABORT_STATEMENT` (default bulk loading): Abort load si any error found en data file
- `CONTINUE`: Keep loading file si errors found
- `SKIP_FILE`: Skip file si error detected (when multiple files uploading)
- `SKIP_FILE_<num>`: Numerical upper limit de failures file puede tolerate before skipped
- `SKIP_FILE_<percentage>`: Percentage upper limit (ej: 10% de rows con errors → skip file)

**PURGE:** Clear down files en stage once successfully loaded en table.

**FORCE:** Circumvent deduplication effort. Ordinarily duplicate files (same name/contents) no reloaded. Set FORCE=TRUE para bypass.

### Output de COPY INTO

Breakdown para cada file: rows loaded, metadata concerning errors encountered. Ayuda validate output de load operation y troubleshoot issues.

### Validation Methods

**1. Validation Mode:** Optional parameter para COPY INTO que permite dry run de load process (no actually loading files). Test files para errors, return result based en value:
- `RETURN_N_ROWS`: Specify number de rows to validate. At first error, copy statement fails
- `RETURN_ERRORS`: Returns all errors (parsing, conversion, etc.) across all files
- `RETURN_ALL_ERRORS`: Returns all errors including files con errors partially loaded durante earlier load (cuando ON_ERROR=CONTINUE)

**Limitación:** Cannot be used con load transformations.

**2. VALIDATE Table Function:** Like validation mode pero intended para use against past execution de COPY INTO command. Takes job ID como parameter. Tampoco supports COPY INTO commands con transformations.

## File Formats

Mechanism para tell Snowflake qué types de files stored en stage y qué properties tienen (ej: identify column delimiters, newline characters).

### Dos Métodos

**1. File_format Options:** Set directly en internal stage o en COPY INTO statement.
```sql
CREATE STAGE ... FILE_FORMAT = (TYPE=CSV SKIP_HEADER=1)
```

**2. File Format Object (Recomendado):** Bundle toda la información required para parse files en separate object. Puede usarse across múltiples COPY INTO statements.
```sql
CREATE FILE FORMAT my_format TYPE=CSV SKIP_HEADER=1;
CREATE STAGE ... FILE_FORMAT = my_format;
```

**Precedencia:** Si file_format set en ambos COPY INTO statement Y stage, **COPY INTO statement takes precedence**.

### Properties

**TYPE:** Format de file expected to load. Natively supported: CSV (any delimited flat file), JSON, AVRO, ORC, PARQUET, XML. Cada type tiene own set de properties.

**Key Properties examples:**
- **SKIP_HEADER:** Number de rows to skip at start de file
- **COMPRESSION:** Tells Snowflake qué compression form usado (varies by type - CSV/JSON: identical options, Parquet: limited a 4 options). Snowflake recomienda compress files stored en stage

**Default:** COPY statements no require file format specified. Por default, COPY statement intenta interpret CSV con no headers usando UTF-8 encoding.

## Snowpipe (Continuous Data Ingestion)

Serverless feature que **automatically load data files en near real-time** en table as soon as files available en stage. Ejemplo: upload file a S3 external stage → Snowflake listening para notification de event → spins up compute para copy new data en table. Solo action que performs user: upload initial data file.

**Contraste con bulk loading:** Manual process de execute COPY INTO table statement.

**Razón de existir:** Response al shift en how data is produced - smaller batches generadas at quicker pace conforme streaming systems (Kafka, Beam, Kinesis) se vuelven más popular. Manually loading data files arriving every minute no sería practical.

### Pipe Object

Functionality enabled con object llamado **pipe**. DDL define COPY statement (similar a manual run) specifying stage y target table. Esto es lo que ejecuta on your behalf. All file formats supported, transformations pueden incorporarse.

### Dos Modos (AUTO_INGEST)

**1. AUTO_INGEST = TRUE (Cloud Messaging):**
- Más comúnmente usado
- Solo works con **external stages**
- Configure cloud blob storage (ej: S3 bucket) para send Snowflake notification telling new file uploaded
- Acts como trigger para Snowflake execute COPY INTO statement defined en pipe

**Process flow (AWS example):**
1. Upload file a S3 bucket usando cloud provider utilities
2. S3 bucket configured (vía event notifications) para send per-object notification a SQS Queue cuando file uploaded
3. **SQS Queue managed por Snowflake** (created cuando created pipe object). Notification holds info en qué files to load
4. Vía SQS Queue, pipe knows spin up compute y perform COPY INTO table statement configured en pipe body
5. **Within minute** de upload file, data should appear en target table (decryption/file size pueden affect time)

**2. AUTO_INGEST = FALSE (REST API):**
- User lets Snowflake know cuando new file uploaded vía call a Snowflake REST endpoint
- Works con **internal y external stages**
- Client applications (Java/Python SDKs provided) call Snowflake vía public **insertFiles REST endpoint** providing list de file names uploaded a stage + reference a pipe
- Snowflake-provided compute resources execute COPY INTO table command en pipe definition para populate target table

### Facts para Exam

✅ **Intended para:** Load many **small files quickly**. Typically takes ~1 minuto load file once receive notification.

✅ **Serverless feature:** Perform COPY INTO operation usando **Snowflake-managed compute resources** (NO user-managed virtual warehouse). Scales compute to meet demands.

✅ **Load history:** Stored en pipe metadata por **14 días** (prevent reloading same files, duplicating data en table).

✅ **Pause capability:** Cuando pipe paused, event messages received enter **limited retention period de 14 días** (default), allowing process cuando pipe resumed. Si pipe paused >14 días = **stale**.

### Snowpipe vs Bulk Loading

| Aspecto | Bulk Loading | Snowpipe |
|---------|--------------|----------|
| **Authentication** | Security options supported por client (ej: username/password en Python) | REST endpoints requieren **key pair authentication con JSON web tokens** |
| **Load history** | Stored en target table metadata **64 días** | Stored en pipe metadata **14 días** |
| **Compute** | Requiere **user-created warehouse** | Usa **Snowflake-supplied compute resources** |
| **Billing** | Billed como queries (time VW active) | **Serverless billing:** Compute resources + overhead (**0.06 credits per 1,000 files** notified/listed vía event notifications o REST API calls) |

### Best Practices (Bulk + Snowpipe)

✅ **Split large files:** Múltiples files around **100-250 MB compressed data**. Single server en warehouse can process up to **8 files in parallel** → providing one large file no utiliza whole warehouse capacity. Files exceeding **100 GB not recommended**, should be split.

✅ **Bundle small files:** Si many very small files (ej: 10 KB messages), bundle up en larger files para avoid too much processing overhead.

✅ **Organize stage data by path:** Copy any fraction de data en Snowflake con single command. Permite execute concurrent COPY statements matching subset de files (taking advantage de parallel operations).

✅ **Separate warehouses:** Tener separate virtual warehouses para data loading tasks vs other tasks. Enables **workload isolation** (COPY INTO statements not queued/causing other queries queue).

✅ **Pre-sort data:** Ordering de data as copied in es important. Organizing stage data creates natural order → translates en micro-partitions más evenly distributed → improved pruning potential.

✅ **Snowpipe frequency:** No upload files at shorter frequency than **1 minute**. Loading very small files more often que once per minute → Snowpipe can't keep up con processing overhead → files back up en queue, incur costs.

## Data Unloading

Proceso similar a how we got data in. Usar **COPY INTO location** command y **GET** command para get data out de Snowflake.

### COPY INTO Location

Unload contents de table a any internal stage, external stage, o external location. También puede copy output de query.

**File formats soportados (más limitado que COPY INTO table):**
- Delimited files: CSV, TSV
- Semi-structured: JSON, Parquet

**Default behavior:**
- Results split en **multiple files** (exact number impacted por size de virtual warehouse executing command)
- Generated files en **CSV format**
- **Compressed usando Gzip**
- Always encoded usando **UTF-8**

**Encryption:**
- Data files unloaded a Snowflake internal location: automáticamente encrypted usando **128-bit keys**. Files unencrypted cuando downloaded a local directory
- Data files unloaded a cloud storage: pueden encrypted si security key provided a Snowflake

### Syntax Examples

**Copy from SELECT query:**
```sql
COPY INTO @stage_name
FROM (SELECT ... FROM table)
FILE_FORMAT = (TYPE=JSON)
```

**File prefix:** Files generated pueden prefixed con string incluyendo at end de stage definition (ej: `@stage_name/data_` → all files start con `data_`)

**File format object:** Control over how files look (change default format a JSON, etc.)

**Benefit de unload from query:** Puede include join clauses para download data from multiple tables.

**Direct to external cloud storage:** Puede copy table contents directly a external cloud provider blob storage (ej: AWS S3 bucket) **without use de stage object**.

### Copy Options

**OVERWRITE (Boolean):** Indica si files generated por COPY INTO command overwrite files ya existentes en stage cuando tienen same name.

**SINGLE (Boolean):** Por default, COPY INTO location splits result en multiple files. Set SINGLE=TRUE para produce **single file**.

**MAX_FILE_SIZE:** Si SINGLE=FALSE (multiple files), controls size de cada file. Default: ~16 MB compressed.

**INCLUDE_QUERY_ID (Boolean):** Uniquely identify loaded files incluyendo query ID de COPY statement en file names de unloaded data files. Helps ensure concurrent COPY statements don't overwrite unloaded files accidentally. **Cannot be set** si SINGLE, OVERWRITE o PARTITION BY in use.

**PARTITION BY:** Partitions unloaded data en directory structure en destination stage. Accepts expression por el cual unload operation partitions table rows en separate files. Example: partition data en directories que map date.

### GET Command

Reverse de PUT. Specify source stage y target local directory para download file. Typically executed **after** using COPY INTO location command.

**Características:**
- **Cannot be used para external stages** (access vía own utilities como AWS console/CLI)
- **Only used con internal stages**
- **Cannot be executed from worksheet en UI**
- Downloaded files automáticamente **decrypted** when using GET

**Parameters:**
- **PARALLEL:** Number de threads to use para downloading files. Increasing threads improve performance con large files. **Default: 10 threads**
- **PATTERN:** Regular expression pattern para select files to download from stage. Si no pattern specified, GET command tries get **all files** en stage

### Download vía UI

**Classic UI:** Limit de **100 MB**

**Snowsight:** **No hard limits**

# Semi-Structured Data

## Qué es Semi-Structured Data

Data formats categorizados como semi-structured si comparten typical features evidentes en JSON y XML:

**Características:**
- **Flexible schema:** Number de fields puede vary from one entity to next (ej: un widget tiene image, otro no). Contrast con CSV (fixed number de columns)
- **Self-describing fields:** Contain way para self-describe cada field included en entity (possible porque schemas pueden vary)
- **Nested data structures:** Ej: image inside widget object
- **Collections:** Arrays
- **Keys y values:** Generally formed de key-value pairs

**Historical challenge:** Difficult accommodate en data analysis systems, particularmente data warehouses. Sistemas antiguos designed para structured data en tables (optimizations para ese kind de data). Coercing field values de flexible file format como JSON en rigid structure como table puede resultar en missing columns (ej: JSON document con new field pero current table doesn't capture column).

## Snowflake's Approach

Snowflake extended standard SQL para include **purpose-built semi-structured data types**. Permiten store semi-structured data inside relational table. Semi-structured data types pueden hold complex/variable structures (arrays, objects) en table column. Existen alongside standard data types (string, date, numeric) en one table → possible combine structured y semi-structured data en single query.

Snowflake también extended SQL para include **semi-structured functions y notation** para access data stored en esos types.

### Semi-Structured Data Types

**1. ARRAY:**
- Own data type, collection holding zero o más elements de data
- Cada element puede ser different data type
- Accessed por position en array
- Can specify como type de column en table
- Crear con `ARRAY_CONSTRUCT()` function
- Access elements: `array_column[position]`

**2. OBJECT:**
- Analogous a JSON object: collection de key-value pairs
- Crear con `OBJECT_CONSTRUCT()` function
- Store complex data structure (keys + values) en one column
- Interact usando notations añadidas por Snowflake a SQL

**3. VARIANT:**
- **Universal semi-structured data type**
- Puede hold **any other data type:** objects, arrays, numbers, anything
- Objects y arrays son really just restrictions de variant type
- **Primary use:** Storing whole semi-structured data files (ej: entire JSON/Parquet document en variant column, o split elements en separate rows during loading)
- Puede hold hasta **16 MB compressed data per row** (si large file exceeds threshold → error thrown. Soluciones: split JSON file prior to loading o usar file format option para separate JSON objects en file para load como separate rows)

### Natively Supported Semi-Structured File Formats

**Snowflake natively supports:**
- **JSON:** Popular plain text format based en subset de JavaScript programming language
- **Avro:** Open source framework originally developed para Apache Hadoop. Stores data definition en JSON format (easy read/interpret). Data stored en binary format (compact/efficient)
- **ORC:** Binary format usado para store Hive data
- **Parquet:** Binary format designed para projects en Hadoop ecosystem
- **XML:** Consisting primarily de tags y elements

**Unloading:** Solo **JSON y Parquet** pueden unloaded usando COPY INTO location command.

**Native support meaning:** Snowflake figured out way para load estos formats en storage, intelligently extracting columns from complex structures y transforming en Snowflake's columnar file format.

## Loading Semi-Structured Data

Data loading process **broadly same** para structured y semi-structured data: files uploaded a stage → COPY INTO command executed (by user o vía Snowpipe) para move stage files en table storage.

### Process Details

Cuando loading any semi-structured data formats, files undergo very similar process como structured data (CSV):
- **Repeating attributes** (ej: keys en JSON) automáticamente identified y extracted en columnar format
- **Rules que prevent extraction:** Value para repeating key tiene different data types between elements, elements tienen string value de null

**Despite limitations:** Query performance comparable entre semi-structured data types (variant) y standard data types.

**Additional processes:** Extracting columns en columnar format + familiar processes: splitting input data en micro-partitions, compression, encryption, gathering statistics.

### File Format Objects

Like structured data, **file format type must match files en stage** para que Snowflake sepa qué parse. Cada supported semi-structured format uniquely structured (some binary, some plain text). Por esto, cada file format object para cada type (JSON, ORC, etc.) tiene own set de options (como CSV file format).

Puede apply file format a: stage, COPY INTO table command, o table itself. Typically found en stage o COPY INTO table command.

**JSON File Format Properties examples:**
- **COMPRESSION:** Usado cuando loading data, configure qué compression algorithm used para extract during loading
- **STRIP_OUTER_ARRAY:** Important option, convenient way para bypass 16 MB limit en variant column durante loading. Top-level elements en JSON file generally listed en array. Property removes outer array y considers cada top-level element en file como own row en table

## Three Main Loading Approaches

### 1. ELT (Extract, Load, Transform)

Load entire semi-structured data file directly en **single variant column** vía COPY INTO table command. Shows power de variant data type: allows **schemaless storage** de hierarchical data en table.

**Idea:** Load file → later query usando semi-structured SQL extensions O pick out columns from variant column y load en new table.

**When to use:** Structure o operations on data **not known upfront**.

**Según Snowflake:** Typically tables para store semi-structured data consist de **single variant column** (common loading pattern expected).

**Under the hood:** Optimizations around column extraction y compression transparently happening para make querying complex hierarchical data quicker.

### 2. ETL (Extract, Transform, Load)

Pick out column values directly from semi-structured data files upfront. Similar a extraction de columns from CSV pero usando **semi-structured SQL extensions**. Notation slightly different entre semi-structured formats por how organized internally.

**Benefits:**
- Values como dates/timestamps stored como **strings** cuando loaded en variant column. ETL approach separating en columns permite store como **native data types** → improves pruning y decreases storage consumption

### 3. Automatic Schema Detection

**INFER_SCHEMA table function:** Detects column definitions en set de staged data files. Output puede passed a `USING TEMPLATE` en CREATE TABLE DDL para define columns.

**MATCH_BY_COLUMN_NAME:** Option para COPY INTO command. Match columns en semi-structured data file a corresponding columns en table during loading, **without manually specifying** cada element (como en ETL approach). **Tricky en practice:** Column names en data file need match exactly column naming convention en Snowflake. Puede match con case sensitivity o without.

## Unloading Semi-Structured Data

Works much same way como structured data:
1. Execute COPY INTO location command para copy data from table/query en stage as file
2. Followed por GET command para download data a local machine

Puede apply file format a: COPY INTO statement, stage, o table definition.

**Limitation:** Solo **JSON y Parquet** pueden unloaded a stage.

## Querying Semi-Structured Data

Assume ELT approach usado (loaded JSON document en single variant column). ¿Cómo query y extract nested elements?

**Variant type:** Exposes semi-structured data en **native form** (when query variant column, ves essentially same structure como prior to ingestion). Behind scenes: Snowflake storing en **columnar form** (more performant para query).

### Dos Métodos para Traverse Semi-Structured Data

**1. Dot Notation:**
- Colon `:` placed after variant column name para extract first-level element
- Subsequent elements accessed con dot `.`
- Syntax: `variant_column:first_level.child_object`
- Column name **case insensitive** (like all SQL columns), pero element names **case sensitive**

**2. Bracket Notation:**
- More akin a high-level programming languages
- Variant column name followed por square brackets enclosing element names
- Syntax: `variant_column['first_level']['child_object']`
- Same case rules apply

**Accessing array element:**
```sql
variant_column:element[integer_position]
```
Extrae single element from array (position-based).

**GET Function:** NOT confused con GET command (used para unload data). Function allows perform same task como dot/bracket notation, accessing repeating elements en form de function call.

### Casting

**Issue:** Si uncast, output de select statements será type variant y **enclosed en double quotes**.

**JSON data types mapping:** JSON puede represent: string literals, integers, arrays, objects, Booleans, null values. Snowflake maps JSON data types a SQL data types. Number, Boolean, null displayed accurately. **Pero string literal values** retornan como string literal (sequence de characters enclosed en double quotes), no string como expected. JSON no representa dates → también true para date columns.

**Important:** Considerarlo cuando comparing structured y semi-structured columns (ej: join between tables - joindría con double quotes included).

**Best practice:** **Explicitly cast results** from variant column.

**Three casting methods:**

**1. Double colon notation (shorthand):**
```sql
variant_column:element::DATE
```

**2. TO_<datatype> function:**
```sql
TO_DATE(variant_column:element)
```

**3. AS_<datatype> function:**
```sql
AS_DATE(variant_column:element)
```
Solo used para converting values **from variant column**.

**Opposite operation:** También puede cast from data types (integer, string) **to variant**.

## Semi-Structured Functions

### FLATTEN Table Function

Takes nested data structure y **explodes/flattens** it → produces row para cada item en nested structure.

**Required parameter:** `INPUT` (thing to flatten - must be variant, object o array data type)

**Wrap en TABLE() function:** Para interact con results de flatten function como si fuera table de columns.

**Optional parameters:**
- **PATH:** Specify path inside object to flatten (no flatten whole thing, take specific value como array)
- **RECURSIVE (Boolean):** Determina si flatten performed en sub-elements además de top-level provided como input

**Output columns (6 total):**
1. **SEQ:** Unique sequence number associated con input record (not guaranteed gap-free o ordered)
2. **KEY:** Si exploding object, holds key de object. Si array → null
3. **PATH:** Path a element dentro data structure being flattened
4. **INDEX:** Si array, holds position de element
5. **VALUE:** **Generally what we want to select.** Contains value de element de flattened array/object
6. **THIS:** Contains element being flattened (useful en recursive flattening)

### LATERAL FLATTEN

Combination de **lateral join** + **flatten function**. Puede ser complicated.

**Purpose:** Normalize table flattening values (ej: skills array). Flatten array duplicating other column values from table para create multiple rows.

**How it works:**
1. **LATERAL join** hands cada row de table (could be view/subquery) a flatten table function
2. From row passed, select thing to flatten (ej: skills array)
3. Associates output de flatten function con row from base table
4. Can select from both base table Y output de flatten function → present nested data alongside other columns

**Use case:** Violates first normal form (contains composite/multi-value attribute) → lateral flatten normalizes.
