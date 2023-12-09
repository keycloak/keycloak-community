## Step-1-a: Keycloak Issuer - Preauthorize Code Flow
This approach appears to be the simplest, as it doesn't necessitate significant extensions in the Keycloak codebase. The [FIWARE](#fiware) implementation previously mentioned encompasses the majority of the required code. However, this implementation depends on components that are not compatible with keycloak licensing model.

The following diagram from [Stefan](https://github.com/wistefan) in issue [OID4VC#17616](https://github.com/keycloak/keycloak/discussions/17616?sort=new#discussioncomment-7326341) displays components needed for the pre-authorized code flow, as currently implemented in the [FIWARE](#fiware) codebase:
```mermaid
sequenceDiagram
    Actor User
    box Mobile Phone
    participant Wallet
    end
    box Keycloak Frontend
    participant AccountConsole
    end
    box Keycloak Backend
    participant Types-Endpoint
    participant Offer-Endpoint
    participant OpendID-Configurations-Endpoint
    participant Token-Endpoint
    participant Credential-Endpoint
    end
    User ->> AccountConsole: logs in
    AccountConsole ->> Types-Endpoint: Fetch types /types
    Note right of AccountConsole: all paths are sub-paths of /verifiable-credentials/<issuer-did>
    Types-Endpoint -->> AccountConsole: Available Types
    User ->> AccountConsole: select Credential Type
    Note right of AccountConsole: Steps until now are just for UX,<br/>not part of the spec
    AccountConsole ->> Offer-Endpoint: Get Offer-URI<br/> /credential-offer-uri?type=<VC>&format=ldp_vc
    Offer-Endpoint ->> Offer-Endpoint: Generate Authorization code for current user session
    Offer-Endpoint -->> AccountConsole: Credential Offer URI + (short lived) Authorization Code
    AccountConsole ->> AccountConsole: generate QR-Code
    User ->> Wallet: Initiate QR-Scan
    Wallet ->> AccountConsole: Scan QR
    AccountConsole -->> Wallet: Credential Offer Uri + Authorization Code
    Wallet ->> OpendID-Configurations-Endpoint: Get OpenID-Config /.well-known/openid-configuration
    OpendID-Configurations-Endpoint -->> Wallet: OpenID-Configuration
    Wallet ->> Token-Endpoint: Get AccessToken(using the Authz-Code) /token
    Token-Endpoint -->> Wallet: AccessToken
    Wallet ->> Credential-Endpoint: Get Credential(using the AccessToken) /credential
    Credential-Endpoint -->> Wallet: The Credential
```

Additional detail from diagram author:

* The flow is using the "Cross-Device Flow"(e.g. Wallet is on a different device than the browser showing the Account Console). 
* Binding the credential to holder-proof is also not yet provided.

As the diagram illustrates, implementation into Keycloak brings following modifications:

### Account Frontend

The modification of the account console to allow for the enrolment of the user wallet.

**To-Do**
* [] Does this lead to the production of any sort of Client Attestation.

### Types Endpoint

**To-Do**

* [] Describe purpose of the Types Endpoint

### Credential Offer Endpoint

See Credential Offer Endpoint in [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-11.2)

**To-Do**

* [] What type of credential will be natively offered by Keycloak?
* [] Are we planing to provide some sort of generic model based on the current KC user and role data model?
* [] Do we want to design something like a CredentialOfferProvider to allow for plugable credential models?

### OAuth 2.0 Authorization Server Metadata

This interface will be extended to add: pre-authorized_grant_anonymous_access_supported=true|false

### Credential Endpoint.
Whereby it is open to check if it does not make sense to have Token Endpoind directly produce the VC.