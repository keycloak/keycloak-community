# Keycloak.X Server Configuration

* **Status**: Draft #1

## How to set configuration

Keycloak.X can be configured in a property file, using command-line arguments or environment variables. All configuration
properties are available in all different approaches.

Properties (conf/keycloak.properties):

    hostname.frontend-url=https://mykeycloak
    
Environment variables:

    export KC_HOSTNAME_FRONTEND_URL=https://mykeycloak
    bin/kc.sh

Command-line arguments:

    bin/kc.sh --hostname-frontend-url=https://mykeycloak
    
When evaluating configuration if a property is defined in several places the order of resolution is first in the list of:
command-line arguments, environment variables, properties.  

The format used in properties will be used throughout the documentation.

The keys used above follows a strict rewrite rule, where the key in properties is the name of the property. Property 
names should follow the following rules:

* Use hyphen `-` to combine multiple word names (`property-name`)
* Use dot (`.`) for separation
* All options must be in a category (`<category-name>.<property-name>`)
* Multiple levels of categories are permitted (`<category-name>.<sub-category-name>.<property-name>`)
* Avoid camelCase

Rules for converting property keys to different formats follow these rules:

* Environment variables: 
  - Prefix with `KC_`
  - Characters converted to uppercase
  - Dot (`.`) replaced with underscore (`_`)
  - Use underscore (`_`) for camel-case names (`PROPERTY_NAME`)
* Command-line arguments:
  - Dot (`.`) converted to hyphen (`-`)
  
## SPIs and Provider Configuration

SPIs and provider names currently use a mix between `-` and camelCase for multi-word names. We should introduce a 
convention here, and use `-`.

Properties for providers follow the following format:

    spi.<spi-name>.<provider-name>.<config-name>

For example:

    spi.hostname.default.frontend-url=https://mykeycloak
    
To override the default provider for a SPI use `spi.<spi-name>.provider=<provider-name>`.

To enable/disable a provider use `spi.<spi-name>.<provider-name>.enabled=true|false`.
    
## Alias

It should not be required to use SPI/provider specific property names for commonly configured properties for default
providers. Here we will support alias for properties.

For example `hostname.frontend-url` is an alias to `spi.hostname.default.frontend-url`.

## Quarkus options

We do not want to expose users to low-level Quarkus properties, and rather define our own Keycloak specific properties
when needed.

Depending on the use-case the strategy here will be one of:

* High-level property - for example `db.vendor=postgres` will behind the scenes set `quarkus.datasource.jdbc.driver` and
  `quarkus.hibernate-orm.dialect`
* Mapping/alias - for example `http.ssl.certificate.file` will map to `quarkus.http.ssl.certificate.file`

We will also allow setting Quarkus properties directly. This will not be recommended approach, but available as an option
to power-users.

## Static and dynamic properties

For some properties Quarkus optimises the build. This is especially relevant to native builds where code may be removed
completely depending on the configuration.

As such there is some properties that require a "re-build", while others properties that can be dynamically changed.
Example of a static property is the database vendor, while an example of a dynamic property is http port.

Documentation of properties should clearly mention what type a property is. Most likely have two separate sections for
static and dynamic properties.

In the JVM distribution we will support a quick "re-build" directly that allows updating static properties easily.

For example to update the database vendor and URL the following would be used:

```
kc.[sh|bat] config --database.vendor=mariadb
kc.[sh|bat] run --database.url=jdbc:mariadb://localhost:3306/kc
```

The default command to kc.[sh|bat] is `start` so that can be omitted if wanted:

```
kc.[sh|bat] --database.url=jdbc:mariadb://localhost:3306/kc
```

## Documenting Properties

We want all properties to be documented properly to make it easy for users to discover what can be configured. This 
documentation will be available through kc.[sh|bat] as well as on the Keycloak website.

To get general help run `kc.[sh|bat] --help`. This will output the list of available commands as well as mention how
to get the list of static and dynamic properties. Something like:

```
Usage: kc.sh COMMAND [OPTIONS]

Commands:
  run       Starts Keycloak
  config    Updates configuration that requires a rebuild of Keycloak
  export    Exports data in the database
  import    Imports data into the database     
  
Run 'kc.sh COMMAND --help' for more information on a command.

Keycloak has two types of configuration properties. Static properties need to be updated using the 'config' command, 
which will result in a "re-build", while dynamic properties can be applied directory to the 'run' command. Both types
of configuration can be added directly to `conf/keycloak.properties`, but if static properties are changed it will require
running `kc.sh config` to take affect. 
```

To list static configuration properties run `kc.[sh|bat] config --help`, which will output something like:

```
Usage: kc.sh config [OPTIONS]

Updates configuration that requires a rebuild of Keycloak

Options:
  --database-vendor        Set the database vendor, available options are h2, mariadb, mssql, mysql, oracle, postgres (default h2)  
```

To list dynamic configuration properties run `kc.[sh|bat] run --help`, which will output something like:

