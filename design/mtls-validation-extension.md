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
- X509ClientAuthenticator.java (existing class bearing the basic SSL validation and storage logics)
- MtlsExtendedValidationSpi.java (SPI meant to build validation plugins for different SSL Certificate types)
- MtlsExtendedValidationProvider.java (provider bearing main methods) 
- MtlsExtendedValidationProviderFactory.java (provider initiating factory)
- org.keycloak.provider.Spi (existing Spi register)
- OIDCAdvancedConfigWrapper.java (existing config wrapper should be extended with additional properties)
- OIDCClientRepresentation.java (existing client representation should be extended with additional properties)
- OIDCConfigAttributes.java (existing config, should be extended)
- OIDCConfigurationRepresentation.java (existing config representation, should be extended with additional properties)
- DescriptionConverter.java (existing converter, should be extended)
- admin-messages_en.properties (existing properties, should be extended with new descriptions for UI)
- client-detail.html (existing htlm page, should be extended with new selectors to apply additional external mtls validation to a certain client and pick desired implementation)
- clients.js (existing js services class, should be extended to get/store mtls validation related data from/to client properties)

### X509ClientCertificateAuthenticator and UI implementation part
- Admin UI section `Client/Advanced Settings` should be extended with a flag `Enable custom certificate validation` to be sure that additional validation should be turned on/off for
  particular client with possibility to pick corresponding implementation from a drop-down list. 
- Adding MtlsExtendedValidationSpi.class as SPI for plugin extension along with corresponding MtlsExtendedValidationProviderFactory class and MtlsExtendedValidationProvider class
- Modification of X509ClientAuthenticator with calls to MtlsExtendedValidationSpi to introspect/validate the certificate if `custom validation is enabled` for current `client`

### MtlsExtendedValidationSpi class
Should extend Spi interface and implement corresponding methods as follows: 
```java
package org.keycloak.mtls;

import org.keycloak.provider.Provider;
import org.keycloak.provider.ProviderFactory;
import org.keycloak.provider.Spi;

public class MtlsExtendedValidationSpi implements Spi {
  @Override
  public boolean isInternal() {
    return false;
  }

  @Override
  public String getName() {
    return "tls-client-extended-validation-impl";
  }

  @Override
  public Class<? extends Provider> getProviderClass() {
    return MtlsExtendedValidationProvider.class;
  }

  @Override
  public Class<? extends ProviderFactory> getProviderFactoryClass() {
    return MtlsExtendedValidationProviderFactory.class;
  }
}

```

### MtlsExtendedValidationProvider class
Should extend Provider interface and have following methods:
- Method `parseAdditionalFields` must handle extraction and storage of additional fields from custom certificates into the AuthenticationFlowContext
- Method `performAdditionalValidation` must handle all the logics concerning certificate validation
```java
package org.keycloak.mtls;

import org.keycloak.provider.Provider;

import java.security.cert.X509Certificate;
import java.util.Map;

public interface MtlsExtendedValidationProvider extends Provider {

    Map<String, String> parseAdditionalFields(X509Certificate[] certs);

    void performAdditionalValidation(X509Certificate[] certs);
}
```

Additionally, it may carry the following logics:
- additional fields validation
- roles validation for presence and syntax
- online check for certificate validity at certificate issuer

### Sample implementation
Should contain the following:
- SampleMtlsVerificationProviderImpl.java - implementing `MtlsExtendedValidationProvider` and carrying corresponding parsing and validation logics, should have `ID` parameter for easy navigation in Keycloak Admin UI and further search 
  among other providers inside Keycloak
- SampleMtlsVerificationProviderFactory.java - implementing `MtlsExtendedValidationProviderFactory`
- org.keycloak.mtls.MtlsExtendedValidationProviderFactory - sould be located in `/resources/META-INF/services/` carrying ProviderFactory initialization
- jboss-deployment-structure.xml - sould be located in `src/main/resources/META-INF/`

#### Samples
#####SampleMtlsValidationProviderImpl.java
```java
import org.jboss.logging.Logger;
import org.keycloak.mtls.MtlsExtendedValidationProvider;

import java.security.cert.X509Certificate;
import java.util.HashMap;
import java.util.Map;

public class SampleMtlsValidationProviderImpl implements MtlsExtendedValidationProvider {
    static final String ID = "Sample External MTLS Verification Implementation"; // This ID is used in Keycloak Admin UI to pick the required implementation from the dropdown in Clients "Advanced Settings" Section 
  // and in X509ClientAuthenticator.class to identify and pick provider corresponding to current client 
    private static final Logger LOG = Logger.getLogger(MtlsExtendedValidationProvider.class);

    @Override
    public Map<String, String> parseAdditionalFields(X509Certificate[] certs) {
        LOG.info("Here we will parse additional fields from our Certificates");
        HashMap<String, String> map = new HashMap<>();
        map.put("Additional field name here","corresponding value for mentioned field");
        return map;
    }

    @Override
    public void performAdditionalValidation(X509Certificate[] certs) {
        LOG.error("Here we will perform our custom validation of MTLS Certificates");
    }

    @Override
    public void close() {

    }
}
```
#####SampleMtlsValidationProviderFactoryImpl.java
```java
import org.jboss.logging.Logger;
import org.keycloak.Config;
import org.keycloak.models.KeycloakSession;
import org.keycloak.models.KeycloakSessionFactory;
import org.keycloak.mtls.MtlsExtendedValidationProvider;
import org.keycloak.mtls.MtlsExtendedValidationProviderFactory;

public class SampleMtlsProviderFactoryImpl implements MtlsExtendedValidationProviderFactory {
    private static final Logger LOG = Logger.getLogger(MtlsExtendedValidationProviderFactory.class);

    @Override
    public MtlsExtendedValidationProvider create(KeycloakSession session) {
        LOG.error("MtlsProviderInitiated!!!");
        return new SampleMtlsValidationProviderImpl();
    }

    @Override
    public void init(Config.Scope config) {

    }

    @Override
    public void postInit(KeycloakSessionFactory factory) {

    }

    @Override
    public void close() {

    }

    @Override
    public String getId() {
        return SampleMtlsValidationProviderImpl.ID;
    }
}
```

#####jboss-deployment-structure.xml
```xml
<jboss-deployment-structure>
    <deployment>
        <dependencies>
            <module name="org.keycloak.keycloak-core" />
            <module name="org.keycloak.keycloak-server-spi" />
            <module name="org.keycloak.keycloak-server-spi-private" />
            <module name="org.keycloak.keycloak-services" />
            <module name="org.keycloak.keycloak-common" />
            <module name="org.jboss.logging" />
        </dependencies>
    </deployment>
</jboss-deployment-structure>
```
#####org.keycloak.mtls.MtlsExtendedValidationProviderFactory
```text
your.sample.packages.SampleMtlsProviderFactoryImpl
```

Compiled JAR should be then deployed in `standalone/deployments` folder of your Keycloak instance.