# Access Control y Security

## Access Control Frameworks

Snowflake usa tres frameworks de access control:

### 1. Role-Based Access Control (RBAC)
Práctica común de adjuntar access privileges (select, modify, remove, etc.) a **roles** que luego se asignan a users. Un role puede asignarse a múltiples usuarios y típicamente se alinea con business functions (departamentos). Habilita el **principio de least privilege** (usuarios reciben solo permisos mínimos necesarios para sus tareas, reduciendo riesgo de seguridad).

### 2. Discretionary Access Control (DAC)
Cada objeto tiene un **owner role** (típicamente el role que creó el objeto). Este role tiene full privileges: select, modify, remove, **grant access** a otros roles, y **transfer ownership** a diferente role. Trabaja junto con RBAC.

**Ejemplo:** User con Role A ejecuta CREATE TABLE → Role A es owner → tiene full privileges → puede grant privilege SELECT a Role B.

**Managed Access Schema:** Tipo especial de schema (keyword durante creación) donde owners de child objects (tables, views) NO pueden grant privileges a otros roles. Solo el owner del schema puede grant privileges a objetos en ese schema.

### 3. User-Based Access Control (UBAC)
Privileges aplicados directamente a users, saltando el paso de asignar a role. Útil para casos temporales (ej: un user necesita acceso a tabla adicional por pocos días sin afectar a otros users con mismo role). **RBAC sigue siendo el mecanismo primario** - UBAC solo para casos específicos.

## Securable Objects

La mayoría de objetos en Snowflake son **securables** (se pueden aplicar granular privileges determinando qué puede hacer un user con ese objeto). Ejemplos de privileges en Virtual Warehouse: USAGE (ejecutar queries), MODIFY (modificar configuración como size).

**Object Hierarchy:** Importante para access control porque para acceder a child object necesitas acceso a parent object. Ejemplo: crear tabla requiere acceso a database y schema donde residirá.

**Facts importantes:**
- Todo securable object es owned por **single role** (típicamente el que lo creó). Ver owner con `SHOW <object>` command (columna owner)
- Owner tiene all privileges por default, incluyendo grant/revoke privileges a otros roles
- Owner puede transfer ownership a otro role
- Acceso a securable object puede compartirse entre users si role es shared
- **MANAGE GRANTS global privilege:** Cualquier role con este privilege puede grant/revoke privileges en cualquier objeto como si fuera el owner
- **Default deny:** A menos que sea permitido por grant, acceso a securable object será denegado

## Roles

Entity al cual privileges en securable objects pueden ser granted o revoked. Roles se asignan a users, dándoles autorización para acciones específicas. Un user puede tener múltiples roles y switch between them en Snowflake session.

**Role Hierarchy:** Roles son securable objects, así que pueden grantarse a otros roles creando jerarquía. Child role granted a parent role → parent hereda privileges de child. Ejemplo: Role 3 granted a Role 2 → Role 2 puede hacer todo lo que Role 3 hace. Role 1 granted a Role 2 → Role 1 puede hacer todo lo que Role 2 y 3 hacen.

### System-Defined Roles

Snowflake accounts incluyen 6 system-defined roles formando jerarquía. No pueden dropearse ni revocar sus privileges, pero sí añadirles privileges (aunque Snowflake desalienta esto, prefiere custom roles). Conforme bajas en hierarchy tree, cada role tiene menos privileges.

**ORGADMIN (Organization Administrator):** Gestiona operaciones a nivel organization. Puede crear accounts en organization, view all accounts, ver regiones enabled, view usage information across organization.

**ACCOUNTADMIN (Account Administrator):** Top-level role para account con muy amplio set de permissions (grant a limitado número de users). Encapsula SYSADMIN y SECURITYADMIN (hereda todos sus privileges vía role inheritance). Único responsable de configurar parameters a nivel account. Puede view/operate en todos los objetos del account, view/manage Snowflake billing y credit data, stop cualquier SQL statement running.

**SYSADMIN (System Administrator):** Main job = gestionar objects en account. Tiene privileges para crear warehouses, databases y muchos otros objects.

