# Observerability

* **Status**: Notes
* **JIRA**: [KEYCLOAK-8288](https://issues.jboss.org/browse/KEYCLOAK-8288)


## Motivation

* Need to know if Keycloak is running
* Need to find and debug issues in production


## Types of Observerability

Health - a health check endpoint returns a simple binary up or down state of the system. This is very useful in managed
environments like Kubernetes and OpenShift where the controller can automatically stop broken instances.

Metrics - metrics provide a more detailed view of the health and state of the system. From metrics it is possible to
find out how the system is behaving, not just if the system is up or down. Examples here could be observing a large
amount of error responses or response times increasing beyond a given threshold.

Logging - metrics provides the details on how the system is behaving, but will not always be able to provide sufficient
information to debug the system. This is where logging comes in. Through well defined logging it is possible to further
obtain information on how the system is behaving and resolving potential issues.

Tracing - in a distributed system it can be difficult to find the journey a single transactions takes through the
systems. With distributed tracing it is possible to do exactly this.


## Observerability in WildFly

WildFly 16 comes with built-in support for Observerability, which will make it relatively straightforward to add the same
to Keycloak.

### Health

Health endpoint is enabled by default in WildFly 16 at:

* http://localhost:9990/health

WildFly implements the [MicroProfile Health](https://microprofile.io/project/eclipse/microprofile-health) specification. 

### Metrics

Metrics endpoint is enabled by default in WildFly 16 at:

* http://localhost:9990/metrics

WildFly implements the [MicroProfile Metrics](https://microprofile.io/project/eclipse/microprofile-metrics) specification. 
Metrics can be exposed in Prometheus or JSON format. 

By default only metrics for the base (JVM) is available. Further metrics can be enabled with `-Dwildfly.statistics-enabled=true`, 
which will enable statistics for all subsystems that support it. It is also possible to enable statistics for specific
subsystems. For example `-Dwildfly.undertow.statistics-enabled=true` will enable statistics for Undertow.

It is also possible to change the prefix of metrics from `wildfly` to `keycloak` by simply changing the `prefix` 
attribute on the microprofile-metrics-smallrye subsystem (or `wildfly.metrics.prefix` system property).

To add application (Keycloak) specific metrics WildFly supports the MicroProfile Metrics specification. This is not
useful to Keycloak as it requires CDI, which Keycloak by design has chosen not to use. This leaves us with the following
options:

* DMR - WildFly automatically exposes metrics from subsystems that expose attributes with the type `metrics`. One big
challenge here is the amount of boilerplate required to work with DMR.

* SmallRye Registry - SmallRye exposes static registries that allows registering additional metrics. One challenge here is
that there is an issue in SmallRye 1 that makes it not possible to use tags/labels. This is resolved in SmallRye 2, but
WildFly is using SmallRye 1. 

* WildFly Registry - WildFly has its own registry used to expose metrics from subsystems. This could be an option, but would
require changes to the API as it is using DMR specific APIs today.

### Logging

JSON formatted logging can be enabled in WildFLy with the following CLI commands:

    /subsystem=logging/json-formatter=json:add(pretty-print=true, exception-output-type=formatted)
    /subsystem=logging/console-handler=CONSOLE:write-attribute(name=named-formatter, value=json)
    :reload
    
More info in [EAP Documentation](https://access.redhat.com/documentation/en-us/jboss_enterprise_application_platform_continuous_delivery/14/html/configuration_guide/logging_with_jboss_eap#configure_json_log_formatter).


## Observerability in Keycloak

### Health

Unclear if we need to do some additional health checks in addition to what WildFly does.

### Metrics

In addition to metrics exposed by WildFly we will expose some Keycloak specific metrics. For example:

* User login
* User login failures
* User count
* Session count
* Client count

Exactly what additional metrics we should expose in the first round is still to be decided.

We will start small with the aim to expand on the available metrics in the future.

### Logging

WildFly covers JSON formatted logging. We will simply add some documentation.

### Tracing

Most transactions to Keycloak are directly from the browser and do not invoke other services. Applications invoke Keycloak
to exchange code for a token, to refresh tokens and finally services may invoke Keycloak to verify tokens. These are
simpler use-cases as we haven't seen much demand for tracing in Keycloak. Hence distributed tracing is a low priority
and will be considered in the future. 


## Resources

* http://www.mastertheboss.com/jboss-server/jboss-monitoring/how-to-manage-wildfly-metrics
* http://www.mastertheboss.com/jboss-server/jboss-monitoring/monitoring-wildfly-with-prometheus
* http://www.mastertheboss.com/jboss-server/jboss-monitoring/wildfly-application-monitoring
* https://microprofile.io/project/eclipse/microprofile-health
* https://microprofile.io/project/eclipse/microprofile-metrics

Non-public resources:

* https://docs.google.com/presentation/d/1YU6kGjereV8-tJKXvYtmNIwiLu1TyF24MtXHMx-HFBI
* https://docs.google.com/document/d/1v0cs_RdcX4ZKuOUpM9frbFjs9KG9DMLFfOnvt124YIY
* https://docs.google.com/spreadsheets/d/1uZnsXTu3W0oXxfSxiB78WBcuKf27hEs_txDsxdkbPjY 

### WildFly DMR example

For example for JMS Queue, you create metric definition for DMR 
[JMSQueueDefinition](https://github.com/wildfly/wildfly/blob/master/messaging-activemq/src/main/java/org/wildfly/extension/messaging/activemq/jms/JMSQueueDefinition.java#L98) 
and you associate a read handler that will read the value for the Artemis objects and return it to DMR 
[JMSQueueReadAttributeHandler](https://github.com/wildfly/wildfly/blob/master/messaging-activemq/src/main/java/org/wildfly/extension/messaging/activemq/jms/JMSQueueReadAttributeHandler.java#L66)

### SmallRye registry example

[MetricRegistries](https://github.com/smallrye/smallrye-metrics/blob/master/implementation/src/main/java/io/smallrye/metrics/MetricRegistries.java) 
to access the static Smallrye registries - they have 3 different ones:

* base (for JVM metrics)
* vendor (for vendor / app server metrics)
* application

### Using WildFly's own registry

The base class is 
[MetricCollector](https://github.com/wildfly/wildfly/blob/master/microprofile/metrics-smallrye/src/main/java/org/wildfly/extension/microprofile/metrics/MetricCollector.java), 
but the API would have to be adapted a bit for our use case.