# Keycloak.X Server Configuration

* **Status**: Draft #2

## Principles

Keycloak.X should be configured through the different configuration options available.

Configuration options can be set using different formats: command-line arguments, environment variables, or a properties file. 

All configuration options are available in all different formats:

Command-line arguments:

    kc.[sh|bat] --<category>-<sub-category>-<property>=<value>
    
Environment variables:

    export KC_CATEGORY_SUB_CATEGORY_PROPERTY=<value>
    kc.[sh|bat]
    
Properties (at `conf/keycloak.properties`):

    <category>.<sub-category>.<property>=<value>
    
When evaluating configuration, if a property is defined in several places the order of resolution is first in the list of:

* command-line arguments
* environment variables
* properties

The aim is to start small with what config options we provide, and extend when there is a need. Great care should be 
taken in reducing the amount of configuration needed, where opinionated/best-practice behaviour should be favoured, with
high-level and simple configuration options introduced only when needed.

The format used in properties will be used throughout the documentation.

### Configuration Option Format

A configuration option follows a strict rewrite rule, where the key is the name of the property. 

Property names must follow these rules:

* Use hyphen `-` to combine multiple word names (e.g.: `category-name`, `sub-category-name`, `property-name`)
* Use dot (`.`) for separation
* All options must be in a category (`<category>.<property>`)
* Multiple levels of categories are permitted (`<category>.<sub-category>.<property>`)
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
  
* Properties
  - No `kc` prefix
  - Internally, they are always prefixed with `kc`.
  
### Quarkus Specific Properties

In some cases, Keycloak properties are basically mapping to a specific property or even to multiple properties in Quarkus.

As a rule, for any property in Quarkus that users need to change in order to change some behavior, we should have a correspondent property
in Keycloak when it makes sense. 

Users should not have to deal with low-level Quarkus properties.

Users should be able to still set any Quarkus property as they do for any Quarkus-based application. However, this is considered
an advanced usage and not supported configuration.

Depending on the use-case the strategy here will be one of:

* High-level property - for example `db=postgres` will behind the scenes set `quarkus.datasource.jdbc.driver` and
  `quarkus.hibernate-orm.dialect`
* Mapping/alias - for example `http.ssl.certificate.file` will map to `quarkus.http.ssl.certificate.file`
  
## Configuring the Server

Keycloak is based on Quarkus and as such it benefits from a lot of optimizations when building and configuring the server. By default, the server distribution provides a minimal server image with only the basic configuration to get it running.

```
# Default and non-production grade database vendor 
db=h2-file

# Default, and insecure, and non-production grade configuration for the development profile
%dev.http.enabled=true
%dev.db.username = sa
%dev.db.password = keycloak
%dev.cluster=local
```

As you can see from above, the default configuration uses defines a `h2-file` database as well as a `dev` profile so that you can
run the server more easily by just running:

    kc.[sh|bat] start-dev
    
Or when explicitly passing the `dev` profile:

    kc.[sh|bat] --profile=dev 

In order to change the configuration, you should either pass any of the available options when starting the server or run the 
`config` command prior to start the server so that any configuration option you set is persisted into the resulting server image. 

### `Config` Command

The `config` command is a very important command that allows the server to optimize and adapt itself based on new configuration options. This is especially relevant to avoid unnecessary steps when starting the server as well as reduce memory footprint where code may be removed completely depending on the choosen configuration.

When a configuration is set through the `config` command, it becomes persisted into the resulting server image and implicitly available when
starting the server. As a result, this new server image is optimized with the new configuration and you don't need to pass the same configuration properties when running the server.

The server image produced after executing the `config` command indicates the intention to define the base configuration that the server should consider when starting new instances.

Users should always prefer to configure the server through this command before starting server instances. The reason being that they
should expect a faster startup time and lower memory footprint due to optimizations performed during this step.

For more information about the different configuration options that can be set when running this command, users should be able to
see the help and usage messages as follows:

