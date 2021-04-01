# Validation SPI

* **Status**: Draft #1
* **JIRA**: [KEYCLOAK-2045](https://issues.redhat.com/browse/KEYCLOAK-2045)

## Motivation

Validation is currently scattered around several places and implemented in different ways 
which makes it very hard to reuse validation logic in a consistent way. 
Besides some static helper functions that are used in different places, there are also many 
component-specific validations that are directly wired to the surrounding business logic.
A unified and consistent validation mechanism is currently missing.

There have been several approaches to establish a validation infrastructure, but they have 
not been followed consistently.

The goal of this proposal is to provide an extensible and unified validation SPI 
that can be used across the Keycloak code-base. The need for a new validation API was highlighted 
by the design discussions for the [user-profile](https://github.com/keycloak/keycloak-community/blob/master/design/user-profile.md) support.

The goals of this proposal can be summarized as:

* Provide a uniform and extensible validation SPI
* Consolidate the scattered validation logic with the new Validation SPI

## Design Principles

The design of the new validation SPI should follow the following design principles:

- Simplicity
The API should have a small API surface and be easy to use.

- API Granularity
The API should support low-level validations.

- Efficiency
Validations should be fast and have little overhead.

- Extensibility
There should be support for built-in as well as user provided validations.
The new SPI should be easy to extend, most custom validations should only need one implementation class. 
Developers should be able to provide reusable custom validation logic components.

- Versatility
Validations should be flexible and should support validatation of simple values and complex objects.
It should be possible to parameterize validation executions. Additionally there should be a way to 
check if the parameterization for a validation is valid.
Validations should be composable and it should be possible to access other validations from within a validation. 

## Considered Options

### Bean Validation API

The JSR 380 Bean Validation 2.0 is a mature and rich validation framework that is extensively used in Java applications.

- Optimized for validation of complex Java bean objects based on declarative configuration rather than single values.

Although, value validations are possible via `javax.validation.Validator#validateValue`, they require a bean class 
with a property with the annotated constraints.
```java
	<T> Set<ConstraintViolation<T>> Validator#validateValue(Class<T> beanType,
												  String propertyName,
												  Object value,
												  Class<?>... groups);
```
[Example for validating a value via Validator#validateValue](https://gist.github.com/thomasdarimont/e6e375819b3b1adff86b7669e5b033b4).

- hibernate-validator is currently not used in Keycloak, thus we'd need to add a dependency for:
```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.0.12.Final</version>
</dependency>
```
- No direct support for validation (Constraint) lookups by name, however the user-profile support needs a way to lookup validations by name.  
`ConstraintValidator` lookups could be done with a custom mapping of constraint name to `ConstraintValidator` class.
- It is possible to dynamically configure constraints programmatically and from external configuration. 
Hibernate-validator supports programmatic validation, see [Hibernate Validator Programmatic constraint definition and declaration](https://docs.jboss.org/hibernate/validator/5.3/reference/en-US/html/ch11.html#section-programmatic-api).
There is also support for [XML Configuration in beanvalidation Spec](https://beanvalidation.org/2.0/spec/#xml).

- Definition of [custom validations is quite involved](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#validator-customconstraints) - you need 2 classes (annotation + constraint) + error message + META-INF service manifest.

All this was considered and evaluated in a small prototype, but then it was concluded that it makes more sense to go 
with a custom implementation that provides a narrow API. Additionally custom extensions should be easier and provide 
a better integration with the existing Keycloak infrastructure. Further more dynamic validation logic lookups 
and configuration should be possible and easy to do. 

Note that it is still possible to provide a validation/validator implementation that leverages bean validation. 
Also, bean-validation still works quite well for declarative validation in JAX-RS endpoints.

### Fully fledged Validation SPI

An earlier attempt at a unified validation SPI can be found here [KEYCLOAK-2045 Add support for flexible Validation](https://github.com/keycloak/keycloak/pull/7324) and was discussed on the [keycloak-dev mailing list](https://groups.google.com/g/keycloak-dev/c/-XjQu7rn56s).

After some discussion, it became apparent that this proposal provides a very comprehensive validation framework that 
offers numerous functionalities that go beyond the requirements of user profile validation and other validations in Keycloak.

However, the API was very complex and not easy to understand. Therefore, it was agreed that @thomasdarimont would revise 
the proposal again and present it in a smaller form. This has resulted in the new "Simple Validation SPI" proposal.

## Proposed Validation SPI

### PoC for new Validator SPI

The Proof of concept implementation can be found here in the PR [KEYCLOAK-2045 Simple Validation API #7887](https://github.com/keycloak/keycloak/pull/7887).

Some usage examples can be found here in the [ValidatorTest](https://github.com/thomasdarimont/keycloak/blob/issue/KEYCLOAK-2045-Simple-Validation-SPI/server-spi-private/src/test/java/org/keycloak/validate/ValidatorTest.java).

### Core API
The proposal focus around the following types:
1. `Validator`: `Provider` interface for validation mechanics.
1. `ValidationContext`: Holds state of a sequence of validations.
1. `ValidationResult`: Denotes the outcome of the validation and contains a collection of all the `ValidationError` found during validation.
1. `ValidatorConfig`: Denotes the configuration for a Validator. A typed wrapper around a `Map<String,Object>`.
1. `ValidationError`: Denotes a concrete error found during validation that was reported by a `Validator`.
1. `Validators`: API Facade to provide lookups for built-in and user provided validations.

### Support API
The following types provide the integration with the Keycloak infrastructure:
1. `ValidatorFactory`: `ProviderFactory` interface for custom validation contributions.
1. `ValidatorSPI`: Provides the `validator` SPI.
1. `CompactValidator`: Convenience class for built-in validations.

The proposed package name is `org.keycloak.validation`. Note the current PR [KEYCLOAK-2045 Simple Validation API](https://github.com/keycloak/keycloak/pull/7887) uses the package name `org.keycloak.validate` to provide a transition path for the current API.

### Module location
Initially, the package `org.keycloak.validation` is treated as an internal API, so the component 
is provided by the module `server-spi-private`. Once the API is declared as a public API, the 
package can be moved to the `server-private` module.

### Validator interface

The proposed Validator interface.

```java
public interface Validator extends Provider {
    /**
     * Validates the given {@code input} with an additional {@code inputHint} and {@code config}.
     *
     * @param input     the value to validate
     * @param inputHint an optional input hint to guide the validation
     * @param context   the validation context
     * @param config    parameterization for the current validation
     * @return the validation context with the outcome of the validation
     */
    ValidationContext validate(Object input, String inputHint, ValidationContext context, ValidatorConfig config);

    // convenience methods, which all delegate to the validate(...) method above
    default ValidationContext validate(Object input) {...}
    default ValidationContext validate(Object input, ValidationContext context) {...}
    default ValidationContext validate(Object input, String inputHint, ValidationContext context) { ... }
    // ... more convenience methods
}
```

### Validator lookups

`Validator` instances can be found by name via the `Validators.validator(keycloakSession, validatorId)` method.
This is necessary to support dynamic validator lookups based on user-profile configurations, for example:
```java
Validators.validator(keycloakSession, LengthValidator.ID);
```

Built-in `Validator` instances can be accessed directly via static methods on the `Validators` class, for example: 
```java
Validators.lengthValidator()
```

`KeycloakSession` could provide like `KeycloakSession#validators()` that returns a `Validators` instance configured with 
the current session.
```java
keycloakSession.validators().validator(LengthValidator.ID);
```

### Validating input

Depending on the use case we support different variants of validating input.

The `Validator#validate(..)` method returns a `ValidationContext` that can be used for consecutive validations.
The result of a validation is denoted by a `ValidationResult` which contains the validation outcome (valid true/false) 
including a collection of `ValidationError`'s found during validation.

For brevity, the result of a single validation can be read via `ValidationContext#isValid()`. 
The result of consecutive validations can be obtained by calling `ValidationContext#toResult` which yields an `ValidationResult`. 

For simple validations we allow single one-short validation executions.
Those validations can use the static methods on `Validators` to access a specific `Validator` instance. 
```java
Assert.assertTrue(Validators.notEmptyValidator().validate("a").isValid());
```

More elaborate validations that need to be performed consecutively can use the `ValidationContext` to accumulate 
validation errors. `Validator` executions can also be customized via an optional `ValidatorConfig`.
```java
String input = "user@acme.com";
String inputHint = "email";

ValidationContext context = new ValidationContext(session);
Validators.emailValidator().validate(input, inputHint, context);
Validators.patternValidator().validate(input, inputHint, context, ValidatorConfig.configFromMap(Collections.singletonMap("pattern", ".*@acme.com")));

ValidationResult result context.toResult();
Assert.assertTrue(result.isValid());
```

Validations that need access context information can use the `KeycloakSession` from the `ValidationContext` to access
the `KeycloakContext` which may expose information like the current `RealmModel` and `ClientModel`.

`Valdiator` implementations can also use the `ValidationContext` to access the `RealmModel` as shown before, to
honor realm settings during validation, e.g. for email uniqueness.

```java
String input = "tester";
String inputHint = "username";

ValidationContext context = new ValidationContext(session);

session.validators().length().validate(input, inputHint, context, ValidatorConfig.configFromMap(Collections.singletonMap("min", "2")));
session.validators().user().usernameUnqiue(input, context);
session.validators().validator("swearwords").validate(input, inputHint, context);

ValidationResult result = context.toResult();

Assert.assertFalse(result.isValid());
Assert.assertEquals(1, result.getErrors().size());

ValidationError[] errors = result.getErrors().toArray(new ValidationError[0]);
ValidationError error = errors[0];
Assert.assertNotNull(error);
Assert.assertEquals(LengthValidator.ID, error.getValidatorId());
Assert.assertEquals(inputHint, error.getInputHint());
Assert.assertEquals(LengthValidator.MESSAGE_INVALID_LENGTH, error.getMessage());
Assert.assertEquals(input, error.getMessageParameters()[0]);
```

### Adding a new Validator

New built-in `Validator`'s can implement the provided `CompactValidator` interface.
A `CompactValidator` needs to provide the following two methods:
1. `String getId()` 
1. `ValidationContext validate(Object input, String inputHint, ValidationContext context, Map<String, Object> config)`

Depending on the requirements the method `ValidationResult validateConfig(Map<String, Object> config)` can be implemented
to support validation of config values that are used to parameterize the validation.

Users can provide their own validations via the `provider` SPI by implementing the `CompactValidator` interface as a singleton 
class or by creating two separate classes by implementing `Validator` and `ValidatorFactory` respectively. 
In either case, users need to register the new validator via Keycloaks SPI mechanism for the `org.keycloak.validation.ValidatorFactory` service.

A simple custom `Validator` might look like this:

```java
@AutoService(ValidatorFactory.class) // annotation processors like googles auto-service can generate the service manifest
public class CustomNonNullValidator implements CompactValidator {

    public static final CustomNonNullValidator INSTANCE = new CustomNonNullValidator();

    public static final String ID = "notNull";

    public static final String ERROR_NULL = "error-invalid-null";

    private CustomNonNullValidator() {}

    @Override
    public String getId() {
        return ID;
    }

    @Override
    public ValidationContext validate(Object input, String inputHint, ValidationContext context, ValidatorConfig config) {

        if (input == null) {
            context.addError(new ValidationError(ID, inputHint, ERROR_NULL, input));
            return context;
        }

        return context;
    }
}
```

### Validating Validator configuration

Validator configurations might be specified via the admin-console. Therefore it must be possible to validate validator configuration. Validator configurations can be valdiated via the `ValidationResult validateConfig(ValidatorConfig config)`
method of the `ValidatorFactory`. Custom Validators which implement the `CompactValidator` interface can implement the method
in the same class. This supports concise and self-contained Validator components.

```java
public interface ValidatorFactory extends ProviderFactory<Validator> {
    /**
     * Validates the given validation config.
     *
     * @param config the config to be validated
     * @return the validation result
     */
    default ValidationResult validateConfig(ValidatorConfig config) {
        return ValidationResult.OK;
    }

}
```

### Reporting Validation Errors

Errors detected during validation are reported via the `ValidationContext#addError(ValidationError)` method.
`ValidationError` provides a constructor which to capture information about:
- The id of the `Validator` where the `ValidationError` was detected.
- The `inputHint` which could be the name of a (nested) attribute that is validated.
- The i18n message (key) for the error message.
- A var args array with the message parameters that can later be used to render error message.

Note that `ValidationError` provides methods like `ValidationError#getMessageParameters` and `ValidationError#getInputHintWithMessageParameters` that help with message rendering too.

Adding an `ValidationError` to a `ValidationContext` toggles it's `valid` status to `false`.

```java
public ValidationContext validate(Object input, String inputHint, ValidationContext context, ValidatorConfig config) {
    // ...
    String string = (String) input;
    if (string.trim().length() == 0) {
        context.addError(new ValidationError(ID, inputHint, MESSAGE_BLANK, input));
    }
    return context;
}
```
### Rendering Error Messages

Validation errors should provide a internationalizable message (key) for `ResourceBundle` lookups as well as optional 
message parameters to provide information about the validated input as well as the valdiated value.

The proposed `ValidationError` class provides a `formatMessage` method which takes a `BiFunction` to abstract
the actual error message rendering. The first parameter is the `message` key followed by an array of message parameters.

The first message parameter will be the `inputHint` passed into the `Validator#validate(..)` followed by the `messageParameters`.

```java
public String formatMessage(BiFunction<String, Object[], String> formatter) {
    Objects.requireNonNull(formatter, "formatter must not be null");
    return formatter.apply(message, getInputHintWithMessageParameters());
}
```

This allows error message rendering like:
```
error-invalid-value={0} is invalid: <{1}>  --> address.zip is invalid: <null>
```

Here is an [example of a custom error message rendering](https://github.com/keycloak/keycloak/pull/7887/files#diff-e8a31e5e873f1c5826f3c2ce581056a85457d21bc60504d06873410ef36f9bcbR119) with the proposed API, look for the `public void formatError()` test.

### Suggest Built-in Validations

It is suggested to provide a set of built-in validations.

1. Length - Checks if a string certain length
1. NotEmpty - Checks if a string, collection, map is not empty
1. Number - Checks if a given value is a number (Integer / Double)
1. Pattern - Checks if the given value matches a pattern
1. URI - Checks if the given value is a valid URI
1. Email - Checks if the given value is a valid email address

### Integration plan

1. Prepare Validation SPI in `server-spi-private` in the `org.keycloak.validation` package and consolidate the other Validation APIs to use the new Validation SPI. Note, that most of the validation code can be directly replaced.
1. Adapt User-Profile support to use the new Validation SPI
1. Promote Validation SPI to `server-spi` as an official API and expose validator lookup via KeycloakSession

### Open Questions
1. Should users be able to override built-in Valdators?
1. How should the error message keys look like?
1. Do we need a concept like a `validation hook` to allow running user provided Validators before/after/instead of built-in validators?
1. Should ValidationErrors be Comparable? Do we need concepts like `order` or `priority`?

### Further Work
1. Add support to report `ValidationWarning`s next to `ValidationError`s to report problems, that don't fail the validation. A generic base type could be `ValidationProblem`.
1. Make Validations executable in the Client side for validation in admin-console.
