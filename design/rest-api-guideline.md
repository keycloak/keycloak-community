# REST API Guideline

* **Status**: Draft #1
* **JIRA**: [KEYCLOAK-9344](https://issues.jboss.org/browse/KEYCLOAK-9344)

## Abstract

The Keycloak REST API Guideline provides a set of design principles and practices that should be considered by developers when designing, implementing and exposing a RESTful API. To provide the best experience for developers consuming Keycloak REST APIs, it's important API developers to follow to the principles and practices defined herein, making these APIs intuitive and easy to use.

The goals of the guideline are:

* Define the [overall principles](#principles) that should be taken into account when designing REST APIs for Keycloak

* Define common practices and patterns for all Keycloak REST APIs such as:
    * [Filtering](#filtering)
    * [Pagination](#pagination)
    * [Error Handling](#error-handling)
    * [Documentation](#documentation)

* Define common practices for [documentation](#documentation)

Although the guideline is based on the REST architectural style, it does not try to strictly adhere to RESTful constraints. The principles covered by the guideline are based on practical experiences in the development of REST APIs in Keycloak while still conforming with most of the constraints of REST architectures.

**Table of Contents**


  - [Principles](#principles)
    - [Level of Service Maturity](#level-of-service-maturity)
    - [Resource Path](#resource-path)
      - [Group](#group)
      - [Versioning](#versioning)
      - [Extensions](#extensions)
      - [Technology Preview](#technology-preview)
      - [Store Resources](#store-resources)
      - [Controller Resources](#controller-resources)
    - [Resource Representation](#resource-representation)
    - [HTTP Status Codes](#http-status-codes)
    - [HTTP Verbs](#http-verbs)
      - [GET](#get)
      - [POST](#post)
      - [PUT](#put)
      - [DELETE](#delete)
  - [Practices](#practices)
    - [Error Handling](#error-handling)
    - [Filtering](#filtering)
    - [Pagination](#pagination)
    - [Concurrency Control and Consistency](#concurrency-control-and-concurrency)
    - [Rate Limiting](#rate-lmiting)
    - [Documentation](#documentation)
  - [Examples](#examples)

## Principles

This section defines the general principles that should be considered by API developers, regardless of the practices defined through this document.

### Level of Service Maturity

Based on [Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html), developers MUST adhere to the maturity Level 2, where interactions with the API endpoints are based on the usage of HTTP Verbs and status codes to communicate status and errors.

With this principle in mind, APIs should leverage the [HTTP Verbs](#http-verbs) defined in this guideline and respect their meaning, idempotency as well as use [HTTP Status Codes](#http-status-codes) in responses to communicate the success and failures when processing a request. For more details about the supported HTTP Verbs and Status Codes, see their respective sections in this document.

By respecting this principle, we expect developers to provide more concise and consistent APIs with a uniform interface. 

### Resource Path

In Keycloak, we choose to stick with the following convention for resource paths:

```
/{realm}/apis/{API_GROUP}/{version}
```

Keycloak APIs usually operate in the context of a realm, the resource path should be prefixed with `/{realm}` where `realm` maps to the the name of an existing realm.

In Keycloak, a realm represents a tenant from where all the configuration is done. By operating in the context of a specific realm, APIs should always consider the realm specified in the path when performing any operation.

#### Group

The `API_GROUP` refers to a set of one or more functionality that can be grouped together. The endpoints associated with a group can manage different resources as far as they are strictly related with the functionality represented by the group.

By including the group in the resource path, we aim to provide a clear and consistency view of the functionalities exposed by the different APIs, with more control over the groups of functionality (in terms of functional and non-functional aspects) as well as make more clear how we communicate the functional structure of these APIs through the documentation.

Here is an example of a resource path related with the group of functionalities provided by the Account Management REST API:

```
/{realm}/apis/accounts/v1/{user_id}
```

The path refers to the `accounts` group and any resource/operation related with user account management (in this case for user with id `user_id`) can be accessed from that path.
Note that by grouping functionalities together and in conjunction with the versioning schema, we can keep the root path and also the semantics of the API without impact the adaptability for new changes or improvements to the API.

Group names should be in lowercase and a single word name.

##### Compound Group Names

Under some circunstances, the group can be a compound name. This is the case when APIs are logically related with a common group. For instance, APIs related with the Keycloak Admin API are grouped under:

```
/{realm}/apis/admin/users/v1/{user_id}
```

The usage of compound names in groups should consider whether or not makes sense to group the API into a logical group. Taking again the Admin API example, the reasoning for using a compound name is to make easier to restrict access to administration related APIs using a Web Application Firewall.

In general, try to stick with single word names for groups.

##### Reserved Group Names

When choosing a group name you should avoid using reserver names such as `apis` or names that conflict fully or partially to any existing group.

As a recommendation for users of Keycloak, realm names should also take these considerations into account otherwise they may end up with a very odd resource path definition.

#### Versioning

Versioning helps APIs to adapt to changes to their fields or restructure resource representations. In addition to that, versioning can help developers to offer non-stable, technology preview features as well as control the end-of-life of APIs.

The `version` refers to the stability and the state of an API. The version does not necessarily refers to a Keycloak release version number so that it can be mapped to different Keycloak release versions.

Here is an example of a resource path related with the group of functionalities provided by the Account Management REST API:

```
/{realm}/apis/accounts/v1/{user_id}
```

From the example above we can infer that the path refers to a specific `v1` version with the possibility to change the version and: 

* completely change the API
* do breaking changes to parts of the API
* remove functionalities or specific parts of the API
* provide non-stable or technology preview features
* the different APIs can evolve separately

#### Extensions

To better accommodate and separate the core API from extensions built on top of the API, developers should consider the following path format for extensions:

```
/{realm}/apis/{API_GROUP}/extensions/{extension_id}/{version}
```

This separation aims to avoid confusion when consuming stable and official Keycloak APIs so that clients can differentiate if they are using some API provided by a third-party with no support from Keycloak team.

When the extension does not fit into any of the existing API groups, it should be possible to use a path with the following format:

```
/{realm}/apis/extensions/{extension_id}/{version}
```

#### Technology Preview

To better accommodate and separate the core API from technology preview features, developers should use a path format as follows:

```
/{realm}/apis/{API_GROUP}/{version}/preview/{preview_id}
```

Technology preview APIs are related with features that have great potential to become fully supported. They provide the latest work in some area of interest and are offered with the intent to gather feedback from the community about new functionalities. However, such functionalities can be removed at any moment and break existing clients consuming the API.

When the API does not fit into any of the existing API groups, it should be possible to use a path with the following format:

```
/{realm}/apis/preview/{preview_id}
```

#### Store Resources

A resource store provides operations for a given resource. For example:

```
/{realm}/apis/accounts/v1/1234/applications
```

The example above is representing a resource store path from where we can manage the `applications` for a user account with ID `1234`. Based on this path, clients can perform different operations based on the HTTP verbs or any controller resource defined to the store. Usually this type of resource is associated with CRUD operations for a specific resource.

When a resource fits under this category, the name should be a plural noun.

#### Controller Resources

Under some circumstances, the HTTP verbs are not enough to represent an operation that can be performed in a resource. In this case,
the resource path may indicate a specific action to be taken when accessed.

```
POST /{realm}/accounts/v1/disable HTTP/1.1
Host: keycloak.org
```

The example above shows the usage of `disable` to operate accounts in order to disable them.

When a resource fits under this category, the name should be a verb, using camel-case if necessary.

##### HTTP Verbs on Controller Resources

Controller resources should use the HTTP `POST` method to perform operations on a resource. There are no restrictions on the request body as long as the [resource representation](#resource-representation) is respected if the action requires a request body.

### Resource Representation

The standard format for resource representation is JSON. There is no rule for schemas or the semantics used by a resource representation, as long as they follow the JSON syntax properly.

However, it is important to taken into account the following principles when defining resource representations:

* Keep it simple for clients, so that a representation can be easily built and processed when making requests/responses to the API
* Be aware of extensive representations as they can affect overall performance and usability, if possible review your API to see if resources can be decomposed into smaller ones
* Avoid deeply nested JSON objects
* Avoid using generic resource representations

### Http Status Codes

The tables below defines the HTTP status code that clients should expect from APIs.

This document also defines which codes are expected to be used for each HTTP verb, please take a look at [HTTP Verbs](#http-verbs) for more details.

Clients should consider the status codes herein defined in order to properly handle responses from APIs, recover from errors, and 
understand the reasoning behind the API responses.

#### Success Codes

|Code|Description|
|---|---|
|200|Indicates the request completed successfully regardless the size of the response body|
|201|Indicates the request completed successfully and a resource was created|
|204|Indicates the request completed successfully and the server did not provide a response body|

#### Redirection Codes

|Code|Description|
|---|---|
|307|Indicates that clients should resubmit the request to another location|

#### Client Side Error Codes

|Code|Description|
|---|---|
|400|Indicates that the request is invalid, usually related with the validation of the payload|
|401|Indicates that clients should provide authorization or the provided authorization is invalid|
|403|Indicates that the authorization provided by the client is not enough to access the resource|
|404|Indicates that the requested resource does not exist|
|405|Indicates that the method chosen by the client to access a resource is not supported|
|409|Indicates that the resource the client is trying to create already exists or some conflict when processing the request|
|415|Indicates that the requested media type is not supported|

#### Server Side Error Codes

|Code|Description|
|---|---|
|500|Indicates that the server could not fulfill the request due to some unexpected error|

### Http Verbs

Based on the REST architectural style, HTTP Verbs should be used as the means to communicate to APIs the operations to be performed
on their resources. While HTTP verbs cover by themselves the most common operations you might want to look the [Controller Resource](#controller-resources) as an alternative on how to overcome any limitation imposed by the usage of HTTP verbs herein defined.

The table below defines the HTTP verbs you should consider when exposing operations through your API and the circumstances where they apply.

|Verb|Description|Idempotent|
|---|---|---|
|GET|Requests the current state of a resource|Yes|
|POST|Creates a new resource or submit a command|No|
|PUT|Replaces a resource state|Yes|
|PATCH|Applies partial updates to a resource|Yes|
|DELETE|Removes a resource|Yes|

#### GET

HTTP `GET` should be used to obtain current state of a resource. It is usually related with operations on resources that return either a list or a single representation, or start some action that does not change the resource state.

##### Expected Status Codes

It is expected that APIs responding to HTTP `GET` requests respect the following HTTP status codes to communicate the result of the operation:

|Status Codes|
|---|
|200|
|307|
|400|
|401|
|403|
|404|
|415|
|500|

#### POST

HTTP `POST` should be used to create new resources. It is a non-safe and non-idempotent operation and clients should be aware of that.

Successful `POST` requests should always result in a `201` status code and empty response. The response should also contain a reference
to the resource that was created:

```
HTTP/1.1 201 Created
Content-Length: 0
Location: http://mykeycloak.com/{realms}/apis/users/{version}/1234
```

##### Expected Status Codes

It is expected that APIs responding to HTTP `POST` requests respect the following HTTP status codes to communicate the result of the operation:

|Status Codes|
|---|
|201|
|400|
|401|
|403|
|404|
|409|
|415|
|500|

#### PUT

HTTP `PUT` should be used to replace a resource state.

Successful `PUT` requests should always result in a `204` status code and empty response.

```
HTTP/1.1 204 No Content
Content-Length: 0
```

#### PATCH

HTTP `PATCH` should be used to partially update the resource state.

Successful `PATCH` requests should always result in a `204` status code and empty response.

```
HTTP/1.1 204 No Content
Content-Length: 0
```

Partial updates should follow the [RFC7396 - Json Merge Patch](https://tools.ietf.org/html/rfc7396) standard. 

The `PATH` operation should be driven by a specific `Content-Type` header as defined by RFC7396 and other patch strategies. By using the
`Content-Type` as a selector, this guideline can support additional patch strategies in the future.

##### Note on RFC7396 - Json Merge Patch

The RFC7396 is the most simple `PATH` strategy, but it has some limitations. The limitations are basically around updating lists on a resource which will be entirely replaced by the patch.

Under certain circumstances, other patch strategies may be useful thus the constraint around the `Content-Type` header of `PATH` requests.

##### Expected Status Codes

It is expected that APIs responding to HTTP `PUT` requests respect the following HTTP status codes to communicate the result of the operation:

|Status Codes|
|---|
|204|
|400|
|401|
|403|
|404|
|409|
|415|
|500|

#### DELETE

HTTP `DELETE` should be used to remove a resource.

Successful `DELETE` requests should always result in a `204` status code and empty response.

```
HTTP/1.1 204 No Content
Content-Length: 0
```

##### Expected Status Codes

It is expected that APIs responding to HTTP `DELETE` requests respect the following HTTP status codes to communicate the result of the operation:

|Status Codes|
|---|
|204|
|400|
|401|
|403|
|404|
|500|

## Practices

### Error Handling

A good error handling mechanism helps to communicate to clients the reasons why a request could not be fulfilled and eventually
recover from this errors.

When returning errors, developers should use a common [HTTP Status Code](#http-status-codes) and include a body using JSON as the format with the information
why a request failed.

The error message should have the following format:

|Claim|Description|Required|
|---|---|---|
|error|The error code|Yes|
|error_description|A text providing more details for clients about the error and eventually how to recover from it|No|

Here is an example of a error response:

```json
HTTP/1.1 400 Bad Request
{
    "error": "invalid_request",
    "error_description": "The field 'email' is mandatory"
}
```

#### Error Codes

Error codes can be any string. They should be lowercase and use `_` when the code has multiple words. However, developers should 
consider existing error codes before creating new ones.

When processing error responses, developers should look the error codes being used by other APIs and see if they can be reused.

### Pagination

When dealing with large data sets, [store resources](#store-resources) should consider paginating the results when fetching resource sets.

In Keycloak, pagination is based on two request parameters:

* `first`

    The position of the first result to retrieve

* `max`
    
    The maximum number of entries to retrieve

These parameters should be sent along with a `GET` request as follows:

```
/{realm}/apis/admin/users/v1?first=1&max=20
```

The example above shows how to request the first page (entries from 1 to 20).

As a result, the server should respond as follows:

```
HTTP/1.1 200 OK
Connection: keep-alive
Cache-Control: no-cache
Content-Type: application/json
Content-Length: 961
Date: Wed, 22 May 2019 13:44:57 GMT

Link: <http://{host}:{port}/auth/{realm}/apis/admin/users/v1?first=20&max=20>; rel="next"
```

As defined by [RFC-5988](https://tools.ietf.org/html/rfc5988), responses from the server should contain a `Link` header indicating whether or not there is a next page and, if there are more pages, a link that can be followed to fetch the next page. If the `Link` header is not present, the client can assume that there are no more pages to fetch.

The possible links and `rel` values are:

|Rel|Description|
|---|---|
|`next`|The link for the next page of results|
|`prev`|The link for the previous page of results|

### Rate Limiting

Keycloak does not provide rate limiting capabilities and for such it is necessary to use a third-party tool such as a API Manager.

### Filtering

Sometimes, we need more control over the content that will return from the server, so that only a subset of the data is available in the response. For example, a list user's groups or permissions. 

The Keycloak admin REST API allows to search for users based on `e-mail`, `first name`, `lastname` and other optional parameters. However, such implementation is limited to a particular context (e.g, collection of users).

Filtering should be implemented as a query parameter, based on what we already do today. To filter on a field, should be as simple as the following request:

```
/{realm}/apis/admin/users/v1?group=users1
```

Multiple filters should result in an implicit `AND`, so in our example `/v1?group=users1&role=role1` would provide results based on `groups` and `roles`. Filtering should take into consideration the list of attributes supported by the API.

### Concurrency Control and Consistency

TODO

### Documentation

TODO

## Examples

### CRUD

The following example demonstrates how a Users API would look and how CRUD operations would be provided:

|Resource Path|GET|POST|PUT|DELETE|
|---|---|---|---|---|
|/{realm}/apis/admin/users/v1|List all users in the realm `acme`|Creates an user in the realm `acme`|Not Supported|Not Supported|
|/{realm}/apis/admin/users/v1/123|Gets a representation for user with id `123`|Not Supported|Updates the user with id `123`|Deletes the user with id `123`|