```
kc.[sh|bat] config --help
```

### Static Options

Some configuration options can only bet set when configuring the server and as such any attempt to change these properties must be done through the `config` command. These options are usually related to optimizations that can be done in order to provide an optimal runtime. 

Example of a static option is how you change the database vendor. 

For example, to use a PostgreSQL database you should execute the following command:

```
kc.[sh|bat] config --db=postgres # re-configure the server so that postgres is set as the default database
```

When running the `config` command any option that was set is going to be persisted so that any attempt
to start the server does not require to pass all the configuration properties again. They are already part of the new server image.

Any attempt to set a static option when starting the server should give an error to the user. Where static options should only be
available through the `config` command.

On the other hand, any other option that can be set when starting the server should also be available through the `config` command 
so that they can be persisted and used during optimizations when creating the new server image.

Static options should be properly documented and available when looking at the help and usage messages for the `start` command.

```
kc.[sh|bat] --help
```

### Dynamic Options

Dynamic options can be set anytime when starting the server without necessarily running the `config` command. An example of a 
dynamic option is `http.port`, which you can define as follows:

```
kc.[sh|bat] --http-port=8180 # dynamic property passed when starting the server, no need to re-configure the server
```

Alternatively, dynamic options can also be used when running the `config` command so that they are persisted and used 
to perform any optimization to the new server image. When doing this, you won't need to pass the same options again when starting the server:

```
kc.[sh|bat] config --http-port=8180 # dynamic option set when configuring the server
kc.[sh|bat] # just start the server
```

Dynamic options should be properly documented and available when looking at the help and usage messages for the `start` command.

```
kc.[sh|bat] --help
```

Users should always prefer to configure the server using the `config` command before starting the server. The reason being that they
should expect a faster startup time and lower memory footprint due to optimizations performed during this step.

### Property Replacement

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

### Configuration Profile

A set of configuration options can be grouped together using a profile.

The profile is set using the `profile` property. For instance:

```
kc.[sh|bat] --profile=dev
```

Profiles should also be used to enforce specific policies when running the server, for example not permitting `*` as a client redirect-uri if running in production.

### Development Profile

Keycloak.X uses a `dev` profile to indicate that the server is running in development mode.

This profile is a shortcut to get started and run the server locally for development and testing purposes. 

It is defined as follows:

```
# Default, and insecure, and non-production grade configuration for the development profile
%dev.http.enabled=true
%dev.db.username = sa
%dev.db.password = keycloak
%dev.cluster=local
```

Basically, it enables the HTTP listener so that you don't need to set up HTTPS, define credentials for the default H2 database, and
set the cluster configuration to use local caches.

Users can always define their own profiles using the `keycloak.properties` file. For that, properties should be
prefixed with `%` following the name of the profile.

### Configuration File

By default, Keycloak reads configuration from the `conf/keycloak.properties` file. Users should be able to change the config file
location by using the `--config-file` CLI option when running the `config` command:

```
kc.[sh|bat] config --config-file=/dir/my.properties
```

Or when starting the server:

```
kc.[sh|bat] --config-file=/dir/my.properties
```

## Configuration Categories

### Database

The aim is that a user should not need to understand Hibernate, JDBC drivers, etc. to be able to configure the database.

As such we should introduce a high-level option `db` instead of requiring users to set JDBC driver and Hibernate
dialects. Usually, database should be configured as follows:

```
kc.[sh|bat] config --db=postgres
```

The database vendor should be configured through the `config` command.

Where the available database vendors should be the following:

