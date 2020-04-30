# Default Hostname Provider

* **Status**: Draft #1
* **JIRA**: [KEYCLOAK-11728](https://issues.jboss.org/browse/KEYCLOAK-11728)

## Changes

### Draft #1

* `url` renamed to `frontendUrl`
* `backChannelUrlFromRequest` renamed to `forceBackendUrlToFrontendUrl`
* Mentioned explicitly that `frontendUrl` and `forceBackendUrlToFrontendUrl` are options in standalone.xml
* Expanded to section for realm URL override

## Motivation

Today we have two hostname providers `request` and `fixed`. The `request` provider will use the `Host` header from 
request to infer the Keycloak hostname, while the `fixed` provider will use a hard-coded value for the hostname. Further,
the `fixed` provider has config options to set ports for http and https, as well as the ability to force to always use
https.

There are a few issues with the current providers that could be addressed with a new and improved provider.

When a fixed hostname is set all URLs advertised by Keycloak uses the same hostname. This means it is not possible to
list different hostname for front-channel URLs (URLs visited by the user-agent) and back-channel URLs (URLs visited by clients).

It is not possible to override the context-path. It may be useful to be able to host Keycloak publicly through a reverse
proxy on for example `https://auth.mycompany.com` which would be mapped to `https://keycloak.mycompany.local/auth/realms/myrealm`.

It is not trivial to setup a fixed hostname. To setup a fixed hostname the provider has to be changed and the fixed
provider has to be configured. Further, to set a fixed hostname for a single realm the fixed provider has to be enabled
first.

## Extending the Hostname SPI

The Hostname SPI should be extended with:

* Ability to retrieve different hostnames for front-channel and back-channel URLs
* Ability to set the context-path

## Default Hostname provider

We should introduce a new default Hostname SPI that will be used by default for new installations. We will recommend
changing to this provider in upgrade guides, but will keep the old providers for a while.

The default provider will use the hostname from the request by default, but will the following configuration options for the provider in standalone.xml:

* `frontendUrl` - value will be the base URL for front-channel requests to Keycloak (for example `https://mycompany.com/auth`)
* `forceBackendUrlToFrontendUrl` - will allow forcing the back-channel URL to be same as front-channel URLs (default will be false)

Realms will have an option `realmFrontendUrl` that allows overriding the base URL for front-channel requests to Keycloak. This will also allow a LB to map for example `https://auth.mycompany.com` to `https://keycloak.mycompany.internal/auth/realms/external-realm`.

### OpenID Connect Discovery endpoint

The OpenID Connect Discovery endpoint will use the front-channel URL for `issuer` and `authorization_endpoint` (and other
front-channel requests) while it will use the back-channel URL for `token_endpoint` and `jwks_uri` (and other
back-channel requests).

By default the back-channel URLs will be based on the request. As such a client that is accessing Keycloak through
`https://mycompany.com/auth/` would get a `token_endpoint` URL starting with `https://mycompany.com/auth`, while a 
client that access Keycloak with `https://keycloak.mycompany.local/auth` would get a `token_endpoint` starting with
`https://keycloak.mycompany.local/auth`.

### Support different URLs for front-channel and back-channel requests in adapters

[KEYCLOAK-6073](https://issues.jboss.org/browse/KEYCLOAK-6073) is a feature request to make it possible to have
adapters use one URL for back-channel requests and another for front-channel requests. The idea there is to basically
have two different URLs configured for the Keycloak server in adapters.

This approach has to downsides. First and most importantly, it does not provide support for third-party libraries.
Secondly, all adapters now have to be configured with two different URLs and there is no control on the Keycloak server
side of what URLs should be used.

Instead of the above the adapters should use the OpenID Connect Discovery endpoint.

As an example the `auth-server-url` in the adapter would be set to `https://keycloak.mycompany.local/auth`. In the
Keycloak server the `url` of the default hostname provider would be set to `https://auth.mycompany.com`.

This would make the adapter and third-party libraries use the correct URLs without additional configuration on the 
adapter side.


### Discussions and relevant information

* [GitHub issues - feedback from Marek](https://github.com/keycloak/keycloak-community/issues/34)
* [GitHub PR - Support different URLs for front and back channel requests in adapters](https://github.com/keycloak/keycloak/pull/6117)
* [JIRA - Support different URLs for front and back channel requests in adapters](https://issues.jboss.org/browse/KEYCLOAK-6073)