**SECURITYADMIN (Security Administrator):** Puede manage any object grant globally, crear/monitor/manage users y roles. Granted el **MANAGE GRANTS security privilege** (modificar/revocar cualquier grant). Hereda privileges de USERADMIN vía system role hierarchy.

**USERADMIN:** Para user y role management. Granted **CREATE USER** y **CREATE ROLE** security privileges (permite crear users y roles en account).

**PUBLIC:** Pseudo-role automáticamente granted a every user y every role en account. Object owned por PUBLIC role es accesible a everyone en account.

### Custom Roles

Permiten crear roles propios con custom fine-grained security privileges. Se alinean a grupos/teams (support team, analysts) o ambientes (dev, prod). Ejercen principio de **least privilege** (usuarios solo reciben privileges necesarios, reduciendo riesgo de error humano o actividad maliciosa).

**Creación:** Cualquier role con CREATE ROLE privilege (USERADMIN, SECURITYADMIN, ACCOUNTADMIN) puede crear custom roles.

**Best practice:** Crear hierarchy de custom roles con top-most custom role assigned a SYSADMIN role. Si custom roles no están assigned a SYSADMIN, system administrators no podrán manage objects owned por custom roles.

## Security Privileges

Define qué nivel de acceso tiene un role en securable object. Típicamente forma de **verb + noun** (ej: GRANT MODIFY ON DATABASE).

### Cuatro Categorías de Privileges

**1. Global Privileges:** Habilitan acciones como create databases y monitor account-level usage. Ejemplo: MANAGE GRANTS.

**2. Privileges for Account Objects:** Diferentes access levels a account-level objects como databases. Ejemplo: MODIFY database.

**3. Privileges for Schemas:** Definen qué puedes hacer dentro de schema. Ejemplo: USAGE en schema.

**4. Privileges for Schema Objects:** Diferentes access levels a schema-level objects como tables. Ejemplos: SELECT, TRUNCATE.

### Gestión de Privileges

**Comandos:** `GRANT` y `REVOKE`. Uso restringido al role owner del objeto o roles con MANAGE GRANTS global privilege.

**Future Grants:** Permiten definir privileges en objects que aún no existen. Ejemplo: grant SELECT privilege en todas las nuevas tables creadas en schema. Facilita gestión (no necesitas añadir grants a cada objeto individual conforme se crean). **No soportados** con: data sharing, data replication, masking policies, row access policies.


## Autenticación

Determina si la persona o programa accediendo Snowflake es realmente quien dice ser (diferente de authorization que determina niveles de acceso).

### Username y Password

Método default de autenticación en Snowflake (vía UI o Snowflake clients como SnowSQL, Python connector). Users con **USERADMIN** system-defined role pueden crear usuarios adicionales (usa CREATE USER privilege).

**Password requirements:**
- Case-sensitive string hasta 256 caracteres (incluye espacios y caracteres especiales como !, %)
- Mínimo 8 caracteres
- Al menos un dígito
- Al menos una uppercase letter y una lowercase letter

**Weak password option:** Posible set weak password que no cumple requirements (para admins usar generic passwords durante creación). Snowflake recomienda set `MUST_CHANGE_PASSWORD = true` para forzar cambio en próximo login.

**Default role:** Si no se set, user obtiene automáticamente **PUBLIC** system-defined role al iniciar sesión.

### Multi-Factor Authentication (MFA)

Capa adicional de seguridad requiriendo probar identidad con password + información adicional (push notification o código). Powered by **Duo Security** (completamente managed por Snowflake, no requiere signup externo). User debe descargar Duo Mobile app, acceso vía QR code en setup process de Snowflake UI.

**Configuración:** MFA habilitado **per-user basis** solo vía UI. No es automático, user debe ir a Preferences → General → Multifactor Authentication. Snowflake recomienda fuertemente que todos los users con **ACCOUNTADMIN** role usen MFA. MFA automáticamente enabled a nivel account sin management adicional.

**Login flow con MFA:** Ingresar credentials → tres formas de probar second factor:
1. Aprobar Duo push notification en phone (más rápido)
2. Click "Call Me" → seguir instrucciones de phone call
3. Click "Enter a passcode" → ingresar passcode generado por Duo app

