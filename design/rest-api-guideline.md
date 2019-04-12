# REST API Guideline

* **Status**: Draft
* **Author**: [pedroigor](https://github.com/pedroigor), [abstractj](https://github.com/abstractj)
* **JIRA**: [KEYCLOAK-9344](https://issues.jboss.org/browse/KEYCLOAK-9344)

## Abstract

The Keycloak REST API Guideline provides a set of design principles and practices that should be considered by developers when designing, implementing and exposing a RESTful API. To provide the best experience for developers consuming Keycloak REST APIs, it's important API developers to follow to the principles and practices defined herein, making these APIs intuitive and easy to use.

The goals of the guideline are:

* Define the [overall principles](#principles) that should be taken into account when designing REST APIs for Keycloak

* Define common practices and patterns for all Keycloak REST APIs such as:
    * [Filtering](#filtering)
    * [Pagination](#pagination)
    * [Error Handling](#error_handling)
    * [Security](#securityp)
    * [Cache](#cache)

* Define common practices for [documentation](#documentation)

Although the guideline is based on the REST architectural style, it does not try to strictly adhere to RESTful constraints. The principles covered by the guideline are based on practical experiences in the development of REST APIs in Keycloak while still conforming with most of the constraints of REST architectures.

## <a id="principles"></a>Principles of API Design

This section defines the general principles that should be considered by API developers, regardless of the practices defined through this document.

### <a id="apipath"></a> Resource Path

In Keycloak, we choose to stick with the following convention for resource paths:

```
/apis/{API_GROUP}/{realm}/{version}
```

#### Group

The `API_GROUP` refers to a set of one or more functionality that can be grouped together. The endpoints associated with a group can manage different resources as far as they are strictly related with the functionality represented by the group.

By including the group in the resource path, we aim to provide a clear and consistency view of the functionalities exposed by the different APIs, with more control over the groups of functionality (in terms of functional and non-functional aspects) as well as make more clear how we communicate the functional structure of these APIs through the documentation.

Here is an example of a resource path related with the group of functionalities provided by the Account Management REST API:

```
/apis/accounts/{realm}/v1
```

The path refers to the `accounts` group and any resource/operation related with user account management can be accessed from that path.
Note that by grouping functionalities together and in conjunction with the versioning schema, we can keep the root path and also the semantics of the API without impact the adaptability for new changes or improvements to the API.

Group names should be in lowercase, a single word name and plural.

#### Version

Versioning helps APIs to adapt to changes to their fields or restructure resource representations. In addition to that, versioning can help developers to offer non-stable, technology preview features as well as control the end-of-life of APIs.

The `version` refers to the stability and the state of an API. The version does not necessarily refers to a Keycloak release version number so that it can be mapped to different Keycloak release versions.

Here is an example of a resource path related with the group of functionalities provided by the Account Management REST API:

```
/apis/accounts/{realm}/v1
```

From the example above we can infer that the path refers to a specific `v1` version with the possibility to change the version and: 

* completely change the API
* do breaking changes to parts of the API
* remove functionalities or specific parts of the API
* provide non-stable or technology preview features
* the different APIs can evolve separately

#### Extensions

To better acommodate and separate the core API from extensions built on top of the API, developers should consider the following path format for extensions:

```
/apis/{API_GROUP}/{realm}/extensions/{version}
```

This separation aims to avoid confusion when consuming stable and official Keycloak APIs so that clients can differentiate if they are using some API provided by a third-party with no support from Keycloak team.

#### Technology Preview

To better acommodate and separate the core API from technology preview features, developers should use a beta version as the API version:

```
/apis/{API_GROUP}/{realm}/v1beta1
```

Technology preview APIs are related with features that have great potential to become fully supported. They provide the latest work in some area of interest and are offered with the intent to gather feedback from the community about new functionalities.

#### Store Resource

A resource store provides operations for a given resource. For example:

```
/apis/accounts/v1/1234/applications
```

The example above is representing a resource store path from where we can manage the `applications` for a user account with ID `1234`. Based on this path, clients can perform different operations based on the HTTP verbs or any controller resource defined to the store. Usually this type of resource is associated with CRUD operations for a specific resource.

When a resource fits under this category, the name should be a plural noun.

#### Controller Resource

Under some circumstances, the HTTP verbs are not enough to represent an operation that can be performed in a resource. In this case,
the resource path may indicate a specific action to be taken when accessed.

```
/apis/accounts/v1/1234/disable
```

The example above shows the usage of `disable` to operate accounts in order to disable them.

When a resource fits under this category, the name should be a verb, using camel-case if necessary.

### <a id="httpverbs"></a> Http Verbs

Based on the REST architectural style, HTTP Verbs should be used as follows:



### <a id="httpstatuscodes"></a> Http Status Codes

TODO

### Level of Service Maturity

Based on [Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html), developers MUST adhere to the maturity Level 2, where interactions with the API endpoints are based on the usage of HTTP Verbs and status codes to communicate status and errors.

### Security

API endpoints should be protected using bearer token authorization as the preferred mechanism.
 
In the absence of tokens and depending on the endpoint capabilities, developers can rely on any cookie created by Keycloak that binds the user with an active session on the server.
 
## Practices of API Design

### <a id="filtering"></a> Filtering

TODO

### <a id="pagination"></a> Pagination

TODO

### <a id="error_handling"></a> Error Handling

TODO

### <a id="securityp"></a> Security

TODO

### <a id="documentation"></a> Documentation

TODO

### <a id="cache"></a> Cache

#### HTTP GET Responses

By default, HTTP `GET` responses are cacheable by default. However, in order to avoid returning stale information API endpoints should not cache their responses when responding to a HTTP `GET` request.

As a general rule, responses from the Keycloak Server should always map to the current state.

[TODO]: Change this to possibly support ETag and Last-Modified.

#### 404 Responses

In order to reduce the amount of load when a resource does not exist, `404` response should include cache directives. This method is also
known as negative caching.

This method should be used carefully to not impact access to resources that can be made available some time in the future.