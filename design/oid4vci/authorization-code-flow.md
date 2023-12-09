## Step-1-b: Keycloak Issuer - Authorization Code Flow

In this use case, the user's wallet serves as a client, utilizing the OIDC authorization code flow to facilitate user authentication and enable the wallet to request a token from Keycloak's token endpoint. The token provided by Keycloak can then be:

* A typical access token that can subsequently be used by the wallet to obtain the VCs from a separate credential issuer endpoint, or
* The VCs themselves, assuming we agree to configure Keycloak's token endpoint to return a different token format.

The following diagram from [Francis](https://github.com/francis-pouatcha) in the issue [OID4VC#17616](https://github.com/keycloak/keycloak/discussions/17616?sort=new#discussioncomment-7326341) displays components involved in the production of the VCs.

```mermaid
sequenceDiagram
Actor User
participant Wallet
box Keycloak
participant CME as Credential Metadata<br/>Endpoint
participant AME as Authorization Matadata<br/>Endpoint
participant PARE as PAR<br/>Endpoint
participant DCR as DCR<br/>Endpoint
participant AZE as Authorization<br/>Endpoint
participant TE as Token<br/>Endpoint
participant CE as Credential<br/>Endpoint
end
User ->> Wallet: interacts
Wallet ->> CME: (1) Obtains Credential Issuer metadata <br>/.well-known/openid-credential-issuer
Note right of Wallet: 10.2 New endpoint<br/>do we need /issuer? see 10.2.1
CME -->> Wallet : Credential issuer metadata
Note right of Wallet: <br/>credential_issuer:<br/>authorization_server:<br/>credential_endpoint:<br/>batch_credential_endpoint:<br/>
Wallet ->> AME: (1) Obtains Authorization Server metadata <br>/.well-known/openid-configuration
AME -->> Wallet: Auth server metadata
Wallet ->> Wallet: create wallet attestation
Note right of Wallet: Self signed attestation<br/>GAIN PoC stage 1
Wallet ->> DCR: Register Wallet(Wallet Attestation)<br/>Endpoint: auth_server_metadata:registration_endpoint<br/>GAIN: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-attestation-based-client-auth-00
Note left of DCR: Configured KC to<br/>consume self signed<br/>attestations
DCR -->> Wallet: Wallet Client Credentials
Wallet ->> Wallet : Construct RAR
Note over Wallet : RAR type openid_credential<br/>[single|batch]<br/>credential=true<br/>all VC|token info
Wallet ->> PARE: Post PAR<br/>attestation-based-client-auth
Note left of PARE: Configured KC to<br/>consume self signed<br/>attestations
PARE-->>Wallet: request_uri
Wallet->>AZE: Authorization Request
AZE ->> AZE: Authorize User
Note right of AZE: evtl. with consent
AZE-->>Wallet: Authorization Response (code)
Wallet->>TE: Token Request (code)
Note left of TE: Configured KC to<br/>consume self signed<br/>attestations
TE-->>Wallet: Token Response
Note right of Wallet: Token might be a<br/>verifiable credential
Wallet ->> CE: Get Credential(using the AccessToken) /credential
CE -->> Wallet: Credentials
```

The key element of concern here is the Client Attestation. The Keycloak client registration interface wil have to be extended to support some sort of client attestations, which are consumable both by Keycloak and eventually by external credential issuers that rely on Keycloak for the authentication of the wallet.

From the diagram, we have following resulting todos:

### Credential Issuer Metadata Endpoint

The Credential Issuer Metadata is specified [here](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html#section-10.2.2).

### OAuth 2.0 Authorization Server Metadata

This interface will be extended to add: pre-authorized_grant_anonymous_access_supported=true|false

### Client Registration Endpoint (DCR)

**To-Do**

* [] Missing clear instruction on how to proceed with the registration of the wallet.

### RAR Support

**To-Do**

* [] Need details on how to proceed with RAR

### Attestation Based Client Auth

See: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-attestation-based-client-auth-00

**To-Do**

* [] Design Keycloak extension to support this.

### Credential Endpoint

See [Credential Endpoint](https://vcstuff.github.io/oid4vc-haip-sd-jwt-vc/draft-oid4vc-haip-sd-jwt-vc.html#section-4.4)
