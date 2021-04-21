# MTLS Validation Extension (MTLS_Ext)

* **Status**: Notes
* **JIRA**: TBD


## Motivation

MTLS validation extension is designed to enable extension of standard SSL Certificate validation already present inside Keycloak (X509) enabling validation of additional fields present in different certificate types (example: QWAC), when MTLS 
support is
enabled.

QWAC is a sample of standard certificate extension that is occurring on OPEN BANKING Environment 
(link to [QWAC spec](https://www.etsi.org/deliver/etsi_ts/119400_119499/119495/01.03.01_60/ts_119495v010301p.pdf)).

Introducing an extra introspection/validation SPI has such main benefits:

* Support validation of additional SSL Certificate fields
* Support for market specific OpenBanking implementations for example European PSD2 (QWAC) or Australian CDR (normative usage of existing fields)
* Support custom (proprietary) certificate validation implementations

## Implementation details
We have to provide new Certificate validation SPI.
This SPI may be implemented by a separate service (plugin), which deals with custom SSL Certificate processing.
This plugin should be deployed inside the Keycloak and should handle all custom (proprietary) validation and parsing logic, so that Keycloak only interacts with it via SPI.

### Affected classes
- X509ClientCertificateAuthenticator (existing class bearing the basic SSL validation and storage logics)
- ExtendedCertificateValidation (SPI meant to build validation plugins for different SSL Certificate types)

### X509ClientCertificateAuthenticator and UI implementation part
- Admin UI section `Client/Advanced Settings/OAuth 2.0 Mutual TLS Certificate Bound Access Tokens Enabled` should be extended with a flag `Enable custom certificate validation` to be sure that additional validation should be turned on/off for
  particular client
- Adding ExtendedCertificateValidation.class as SPI for plugin extension
- Modification of X509ClientCertificateAuthenticator to with calls to ExtendedCertificateValidation SPI to introspect/validate the certificate if `custom validation is enabled` for current `client`

### ExtendedCertificateValidation SPI class
Sample implementation for QWAC SSL Certificate validation must include:
Method `introspect` must handle extraction and storage of additional fields from custom certificates into the AuthenticationFlowContext
Method `validate` must handle all the logics concerning certificate validation

Additionally, it may carry the following logics:
- additional fields validation
- roles validation for presence and syntax
- correspondence of request(PAR/RAR) to effective roles in certificate
- online check for certificate validity at certificate issuer