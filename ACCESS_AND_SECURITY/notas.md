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
