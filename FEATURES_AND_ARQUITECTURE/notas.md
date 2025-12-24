# ¿Qué es Snowflake?

Snowflake es una **plataforma de datos cloud-native ofrecida como SaaS**. A diferencia de un data warehouse tradicional, soporta múltiples workloads: data warehouse (SQL, ACID), data lake (semi-estructurados, JSON, Avro), data engineering (Snowpipe, Tasks, Streams), data science (Snowpark, integración ML), data sharing (marketplace) y application development (UDFs, stored procedures, conectores).

**Cloud-native significa que fue construido desde cero para la nube**, no es un port de tecnología legacy (NO es "cloud-wash" - importante para el examen). Corre en AWS, Azure o GCP, heredando alta disponibilidad, escalabilidad, elasticidad y durabilidad de la nube.

Como **SaaS puro**, no hay gestión de infraestructura ni patches manuales. Updates semanales transparentes sin downtime. Modelo pay-as-you-go. Acceso simple vía UI o conectores programáticos.

**Key exam points:** Cloud-native (no cloud-wash), data platform (no solo warehouse), SaaS puro, optimizaciones/mejoras automáticas.

# Multi-Cluster Shared Data Architecture

Snowflake usa una arquitectura nueva construida específicamente para la nube, diferente a las tradicionales shared-disk y shared-nothing.

**Arquitecturas tradicionales:** Shared-disk tiene storage centralizado (simple pero con punto único de fallo, latencia de red, limitaciones de escalabilidad). Shared-nothing tiene storage y compute en cada nodo (mejor rendimiento pero acoplado, difícil redistribuir datos, tendencia a over-provisioning, scaling limitado).

**Arquitectura de Snowflake:** Multi-cluster Shared Data con tres capas físicamente separadas pero lógicamente integradas:

1. **Storage Layer** : Almacenamiento centralizado en la nube con todos los datos de tablas, similar a shared-disk.
**Profundizando más en la storage layer**: 
La storage layer es el blob storage del cloud provider donde está deployada tu cuenta (S3 en AWS, Azure Blob en Azure, GCS en GCP). Heredamos la escalabilidad casi infinita, disponibilidad y durabilidad del proveedor. Por ejemplo, S3 replica datos en tres availability zones físicamente separadas.

Los datos se organizan en databases, schemas y tables. Snowflake soporta archivos estructurados (CSV, TSV) y semi-estructurados (JSON, Avro, Parquet). Al cargar datos, Snowflake los reorganiza transparentemente a su **formato columnar propietario**, que permite read optimization (saltar columnas no necesarias en queries OLAP). Los archivos se comprimen automáticamente (más eficiente en columnar) y se encriptan por defecto con AES-256.

Los datos se dividen en **micro-partitions** para que Snowflake ignore particiones innecesarias durante queries. Todo este proceso (encriptación, compresión, columnar storage, micro-partitioning) es automático y gestionado por Snowflake. Los datos solo son accesibles vía SQL, no directamente en blob storage correspondiente.

**Billing:** Tarifa plana por TB/mes de datos almacenados, calculado a fin de mes.

2. **Query Processing Layer**: Múltiples compute clusters llamados Virtual Warehouses, ejecutan queries, tienen acceso consistente al storage y cachean datos localmente, similar a shared-nothing.
**Profundizando más en la compute o query processing layer**:
Realiza las tareas de procesamiento sobre los datos del storage layer para responder queries. Consiste en **Virtual Warehouses** (VWs): clusters de compute gestionados por Snowflake que se crean vía SQL. Son abstracciones sobre instancias cloud (EC2 en AWS) que ejecutan el motor de ejecución de Snowflake. Como es SaaS, no tenemos acceso directo a los nodos, solo interactuamos con el objeto warehouse.

Los VWs funcionan similar a shared-nothing: hacen llamadas remotas al storage layer y cachean los datos en almacenamiento local de alta velocidad para queries subsecuentes. Son **ephemeral y flexibles**: se crean/dropean instantáneamente, pueden pausarse (sin cobro en pausa) y resumirse. Esto permite ajustar compute a la demanda real en lugar de provisionar para capacidad pico.

**Ventajas clave:** Múltiples VWs permiten scale-out para workloads concurrentes y **workload isolation** (un VW para data loading, otro para analytics, sin contención de recursos). Cada VW puede tener su propia configuración.

**Tamaños:** Extra Small a 6XL, indicando poder de compute (número y capacidad de instancias). Permite scale-up individual según requisitos del workload.

Todos los VWs tienen acceso a los mismos datos en storage layer simultáneamente. Snowflake usa **procesamiento ACID estricto** (no eventually consistent) mediante el Transaction Manager en Cloud Services Layer, que sincroniza accesos para que todos los updates/inserts estén inmediatamente disponibles para todos los VWs.

