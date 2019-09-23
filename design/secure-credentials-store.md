# Secure Credentials Store - Vault SPI (Part I)

* **Status**: Draft #1
* **JIRA**: [KEYCLOAK-3205](https://issues.jboss.org/browse/KEYCLOAK-3205)

## Motivation

A number of credentials are currently stored in plaintext in the Keycloak database. This is less than ideal as it
can be costly and hard to provide the sufficient layer of protection at the database level.

Credentials include:

* Realm Keys
* Client credentials (secret)
* Identity brokering credentials
* LDAP credentials
* External tokens from identity brokering
* SMTP credentials
* User credentials such as OTP secrets (excluding passwords as they are securely hashed)

In addition custom providers such as user storage providers may also add additional credentials.

## Overview

The task will be split into two separate parts:

1. Read-only access to a secure storage (i.e. vault). Vault is managed and written-to by external
   tools specific to vault implementation.

2. Encryption / decryption SPI for secrets. There are services providing encryption / decryption
   of secrets, while abstracting from key management and other stuff. Note this does *not*
   mean that Keycloak would automatically support writing to vaults, the two functionalities
   are independent.

The rest of this document will deal with the first part (Vault SPI) only. **The second part
(encryption / decryption) will be treated in a separate document.**


## Scope

Generally, any secret can be stored in a vault. The value stored in Keycloak database should
however be only a reference to the vault provided by the administrator.

For some types like user OTP secret or client credentials it might be impractical to store
them into a vault: to change a secret, the user would have to change it in the vault. Yet the
decision what is practical and what not needs to be kept in administrator's hands. Hence any
place where a secret is retrieved should support a way to obtain it from the vault that needs to be
consistent across the whole product.

Generally, there might be more than one vault SPI provider available at the same time.
For example, when a new vault is plugged in addition to an existing one, the values
from the original vault are likely not be migrated immediately, and thus the references
to the original vault should work.

### Realm Keys

Today there are SPIs both for realm keys as well as signing of tokens. Keys SPI enables storing keys in Java KeyStore as well as adding custom support for a vault. Signature SPI enables delegating token signature to external HSM devices.

## Design

### Vault Provider SPI

It may be beneficial to allow per-type configuration. For example retrieving LDAP and SMTP credentials from Kubernetes secrets, while using HashiCorp Vault Transit Secrets for external tokens and not encrypting user secrets like otp secrets. That would allow more flexiblity as well as tuning security vs performance.

Vault will manage access to vault providers and ensure proper vault is used to obtain the secret.

For the consumer, the interface would be like this:

```java
public interface VaultServiceProvider {

    /**
     * Retrieves a secret from vault. The implementation should respect
     * at least the realm ID to separate the secrets within the vault.
     * If the secret is retrieved successfully, it is returned;
     * otherwise this method results into an empty {@link java.util.Optional}.
     *
     * @param vaultSecretId Identifier of the secret. It corresponds to the value
     *        entered by user in the respective configuration, which in turn
     *        is obtained from the vault when storing the secret.
     *
     * @return The secret or {@code null} if the secret was successfully resolved, or
     *         an empty {@link java.util.Optional}
     */
    Optional<byte[]> obtainSecret(String vaultSecretId);
}
```

### Accessing secrets

Accessing secrets from the code must be done in a way that prevents reading secrets from memory. Hence beware of using `String`s - these are cached. Prefer using `char[]` and clearing it up after usage similarly to e.g. [KeySpec](https://docs.oracle.com/javase/1.5.0/docs/api/javax/crypto/spec/PBEKeySpec.html#clearPassword()).

### Default vault service providers to consider

This is the list of vaults considered for a default implementations (TODO: to be shortlisted):

* [Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/) / [OpenShift](https://docs.openshift.com/container-platform/4.1/nodes/pods/nodes-pods-secrets.html) secrets
* Plain text (for testing purposes)
* Database - use application level encryption to securely store secrets in the database
  in a portable way. This would require dealing with keys as well as key rotation.
* [HashiCorp vault](https://www.vaultproject.io/)
* [Keywhiz](https://square.github.io/keywhiz/)
* [Elytron Credential Store](https://docs.wildfly.org/17/WildFly_Elytron_Security.html#secure-credential-storage)

#### Plain-text file per secret (Kubernetes / OpenShift)

Kubernetes supports mounting secrets as volumes. Each key from a secret is then available as a file with that name with contents of the corresponding secret value.

_Realm separation_ is provided by prefixing the file name with a realm name and underscore. Hence if there are two realms called `realm-a` and `realm.b`, the secret `smtp.password` would be looked up as contents of files `realm-a_smtp.password` and  `realm.b_smtp.password`, respectively. Every occurence of an underscore in both realm name and secret key is replaced with two underscores, so e.g. for realm `realm_c` and Keycloak key `smtp_password`, the secret key (and thus file name) would be `realm__c_smtp__password`.

*References:* [Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/) / [OpenShift](https://docs.openshift.com/container-platform/4.1/nodes/pods/nodes-pods-secrets.html) secrets

### Supported fields (TBD)

The following fields will have support for vault access:

* Realm private keys (via key provider)
* Client secret
* Identity brokering credentials
* LDAP credentials
* SMTP credentials

The following fields will *not* have support for vault access:

* Realm private keys (directly)
* External tokens from identity brokering
* User credentials such as OTP secrets (excluding passwords as they are securely hashed)