**Propiedades configurables:**
- **BYPASS MFA:** User con ALTER USER privilege puede temporalmente disable MFA para un user (configurable número de minutos). Útil si trusted user no puede acceder device con Duo Security
- **DISABLE MFA:** Cancela enrollment del user. Para usar MFA nuevamente, debe re-enroll
- **ALLOW_CLIENT_MFA_CACHING:** Account-level parameter que permite users almacenar MFA token en client-side cache. User no necesita responder tantos prompts al conectar/autenticar con Snowflake usando MFA. Útil si logueando múltiples veces en corta duración

### Federated Authentication (SSO)

Permite conectar a Snowflake usando secure **single sign-on (SSO)**: loguearse a varios sistemas software independientes con single ID y password. Snowflake delega authentication responsibility a external **identity provider (IdP)** SAML 2.0 compliant.

**Native support:** Okta y ADFS IdPs. Para otros IdPs, debes definir custom application para Snowflake en tu IdP.

**IdP:** Servicio independiente responsable de crear/mantener user credentials y autenticar users para SSO access. Snowflake verifica con third party para confirmar identidad. En ambiente federated, Snowflake es **service provider (SP)**.

**Login workflow:** Dos formas de iniciar:
1. **Snowflake-initiated:** Vía Snowflake UI → botón adicional habilitado → link a IdP login page → ingresar IdP credentials → load Snowflake con nueva session
2. **IdP-initiated:** Ir directo a IdP login page → login → usar IdP facilities para start Snowflake session

**Account properties (Snowflake side):**
- **SAML_IDENTITY_PROVIDER:** Account-level property que habilita federated authentication. Acepta JSON object con: certificate (IdP certificate verifying communication), ssoUrl (IdP endpoint donde Snowflake envía SAML requests), type (tipo de IdP: Okta, ADFS, custom), label (texto en botón de Snowflake login page)
- **SSO_LOGIN_PAGE:** Controla si SSO button se muestra en main login page

### Key Pair Authentication

Método usando par de cryptographic keys (public y private) como alternativa a username/password. Para conectar vía **Snowflake clients, NO UI**.

**Setup process:**
1. User genera private-public key pair usando OpenSSL (mínimo 2048-bit keys)
2. Public key se asigna a user con comando específico
3. Private key se mantiene safe locally por user
4. User configura Snowflake connector (Python, etc.) para autenticar usando key pair. Cada connector tiene método propio

**Key Pair Rotation:** Permite update active public key sin downtime. Habilita dos active keys simultáneamente para swap. Comandos: set second key → unset first key (ahora out of date).

### OAuth y SCIM (Conocimiento Básico)

**OAuth:** Snowflake soporta OAuth 2.0 protocol. Open standard protocol que permite supported clients authorized access a Snowflake **sin compartir/almacenar user login credentials**. Ejemplo: Tableau client conecta a Snowflake sin mantener usernames/passwords (delegated authorization - user delega habilidad de read data a client). Dos pathways: Snowflake OAuth y External OAuth.

**SCIM (System for Cross-domain Identity Management):** Facilita management de user identities y groups (en Snowflake = roles). IdP como ADFS usa SCIM client para hacer RESTful API requests a Snowflake SCIM server. Snowflake verifica request y realiza acciones en roles/users. Usos: manage user lifecycle (crear/delete users, update settings), map Active Directory groups a Snowflake roles.

## Network Policies

Por default, Snowflake no bloquea IPs de acceder al account URL. **Network policies** permiten allow/deny acceso basado en IP addresses (capa adicional de seguridad sobre authentication).

### Componentes

**Network Rules:** Objects que almacenan single IP o range de IPs. Parámetros: MODE (ingress para inbound traffic), TYPE (IPv4, IPv6 no soportado), VALUE_LIST (IPs en CIDR notation o single IPs separadas por comas).

**Network Policy:** Referencia network rules para allow/block traffic. Tiene ALLOWED_NETWORK_RULE_LIST y BLOCKED_NETWORK_RULE_LIST (ambos opcionales). Si solo defines allowed list, todo fuera del range es denied. Si IP está en ambas listas, **blocked se aplica primero**.