3. **Cloud Services Layer**: Coordina todo - autenticación, infraestructura, optimización de queries.
**Porfundizando más en la service layer**
Colección de servicios altamente disponibles y escalables que coordinan actividades across todas las cuentas Snowflake para procesar requests de usuarios. No se explora profundamente en la documentación porque es SaaS (no necesitamos entender los internals). Usa un modelo **global multi-tenancy** en lugar de crear servicios específicos por cuenta, logrando economías de escala y simplificando features como secure data sharing.

**Servicios principales:**
- **Authentication & Access Control:** Verifica identidad, credenciales y privilegios para acciones.
- **Infrastructure Management:** Crea y gestiona recursos cloud subyacentes (blob storage, compute instances).
- **Transaction Management:** Garantiza ACID compliance y acceso consistente de datos por todos los VWs.
- **Metadata Management:** Mantiene información y estadísticas sobre objetos y datos (metadata cache).
- **Query Parsing & Optimization:** Convierte SQL queries en planes ejecutables para VWs.
- **Security:** Maneja encriptación de datos y rotación de keys.

Corre en instancias cloud como los VWs, pero sin control ni visibilidad del usuario.

**Resumen de la Multi-Cluster Shared Data Architecture**

**Ventajas clave:** Storage y compute **desacoplados** (se escalan independientemente), sin límites duros de escalabilidad, múltiples clusters aislados (workload isolation - analytics y ETL no compiten por recursos). Arquitectura service-oriented donde cada capa es un servicio físico separado comunicándose vía REST.

**Exam focus:** Este desacoplamiento y la capacidad multi-cluster son los diferenciadores principales de Snowflake, resultado directo de ser cloud-native.

# Snowflake Editions y Object Model

## Snowflake Editions

Snowflake ofrece cuatro ediciones que afectan el pricing de compute y storage, así como las features disponibles:

**Standard:** Edición introductoria con funcionalidad core. Incluye SQL ANSI estándar (DDL, DML, multi-table insert, merge, windowing functions) y suite de seguridad Continuous Data Protection (Time Travel, network policies).

**Enterprise:** Incluye todo de Standard más features para empresas grandes como multi-cluster warehouses y database failover.

**Business Critical:** Ofrece mayor protección de datos y seguridad para datos muy sensibles o que deben cumplir regulaciones. Incluye private connectivity y customer-managed encryption keys. Tiene todo de Enterprise y Standard.

**VPS (Virtual Private Snowflake):** Máximo nivel de seguridad para organizaciones con requisitos estrictos (gobiernos, instituciones financieras). Incluye todo de Business Critical pero en un **ambiente completamente separado**, sin compartir la services layer con otras cuentas (metadata store tampoco compartido).

## Object Model

Todo en Snowflake es un **objeto** (algo con lo que puedes interactuar vía comandos como CREATE o DROP). Todos los objetos son **securables** (tienen privilegios que se otorgan a roles, roles que se asignan a usuarios, determinando qué pueden ver y hacer).

**Jerarquía de objetos:**
- **Organization:** Feature opcional (no por defecto) para gestionar una o más cuentas Snowflake, se configura vía organization object.
- **Account:** Colección de servicios Snowflake (UI, compute, storage) accesible vía URL, también es un account object para cambiar parámetros a nivel cuenta.
- **Account-level objects:** Objetos que no contienen datos pero configuran la cuenta (users, warehouses, roles, etc.).
- **Databases:** Primera forma de organizar datos, puede haber muchos por cuenta.
- **Schemas:** Subdivisión de databases, un schema pertenece a un database.
- **Schema-level objects:** Tablas (representación lógica de datos almacenados), views, stages, streams, pipes, tasks, y otros objetos con propósitos específicos.

## Jerarquía de Contenedores

### Organization y Account

**Organization:** Metadata container para gestionar una o más cuentas. Se genera automáticamente al crear una cuenta vía self-service. **Single account organization** (una cuenta para empresas pequeñas/medianas) o **multiple account organization** (muchas cuentas para corporaciones con diferentes departamentos/proyectos/locaciones).

Para gestionar múltiples cuentas, las standard accounts tienen el role **org_admin** que permite crear un **organization account** (tipo especial con URL diferente) con el role **global_org_admin**. Desde ahí se usa `CREATE ACCOUNT` para añadir más standard accounts al contenedor de la organización. El organization account gestiona todo el lifecycle: crear, renombrar, eliminar, ver cuentas (`SHOW ACCOUNTS`), configurar database replication/failover, y monitorear usage/billing across cuentas vía organization usage views.

**Account:** Colección privada de servicios con URL única. Asociado a un single cloud provider (AWS/Azure/GCP) y una región geográfica específica. La región afecta pricing, regulatory certifications, features disponibles y network latency. Cada account se crea como una Snowflake edition (puede cambiarse después) y contiene system-defined roles como **account_admin** para gestionar account-level objects.

