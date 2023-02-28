# Keystore SPI
* **Status**: Draft #1
* **GitHub issue**: https://github.com/keycloak/keycloak/pull/7365

## Motivation

This document proposes a new SPI called "KeyStore".
Intended consumers for the API are the various TLS clients and servers in Keycloak.

The purpose of the SPI is to allow explicitly setting specific credentials and trust anchors for given use cases and realms.
Currently, there are a couple of alternatives for the configuration:


The JDK default key/trust store is used in use cases that utilize the Java default SSL socket factory.
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

KeyStore SPI shall allow implementation following use cases:

1. The administrator can separately configure credentials and trust anchors for each TLS client/server implemented by Keycloak.
2. The administrator can separately configure credentials and trust anchors for each realm.
3. The administrator can configure credentials and trust anchors in PKCS#12 and JKS format, as well as directly as PEM files.
4. The administrator can renew TLS client and server credentials by swapping the file(s) on disk, without requiring a server restart.

## Design proposal

The API proposal for KeyStore SPI is following

```java
public interface KeyStoreProvider extends Provider {

    /**
     * Creates KeyStore builder for given PKCS#12 or JKS file.
     *
     * @param type KeyStore type such as "PKCS12" or "JKS".
     * @param path Path to the keystore file.
     * @param password Password used to decrypt the KeyStore.
     * @return KeyStore builder.
     */
    KeyStore.Builder fromKeyStoreFile(String type, String path, String password);

    /**
     * Creates KeyStore builder for given certificate and private key files.
     *
     * @param certs List of paths to certificates.
     * @param keys List of keys to private keys.
     * @return KeyStore builder.
     */
    KeyStore.Builder fromPem(List<String> certs, List<String> keys);

    /**
     * Creates KeyStore builder for certificate path(s).
     *
     * A key store with only certificates is used as trust store.
     *
     * @param cert Path to certificate.
     * @return KeyStore builder.
     */
    KeyStore.Builder fromPem(String... cert);
}
```

The certificate and key files must be accessible as files on the filesystem.
The methods return [`java.security.KeyStore.Builder`](https://docs.oracle.com/javase/8/docs/api/java/security/KeyStore.Builder.html) which encapsulates both KeyStore and KeyEntry password(s).
`KeyStore.Builder` can be used with `KeyManagerFactory` and `TrustManagerFactory`.

```java
// Initialize KeyManagerFactory with PKCS#12 file.
keyManagerFactory.init(
    new KeyStoreBuilderParameters(
        keyStoreSpi.fromKeyStoreFile("PKCS12", "credentials.p12", "password")));

// or with PEM files.
keyManagerFactory.init(
    new KeyStoreBuilderParameters(
        keyStoreSpi.fromPem(Arrays.asList("cert.pem"), Arrays.asList("key.pem"))));

// Initialize TrustManagerFactory with PEM file.
trustManagerFactory.init(
    keyStoreSpi.fromPem("ca.pem").getKeyStore());
```

The user of the SPI needs to be aware of the paths and the format of the files.
It is assumed that the paths are provisioned in the following ways:

- Using existing global configuration options such as [HTTPS server certificate](https://www.keycloak.org/server/enabletls).
- As part of the realm-specific configuration. This option does not exist today.

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
Currently, PEM files are supported for HTTPS credentials only.
Internally the PEM files can be loaded into `KeyStore`, allowing consumers to remain unaware of the format on disk.

### Realm-specific configuration

An example of realm-specific KeyStore configuration are credentials and trust anchors for LDAP federation.
Currently, all LDAP connections share the same credentials and trust anchors.
It should be possible to configure separate client credentials and trust anchors for each LDAP federation.


## Milestones

1. Implement KeyStore SPI and default implementation.
2. Finalize [LDAP SASL EXTERNAL PR](https://github.com/keycloak/keycloak/pull/7365) which initially triggered the work on KeyStore SPI.
Utilize KeyStore SPI for loading client credentials for SASL EXTERNAL.
This allows Keycloak to hot-reload client credentials for the LDAP federation.
3. Migrate the HTTPS server to use KeyStore SPI.
This allows Keycloak to hot-reload server credentials.
Neither Quarkus or Vert.X support hot-reload but Keycloak can pass `KeyManager` (enabled by [Quarkus PR](https://github.com/quarkusio/quarkus/pull/27682)) which can be initialized with `KeyStore` from the default KeyStore SPI implementation.
