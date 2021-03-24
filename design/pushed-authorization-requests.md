# Pushed authorisation requests (PAR)

* **Status**: Draft #1
* **JIRA**: 


## Motivation

[Pushed authorisation requests](https://tools.ietf.org/html/draft-ietf-oauth-par-06) is designed to enable OAuth clients to push the payload of an authorization request directly to the authorization server in exchange for a request URI value, which is used as reference to the authorization request payload data in a subsequent call to the authorization endpoint via the user-agent. The spec is still draft, but it has already been implemented by many major IdPs. By supporting this spec, we will be able to use Keycloak in more fields such as FAPI 2.0.

Introducing an extra back-end call to submit the authorisation parameters has three main benefits:

* Keeps the parameters confidential between client and server.
* Frees the authorisation request from any browser URL length limits. They can become an issue with complex requests, such as Rich Authorization Requests.
* Confidential OAuth clients will be authenticated up-front and the request parameters will be checked for errors before sending the end-user to the authorisation endpoint for login and consent.


## Specifications

### Authorization Server Metadata

These parameters should be added to the .well-known/openid-configuration:

- **pushed_authorization_request_endpoint**

The URL of the pushed authorization request endpoint at which a client can post an authorization request in exchange for a "request_uri".

The presence of "pushed_authorization_request_endpoint" is sufficient for a client to determine that it may use the pushed authorization requests flow. A "request_uri" value obtained from the PAR endpoint is usable at the authorization endpoint regardless of other authorization server metadata such as "request_uri_parameter_supported" or "require_request_uri_registration"

- **require_pushed_authorization_requests**

Boolean parameter indicating whether the authorization server accepts authorization request data only via the pushed authorization request method. If omitted, the default value is "false"


### Client Metadata

The following client metadata parameter indicate whether pushed authorization requests are required for the given client

- **require_pushed_authorization_requests**

Boolean parameter indicating whether the only means of initiating an authorization request the client is allowed to use is a pushed authorization request. If omitted, the default value is "false"



### PAR endpoint URL

It should be found out from the pushed_authorization_request_endpoint in the Authorization server metadata and has this form:

````
{scheme}://{host}[:{port}]/par
````


### API overview

The pushed authorization request endpoint is an HTTP API at the authorization server that accepts HTTP "POST" requests with parameters in the HTTP request entity-body using the "application/x- www-form-urlencoded" format with a character encoding of UTF-8.

##### Header parameters:

-   **[ Authorization ]** Used for HTTP basic authentication of the client, if applicable.

-   **Content-Type** Must be set to *`application/x-www-form-urlencoded`*

##### Body:

-   The OAuth 2.0 authorisation request parameters.

##### Success:

-   Code: `201`

-   Content-Type: `application/json`

-   Body: {object} The PAR Response.

##### Errors:

-   400 (Bad Request) with a PAR Error 

The authorization server returns an error response with the same format as is specified for error responses from the Authorization endpoint and MUST refuse any other request with HTTP status code 400 and error code "invalid_request".

In addition to the above, the pushed authorization request endpoint can also make use of the following HTTP status codes:

-   405 (Method Not Allowed)

If the request did not use the "POST" method, the authorisation server responds with an HTTP 405 (Method Not Allowed) status code.

-   413 (Payload Too Large)

If the request size was beyond the upper bound that the authorization server allows, the authorization server responds with an HTTP 413 (Payload Too Large) status code

-   429 (Too Many Requests)

If the number of requests from a client during a particular time period exceeds the number the authorization server allows, the authorization server responds with an HTTP 429 (Too Many Requests) status code.

##### Example PAR request

````
POST /as/par HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic czZCaGRSa3F0Mzo3RmpmcDBaQnIxS3REUmJuZlZkbUl3

response_type=code&state=af0ifjsldkj&client_id=s6BhdRkqt3
&redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
&code_challenge=K2-ltc83acc4h0c9w6ESC_rEMTJ3bww-uCHaoeK1t8U
&code_challenge_method=S256&scope=account-information
````

##### Example PAR Response

````
HTTP/1.1 201 Created
Content-Type: application/json
Cache-Control: no-cache, no-store

{
   "request_uri":
   "urn:ietf:params:oauth:request_uri:bwc4JK-ESC0w8acc191e-Y1LTC2",
   "expires_in": 60
}
````

##### Example PAR Error
````
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-cache, no-store

{
   "error": "invalid_request",
   "error_description": "The redirect_uri is not valid for the given client"       
}
````


### Representations

#### PAR Response

- **request_uri**

The request URI corresponding to the authorization request posted. This URI is used as reference to the respective request data in the subsequent authorization request only.

The format of the "request_uri" value is at the discretion of the authorization server but it MUST contain some part generated using a cryptographically strong pseudorandom algorithm such that it is computationally infeasible to predict or guess a valid value. The authorization server MAY construct the "request_uri" value using the form "urn:ietf:params:oauth:request_uri:<reference-value>" with "<reference-value>" as the random part of the URI that references the respective authorization request data. The string representation of a UUID as a URN per [[RFC4122](https://tools.ietf.org/html/rfc4122)] is also an option for authorization servers to construct "request_uri" values. The "request_uri" value MUST be bound to the client that posted the authorization request.

- **expires_in**

A JSON number that represents the lifetime of the request URI in seconds as a positive integer. The request URI lifetime should be configurable (e.g., between 5 and 600 seconds)

#### PAR Error

-   **error** An error code which can take one of the following values :

    -   **invalid_request** The request is missing a required parameter, includes an invalid parameter value, includes a parameter more than once, or is otherwise malformed.
    -   **invalid_client** Client authentication failed, due to missing or invalid client credentials.
    -   **unauthorized_client** The client is not registered for the requested `response_type`.
    -   **unsupported_response_type** The server doesn't support the requested `response_type`.
-   **[ error_description ]** Optional text providing additional information about the error that occurred.

-   **[ error_uri ]** Optional URI for a web page with information about the error that occurred.


### Authorization Request

The client uses the request URI value to create the subsequent authorization request by directing the user-agent to make an HTTP request to the authorization server's authorization endpoint like the following:

````
GET /authorize?client_id=s6BhdRkqt3&request_uri=urn%3Aietf%3Aparams
       %3Aoauth%3Arequest_uri%3Abwc4JK-ESC0w8acc191e-Y1LTC2 HTTP/1.1
     Host: as.example.com
````

## Implementation details

### PAR configuration variable
Add a new Boolean `PAR` to enable/disable the PAR flow

when `PAR` is `false`

`pushed_authorization_request_endpoint` is not present in the server metadata

The Authorisation server doesn't support the PAR flow

when `PAR` is `true`

`pushed_authorization_request_endpoint` is present in the server metadata

The Authorisation server support the PAR flow and a client may use the PAR flow

### require_pushed_authorization_requests parameter 

when `false`

there is no mandatory PAR request before autorization endpoint, the client may use it or not

when `true`

The PAR is automatically enable and PAR request is mandatory prior autorization endpoint and reject any authorization request without a request URI issued from the PAR endpoint, clients must use it

### expires_in configuration
The default lifetime of request_uri should be 60 seconds

### Client Metadata require_pushed_authorization_requests

when `false`, PAR is not mandatory

when `true`, PAR is mandatory

if client metadata require_pushed_authorization_requests `true` and server metadata require_pushed_authorization_requests `false`

NOT SUPPORTED???

### The PAR endpoint 

Authenticate the client in the same way as at the token endpoint

Accept The OAuth 2.0 authorisation request parameters

Check if the configuration allow the client to make a PAR request

Reject the request if the "request_uri" authorization request parameter is provided.

Validate the pushed request as it would an authorization request sent to the authorization endpoint

Generate request_uri

save the Auth Request+request_uri+expires_in

return PAR Response

### Generation of request_uri
The format of the "request_uri" value is at the discretion of the authorization server but it MUST contain some part generated using a cryptographically strong pseudorandom algorithm such that it is computationally infeasible to predict or guess a valid value. The authorization server MAY construct the "request_uri" value using the form "urn:ietf:params:oauth:request_uri:<reference-value>" with "<reference-value>" as the random part of the URI that references the respective authorization request data. The string representation of a UUID as a URN per [RFC4122] is also an option for authorization servers to construct "request_uri" values. The "request_uri" value MUST be bound to the client that posted the authorization request.

### Retrieve the authorization request with the request_uri

At the Authorisation Endpoint, keycloak must be able to retrieve the authorization request with the request_uri
Must check if request_uri still valid

### Save the authorization request with the associated request_uri generated



