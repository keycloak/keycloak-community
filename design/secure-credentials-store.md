# Secure Credentials Store

* **Status**: Notes
* **Author**: [stianst](https://github.com/stianst)
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


## Current Recommendation

 The current recommendation is to enable encryption at the database level. However, this is not well documented and
 may also have performance impacts.
 
 
 ### Realm Keys
 
 There is today SPIs both for realm keys as well as signing of tokens. 
 
 Keys SPI enables storing keys in Java KeyStore as well as adding custom support for a vault.
 
 Signature SPI enables delegating token signature to external HSM devices.
 
 
 ## Key Rotation
 
 It is important that a built-in approach supports key rotation.
 
 
 ## Vault SPI
 
 To allow pluggability of the vault a Vault SPI should be created that allows storing and loading secrets from an
 external vault. This SPI should also be leveraged by the built-in provider if one is provided.
 
 
 ### Default providers to consider
 
 Database - use application level encryption to securely store secrets in the database in a portable way. This would 
 require dealing with keys as well as key rotation. 
 
 Kubernetes secrets - Kubernetes has built-in support for secrets. Secrets in Kubernetes are by default stored in 
 plain-text in etcd, but for a production system this is not recommended as such Kubernetes secrets may be a good option
 for deployments on Kubernetes/OpenShift as there would be a single place to protect.
 
 HashiCorp Vault - a popular open-source vault.  
 
 
 ## Should everything be stored in a vault?
 
 Client credentials, user credentials and external tokens may not be suitable to store in a vault due to the high number
 of such items. It may be better to encrypt these within the database.