### Aplicación

**Privileges:** SECURITYADMIN, ACCOUNTADMIN, o custom role con CREATE NETWORK POLICY/RULE privilege. Crear policy NO la activa, debe **aplicarse** (account o user level).

**Account-level:** Solo una policy activa a la vez (reemplaza anterior). Ver con `SHOW PARAMETERS`.

**User-level:** Restringe IPs desde donde user puede conectar. Usa `ALTER USER`. Si user tiene ambas, **user-level toma precedencia**.

**Bypass temporal:** Property `MINS_TO_BYPASS_NETWORK_POLICY` solo configurable por Snowflake Support.

**Workflow:** Crear network rule (IPs) → crear network policy (referencia rules) → aplicar a account/user.


# Data Encryption

## Encryption at Rest y in Transit

**At Rest:** Todos los datos en storage layer (tables), internal stages (files para loading/unloading), virtual warehouse y query result caches se encriptan automáticamente con **AES-256 strong encryption**. Todo lo que Snowflake controla está encrypted at rest. Transparente, sin configuración o management del user.

**In Transit:** Secure **HTTPS** siempre usado al conectar a Snowflake account URL (browser UI, JDBC driver, etc.). Snowflake usa **TLS 1.2 protocol** para encriptar todas las network communications desde client machine a Snowflake endpoints.

**End-to-End Encryption:** Combinación de encryption at rest e in transit en cada etapa del data loading/unloading process. Reduce attack surface, expone data solo a authorized users.

## Data Loading Scenarios

Dos flows principales determinados por tipo de stage:

**Internal Stage:** Al upload file (ej: comando PUT), Snowflake transparentemente encripta loaded files. Encryption computation se realiza en **client machine**. Luego, usando different key, data se encripta al cargar en long-term table storage.

**External Stage (ej: S3 bucket, no managed por Snowflake):** Puedes dejar contents unencrypted. Data se encripta al cargar en Snowflake tables. Si usas client-side encryption en external stage antes de loading, debes proveer decryption information para que Snowflake decrypt y re-encrypt al cargar en table storage.

## Hierarchical Key Model

Cada key más arriba en hierarchy encripta la key debajo (proceso llamado **wrapping**). File key final encripta user's data.

**Jerarquía:**
- **Account Master Key:** Corresponde a one customer account
- **Table Master Key:** Corresponde a one database table
- **File Key:** Cada individual data file o micro-partition encriptado con separate key

**Ventaja:** Limita cantidad de data que una key protege. Si file key es exposed, attacker no puede decrypt data de otras tables o accounts (reduce attack surface si key es comprometida).

**Root Key Storage:** Top-most encryption keys almacenadas en **AWS Cloud HSM Classic**. Lower-level keys generadas usando Cloud HSM's random number generation (hardware-based solution integrada con Snowflake security framework).

## Key Rotation y Re-keying

**Key Rotation:** Reemplazar existing account y table encryption keys cada **30 días** con new key. Existing key se retira y solo se usa para decrypt data. Example: table key rotada monthly, encrypting new files con new key cada mes. Se mantienen retired keys para decrypt data files encriptados anteriormente.

**Periodic Re-keying:** (Enterprise Edition feature) Cuando retired key excede **1 año**, Snowflake automáticamente crea new encryption key y re-encripta toda la data previamente protegida por retired key. New key se usa para decrypt table data going forward. Habilitar con ACCOUNTADMIN role: set `PERIODIC_DATA_REKEYING = true`. **Costo adicional** por extra fail-safe storage de data files re-keyed.

## Tri-Secret Secure

Feature que permite encrypt data en Snowflake con **composite master key**: parte customer-managed key + parte account-managed key (por Snowflake). Customer-managed master key mantenida en key management service del cloud provider (AWS KMS, Google Cloud KMS, Azure Key Vault).

**Ventajas:** Mayor control sobre data. Imposible decrypt data sin customer-managed key. En data breach, puedes withdraw access a customer-managed key (disabling access a data).

**Desventaja:** Más management overhead para protect y ensure key availability.

