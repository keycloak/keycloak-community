# Keycloak.X Server Configuration

* **Status**: Draft #1

## How to set configuration

Keycloak.X can be configured in a property file, using command-line arguments or environment variables. All configuration
properties are available in all different approaches.

Properties (conf/keycloak.properties):

    hostname.frontendUrl=https://mykeycloak
    
Environment variables:

    export KC_HOSTNAME_FRONTEND-URL=https://mykeycloak
    bin/kc.sh

Command-line arguments:

    bin/kc.sh --hostname-frontendUrl=https://mykeycloak
    
When evaluating configuration if a property is defined in several places the order of resolution is first in the list of:
command-line arguments, environment variables, properties.  

The format used in properties will be used throughout the documentation.

The keys used above follows a strict rewrite rule, where the key in properties is the name of the property. Property 
names should follow the following rules:

* Use camel-case (`propertyName`)
* Use dot (`.`) for separation
* All options must be in a category (`<categoryName>.<propertyName>`)
* Multiple levels of categories are permitted (`<categoryName>.<subCategoryName>.<propertyName>`)

Rules for converting property keys to different formats follow these rules:

* Environment variables: 
  - Prefix with `KC_`
  - Characters converted to uppercase
  - Dot (`.`) replaced with underscore (`_`)
  - Use hyphen (`-`) for camel-case names (`PROPERTY-NAME`)
* Command-line arguments:
  - Dot (`.`) converted to hyphen (`-`)
  
## SPIs and Provider Configuration

Properties for providers follow the following format:

    spi.<spi-name>.<provider-name>.<config-name>

For example:

    spi.hostname.default.frontendUrl=https://mykeycloak
    
## Alias

It should not be required to use SPI/provider specific property names for commonly configured properties for default
providers. Here we will support alias for properties.

For example `hostname.frontendUrl` is an alias to `spi.hostname.default.frontendUrl`.

## Quarkus options

We do not want to expose users to low-level Quarkus properties, and rather define our own Keycloak specific properties
when needed.

Depending on the use-case the strategy here will be one of:

* High-level property - for example `db.vendor=oracle12` will behind the scenes set `quarkus.datasource.jdbc.driver` and
  `quarkus.hibernate-orm.dialect`
* Mapping/alias - for example `http.ssl.certificate.file` will map to `quarkus.http.ssl.certificate.file`

We will also allow setting Quarkus properties directly. This will not be recommended approach, but available as an option
to power-users.

## Documenting Properties

We want to document all all properties that a user can set (this does not include the low-level Quarkus properties). We
would like to achieve something similar to https://quarkus.io/guides/all-config.

In the documentation the property key used in the properties file should be used. It should include a description, the 
type and the default value.

We would also like the ability to list properties on the command-line by running `bin/kc.sh --list-config-options`.

For SPIs and Providers we should extend the ProviderFactory to add an option method to return information on what
properties it supports.

## Config Options

The aim is to start small with what config options we provide, and extend when there is a need. Great care should be 
taken in reducing the amount of configuration needed, where opinionated/best-practice behaviour should be favoured, with
high-level and simple configuration options introduced only when needed.

Below is a list of config by category we should introduce initially:

### HTTP

Keycloak.X will not come with the capability to generate a self-signed certificate. Instead, Keycloak by default will
complain if a certificate has not been configured, but will allow explicitly disabling SSL for development purposes.

* http.mode - Options `ssl` (default), `unsecure`  
* http.port
* http.ssl-port
* http.ssl.certificate.file
* http.ssl.certificate.key-file
* http.ssl.certificate.key-store-file
* http.ssl.certificate.key-store-file-type
* http.ssl.certificate.key-store-password
* http.ssl.certificate.trust-store-file
* http.ssl.certificate.trust-store-file-type
* http.ssl.certificate.trust-store-password
* http.ssl.cipher-suites
* http.ssl.protocols
* http.ssl.client-auth

### Proxy Mode

The proxy mode will allow an easy way to configure if Keycloak is hosted behind a reverse-proxy or not, including how
SSL is terminated.

* proxy.mode - Options are `none` (default), `reencrypt`, `edge` and `passthrough`.

Setting `proxy.mode` will under the covers configure `http.mode` as well as `quarkus.http.proxy-address-forwarding`.

## Database

The aim is that a user should not need to understand Hibernate, JDBC drivers, etc. to be able to configure the database.
As such we should introduce a high-level option `db.vendor` instead of requiring users to set JDBC driver and Hibernate
dialects.

We will support either `db.url` which will support a JDBC URL, or using separate properties to build-up the URL behind the
scenes.

* db.vendor
* db.url 
* db.hostname
* db.port - defaults to the default for the configured vendor 
* db.schema
* db.database
* db.username
* db.password
* db.pool.initialSize
* db.pool.minSize - default to 0
* db.pool.maxSize - default to 100

## Caches (Infinispan and JGroups)

As we aim to not use an embedded Infinispan server in the future, configuration of caches will not for now follow the
standard way of configuring Keycloak. Instead there will be a separate XML file for configuring this (`conf/cluster.xml`).

## Logging

TBD

## Metrics

TBD