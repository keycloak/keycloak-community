# KeyCloak Gatekeeper Logging

* **Status**: Notes
* **JIRA**: [KEYCLOAK-12100](https://issues.jboss.org/browse/KEYCLOAK-12100)

## Motivation

Keycloak gatekeeper currently is setup with logging with production configuration or verbose configuration.
Adding further configuration options to specify below details would allow better security audits and easier debugging at different environments.

1. Log level
2. Time encoding
3. Log sampling
4. Log encoding


## Implementation details

### Current Configuration Options

We have below mentioned flags related to zap logging configuration for customization already in keycloak gatekeeper.

1. `--enable-json-logging`: To switch to JSON or back to console(text) logging.
2. `--disable-all-logging`: Disable All Logging
3. `--verbose`: Uses Development and Debug Logging Level.

To add further customizations, users can use the zap flagset and specify flags on the command line.
The zap flagset includes the following flags that can be used to configure the logger:

* `--zap-level` string or integer - Sets the zap log level (`debug`, `info`, `error`, or an integer value greater than 0). If 4 or greater the verbosity of client-go will be set to this level.
* `--zap-stacktrace-level` - Set the minimum log level that triggers stacktrace generation (default: `error`)
* `--zap-time-encoding` string - Sets the zap time format (`epoch`, `millis`, `nano`, or `iso8601`)

These flag sets would override the existing configuration if they are set otherwise, the configuration specified from existing flags would be valid.

This idea is based on operator sdk logging configuration.
https://github.com/operator-framework/operator-sdk/blob/master/doc/user/logging.md#default-zap-logger


Current Example Implementation:

https://github.com/keycloak/keycloak-gatekeeper/pull/512