```
Usage: kc.sh config [OPTIONS]

Updates configuration that requires a rebuild of Keycloak

Options:
  --database-url                        Set the database JDBC URL (default depends on database-vendor which allows setting hostname, 
                                        database, etc. as individual properties)
  --hostname.frontend-url               Set the URL used by Keycloak for frontend requests (requests from user-agent) 

Provider Options:
  --spi.hostname.default.admin-url      Set the URL used by Keycloak for Admin Console and API
```

On keycloak.org we will add a page that lists all config properties, which is generated from the Keycloak code-base to
keep it up to date with the latest release.

For SPIs and Providers we will extend the ProviderFactory to add an option method to return information on what
properties it supports. This will allow including SPI/provider specific configuration in the documentation.

## Property replacement

When setting the value of a property it is possible to include other properties or environment variables in the value. 

Including a property:
```
spi.hostname.custom.url=${hostname.frontend-url}
```

Including environment variables:
```
spi.hostname.custom.url=${env.MY_KC_URL}
```

It is also possible to use alternatives, including a fallback:
```
spi.hostname.custom.url=${env.MY_KC_URL,hostname.frontend-url:localhost}
```

In the above example `env.MY_KC_URL` will be used if set, if not hostname.frontend-url will be used. If neither is set
it will fallback to `localhost`.

## Config Options

The aim is to start small with what config options we provide, and extend when there is a need. Great care should be 
taken in reducing the amount of configuration needed, where opinionated/best-practice behaviour should be favoured, with
high-level and simple configuration options introduced only when needed.

Below is a list of config by category we should introduce initially:

### Installation layout

* keycloak.home - Defaults to installation directory of Keycloak
* keycloak.config-file - Defaults to `${keycloak.config-dir}/keycloak.properties`
* keycloak.config-dir - Defaults to `${keycloak.home}/conf`
* keycloak.data-dir - Defaults to `${keycloak.home}/data`
* keycloak.log-dir - Defaults to `${keycloak.home}/log`
* keycloak.provider-dir - Defaults to `${keycloak.home}/providers`
* keycloak.theme-dir - Defaults to `${keycloak.home}/themes`

### Profile

* profile.dev - boolean - defaults to `false` 
* profile.preview - boolean - defaults to `false`

If `profile.dev` is enabled it will change the default configuration for `http.enabled` to `true`. We also plan to use
this to enable other secure by default configuration, for example only permitting `*` as a client redirect-uri if 
`profile.dev` is enabled. `profile.dev` will also enabled an embedded H2 database as the default database.

Enabling/disabling individual features are done with `profile.feature.<featureName>=true|false`

### HTTP

Keycloak will use a secure by default approach and require `http` to be explicitly enabled. `http` can also be enabled
by enabling `profile.devMode`, or with the `proxy.mode` option.

If `https` is enabled and no certificate is configured Keycloak will show a warning. It will not generate a self-signed
certificate like the old Keycloak distribution does.
 
* http.enabled - boolean - defaults to `false`  
* http.port - int - defaults to `8080`
* https.enabled - boolean - defaults to `true`
* https.port - int - defaults to `8443`
* https.certificate.file
* https.certificate.key-file
* https.certificate.key-store-file
* https.certificate.key-store-file-type
* https.certificate.key-store-password
* https.certificate.trust-store-file
* https.certificate.trust-store-file-type
* https.certificate.trust-store-password
* https.cipher-suites
* https.protocols
* https.client-auth

### Proxy Mode

The proxy mode will allow an easy way to configure if Keycloak is hosted behind a reverse-proxy or not, including how
SSL is terminated.

* proxy.mode - none|reencrypt|passthrough - default value is `none`

Setting `proxy.mode` will under the covers change the default values for `http.enabled`, `https.enabled` 
and `quarkus.http.proxy-address-forwarding`.

## Database

The aim is that a user should not need to understand Hibernate, JDBC drivers, etc. to be able to configure the database.
As such we should introduce a high-level option `db.vendor` instead of requiring users to set JDBC driver and Hibernate
dialects.

* database.vendor - h2|mariadb|mssql|mysql|oracle|postgres
* database.schema
* database.username
* database.password

* database.pool.initial-size
* database.pool.min-size - default to 0
* database.pool.max-size - default to 100

* database.url - JDBC URL

Setting the db.vendor will set a default JDBC URL with property expressions allowing setting parts of the URL with 
individual properties.
 
Below are the default JDBC URL values that will set if `database.url` has not been configured:

* h2 - `jdbc:h2:${database.url.path:${keycloak.data-dir}/db}` - use `database.url.path=mem:kc` for in-memory
* mariadb - `jdbc:mariadb://${database.url.host}/${database.url.database:keycloak}${database.url.properties}`
* mssql - `jdbc:sqlserver://${database.url.host};databaseName=${database.url.database}${database.url.properties}`
* mysql - `jdbc:mysql://${database.url.host}/${database.url.database}${database.url.properties}`
* oracle - `jdbc:oracle:thin:${database.url.host}:${database.url.database}${database.url.properties}`
* postgres - `jdbc:postgresql://${database.url.host}/${database.url.database}${database.url.properties}`

Note: `database.url.properties` must be prefixed with `?` (or `;` for mssql) (for example `database.url.properties=?key=value`)


## Caches (Infinispan and JGroups)

TBD

## Logging

TBD

## Metrics

TBD