**Account URL format:** `organization_name-account_name` (strings alfanuméricos de 7 caracteres auto-generados, o nombres personalizados vía Snowflake Support). Para herramientas non-UI (SnowSQL, Python connectors), se usa el **account identifier**: combinación de ambos separados por guion.

### Database

Primer contenedor lógico para datos, agrupa schemas que a su vez contienen schema-level objects (tables, views). Un account puede tener muchas databases.

**Propiedades:** Nombre único en el account, debe empezar con carácter alfabético. No puede contener espacios/caracteres especiales a menos que esté entre comillas dobles (con comillas = case-sensitive, sin comillas = case-insensitive).

**Capacidades:** Pueden crearse desde un clone de otra database en el mismo account, replicarse a otro account (con sus child objects), o crearse desde un share object de otra cuenta Snowflake.

### Schema

Agrupación lógica de database objects (tables, views, stages). Un database puede contener muchos schemas, cada schema pertenece a un database. Permite segmentar aún más los databases.

**Propiedades:** Nombre único dentro del database, mismas reglas de naming que databases (empieza con alfabético, sin espacios/especiales salvo con comillas dobles, case-sensitivity según comillas).

**Namespace:** `database_name.schema_name` forma el namespace. Puedes usarlo como prefijo para referenciar globalmente una tabla, o establecerlo en el contexto (UI/command line) con `USE DATABASE` y `USE SCHEMA`. Una vez establecido, no necesitas calificar table names con database y schema.

**Cloning:** Los schemas pueden clonarse, produciendo una versión espejo del schema y sus child objects.

### Tables

Abstracción lógica sobre los datos en storage layer que describe la estructura de tus datos para consultarlos. Existen cuatro tipos con diferentes requisitos de data retention:

**Permanent Table (default):** Existe hasta que se dropea explícitamente. Time Travel hasta 90 días (Enterprise+) o 1 día (Standard). Tiene 7 días de Fail-Safe (no configurable) donde Snowflake puede restaurar datos borrados.

**Temporary Table:** Para datos transitorios no permanentes (ej: pasos en ETL complejo). Persiste solo durante la sesión, después se purga completamente del sistema. No se puede convertir a otro tipo. Max Time Travel de 1 día (cualquier edition), sin Fail-Safe.

**Transient Table:** Similar a permanent (existe hasta dropeo explícito) pero con max Time Travel de 1 día (cualquier edition) y sin Fail-Safe. Útil para reducir storage costs ya que Fail-Safe contribuye a costos.

**External Table:** Permite consultar datos fuera de Snowflake (ej: Azure container). Overlays estructura sobre datos externos, Snowflake registra metadata de los archivos. Read-only, más lentas, sin Time Travel ni Fail-Safe.

**Identificar tipo:** Comando `SHOW TABLES` (columna kind) o iconos en UI object browser.

### Views

**Standard View:** Almacena definición de query SELECT, no datos. Al consultar la view, recupera datos de la source table. Se usan como tables (joins, subqueries, ORDER BY, GROUP BY, WHERE). No contribuyen a storage costs. Si se dropea la source table, la view retorna error "object does not exist". Función común: restringir contenidos de tabla (subset de columnas/filas).

**Materialized View:** Almacena query definition pero mantiene activamente el resultado (pre-computed dataset). Snowflake cobra compute por proceso background que actualiza periódicamente la view (serverless feature - no usa user-managed virtual warehouses). Costos adicionales de storage para resultados. Puede usarse para boost performance de external tables.

