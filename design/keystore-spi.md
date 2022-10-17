# Keystore SPI
* **Status**: Draft #1
* **GitHub issue**: https://github.com/keycloak/keycloak/pull/7365

## Motivation

This document proposes a new SPI called "KeyStore".
The purpose of the SPI is to allow an arbitrary number of key stores to be provisioned.
It also proposes a unified API for handling both credentials and trust anchors, allowing provisioning to be done using both key store files (PKCS#12 or JKS) and PEM files.
Intended consumers for the API are the various TLS clients and servers in Keycloak.

Currently, the configuration of key stores depends on the TLS client/server implementation and there are a couple of alternatives.
The JDK default KeyStore is used for clients that rely on the default SSL socket factory.
HTTPS server uses credentials set using Quarkus configuration (mapped from Keycloak configuration).

For trust stores, Keycloak currently implements TrustStore SPI.
It allows users to provide a single additional TrustStore (besides the JDK default), which is supported in specific use cases (see table).
Since TrustStore SPI provides only one trust store the same set of CA certificates will be shared for all uses.
For example, it is not possible to differentiate between the CA certificates used to validate the LDAP server certificate and the CA certificate that X.509 client certificate authenticator uses to validate client certificates, even though these two might be completely different PKI domains.

Table: Current TrustStore SPI users

| User | Socket factory | TrustStore or CA certificates | Validation policy | Usage description |
|---|---|---|---|---|
| `LDAPContextManager` | &check; | | | SSL context for LDAPS and StartTLS |
| `LDAPOperationManager`  | &check; | | | SSL context for LDAPS and StartTLS |
| `LdapMapContextManager` | &check; | | | SSL context for LDAPS and StartTLS |
| `LdapMapOperationManager`  | &check; | | | SSL context for LDAPS and StartTLS |
| `CertificateValidator` |  | &check; | | X.509 client certificate authenticator |
| `WebAuthnRegisterFactory` |  | &check; | | WebAuthn attestation |
| `DefaultHttpClientFactory` |  | &check; | &check; | TrustStore for HTTP client, server certificate validation policy |
| `NginxProxySslClientCertificateLookup`|  | &check; | | X.509 client certificate validation |
| `CRLUtils` |  | &check; | | X.509 client certificate authenticator |
| `DefaultEmailSenderProvider` | &check; | | &check; | SSL context for SMTPS and StartTLS |

## Use cases

KeyStore SPI shall support the following use cases:

1. The administrator can configure several individual key stores.
2. KeyStore SPI consumer requests KeyStore using an identifier.
The identifier is used to select one of the configured key stores.
3. The administrator can provision both credentials and trust anchors with KeyStore SPI.
4. The administrator can configure key stores in PKCS#12 and JKS format, as well as directly as PEM files.
5. The administrator can renew TLS client and server credentials by swapping the file(s) on disk, without requiring a server restart.

## Design proposal

The API proposal for KeyStore SPI is following

```java
public interface KeyStoreProvider extends Provider {

    /**
     * Load KeyStore of a given identifier.
     *
     * @param keyStoreIdentifier Identifier of the wanted KeyStore.
     * @return KeyStore.
     */
    KeyStore loadKeyStore(String keyStoreIdentifier);

    /**
     * Loads KeyStore of given identifier and returns a KeyStore.Builder.
     * Builder encapsulates both KeyStore and KeyEntry password(s).
     *
     * @param keyStoreIdentifier Identifier of the wanted KeyStore.
     * @return Builder for KeyStore.
     */

    KeyStore.Builder loadKeyStoreBuilder(String keyStoreIdentifier);

}
```

The identifier is a string that is used to select the correct KeyStore from the configured key stores.
The method `loadKeyStore()` returns `KeyStore` which can be used with `TrustManager` since no password is required.
The method `loadKeyStoreBuilder()` returns [`KeyStore.Builder`](https://docs.oracle.com/javase/8/docs/api/java/security/KeyStore.Builder.html) which can be used with `KeyManager`, since the builder encapsulates both `KeyStore` and `KeyEntry` passwords as well, which are required to access the private keys.

The key store identifier is a hierarchical name that identifies the key store.
For example, the following names could be used:

| Key store identifier | Description |
|---|---|
| `ldap-client-keystore` | Key store that includes client certificate, private key and chain certificates for LDAP client |
| `ldap-client-truststore` | Key store that includes CA certificates for LDAP client |

The configuration options that are exposed to the administrator could be constructed from the above identifier in the following way:

| Configuration option | Description |
|---|---|
| `spi-keystore-default-ldap-client-keystore-certificate-file` | Certificate for LDAP client as PEM file |
| `spi-keystore-default-ldap-client-keystore-certificate-key-file` | Private key for LDAP client as PEM file |
| `spi-keystore-default-ldap-client-keystore-keystore-file` | Client credentials for LDAP client as PKCS#12 or JKS file |
| `spi-keystore-default-ldap-client-keystore-keystore-password` | Password for the key store file |
| `spi-keystore-default-ldap-client-keystore-keystore-type` | Key store type: JKS (default) or PKCS12 |
| `spi-keystore-default-ldap-client-truststore-certificate-file` | Trusted CA certificate(s) for LDAP client as PEM file |
| `spi-keystore-default-ldap-client-truststore-keystore-file` | Trusted CA certificate(s) for LDAP client as PKCS#12 or JKS file |
| `spi-keystore-default-ldap-client-truststore-keystore-type` | Key store type: JKS (default) or PKCS12 |

Alternatively, the configuration options could be mapped to shorthand options.

## Implementation details

### Support for hot-reloading client and server credentials

Currently, Keycloak needs to be manually restarted for example when HTTPS server credentials are renewed.
Why is this a problem?
Let's Encrypt recommends renewing certificates [every 60 days](https://letsencrypt.org/docs/faq/#what-is-the-lifetime-for-let-s-encrypt-certificates-for-how-long-are-they-valid).
A lot shorter renewal period may be used in certain deployments as internal "private" PKI certificates, even down to hours or minutes.
It becomes inconvenient to restart Keycloak every time certificate is rotated.
For critical and highly available services it would be desirable to support automatic certificate reload after rotation.

The default KeyStore SPI implementation shall implement hot-reload support.
JDK `KeyStore` internally provides [`KeyStoreSpi`](https://docs.oracle.com/javase/8/docs/api/java/security/KeyStoreSpi.html) which is meant to be used to build custom KeyStores.
Default KeyStore SPI shall provide a custom implementation of `KeyStoreSpi` that reloads the KeyStore from disk when the files have been changed.

### Support for PEM files

The default KeyStore SPI implementation shall support loading KeyStore from PEM files.
This allows aligning the configuration of different TLS interfaces that Keycloak has.
Currently, PEM files are supported for the HTTPS credentials only.
Internally the PEM files can be loaded into `KeyStore`, allowing consumers to remain unaware of the format on disk.

## Milestones

1. Implement KeyStore SPI and default implementation.
2. Finalize [LDAP SASL EXTERNAL PR](https://github.com/keycloak/keycloak/pull/7365) which initially triggered the work on KeyStore SPI.
Utilize KeyStore SPI for loading client credentials for SASL EXTERNAL.
This allows Keycloak to hot-reload client credentials for the LDAP federation.
3. Migrate HTTPS server to use KeyStore SPI.
This allows Keycloak to hot-reload server credentials.
Neither Quarkus or Vert.X support hot-reload but Keycloak can pass `KeyManager` (enabled by [Quarkus PR](https://github.com/quarkusio/quarkus/pull/27682)) which can be initialized with `KeyStore` from the default KeyStore SPI implementation.

Further considerations:

It is not necessary to migrate existing TrustStore SPI consumers to KeyStore SPI even if trust anchors can be provisioned using KeyStore SPI.
However, it can be beneficial in the future for a unified API and implementation to load both credentials and trust anchors.
It allows having a single place in code to handle key store files, regardless of whether they contain credentials or trust anchors.
