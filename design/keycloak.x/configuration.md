# Keycloak.X Server Configuration

* **Status**: Draft #2

## How To Set Configuration

Keycloak.X can be configured using command-line arguments, environment variables, or a property file. All configuration
properties are available in all different approaches.

Command-line arguments:

    kc.[sh|bat] --hostname-frontend-url=https://mykeycloak
    
Environment variables:

    export KC_HOSTNAME_FRONTEND_URL=https://mykeycloak
    kc.[sh|bat]
    
Properties (conf/keycloak.properties):

    hostname.frontend-url=https://mykeycloak
    
When evaluating configuration if a property, is defined in several places the order of resolution is first in the list of:
command-line arguments, environment variables, and properties.

The format used in properties will be used throughout the documentation.

The keys used above follows a strict rewrite rule, where the key in properties is the name of the property. Property 
names should follow the following rules:

* Use hyphen `-` to combine multiple word names (`property-name`)
* Use dot (`.`) for separation
* All options must be in a category (`<category-name>.<property-name>`)
* Multiple levels of categories are permitted (`<category-name>.<sub-category-name>.<property-name>`)
* Use only lowercase characters
* Avoid camel-case

Rules for converting property keys to different formats follow these rules:

* Command-line arguments:
  - Dot (`.`) converted to hyphen (`-`)

* Environment variables: 
  - Prefix with `KC_`
  - Characters converted to uppercase
  - Dot (`.`) replaced with underscore (`_`)
  - Use underscore (`_`) for camel-case names (`PROPERTY_NAME`)
  
## SPIs and Provider Configuration

SPIs and provider names currently use a mix between `-` and camelCase for multi-word names. We should introduce a 
convention here, and use `-`.

Properties for providers follow the following format:

    spi.<spi-name>.<provider-name>.<config-name>

For example:

    spi.myspi.myprovider.some-property=https://mykeycloak
    
### Enable/Disable Provider

To enable/disable a provider use `spi.<spi-name>.<provider-name>.enabled=true|false`.

By default, only the providers we find necessary are marked as enabled when building the distribution are available. 
In order to disable a provider, it is necessary to run the `config` command so that the provider is excluded from the runtime. 
The same goes when enabling a provider that was previously disabled. For instance:

    kc.[sh|bat] --spi-myspi-myprovider-enabled=false config
    
By disabling unwanted providers when configuring the server, users should be able to get a smaller footprint and faster startup time.

## Quarkus Options

We do not want to expose users to low-level Quarkus properties, and rather define our own Keycloak specific properties
when needed.

Depending on the use-case the strategy here will be one of:

* High-level property - for example `db=postgres` will behind the scenes set `quarkus.datasource.jdbc.driver` and
  `quarkus.hibernate-orm.dialect`
* Mapping/alias - for example `http.ssl.certificate.file` will map to `quarkus.http.ssl.certificate.file`

We will also allow setting Quarkus properties directly. This will not be recommended approach, but available as an option
to power-users.

## Configuring the Server

Keycloak is based on Quarkus and as such it benefits from a lot of optimizations when building the server. By default, the server distribution
provides a minimal server image with only the basic configuration to get it running.

```
# Default and non-production grade database vendor 
db=h2-file

# Default, and insecure, and non-production grade configuration for the development profile
%dev.http.enabled=true
%dev.db.username = sa
%dev.db.password = keycloak
```

As you can see from above, the default server image is built using the `h2-file` database as well as defining a `dev` profile so that you can
run the server more easily by just running:

    kc.[sh|bat] --profile=dev

In order to change the options that were previously defined for a server image, you should then run the `config` command.

### `Config` Command

The `config` command is a very important command that allows the server to optimize itself based on new configuration properties. This is especially relevant to avoid unnecessary steps when starting the server as well as reduce memory footprint where code may be removed completely depending on the configuration. 

In a nutshell, what the `config` command does is to persist the configuration properties that were set when running the command into a new server image. As a result, this new server image is optimized with the new configuration and you don't need to pass the same configuration properties when running the server.

The server image produced after executing the `config` command indicates the intention to define the base configuration that the server should consider when
starting new instances. 

### Static Properties

Some configuration properties can only bet set when configuring the server (build time configuration) and as such any attempt to change these properties must be
done through the `config` command. Example of a static property is changing the db type (--db=postgres for example). 

In the JVM distribution we will support a quick "re-build" directly that allows updating static properties easily.

For example to update the database vendor and URL the following would be used:

```
kc.[sh|bat] --db=mariadb config # re-configure the server so that mariadb is set as the default database
kc.[sh|bat] --db.url=jdbc:mariadb://localhost:3306/kc # start the server passing the desired JDBC URL
```