**Habilitación:** Contactar Snowflake Support.


# Column-Level y Row-Level Security

## Column-Level Security (Dynamic Data Masking)

Security feature donde policy se aplica a column en table/view para **mask data at query runtime** (no stored en formato masked - eso es static masking). Masking policy define reglas determinando quién tiene autorización para ver column's data y cómo maskearlo.

**Funcionamiento:** User no autorizado ejecuta SELECT con masked column → query result muestra masked o partially masked data (transparente al user, no necesita conocer masking policy). User autorizado ejecuta mismo query → result no masked, plaintext values.

### Crear y Aplicar Masking Policy

**Componentes CREATE MASKING POLICY:**
- **Identifier:** Nombre de policy
- **val <datatype>:** Val referencia column value en policy body. Input datatype determina en qué columns puede aplicarse policy (ej: STRING solo en columns tipo string)
- **RETURNS <datatype>:** Datatype del masked value a retornar. Input y output **deben tener mismo datatype**
- **Arrow (→):** Separa policy signature de policy body
- **Policy body:** CASE statement con condiciones determinando si retornar masked value y qué masking aplicar

**Ejemplos de masking:**
- Full masking: Retornar valor completamente masked (ej: '***')
- Partial masking: Mostrar parte del valor (ej: dominio de email, no identifier antes de @)
- Hashing: Usar funciones como SHA-2 (hashing algorithm). UDFs también pueden usarse
- Semi-structured values: Retornar valores semi-estructurados

**Aplicar policy:** `ALTER TABLE` statement. Pueden aplicarse a thousands de columns across databases/schemas.

### Características Adicionales

- Masking policies son **schema-level objects**
- Crear/aplicar puede hacerse **independientemente de object owners** (permite security team separado determinar policies - segregation of duties)
- **Nested policies:** Pueden existir en tables y views que referencian esas tables. Lowest masking policy aplica primero
- Policy aplica **no matter where column is referenced** en SQL statement. Puede causar unexpected behavior (ej: join con masked data = join based on masked data, no underlying data)
- Policy puede añadirse cuando object es creado o después

### External Tokenization

Masking policies pueden usar **external functions** para mask data. Permite ingestar scrambled/tokenized data en Snowflake y detokenize at query runtime.

**Proceso:**
1. Tokenized data ingested en table (data **stored en tokenized form** en Snowflake storage, no plaintext)
2. Masking policy creada con detokenization external function configurada en policy body
3. Al usar column en SQL query: si user autorizado → policy hace call a external service vía external function con tokenized data → external function retorna detokenized data. Si user no autorizado → masked value retornado

**Beneficio:** Store underlying data en tokenized form, ni Snowflake ni cloud provider pueden leer contents.

## Row-Level Security (Row Access Policies)

Permite restringir qué rows se retornan en query basado en condition (ej: current role). Sintácticamente similar a masking policies pero en lugar de mask single column, entire row está presente u omitida del output.

**Funcionamiento:** User no autorizado → SELECT query tiene all rows filtered out o partially filtered out. User autorizado → complete result set retornado. Transparente al user al querying table.

### Crear y Aplicar Row Access Policy

**Componentes CREATE ROW ACCESS POLICY:**
- **Identifier:** Nombre de policy
- **Column y datatype:** Column como "anchoring point" (idealmente relacionada con lo que policy intenta lograr, ej: account_ID column para policy sobre bank accounts)
- **RETURNS BOOLEAN:** Return type siempre Boolean (diferente de masking policies donde input/output deben ser mismo type)
- **Arrow (→):** Separa policy definition de policy body
- **Policy body:** CASE statement con condiciones. Retorna all rows o none based on conditions (ej: current role, current user, current region)

**Row access policies complejas:** Pueden incorporar lookup table, limitando rows que user puede ver based on otra column's value.

**Aplicar policy:** `ALTER TABLE` command a column.

### Similitudes con Column Masking Policies

- Schema-level objects (database y schema deben existir antes)
- Permiten segregation of duties
- Mismo pattern: crear primero, luego aplicar
- Pueden nested (first policy in chain aplica primero)
- Pueden añadirse cuando object es creado o después