| Vendor                            | Description   | Default URL |
|:--------:                         |:-------------:|:--------|
|h2-mem                             | H2 In-Memory  | jdbc:h2:mem:keycloakdb${kc.db.url.properties:} |
|h2-file                            | H2 File       | jdbc:h2:file:${kc.home.dir:${kc.db.url.path:~}}/data/keycloakdb${kc.db.url.properties:;;AUTO_SERVER=TRUE}|
|postgres, postgres-95, postgres-10 | PostgreSQL    | jdbc:postgresql://${kc.db.url.host:localhost}/${kc.db.url.database}${kc.db.url.properties:} |
|mariadb                            | MariaDB       | jdbc:mariadb://${kc.db.url.host:localhost}/${kc.db.url.database:keycloak}${kc.db.url.properties:} |
|mysql                              | MySQL         | jdbc:mysql://${kc.db.url.host:localhost}/${kc.db.url.database:keycloak}${kc.db.url.properties:}"; |

#### JDBC URL

Setting the database vendor will set a default JDBC URL with property expressions allowing setting parts of the URL with 
individual properties:

* `kc.db.url.host`       - Change the host where the database is running
* `kc.db.url.database`   - Change the database name
* `kc.db.url.properties` - Set any database specific property

By default, the default JDBC URLs use `localhost` as the host where the database is running. Users should be able to change
the database host by using the `kc.db.url.host` system property as follows:

```
kc.[sh|bat] -Dkc.db.url.host=my.postgres.local # start the server using this host to connect to the database
```

The same goes for the other properties.

Note: `db.url.properties` must be prefixed with `?` (or `;` for mysql) (for example `db.url.properties=?key=value`). However, 
both the prefix and the character that separates multiple values depend on the database vendor you are using.

However, it should still be possible to provide a custom JDBC URL through the `db-url` option:

```
kc.[sh|bat] --db-url=jdbc:postgresql://my.postgres.local/keycloak
```

#### Available Options

| Name                            | Description   | Requires `config` |
|:--------                         |:-------------|:--------|
|db                             | The database vendor. Possible values are: h2-mem, h2-file, mariadb, mysql, postgres95, postgres10.  | true |
|db.url                            | The database JDBC URL. If not provided a default URL is set based on the selected database vendor. For instance, if using 'postgres', the JDBC URL would be 'jdbc:postgresql://localhost/keycloak'. The host, database and properties can be overridden by setting the following system properties, respectively: -Dkc.db.url.host, -Dkc.db.url.database, -Dkc.db.url.properties.       | false|
|db.username | The database username.    | false|
|db.password                            | The database password.       | false |
|db.schema                              | The database schema.         | false |
|db.pool.initial-size                              | The initial size of the connection pool.         | false |
|db.pool.min-size                              | The minimal size of the connection pool.         | false |
|db.pool.max-size                             | The maximum size of the connection pool.         | false |

### HTTP

Keycloak.X will use a secure by default approach and require HTTPS to be configure prior to running the server in production. It will not generate a self-signed certificate like the old Keycloak distribution does. 

However, it should be possible to start the server and still accept HTTP (insecure) requests by using the `http.enabled` property.

```
kc.[sh|bat] --http-enabled=true
```

#### Default HTTPS Configuration

In order to make easier to configure HTTPS, it should be possible to set up keys and certificates by just placing a `server.keystore` Java KeyStore file
in the `conf` directory:

```
keytool -genkeypair -storepass password -storetype PKCS12 -keyalg RSA -keysize 2048 -dname "CN=server" -alias server -ext "SAN:c=DNS:localhost,IP:127.0.0.1" -keystore conf/server.keystore
```

By default, the server will look for a `conf/server.keystore` file and try to fetch the keys and certificates from there to properly configure HTTPS. The default password to open the
keystore should be `password` but it is highly recommended using a stronger password if using a Java KeyStore to configure HTTPS:

```
kc.[sh|bat] --https-certificate-key-store-password=changeme # use a different password to access the key store
```

#### Context Root

Keycloak.X should use `/` as the context root path as follows:

```
http://localhost:8080
```

There is no need to keep the `/auth` context path currently used by the Wildfly distribution.

#### Available Options