It is worth mention that when running the `config` command any property that was set is going to be persisted so that any attempt
to start the server does not require to pass all the configuration properties again. They are already part of the new server image.

Any attempt to override a configuration that was previous set when running the `config` command is going to be ignored.

### Dynamic Properties

Dynamic properties can be set without necessarily running the `config` command but when starting the server. An example of a dynamic property is http port,
which you can define as follows:

    kc.[sh|bat] --http-port=8180 # dynamic property passed when starting the server, no need to re-configure the server

However, if the dynamic property is already set to the server image after running the `config` command, any attempt to override its
value will be ignored:

    kc.[sh|bat] --http-port=8080 config # creates a new server image setting the HTTP port to 8080
    kc.[sh|bat] --http-port=8180        # it won't change the port to 8180 because 8080 is persisted into the server image

### Documenting Properties

We want all properties to be documented properly to make it easy for users to discover what can be configured. This 
documentation will be available through kc.[sh|bat] as well as on the Keycloak website.

To get general help run `kc.[sh|bat] --help`. This will output the list of available commands as well as mention how
to get the list of static and dynamic properties. Something like:

```
Keycloak - Open Source Identity and Access Management

Find more information at: https://www.keycloak.org/

Usage: kc.sh [--config-file=<path>] [--profile=<profile>] [system properties...] [COMMAND]
Use this command-line tool to manage your Keycloak cluster

Additional Options

      [system properties...] Any Java system property you want set

Configuration Options

      --config-file=<path>   Set the path to a configuration file
      --profile=<profile>    Set the profile. Use 'dev' profile to enable development mode

Commands

  config       Update the server configuration
  options      Display the list of all command-line options
  show-config  Print out the current configuration
  start        Start the server
  start-dev    Start the server with the dev profile

Use "kc.sh <command> --help" for more information about a command.
```

Documentation of properties should clearly mention what type a property is. Most likely have two separate sections for
static and dynamic properties.

To list static configuration properties run `kc.[sh|bat] config --help`, which will output something like:

```
Usage: kc.sh config [OPTIONS]

Updates configuration that requires a rebuild of Keycloak

Options:
  --db        Set the database type, available options are h2-file, h2-mem, mariadb, mssql, mysql, oracle, postgres. Default is h2-file.  
```

To list dynamic configuration properties run `kc.[sh|bat] start --help`, which will output something like:

```
Usage: kc.sh start [OPTIONS]

Starts Keycloak

  --db-url                            Set the database JDBC URL (default depends on database-vendor which allows setting hostname, 
                                      database, etc. as individual properties)
  --hostname-frontend-url             Set the URL used by Keycloak for frontend requests (requests from user-agent) 

Provider Options:
  --spi.hostname.default.admin-url    Set the URL used by Keycloak for Admin Console and API
```

On keycloak.org we will add a page that lists all config properties, which is generated from the Keycloak code-base to
keep it up to date with the latest release.

For SPIs and Providers we will extend the ProviderFactory to add an option method to return information on what
properties it supports. This will allow including SPI/provider specific configuration in the documentation.

### Property replacement

When setting the value of a property it is possible to include system properties or environment variables in the value. This approach
is especially useful when configuration properties defined in `keycloak.properties` which values can be overridden using the CLI when starting
or configuring the server. 

Including a system property:
```
spi.hostname.default.frontend-url=${myKcUrl}
```

For example this would be used with: 
```
kc.sh -DmyKcUrl=https://mykeycloak.com # the server's frontend url will be set to https://mykeycloak.com
```

Note: Do not use system properties with the prefix `kc.` as these are reserved for future use.

Including environment variables:
```
spi.hostname.custom.url=${env.MY_KC_URL}
```

For example this would be used with: 
```
export MY_KC_URL=https://mykeycloak.com && kc.sh
```

It is also possible to use alternatives, including a fallback:
```
spi.hostname.default.frontend-url=${env.MY_KC_URL,myKcUrl:localhost}
```

In the above example `env.MY_KC_URL` will be used if set, if not `myKcUrl` will be used. If neither is set
it will fallback to `localhost`.

## Config Options

The aim is to start small with what config options we provide, and extend when there is a need. Great care should be 
taken in reducing the amount of configuration needed, where opinionated/best-practice behaviour should be favoured, with
high-level and simple configuration options introduced only when needed.

Below is a list of config by category we should introduce initially:

### Profile

A profile is selected by the `profile` property. For instance:

```
kc.[sh|bat] --profile=dev
```