### Row + Column Policies en Mismo Table/View

**Restricciones:**
- No puedes apply masking policy a column si ya está referenced por row access policy
- **Row access policies evaluadas ANTES que masking policies**
- Misma column no puede especificarse en ambas policy signatures simultáneamente

# Secure Views

Tipo de view diseñada para **limitar acceso a underlying tables o internal structural details** de la view. Tanto standard como materialized views pueden designarse como secure.

## Creación y Conversión

**Crear:** Añadir keyword `SECURE` antes del view identifier en CREATE VIEW DDL.

**Convertir back a standard view:** Owner role puede ejecutar `ALTER VIEW ... UNSET SECURE` command.

## Cómo Asegura la View (Dos Formas)

### 1. Oculta Underlying Tables y Structural Details

View definition y details solo visibles para **authorized users** (users granted el role que owns la view). 

**Commands que NO muestran secure views:**
- `SHOW VIEWS`
- `SHOW MATERIALIZED VIEWS`
- `GET_DDL` utility function
- `INFORMATION_SCHEMA.VIEWS`
- `ACCOUNT_USAGE.VIEWS`

### 2. Bypasea Query Optimizer Optimizations

**Problema con standard views:** Query optimizer construye execution plan con optimizations (ej: push down - colocar filter primero para reducir data size). Esto podría permitir a user sin acceso a all data en underlying table determinar values que normalmente no tendría acceso.

**Solución secure views:** Bypasea algunas view optimizations del Query Optimizer para prevenir exposición a data de underlying table que debería ser filtered por la view.

**Trade-off:** Secure views pueden ser **más lentas** que standard views porque no usan query plan optimizations. Por esto, Snowflake desalienta hacer secure views que no tengan strict security requirements.


# Account Usage y Information Schema

## SNOWFLAKE Database (Account Usage)

Database read-only compartida por default con todas las accounts. Importada usando secure data share llamado **ACCOUNT_USAGE**. Propósito: compartir views con fine-grained usage metrics para query y report sobre account y object usage (ej: queries ejecutadas última hora, credits usados por warehouse en 3 meses). Home para **long-term historical metadata** sobre Snowflake account.

### Schemas en SNOWFLAKE Database

**ACCOUNT_USAGE (main schema):** Views que muestran object metadata y historical usage metrics para account. Ejemplo: view TABLES con metadata de todas las tables creadas en account. Usado para govern/maintain Snowflake account.

**CORE:** Relativamente nuevo, contiene solo system tags usados por data classification. Más views/objects en future releases.

**READER_ACCOUNT_USAGE:** Views con object metadata y usage metrics para reader accounts creados para main account.

**DATA_SHARING_USAGE:** Views con información sobre listings publicados en data exchange.

**ORGANIZATION_USAGE:** Historical usage data para todas las accounts en organization.

**INFORMATION_SCHEMA:** Creado por default para every database en account (no único de SNOWFLAKE database).

### Características Account Usage Views

**Acceso:** Por default solo ACCOUNTADMIN role. Privileges pueden grantarse a otros roles.

**Dropped objects:** Record dropped objects, no solo activos. Column DELETED (timestamp cuando object fue dropped). Field INTERNAL_ID para diferenciar objects dropped/recreated con mismo nombre.

**Latency:** ~2 horas para mayoría de views (tiempo para changes aparecer en view). Breakdown completo en Snowflake documentation.

**Retention period:** 1 año para most views con historical usage metrics (ej: METERING_HISTORY view con historical metadata sobre credit consumption).

## Information Schema

Cada database creado en account automáticamente incluye read-only schema **INFORMATION_SCHEMA** (basado en SQL-92 ANSI Information Schema + queryable views/functions específicas de Snowflake).

### Contenido

**Views con información de objects en database:** Ej: TABLES view con metadata de tables en ese specific database.

**Views con account-level objects:** Objects fuera del scope del database (roles, warehouses, other databases). Algunas views hold metadata específico de database, otras hold identical metadata across cada database's information schema. Ej: DATABASES view tiene metadata de databases across account.