| Name                            | Description   | Requires `config` |
|:--------                         |:-------------|:--------|
|http.enabled                             | Enables the HTTP listener.  | false |
|http.host                            | The HTTP host. | false|
|http.port | The HTTP port.    | false|
|https.port                            | The HTTPS port. | false |
|https.client-auth                              | Configures the server to require/request client authentication. none, request, required. | false |
|https.cipher-suites                              | The cipher suites to use. If none is given, a reasonable default is selected. | false |
|https.protocols                              | The list of protocols to explicitly enable.         | false |
|https.certificate.file                             | The file path to a server certificate or certificate chain in PEM format. | false |
|https.certificate.key-file                             | The file path to a private key in PEM format. | false |
|https.certificate.key-store-file                             | An optional key store which holds the certificate information instead of specifying separate files. | false |
|https.certificate.key-store-password                             | A parameter to specify the password of the key store file. If not given, the default ("password") is used. | false |
|https.certificate.key-store-file-type                             | An optional parameter to specify type of the key store file. If not given, the type is automatically detected based on the file name. | false |
|https.certificate.trust-store-file                             | An optional trust store which holds the certificate information of the certificates to trust. | false |
|https.certificate.trust-store-password                             | A parameter to specify the password of the trust store file. | false |
|https.certificate.trust-store-file-type                            | An optional parameter to specify type of the trust store file. If not given, the type is automatically detected based on the file name. | false |

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

### Cluster

Keycloak.X is still using Infinispan and JGroups for clustering and HA deployments.

However, the configuration should use Infinispan’s native configuration as opposed to using an abstraction as in the Wildfly Infinispan Subsystem. That should give much more flexibility in terms of configuration, support, as well as documentation.

XML Configuration files should be within the `conf` directory and the name should be prefixed by `cluster-` plus the name of the cluster configuration. For instance:

* cluster-default.xml
* cluster-local.xml

#### Default Configuration

By default, clustering is enabled and you are ready to build a Keycloak cluster using the default configuration.

The default configuration should be located at the `conf` directory, the file name is `cluster-default.xml`.

#### Local Cache

Users should also be able to run the server using local caches for development and testing purposes. For that a `cluster-local.xml` file should be provided 
to configure all caches as local, no clustering.

```
kc.[sh|bat] --cluster=local
```

#### Custom Configuration

Users may define their own cluster configuration by just creating a file in the `conf` directory with the `cluster-` prefix, just like `cluster-local.xml` and `cluster-default.xml` files.

The name used after the prefix should then be used to specify the value for the `--cluster` configuration option. For instance, for a 
`cluster-my-config.xml` file at the `conf` directory users would run either the `config` or `start` commands as follows:

```
kc.[sh|bat] --cluster=my-config
```

#### Cluster Stacks

We also provide some good defaults for specific platforms such as Kubernetes and EC2. For instance, to run a cluster in Kubernetes you could run the following command:

```
kc.[sh|bat] --cluster-stack=kubernetes
```

The default configuration for these platforms are based on the defaults provided by Infinispan. 

In the example above, the default configuration for Kubernetes is going to be based on UDP for node communication and DNS_PING for node discovery. 

Any parameter you can use to customize the default configuration (usually done through system properties) can be obtained from Infinispan documentation.

### SPIs and Provider Configuration

SPIs and provider names currently use a mix between `-` and camelCase for multi-word names. We should introduce a 
convention here, and use `-`.

Properties for providers follow the following format:

    spi.<spi>.<provider>.<property>

For example:

    spi.myspi.myprovider.some-property=value
    
Whenever makes sense, we should avoid forcing users to use this format if a provider configuration is widely and commonly used by
users so that we provide a more specific and reduced property name. 

For instance, the `spi-hostname-default-frontend-url` property is a very common configuration when deploying in production and to set it
users would just use a shorthand `hostname-frontend-url` property.
    
#### Enable/Disable Provider

