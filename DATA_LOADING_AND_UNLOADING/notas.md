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