**Table functions:** Para historical/usage data across account. Functions que retornan múltiples rows, queryable como table. Ejemplo: COPY_HISTORY table function (retrieve metadata sobre files loaded en table).

**Output basado en privileges:** Al query information schema view/table function, solo objects a los que current role tiene granted access son returned.

## Account Usage vs Information Schema

Ambos tienen overlapping functionality con identical structures y naming conventions, pero diferencias cruciales:

| Característica | Account Usage Views | Information Schema |
|---|---|---|
| **Dropped objects** | Sí | No |
| **Latency** | Hasta 3 horas | Sin latency (más up-to-date) |
| **Retention period** | Hasta 1 año | 7 días a 6 meses |
| **Uso recomendado** | Past objects, historical data | Most up-to-date metadata |


# Object Tagging

Schema-level object que permite asignar **metadata específico** a otros database objects, sirviendo como labeling/classification mechanism. Por ejemplo: tag llamado `business_unit` asignado a table/warehouse con string value (sales, HR, operations). Puede aplicarse at object creation time o después con ALTER statement.

**Propósito:** Agrupar objects independientemente de su ubicación en object hierarchy del account. Ayuda con **data governance** (managing data para que esté organized, secure, compliant con regulaciones).

## Data Governance en Snowflake

Snowflake divide data governance capabilities en cuatro categorías. Object tagging cae bajo **data sensitivity y access visibility** (junto con sensitive data classification). Ya cubrimos **protecting data** (masking policies, row access policies).

**Uso con security:** Combinar object tagging con masking policies. Cuando tag se asigna a sensitive column, puede automáticamente enforce masking policy.

## Gestión de Tags (Centralized vs Decentralized)

**Responsable:** Típicamente **data steward** (team/individual responsible de implementar robust data governance strategy). Snowflake recomienda crear custom role como `TAG_ADMIN` granted a data steward user.

**Centralized approach:** Tag admin tiene privileges para CREATE y APPLY tags. Un team/individual crea y aplica tags.

**Decentralized approach:** Tag admin crea tags, pero qué objects se tagean depende de individual users.

## Crear y Aplicar Tags

**CREATE TAG:** Define tag object con identifier. Opcionalmente especificar **allowed values** (comma-separated list de strings). Si se intenta aplicar tag con value fuera de allowed list, falla.

**Allowed values management:**
- Ver allowed values: `SYSTEM$GET_TAG_ALLOWED_VALUES()` function (requiere USAGE privilege en parent database/schema del tag o global APPLY TAG privilege)
- Modificar: `ALTER TAG ... ADD/REMOVE ALLOWED VALUES`

**Aplicar tag:**
- Existing object: `ALTER <object> SET TAG tag_name = 'value'`
- At creation: `CREATE <object> ... TAG (tag_name = 'value')`

## Tag Inheritance

**Child objects heredan tags** de parent objects. Ejemplo: database tagged como `operations` → schema y tables en ese database también inherited tag `operations`.

**Verificar tags:**
- **INFORMATION_SCHEMA.TAG_REFERENCES** table function: Sin latency, muestra direct tags Y inherited tags
- **ACCOUNT_USAGE.TAG_REFERENCES** view: Solo direct tags (no inherited), latency 45 min - 3 horas

**Column-level tags:** Pueden setear tags en cada column de table (necesario cuando combinando tags con masking policies para auto-apply a sensitive columns). Por tag inheritance, todas las columns heredan parent table tag, pero pueden tener additional explicit tags.

**Límite:** Hasta **50 unique tags** pueden setarse en single object.

## Governance Dashboard

**Acceso:** Monitoring → Governance. Proporciona snapshot del governance posture de data.

**Actualización:** Dashboard updates cada **12 horas**. Tags resultantes de tag lineage **no se muestran**. Dashboard solo muestra tags para **tables y columns**.

## Comandos y Características Adicionales

**Comandos disponibles:** `SHOW TAGS`, `DROP TAG`, `UNDROP TAG`.

**Database replication:** Tags y assignments se replican desde source account a target account.

**Object cloning:** Tag associations en source object (ej: table) están presentes en cloned object.