To enable/disable a provider use `spi.<spi>.<provider>.enabled=true|false`.

By default, only the main providers are enabled and available when building the distribution.
 
In order to disable a provider, it is necessary to run the `config`. The same goes when enabling a provider. For instance:

    kc.[sh|bat] config --spi-myspi-myprovider-enabled=true|false
    
By disabling unwanted providers when configuring the server, users should be able to get a smaller footprint and faster startup time given that
the provider won't be loaded and initialized when the server starts.

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

### Health and metrics

Enabling/disabling health and metrics information is done with `metrics.enabled=true|false`.

When enabled, the application will expose the following endpoints:

* `/health`
* `/health/live`
* `/health/ready`
* `/metrics`

This feature is disabled by default.

## Commands

Commands should represent common operations that users should be doing when configuring, starting or managing server instances.

Great care should be taken in reducing the number of available actions to keep the user experience simple with meaningful commands.

As a rule, command names should follow these rules:

* Verb
* If multi-word, separate words by `-`

Examples of commands are:

* `config`
* `start`
* `import`
* `export`
* `show-config`

Commands may support additional options to change how they are executed and behavior. For that, commands should provide the
help and usage messages to document how they can be used as well as the different options they support. 

## Documentation

All options and commands should be documented properly to make it easy for users to discover what can be configured and which actions can be performed. This 
documentation will be available through `kc.[sh|bat]` as well as on the Keycloak website.

### Using Help Command

To get general help run `kc.[sh|bat] --help`. This will output the list of available commands as well as mention how
to get the list of static and dynamic options. Something like:

```
Keycloak - Open Source Identity and Access Management

Find more information at: https://www.keycloak.org/

Usage: kc.sh [--help] [--version] [--config-file=<path>] [--profile=<profile>] [COMMAND]
Use this command-line tool to manage your Keycloak cluster

Configuration Options

  --config-file=<path>   Set the path to a configuration file.
  --help                 This help message.
  --profile=<profile>    Set the profile. Use 'dev' profile to enable development mode.
  --version              Show version information

Commands

  config       Creates a new server image based on the options passed to this command. Once created, configuration will be read from the server image
                 and the server can be started without passing the same options again. Some configuration options require this command to be executed
                 in order to actually change a configuration. For instance, the database vendor.

  export       Export data from realms to a file or directory.

  import       Import data from a directory or a file.

  show-config  Print out the current configuration.

  start        Start the server.

  start-dev    Start the server in development mode.


Use "kc.sh <command> --help" for more information about a command.
Use "kc.sh options" for a list of all command-line options.
by Red Hat
```

To list static configuration options run `kc.[sh|bat] config --help`, which will output something like:

```
Usage: kc.sh config [OPTIONS]

Updates configuration that requires a rebuild of Keycloak

Options:
  --db        Set the database type, available options are h2-file, h2-mem, mariadb, mssql, mysql, oracle, postgres. Default is h2-file.  
```

To list dynamic configuration options run `kc.[sh|bat] start --help`, which will output something like:

```
Usage: kc.sh start [OPTIONS]

Starts Keycloak

  --db-url                            Set the database JDBC URL (default depends on database-vendor which allows setting hostname, 
                                      database, etc. as individual properties)
  --hostname-frontend-url             Set the URL used by Keycloak for frontend requests (requests from user-agent) 
```

### Documentation from Website

On keycloak.org we will add a page that lists all config properties, which is generated from the Keycloak code-base to
keep it up to date with the latest release.

### SPI and Provider Configuration Options  

For SPIs and Providers we will extend the ProviderFactory to add an option method to return information on what
properties it supports. This will allow including SPI/provider specific configuration in the documentation.

## System properties defined by Keycloak

The following system properties are defined by Keycloak, and can be used for property replacement.

* kc.home - Defaults to installation directory of Keycloak
* kc.config-file - Defaults to `${kc.config-dir}/keycloak.properties`