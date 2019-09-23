# OAuth 2.0 Device Authorization Grant

* **Status**: Draft #1
* **JIRA**: [KEYCLOAK-7675](https://issues.jboss.org/browse/KEYCLOAK-7675)


## Motivation

[The OAuth 2.0 Device Authorization Grant](https://tools.ietf.org/html/draft-ietf-oauth-device-flow-15) is designed for internet-connected devices that have limited input capabilities or lack a suitable browser. The spec is still draft, but it has already been implemented by many major IdPs. By supporting this spec, we will be able to use Keycloak in more fields.

## Endpoint

### Device Authorization Endpoint

In the spec, Device Authorization Endpoint is introduced for authorization request from the device which is an OAuth client. Keycloak has common AuthorizationEndpointBase class which provides common logic like checking realm, SSL and so on. We implement this new endpoint using the common base class. The URL is defined as `/realms/{realmName}/openid-connect/device/auth` in current implementation proposal.

### Verification Endpoint

Keycloak must provide a new endpoint for the verification process of the user code which is returned by device authorization request. Also, it's good UX that providing a shorter verification URI because the end-user need to enter the URI manually into their browser if the device has a limited display. So, it's defined as `/realms/{realmName}/device` in current implementation proposal.

This endpoint provides user code verification and user authentication, which we will discuss in User Interaction section.

### Token Endpoint

Token request for the spec is represented with new grant type of `urn:ietf:params:oauth:grant-type:device_code`. We add the process of this new grant type into an existing TokenEndpoint class for OAuth 2.0/OIDC.


## User Interaction when verifying a user code

The spec says that the details of the user interaction when processing the verification are up to the authorization server in [3.3. User Interaction](https://tools.ietf.org/html/draft-ietf-oauth-device-flow-15#section-3.3). There are two methods for the process which are imeplemented by major IdPs. *Login First* or *User Code Verification First*.

### Login First

In this case, the end-user need to login first before entering a user code in the user interaction. It means the authorization server must provide a shared authentication flow because the server can't detect the client ID yet which is bind to the user code. However, the risk getting user code brute force attack is low as attackers must log in first.

### User Code Verification First

In this case, the end-user need to enter a user code first in the user interaction. It means the authorization server can provide different authentication flow per client because the server can detect the client ID from the user code. However, the risk getting user code brute force attack is higher than Login First as attackers can try to enter user codes freely.

### Which user interaction does keycloak should support?

User Code Verification First is adopted by major IdPs like Google, Microsoft, and Salesforce. So it seems to be a common practice. Also, it can provide a more flexible authentication flow per device client. Keycloak already provides a feature that a client can use specific browser flow by using the Authentication Flow Overrides option. Therefore, it's suitable for supporting User Code Verification First.

## Consent

The spec says that the authorization server SHOULD display information about the device so that the person can notice if a software client was attempting to impersonating a hardware device. So It should always display consent screen when end-user do log in even if consent required is true or the user already consented.

## User Code Format

The spec says about user code format patterns in [6.1. User Code Recommendations](https://tools.ietf.org/html/draft-ietf-oauth-device-flow-15#section-6.1). It's good to be able to provide an SPI for user code format customization. We implement a case-insensitive eight-letter format as a default implementation of the SPI.

## Configuration

### Realm Configuration

In the token tab of the realm configuration, we add two options for the spec.

* **OAuth 2.0 Device Code Lifespan:** Max time before the device code and user code are expired. This value needs to be a long enough lifetime to be usable (allowing the user to retrieve their secondary device, navigate to the verification URI, login, etc.) but should be sufficiently short to limit the usability of a code obtained for phishing. The default value is 600 seconds (10 minutes).
* **OAuth 2.0 Device Polling Interval:** The minimum amount of time in seconds that the client should wait between polling requests to the token endpoint. The default value is 5 seconds.

### Client Configuration

In the client edit page for openid-connect protocol, we add an option for the spec.

* **OAuth 2.0 Device Authorization Grant Enabled:** This enables support for OAuth 2.0 Device Authorization Grant, which means that the client is an application on the device that has limited input capabilities or lacks a suitable browser.

Also, we add some extra options when the above option is enabled.

* **OAuth 2.0 Device Code Lifespan:** This is used for overriding realm configuration per client.
* **OAuth 2.0 Device Polling Interval:** This is used for overriding realm configuration per client.

## Database changes

There are no database schema changes since we can store new configuration for the spec into `REALM_ATTRIBUTE` and `CLIENT_ATTRIBUTES` tables.

## Testing

We can implement test code using an existing test framework.

## Documentation

We need to add explanation about supporting the spec into [keycloak-documentation](https://github.com/keycloak/keycloak-documentation). [Server Administration Guide](https://www.keycloak.org/docs/latest/server_admin/index.html) looks like a better place to explain it.

## Implementation proposal

**Source:** https://github.com/openstandia/keycloak/tree/oauth2-device-authorization-grant

**Current limitations:**
- Only basic test cases
- All config options are not implemented yet
- SPI for custom user code format is not implemented yet

### How to try it

Create a realm and Add a client and enable `OAuth 2.0 Device Grant Enabled` as a public client.

Send POST request to the `Device Authorization Endpoint` like below. 

```
curl -X POST \
    -d "client_id=foo" \
    "http://localhost:8080/realms/test/openid-connect/device/auth"
```

It returns a response like below.

```
{
  "device_code": "9tDoVhtMME84R0YNcyZjid54EAaqOXTL-Uoml6WKbjQ",
  "user_code": "PPBWEREJMQ",
  "verification_uri": "http://localhost:8080/auth/realms/test/device",
  "verification_uri_complete": "http://localhost:8080/auth/realms/test/device?user_code=PPBWEREJMQ",
  "expires_in": 600,
  "interval": 5
}
```

Open your browser and go to the `verification_uri`. Then enter the `user_code`, log in, and consent giving information to the client.

Finally, you can get an access token using `device_code` like below.

```
curl -X POST \
    -d "grant_type=urn:ietf:params:oauth:grant-type:device_code" \
    -d "client_id=foo" \
    -d "device_code=9tDoVhtMME84R0YNcyZjid54EAaqOXTL-Uoml6WKbjQ" \
    "http://localhost:8080/auth/realms/test/protocol/openid-connect/token"
```

It returns a response like below if the `device_code` was approved.

```
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "expires_in": 300,
  "refresh_expires_in": 1800,
  "refresh_token": "eyJhbGciOiJIUzI1NiI...",
  "token_type": "bearer",
  "not-before-policy": 0,
  "session_state": "48dc7735-1762-4bb6-bf1d-6ec21267587d",
  "scope": "profile email"
}
```
