# Keycloak X - Providers

* **Status**: Notes
* **JIRA**: TBD

How providers will look like in the future is still very much to be decided.

One option is to only support remote providers through REST and GRPC. This approach would have loads of benefits:

* Versioned and more self-contained APIs
* Simpler to test
* Support any language and framework you can think of
* Sandboxing, supporting custom providers in a SaaS world

It'll also have downsides as well, including performance overhead, especially in more fine-grained providers such as
protocol mappers.

There are options to do polyglot providers deployed in-process as well. We already have JS support for some providers 
such as protocol mappers. This does have the downside of it being much harder to support versioned APIs.

Quite likely we'll end up supporting both in-process and remote providers. 