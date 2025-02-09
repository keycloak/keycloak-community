# FAPI Grant Management

* **Status**: Notes
* **JIRA**: TBD


## Motivation

[FAPI Grant Management ][1] defines an extension of OAuth 2.0 to allow clients to explicitly manage their grants with the authorization server.

## Specifications

### Authorization Server Metadata

These parameters should be added to the .well-known/openid-configuration:

- **grant_id_supported**

Boolean indicating support for provision of grant ids in token responses and use of such grant ids in authorization requests.

- **grant_management_endpoint**
  URL of the authorization server's Grant Management Administration Endpoint.

  
FAPI Grant Management consist of 2 parts:
- *Grant Management API*
- *OAuth Protocol Extensions*

### Grant Management API
The Grant Management API allows clients to perform various actions on a grant whose grant_id the client previously obtained from the authorization server.

Currently supported actions are:

- `Query`: Retrieve the current status of a specific grant
- `Revoke`: Request the revocation of a grant
#### API authorization
Using the grant management API requires the client to obtain an access token authorized for this API.The token is required to be associated with the following scope value:

`grant_management_query`: scope value the client uses to request an access token to query the status of its grants.

`grant_management_revoke`: scope value the client uses to request an access token to revoke its grants.
#### Grant Resource URL

It should be found out from the grant_management_endpoint in the Authorization server metadata and has this form:

````
{scheme}://{host}[:{port}]/grants
````

#### API Query Status of Grant overview

The status of a grant is queried by sending a HTTP GET request to the grant's resource URL as shown by the following example.
````
GET /grants/TSdqirmAxDa0_-DB_1bASQ 
Host: as.example.com
Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
````

The authorization server will respond with a JSON-formated response as shown in the folling example:
````
HTTP/1.1 200 OK
Cache-Control: no-cache, no-store
Content-Type: application/json

{
   "scopes":[
      {
         "scope":"contacts read write",
      },
      {
         "scope":"openid"
      }
   ],
   "claims":[
      "given_name",
      "nickname",
      "email",
      "email_verified"
   ],
   "authorization_details":[
      {
         "type":"account_information",
         "actions":[
            "list_accounts",
            "read_balances",
            "read_transactions"
         ],
         "locations":[
            "https://example.com/accounts"
         ]
      }
   ]
}
````

#### API Revoke Grant overview
To revoke a grant, the client sends a HTTP DELETE request to the grant's resource URL. The authorization server responds with a HTTP status code 204 and an empty response body.

This is illustrated by the following example.

````
DELETE /grants/TSdqirmAxDa0_-DB_1bASQ 
Host: as.example.com
Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA

HTTP/1.1 204 No Content
````
The AS revoke the grant and all refresh tokens issued based on that particular grant, it revoke all access tokens issued based on that particular grant.
#### Error Responses:

If the resource URL is unknown, the authorization server responds with HTTP status code 400.

If the client is not authorized to perform a call, the authorization server responds with HTTP status code 403.

If the request lacks a valid access token, the authorization server responds with HTTP status code 401.



#### Representations of Grant Entity

- **grant_id**

String value identifying an individual grant managed by the AS for a certain client and a certain resource owner. 

- **scopes**

JSON array where every entry contains the scope parameter value.

- **claims**

JSON array containing the names of all OpenID Connect claims (see [@!OpenID]) as requested and consented in one or more authorization requests associated with the respective grant.

- **authorization_details**

JSON Object as defined in [@!I-D.ietf-oauth-rar] containing all authorization details as requested and consented in one or more authorization requests associated with the respective grant.

- **user_id**

Id of user

- **created_at**

Creation date

- **updated_at**

Date uptade

- **userConsent**

a ManyToOne relation to UserConsentEntity


### OAuth Protocol Extensions
#### Authorization Request
The spec introduces the authorization request parameter `grant_id`. This parameter can be used with any request serving as authorization request.

- `grant_id`: OPTIONAL. String value identifying an individual grant managed by a particular authorization server for a certain client and a certain resource owner.

````
GET /authorize?response_type=code&
     client_id=s6BhdRkqt3
     &grant_id=TSdqirmAxDa0_-DB_1bASQ
     &scope=write
     &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
     &code_challenge_method=S256
     &code_challenge=K2-ltc83acc4h... HTTP/1.1
Host: as.example.com 
````

#### Authorization Error Response
- In case the `grant_id` is unknown or invalid, the authorization server will respond with an error code `invalid_grant_id`.

- In case the AS does not support `grant_id`, it will respond with the error code `invalid_request`.


#### Token Response
The spec introduces the token response parameter `grant_id`:

`grant_id`: URL safe string value identifying an individual grant managed by the AS for a certain client and a certain resource owner. The grant_id value MUST be unique per client.

The grant_id will be a string representation of a UUID as a URN per [[RFC4122](https://tools.ietf.org/html/rfc4122)]

we will use `Base64Url.encode(KeycloakModelUtils.generateSecret())` to generate

Here is an example response:
````
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-cache, no-store

{
   "access_token": "2YotnFZFEjr1zCsicMWpAA",
   "token_type": "example",
   "expires_in": 3600,
   "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
   “grant_id”:”TSdqirmAxDa0_-DB_1bASQ”
}
````

## Implementation details
### save grant
We will add a Grant entity to the Keycloak JPA EntityManager with the help of JpaEntityProvider

````
public class GrantJpaEntityProvider implements JpaEntityProvider {}
...
em.persist(grant);
em.flush();
````
### query grant

````
TypedQuery<String> query = entityManager.createNamedQuery("findGrantByClientIdAndGrantId", Grant.class);

        query.setFlushMode(FlushModeType.COMMIT);
        query.setParameter("client_id", client_id);
        query.setParameter("grant_id", grant_id);

        List<Grant> result = query.getResultList();
````
### revoke grant
````
revokeAllAccessAndRefreshTocken(grant_id)
em.remove(grant)
````
### authorisation request
AS should accept grant_id in all endpoint where Authorization request is accepted. AS should create or update the Grant depending if grant_id is present or not and if `grant_id_supported` is `true`.

Classes/methods affected:

## Admin UI


## Tests
Grant Management should be properly covered by unit and integration tests.

## Documentation
Grant Management usage should be properly documented.

Affected documents: Securing Applications and Services Guide

## Resources
* [Financial_API_Grant_Management][1]

[1]: https://openid.net/specs/fapi-grant-management-01.html

