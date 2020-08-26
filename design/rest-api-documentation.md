# REST API documentation

* **Status**: Notes
* **JIRA**: 
  * [KEYCLOAK-9655](https://issues.jboss.org/browse/KEYCLOAK-9655)
  * [KEYCLOAK-14744](https://issues.jboss.org/browse/KEYCLOAK-14744)


## Motivation

The value of an API to a large extent depends on how easy it is to consume. Documentation plays an important part in whether an API is useful. The main issue with the Keycloak documentation these days is to keep it up to date with the changes into the code, forcing developers to dig into the sources sometimes to understand how the API works.

## Microprofile OpenAPI

The Microprofile OpenAPI specification defines a standard mechanism to document REST endpoints using annotations or a pre-generated JSON. That was introduced since *WildFly 19* making things easier to be consumed by [Swagger](https://swagger.io/) tooling. 

To document the external API that's exposed by the [Account REST endpoint](https://github.com/keycloak/keycloak/blob/9369c7cf4d2227ae8be40a6129ca21200ae16a9d/services/src/main/java/org/keycloak/services/resources/account/AccountRestService.java#L464-L517), we need to update the source code:

```
public class AccountRestService {

    ...

  @Path("/applications")
  @GET
  @Produces(MediaType.APPLICATION_JSON)
  @Operation(description = "List of applications")
  @APIResponses(value = {
  	@APIResponse(
  			responseCode = "200",
  			description = "Successful, List of applications"
      ),
  	@APIResponse(
  			responseCode = "401",
  			description = "Unauthorized, Failure"
      ),
  	@APIResponse(
  			responseCode = "500",
  			description = "Error in getting the list of applications"
      )
  })
  public Response applications(@QueryParam("name") String name) {
  	...
  }

}
```

A new dependency must be introduced in the Keycloak `pom.xml` file:

```
<dependency>
    <groupId>org.eclipse.microprofile.openapi</groupId>
    <artifactId>microprofile-openapi-api</artifactId>
</dependency>
```

By spinning up the Keycloak server, developers should be able to send an HTTP request to http://localhost:8080/auth/openapi and get the OpenAPI document. Here is an example:

```
$ curl http://localhost:8080/auth/openapi
---
openapi: 3.0.1
info:
  title: Account REST API
  contact:
    name: Keycloak team
    url: https://keycloak.org
    email: keycloak-dev@googlegroups.com
  license:
    name: Apache 2.0
    url: http://www.apache.org/licenses/LICENSE-2.0.html
  version: v1alpha1
servers:
- url: /auth
tags:
- name: Account REST API
  description: Account REST API operations.
- name: OpenAPI Example
  description: Get a text in various formats
- name: Account REST service
  description: REST API for the Account REST service
paths:
  /account/applications:
    get:
      tags:
      - Account REST API
      description: List of applications
      parameters:
      - name: name
        in: query
        schema:
          type: string
      responses:
        "401":
          description: Unauthorized, Failure
        "500":
          description: Error in getting the list of applications
        "200":
          description: Successful, List of applications
```

As we can see, with minimal effort it is possible to produce a JSON document that describes the functionalities of our API and can be used as a foundation for UI tools like Swagger to visualize and interact with our APIs. This document is served as a static file.

## Swagger UI

Swagger UI allows developers to visualize and generate documentation based on the OpenAPI specification. There are several ways of doing this:

1. Inserting the URL in the [Swagger generator](https://generator.swagger.io/) which is available online. For example: http://localhost:8080/auth/openapi
2. Spinning up a Docker image: 
```
docker pull swaggerapi/swagger-ui
docker run -p 80:8080 swaggerapi/swagger-ui
```   
3. [Swagger Codegen](https://github.com/swagger-api/swagger-codegen#building). The best way to generate static HTML files for documentation.

## Implementation plan

1. Upgrade Keycloak distribution to WildFly 20.0.1.Final

The current Keycloak distribution still use legacy feature packs from WildFly. In order to provide a subsystem xml file so that it can be used, we need the latest changes from [introduced in WildFly 20.0.1.Final](https://issues.redhat.com/browse/WFLY-13766).

2. Introduce Microprofile OpenAPI subsystem in the Keycloak distributions
3. Update the new Account REST API with the proper annotations
4. Make sure that nothing is broken with the Quarkus distribution

### Future work items

The items below were included as nice to have because it's fairly easy to get these documents out of date and introduce it requires more discussion.

* Generate static HTML files for the REST API
* Generate static OpenAPI document and add to the release distribution
  