By default, a `dev` profile is provided so that it will change the default configuration for `http.enabled` to `true` as well as set the database related properties to use a `h2-file` database. We also plan to use this to enable other secure by default configuration, for example only permitting `*` as a client redirect-uri if `profile=dev` is enabled.

Users can always define their own profiles by defining properties in `keycloak.properties` and prefix these properties with the profile they are related to. The example below shows how the `dev` profile is defined, by default:

```
# Default, and insecure, and non-production grade configuration for the development profile
%dev.http.enabled=true
%dev.db.username = sa
%dev.db.password = keycloak
```

### Database

The aim is that a user should not need to understand Hibernate, JDBC drivers, etc. to be able to configure the database.
As such we should introduce a high-level option `db` instead of requiring users to set JDBC driver and Hibernate
dialects.

* db - h2-file|h2-mem|mariadb|mysql|postgres-95|postgres-10
* db.schema
* db.username
* db.password

* db.pool.initial-size
* db.pool.min-size - default to 0
* db.pool.max-size - default to 100

* db.url - JDBC URL

Setting the db.vendor will set a default JDBC URL with property expressions allowing setting parts of the URL with 
individual properties.
 
Below are the default JDBC URL values that will set if `db.url` has not been configured:

* h2-file - `jdbc:h2:file:${kc.home.dir:${kc.db.url.path:~}}/data/keycloakdb${kc.db.url.properties:;;AUTO_SERVER=TRUE}`
* h2-mem  - `jdbc:h2:mem:keycloakdb${kc.db.url.properties:}`
* mariadb - `jdbc:mariadb://${kc.db.url.host:localhost}/${kc.db.url.database:keycloak}${kc.db.url.properties:}`
* mysql   - `jdbc:mysql://${kc.db.url.host:localhost}/${kc.db.url.database:keycloak}${kc.db.url.properties:}";`
* postgres-95|postgres-10 - `jdbc:postgresql://${kc.db.url.host:localhost}/${kc.db.url.database}${kc.db.url.properties:}`

Note: `db.url.properties` must be prefixed with `?` (or `;` for mssql) (for example `db.url.properties=?key=value`). However, 
both the prefix and the character that separates multiple values depend on the database vendor you are using.

### HTTP

Keycloak will use a secure by default approach and require `http` to be explicitly enabled. `http` can also be enabled
by using the dev profile, or with the `proxy.mode` option.

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

* proxy.mode - none|edge|reencrypt|passthrough

Setting `proxy.mode` will under the covers change the default values for `http.enabled`, `https.enabled` 
and `quarkus.http.proxy-address-forwarding` depending on the selected mode.

The `none` value disables proxy support and is the default value.

The `edge` value automatically sets `http.enabled=true` and `http.proxy-address-forwarding=true`. This mode is suitable for deployments
with a highly secure internal network where the reverse proxy keeps a secure connection (HTTP over TLS) with clients while communicating with Keycloak using HTTP.

The `reencrypt` value automatically sets `http.proxy-address-forwarding=true` and require the server to be configured with its own pair of keys and certificates so that the HTTPS listener can be properly set. This mode is suitable for deployments where internal communication between the reverse proxy and Keycloak should also be protected where different keys and certificates can be used on the reverse proxy as well as on Keycloak.

The `passthrough` value automatically sets `http.proxy-address-forwarding=true`. This mode is suitable for deployments where the
reverse proxy is only forwarding the requests to the Keycloak server so that secure connections between the server and clients are based on the keys and certificates used by the Keycloak server itself.
 
### Features

Enabling/disabling individual features are done with `features.<featureName>=enabled|disabled`.

Individual features have the following different support levels:

* supported
* deprecated
* preview
* experimental

By default only supported features are enabled. In addition there are some supported features that are not enabled out of the box.

To enable non-supported features you can either enable individual features or change the feature level with `--features=<level>`. 
For example `--features=preview` would enable all preview and deprecated features.

## Show Configuration

Keycloak can read configuration properties from multiple sources (environment variables, CLI, and property files). In addition to 
that, the server's configuration can also be persisted when running the `config` command.

## System properties defined by Keycloak

The following system properties are defined by Keycloak, and can be used for property replacement.

* kc.home - Defaults to installation directory of Keycloak
* kc.config-file - Defaults to `${kc.config-dir}/keycloak.properties`
* kc.config-dir - Defaults to `${kc.home}/conf`
* kc.data-dir - Defaults to `${kc.home}/data`
* kc.log-dir - Defaults to `${kc.home}/log`
* kc.provider-dir - Defaults to `${kc.home}/providers`
* kc.theme-dir - Defaults to `${kc.home}/themes`

## Caches (Infinispan and JGroups)

TBD

## Logging

TBD

## Metrics

TBD