**Secure View:** Tanto standard como materialized pueden designarse secure añadiendo keyword `SECURE`. La query definition solo es visible para usuarios autorizados (comandos como `GET DDL` o `DESCRIBE` no retornan definition a no autoriza

### User-Defined Functions (UDFs)

Schema-level objects que permiten escribir funciones propias en SQL, JavaScript, Python o Java, extendiendo las system-defined functions. Componentes: function name, input parameters (cero o más), return type, y function definition (delimitada por `$$`).

**Tipos de retorno:**
- **Scalar:** Retorna una fila con un valor de una sola columna
- **Tabular (UDTF - User-Defined Table Function):** Retorna cero, una o múltiples filas. Requiere cláusula RETURNS con keyword TABLE especificando nombres y tipos de columnas

**Características:** Pueden llamarse como parte de un SQL statement (a diferencia de stored procedures). Pueden overloadearse (múltiples funciones con mismo nombre si parámetros difieren).

**SQL UDFs:** Language por defecto, no requiere especificar LANGUAGE parameter.

**JavaScript UDFs:** Requiere `LANGUAGE JAVASCRIPT`. Permite branching, looping, error handling y funciones built-in de la librería estándar JavaScript. No puede incluir librerías adicionales. Soporta recursividad (auto-llamarse). Data type mapping necesario entre Snowflake y JavaScript (ej: números se representan como doubles en JS).

**Java UDFs:** (Preview mode) Requiere `LANGUAGE JAVA`. Snowflake spinnea JVM para ejecutar código. Soporta Java 8-11. Restringido a librerías estándar Java. Definition puede ser inline code o pre-compiled code (JAR file) usando parámetros HANDLER (clase y función) y TARGET_PATH (ubicación JAR). **No pueden designarse como secure** (SQL y JavaScript sí).

**External Functions:** UDF que llama código mantenido y ejecutado fuera de Snowflake. Funciona como UDF regular desde perspectiva del usuario. Permite usar lenguajes como Go, C#, y referenciar librerías third-party. Ejemplo: AWS Lambda function.

Requiere **API_INTEGRATION object** con información para autenticarse con cloud provider (AWS, Azure, GCP). Usa **proxy service** (ej: AWS API Gateway) entre Snowflake y código externo como capa adicional de seguridad. Flow: SQL call → API_INTEGRATION credentials → proxy service → remote service (Lambda/Azure Functions/GCP Cloud Functions) → resultado back.

**Limitaciones external functions:** Query optimizer no puede ver código remoto (menos optimizaciones, ejecución más lenta). Latencia por ser remotas. Solo scalar (no tabular). No se pueden compartir vía secure data sharing. Potenciales security concerns (librerías third-party podrían almacenar datos sensibles fuera). Posibles charges por data movement entre cloud platforms/regions.

### Stored Procedures

Colecciones nombradas de SQL statements que pueden combinarse con lógica procedural para bundlear/modularizar queries comúnmente ejecutadas que realizan tareas administrativas. En Snowflake, son database objects creados en un database y schema específicos.

**Tres métodos de implementación:**
- JavaScript
- Snowflake Scripting (SQL con soporte para lógica procedural)
- Snowpark (Python, Java, Scala)

**Componentes (JavaScript method):**
- **Signature:** Identifier y parámetros de input (cero o más)
- **Return type:** Obligatorio especificarlo, pero opcional retornar valor en el código. Común no retornar nada o solo warning/error code. JavaScript stored procedures solo retornan scalar values, Snowflake Scripting (`LANGUAGE SQL`) puede retornar tabular data
- **Privilege model único:** Ejecuta con privilegios del role que creó el procedure (owner's rights) o del role que lo ejecuta (caller's rights)
- **Body:** Delimitado por `$$`, contiene código JavaScript con branching, looping, error handling

**Feature única de Snowflake:** Pueden mezclar JavaScript y SQL en su definition. Se crean SQL commands dinámicamente en variables JavaScript y se ejecutan usando Snowflake's JavaScript API (objeto Snowflake in-built). Resultados pueden almacenarse en variables y loopearse.

**Invocación:** Usando keyword `CALL` como statement independiente, no como parte de otro statement. Un single executable statement solo puede llamar un stored procedure.

**Diferencias vs UDFs:**
- Solo UDFs pueden llamarse como parte de SQL statement (ej: en SELECT)
- Ambos pueden overloadearse y tomar cero o más input arguments
- Procedures usan JavaScript API (mezclan SQL y JavaScript), UDFs no
- Procedures no necesariamente retornan valor, UDFs siempre retornan
- Valor retornado por procedure no puede usarse directamente en SQL (ej: guardar en SQL variable)
- No todos los UDFs soportan recursividad, procedures sí

**Filosofía:** Stored procedures para **realizar acciones** (DML: update, delete, truncate) más que retornar valores. UDFs para **calcular algo y retornar valor** al usuario.

### Sequences

Schema-level object usado para generar números secuenciales y únicos automáticamente. Caso de uso común: incrementar employee IDs o transaction IDs cuando se añade una fila a una tabla.

**Parámetros opcionales:**
- **START:** Integer desde el cual empezar a producir valores (default: 1)
- **INCREMENT:** Cuánto subir/bajar cada vez (default: 1, puede ser negativo)

**Uso:** Acceder a valores con `sequence_name.NEXTVAL`. Valores son **globally unique** (dos queries simultáneas no retornan el mismo número).

**Comportamiento con múltiples calls:** En Snowflake, múltiples `NEXTVAL` en una expresión incrementan across columnas (no retornan mismo número). Gotcha: si reruneamos query con múltiples `NEXTVAL`, el increment se multiplica por número de operaciones nextval, causando gaps. Snowflake advierte que sequences **no garantizan valores gap-free**.

**Usos comunes:**
1. Insert directo: `INSERT INTO table VALUES (seq.NEXTVAL, ...)`
2. **Default value para columna** (más común): `CREATE TABLE ... (id INT DEFAULT seq.NEXTVAL, ...)`

### Tasks

Objeto para **schedule ejecución** de SQL statement, stored procedure o lógica procedural (Snowflake Scripting). Casos de uso: copiar datos periódicamente, routines de mantenimiento (limpiar tablas antiguas), poblar reporting tables.

**Requisitos:** Rol ACCOUNTADMIN o custom role con privilege `CREATE TASK`.

**Configuración CREATE TASK:**
- Task identifier (único en schema)
- Virtual warehouse para ejecución (opcional: usar Snowflake-managed compute serverless, pero con limitaciones como no ejecutar SQL con Python/Java UDFs)
- **Triggering mechanism:** Intervalo en minutos, CRON expression, o después de completar otro Task (sin SCHEDULE parameter)
- Command a ejecutar

**Lifecycle:** Crear Task no lo inicia automáticamente. Requiere `ALTER TASK RESUME` (necesita privilege global `EXECUTE TASK` - solo ACCOUNTADMIN por default - y ownership/operate privilege en el Task). Inicia timer según config. Pausar con `ALTER TASK SUSPEND`.

**DAG (Directed Acyclic Graph):** Tasks pueden encadenarse. **Root task** define schedule para iniciar DAG run. **Child tasks** se activan tras successful completion del parent (no definen schedule). Tasks fluyen en una dirección. Un child task puede tener múltiples dependencias (espera a todas antes de ejecutar). Keyword `AFTER` seguido de parent task names separados por coma.

**Límites DAG:** Max 1,000 Tasks por DAG. Un Task solo puede linkarse a 100 otros Tasks. Todos los Tasks en DAG deben tener mismo Task owner y estar en mismo database/schema.

### Streams

Schema-level object que permite ver inserts, updates y deletes hechos a una tabla **entre dos puntos en tiempo**. Se crea sobre una tabla y es queryable. Output tiene estructura idéntica a base table pero solo contiene **changed records**.

**Metadata columns adicionales:**
- `METADATA$ACTION`: Indica si operación DML fue INSERT o DELETE
- `METADATA$ISUPDATE`: Indica si operación fue parte de UPDATE statement. Updates se representan como par DELETE + INSERT con ISUPDATE=true
- `METADATA$ROW_ID`: ID único para trackear cambios a una fila específica over time

**Funcionamiento:** Stream almacena un **offset** (punto desde el cual se registran cambios). Inicialmente vacío (solo muestra cambios after offset). Cambios se acumulan en Stream conforme se modifica base table.

**Progresión del offset:** Usar Stream en DML statement (ej: `INSERT INTO downstream_table SELECT * FROM stream`). Esto consume los cambios y progresa el offset.

**Integración con Tasks:** Task puede ejecutarse solo cuando Stream tiene datos usando `SYSTEM$STREAM_HAS_DATA()` (retorna Boolean). Task inserta cambios del Stream en downstream table, consumiendo cambios y progresando offset para que siguiente ejecución solo capture nuevos cambios.

**Resumen:** Tasks + Streams = método para **procesamiento continuo** de nuevos/changed records. Crean continuous transformation pipelines cuando se encadenan.


# Billing

## Modelos de Pricing

**On Demand:** Similar a pay-as-you-go de otros cloud providers. Solo pagas por recursos usados, invoice mensual. Mínimo $25/mes. Sin acuerdos de licencia a largo plazo.

**Pre-Purchased Capacity:** Comprar cantidad dollar upfront. Rates significativamente más bajos que on demand. Ideal si uso es predecible (ej: mínimo 100 horas compute/mes). Si uso es variable, on demand tiene más sentido.

## Áreas de Billing

**1. Virtual Warehouse Services:** Costo de customer-managed virtual warehouses creados y usados para ejecutar queries.

**2. Cloud Services:** Operaciones que no usan user-managed VWs pero cuestan compute. Ejemplos: metadata operations (CREATE TABLE, SHOW, DESCRIBE commands).

**3. Serverless Features:** Snowflake gestiona compute resources (no customer-managed VWs). Ejemplos: Snowpipe (auto-ingesta files), clustering, materialized views. Snowflake spinnea compute behind the scenes, gestiona scaling, resizing y duration.

**4. Data Storage:** Storage en stages (temporal) o tables (long-term).

**5. Data Transfer:** Transferir datos out of Snowflake o entre accounts si destino está en diferente región o cloud provider.

## Snowflake Credits

**Definición:** Unidades de billing para medir consumo de compute resources. Un credit representa valor monetario basado en cloud provider, región y Snowflake edition. Services con compute (VWs, cloud services, serverless) usan credits. Storage y data transfer usan moneda directa ($/TB).

### Virtual Warehouse Services

**Factores:** (1) VW size - cada tamaño tiene hourly credit rate diferente (XS = 1 credit/hora), (2) Duration en started state - consumo calculado per-second mientras está activo. **Suspended = no consume credits**.

**Importante:** Costo NO se basa en número/complejidad de queries, sino en started vs suspended state. Minimum billable period: 60 segundos (si corre 30 seg, se cobra 1 min). Después de 60 seg, billing per-second.

**Ejemplo:** XS VW por 2.5 horas = 2.5 credits. Si credit cuesta $4 en tu región = $10 total.

### Cloud Services

Queries que usan cloud services se cobran a **4.4 credits/compute hour**. Ejemplos: CREATE TABLE (no usa VW, solo cloud services). Aunque rate parece alto, queries son muy rápidas.

**Cloud Services Adjustment:** Solo se cobra cloud services usage que **excede 10% del daily usage de VW compute**. Se calcula diariamente. Ejemplo: 16 credits VW + 1.1 credits cloud services → no pagas cloud services (1.1 < 10% de 16).

### Serverless Features

Cada feature (clustering, Snowpipe, database replication, materialized views) tiene su propio credit rate per compute hour. Usan compute Y cloud services en background, cada tipo con su rate. Ejemplo: Materialized view maintenance = 10 credits/hora (compute) + 5 credits/hora (cloud services).

**Cloud services adjustment NO aplica a serverless features.**

## Data Storage y Transfer

**Data Storage:** Flat rate (no credits), calculado mensualmente basado en daily average de on-disk bytes. Incluye: database tables (con Time Travel y Fail-Safe data), files en Snowflake-managed stages. Files pueden comprimirse en stages. Costo: flat rate per TB según account type (capacity/on demand), cloud provider y región.

**Data Transfer:** Per-byte fee si transfieres datos entre regiones o entre cloud platforms. Costo depende de región y platform del account. 

**Casos con charges:**
- `COPY INTO location` unloading data a diferente cloud provider/región
- Database replication a account en diferente región/platform
- External functions (procesan data outside Snowflake)

**Ejemplo:** Unload data desde AWS London a Azure Tokyo = costo adicional vs mismo provider/región.

# SnowSQL

Command-line utility para ejecutar SQL statements en tu Snowflake account desde máquina local. Útil para automatizar jobs, integración con CI/CD pipelines, o si prefieres command line sobre UI.

## Capacidades

Ejecuta casi todos los comandos disponibles en UI: DML (INSERT, UPDATE, DELETE, MERGE), DDL (CREATE, ALTER), DQL (SELECT), account management (set parameters), loading/unloading data. Limitaciones: funcionalidades únicas de UI como graphing (ej: credit consumption).

## Instalación y Configuración

Disponible para Linux, macOS, Windows en developers.snowflake.com/snowSQL. Instalación crea carpeta hidden `.snowsql` en user directory con:
- **Binaries folder** (versión SnowSQL)
- **auto_upgrade:** Para upgrades automáticos
- **config:** Configurar connection parameters por defecto (account, username, password) para evitar supplierlos cada vez
- **history:** Almacena comandos ejecutados

## Connection Parameters

**Required:** `-a` (account identifier = `organization_name-account_name`, obtener con `SHOW ACCOUNTS` usando ACCOUNTADMIN/ORGADMIN role)

**Opcionales:** `-u` (username), `--database`, `--schema`, `--role`, `--warehouse`

**Security:** Passwords NO pueden pasarse vía parameters. Deben ingresarse vía interactive prompt, config file (option `password`), o environment variable.

## Modos de Uso

**1. Interactive Mode:** Construir SQL statements con auto-complete (session IDs, databases, schemas, tables). Salir con CTRL+D.

**2. One-off Query:** `-q` parameter para ejecutar query única sin entrar a interactive mode.

**3. SQL File Execution:** `-f` parameter para especificar SQL file. Ejecuta statements secuencialmente, retorna resultados y sale. Útil para ambientes automatizados.

## Comandos Predefinidos

**!help:** Lista comandos disponibles (administración SnowSQL + functional SQL commands)

**!options:** Ver opciones disponibles y valores actuales (ej: variable_substitution, text color)

**!set:** Configurar opciones (ej: `!set variable_substitution=true`)

**!define:** Definir SQL variables (ej: `!define table_name=customer`)

**!variables:** Listar variables definidas

**Uso de variables:** Prefix con `&` en SQL statements (ej: `SELECT * FROM &table_name`). Pueden supplirse al invocar SnowSQL para parametrizar SQL files.

## Data Loading

**PUT command:** Subir file desde local filesystem a Snowflake stage. Syntax: `PUT file://path/to/file @stage_name`

**COPY INTO:** Convierte raw file (ej: CSV) a Snowflake table storage format. Syntax: `COPY INTO table_name FROM @stage_name`

**!exit:** Salir de CLI.

# Conectividad

## Drivers y Connectors

Snowflake mantiene drivers y connectors open-source para conectar programáticamente y desarrollar aplicaciones. Soportan: **Python** (snowflake-connector-python package), **Go**, **PHP**, **.NET**, **Node.js**.

**Spark Connector:** Permite a Spark cluster read/write en Snowflake tables. Snowflake aparece como otra data source (similar a Postgres, HDFS, S3).

**Kafka Connector:** Para Confluent o open-source Kafka. Lee datos de Kafka topics y los carga en Snowflake tables.

**JDBC/ODBC:** Permiten conectividad con third-party tools no oficialmente partnered y lenguajes como Java o C++.

**Ejemplo Python:** Install package → import → crear connection object (username, password, account identifier) → crear cursor para ejecutar SQL statements y traverse results → ejecutar queries.

## Technology Partners

Third-party solutions certificadas por Snowflake para native connection/integration. Cinco categorías:

**1. Business Intelligence:** Software para análisis de datos, visualizaciones, dashboards. Ejemplos: Tableau, Power BI, QlikView, ThoughtSpot.

**2. Data Integration:** Tomar datos de source system, ponerlos en Snowflake con transformaciones. Ejemplos: dbt, Informatica, Pentaho, Fivetran.

**3. Security & Governance:** Data governance (Collibra), monitoring (Datadog), secrets storage (HashiCorp Vault), data cataloging (data.world).

**4. Machine Learning & Data Science:** Herramientas especializadas para workloads ML/DS con Snowflake data. Ejemplos: DataRobot, Dataiku, Amazon SageMaker, Zepl.

**5. SQL Development & Management:** Gestión de modeling, development y deployment de SQL code. Ejemplos: SqlDBM, SeekWell, Agile Data Engine.

**Partner Connect:** Feature en Snowflake UI para crear trial accounts con partner tools expeditamente. Usa setup wizard que crea objects en Snowflake (database, warehouse, roles) específicos para el partner tool.

## Snowpark

API para query y procesar datos usando lenguajes high-level (Java, Scala, Python) como alternativa a SQL. Abstracción principal: **DataFrame** (estructura 2D de rows y columns, similar a spreadsheet o pandas/Spark DataFrames).

**Modelo de computación:**
- **Lazy evaluation:** Operaciones se ejecutan solo cuando se solicita una acción (ej: `collect()`, `show()`), no cuando se declaran
- **Pushdown model:** Todas las operaciones usan Snowflake compute, **no se transfiere data** fuera de Snowflake (contraste con Spark connector que mueve data a Spark cluster)

**API Methods:** Proporciona métodos como `select()`, `join()`, `drop()`, `union()`, `filter()`, `groupBy()`, `count()`, `withColumnRenamed()`, `write()`.

**Workflow típico:**
1. Import Snowpark libraries
2. Crear session con connection parameters (account, user, password, role, warehouse, database, schema)
3. Crear DataFrame desde tabla (`session.table("table_name")`) o SQL query (`session.sql("SELECT...")`)
4. Aplicar transformaciones (filter, groupBy, etc.) encadenándolas con dot operator
5. Escribir resultados a tabla (`write.mode("overwrite/append").saveAsTable()`)
6. Cerrar session

**DataFrames:** Colección de row objects con schema definido. Cada row contiene column names y values. Lazily evaluated (resultados se recuperan al llamar `collect()` o `show()`, no al crear DataFrame).

**Objetivo:** Evitar que developers tengan que mover data fuera de Snowflake para usar su lenguaje preferido.

# Snowflake Scripting

Extensión de Snowflake SQL que añade soporte para **lógica procedural**: looping, branching, exceptions, declaración/asignación de variables. Código puede implementarse en stored procedure o ejecutarse directamente en worksheet.

## Scripting Block Structure

**1. DECLARE (opcional):** Define variables, cursors, result sets, exceptions.

**2. BEGIN...END:** SQL statements y scripting constructs (loops, branching). `BEGIN` también inicia transacciones. Snowflake recomienda `BEGIN TRANSACTION` para evitar confusión.

**3. EXCEPTION (opcional):** Exception handling code (qué hacer si hay error).

## Anonymous Block vs Stored Procedure

**Anonymous Block:** Ejecutado fuera de stored procedure. Variables solo accesibles dentro del scope del block. Objects creados (ej: tables) accesibles fuera del block.

**Stored Procedure:** Contiene misma lógica procedural en long-lived object, bueno para sharing entre usuarios y código repeatable.

**Nota:** SnowSQL y classic console requieren wrapping scripting blocks en `$$`. Snowsight no lo necesita.

## Características Principales

**Variables:** Declarar con data type o inferir tipo con `LET`. Asignar con `:=` o `LET`. Usar funciones en blocks.

**Branching:** IF-ELSE statements, CASE statements.

**Looping:** FOR loop, WHILE loop, REPEAT, LOOP. FOR loop example: inicializar valor, iterar hasta max, ejecutar cálculos por iteración.

**Cursors:** Para loop sobre records en tabla. Declarar: `cursor_name CURSOR FOR query`. Ejecuta cuando se llama (no cuando se declara). Loop through cursor para procesar cada row.

**Result Sets:** SQL data type similar a cursor pero ejecuta query cuando se asigna a variable (no cuando se llama), por eso "holds results of query". Acceso vía `TABLE()` function (retorna all/subset de rows) o iterando con cursor.

**RETURN:** Keyword para retornar resultado del block.


# Snowflake AI Features: Cortex y Machine Learning

Snowflake cambió su nombre a "Snowflake AI Data Cloud" para señalar el enfoque en AI technologies, principalmente large language models (LLMs). Dos grupos principales: **Cortex** (features usando LLMs) y **Machine Learning** (ML functions in-built para técnicas tradicionales como classification/forecasting, más capacidad de crear custom ML functions).

## Cortex Features

### Cortex AI SQL Functions

Extienden SQL con funciones AI-related ejecutables como cualquier función normal. Ejemplo principal: **AI_COMPLETE**

**Parámetros:**
- **Prompt (string):** Pregunta a responder o instrucción creativa
- **Model name (string):** LLM a usar (Snowflake propios, OpenAI, Anthropic, Meta, Mistral AI, DeepSeek). Elección afecta output y precio (modelos Snowflake más baratos)
- **Model parameters opcionales:** Temperature (controla randomness), guardrails (feature Snowflake que filtra respuestas potencialmente dañinas)

**Uso práctico:** Responder preguntas sobre datos. Ejemplo: `AI_COMPLETE(prompt, model)` para resumir columnas de texto (ej: reviews), output puede poblar nueva columna para facilitar análisis de free text.

### Snowflake Copilot

LLM-powered SQL assistant en panel UI. Genera SQL code desde preguntas en plain English. Entiende contexto de datos (database structure, tables, columns, objects) pero **no puede generar cross-database o cross-schema queries**. También sugiere mejoras a queries escritos (performance, concisión) y ayuda a descubrir Snowflake features desde documentación.

**Funcionalidad:** Click "Run" (añade query a worksheet y ejecuta automáticamente) o "Add" (añade a worksheet para ejecutar después).

**Limitación:** Depende de object comments (table/column comments). Si no están bien documentados, puede ser limitado en entender datos out-of-the-box.

### Document AI

Extrae datos (nombres, fechas, checkbox values) desde documentos no estructurados (PDFs). Usa LLM propietario de Snowflake llamado **Arctic Tilt**.

**Workflow:**
1. **Crear model build** (vía UI → AI and ML → Document AI): Representa un tipo de documento del cual extraer datos
2. **Apuntar a sample PDFs** previamente uploaded a stage
3. **Definir questions:** Prompts para extraer tipos específicos de datos usando OCR. Model retorna answer + confidence score (0.9 = 90% confidence)
4. **Review y fine-tuning:** Verificar accuracy en samples, corregir respuestas incorrectas, opcionalmente fine-tune foundational model
5. **Publish model build:** Usar en SQL code con syntax `!PREDICT(model_build_name, file_url, optional_version)`. Retorna JSON con confidence scores y valores extraídos

### Cortex Fine-Tuning

Tomar output de large model para entrenar smaller model, más rápido, barato y mejor en tarea específica. Use case: millones de social media posts, calcular sentiment.

**Proceso:**
1. Usar large model para generar accurate labels en sample dataset (ej: 10K posts con sentiment via `AI_COMPLETE`)
2. **Crear fine-tuning job** con `FINETUNE()` function: especificar output model name, base model, training data query
3. Usar fine-tuned smaller model para analizar resto de datos (mucho más eficiente)

**Ahorro ejemplo:** Claude Opus (12 credits/1M tokens) vs Mistral 7B fine-tuned (0.12 credits/1M tokens) = 100x menos costo.

**Comandos:** `FINETUNE()` para crear job, `SHOW`/`DESCRIBE` para check status, `CANCEL` para stop job.

### Cortex Search

Combina **keyword search** con **semantic search** (busca por significado, no solo palabras exactas). Usa **embeddings** (representaciones numéricas de texto) y **vector search** (comparar vectors usando matemáticas). Si dos textos tienen significado similar, sus vectors están cerca.

**Implementación:**
1. Crear **Cortex Search Service** que indexa columna de texto para semantic vector-based searching
2. **Attributes:** Filtrar resultados por columnas específicas (ej: product_ID, customer_ID, review_date)
3. **Target lag:** Refresh frequency del índice (ej: 30 minutos)
4. Especificar embedding model default de Snowflake

**Uso:** REST API endpoint (enviar POST requests) o Snowflake Python SDK. Retorna JSON payload con resultados.

### Cortex Analyst

REST API fully-managed que convierte plain text questions sobre structured data en SQL commands y retorna resultados. Para construir chat interface sobre datos Snowflake, útil para non-technical business users.

**Arquitectura:**
- **Bottom:** Structured data en Snowflake tables
- **Middle:** Cortex Analyst REST API (entre tables y frontend app)
- **Top:** Frontend app (comúnmente Streamlit - open-source tool para interactive web apps en Python, natively supported por Snowflake)

**Uso en Streamlit:**
- Call API vía `get_analyst_response()` function
- Proveer: (1) **Semantic model** (YAML document con tables, columns, elementos accesibles - bridges gap entre data definition en metadata y cómo business user phrase questions), (2) **Prompt/chat history**
- API retorna JSON con response y SQL usado, mostrado en Streamlit app

