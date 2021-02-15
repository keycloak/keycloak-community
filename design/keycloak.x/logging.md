# Keycloak X - Logging

* **Status**: Notes
* **JIRA**: TBD

## Current state

Keycloak.X leverages uses JBoss Log Manager and the JBoss Logging facade thats provided by Quarkus.

By default the log messages are unstructured text with timestamps and thread and call-site information
printed to the console or a log file. There are many ways to parse this format, which often
require an additional translation step to create a structured log message from a given log line.

## Desired state

Keycloak should provide native support for structured logging to avoid additional translation steps.
JSON is a commonly used format for structured logging and should be supported natively, and other formats like GELF or binary encodings should also be possible.

It should also be possible to add additional context information to a log message, 
e.g. by leveraging the MDC (Mapped Diagnostic Context) support provided by JBoss-Logging.

## Additional Thoughts

- I think it makes sense to keep the log messages plain text in %dev profile.
- Should we enable JSON Logging by default in prod, or put it behind a short-flag? (`log-format=json` vs. `log-format=plain`)
- Should be already allow Quarkus JSON logging support? KEYCLOAK-17057
- How should we handle open-tracing information in logs? Already supported via MDC Logging?
- Could / Should we provide more advanced log patterns (e.g. with OpenTracing log fields)?

## Related Designs 

- [Observerability](https://github.com/keycloak/keycloak-community/blob/master/design/observerability.md)

## Related Issues

- [KEYCLOAK-17057 | Add support for JSON Logging in Keycloak.X](https://issues.redhat.com/browse/KEYCLOAK-17057)