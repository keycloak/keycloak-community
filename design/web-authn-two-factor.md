# W3C Web Authentication - Two-Factor

* **Status**: Draft #1
* **JIRA**: [KEYCLOAK-7159](https://issues.jboss.org/browse/KEYCLOAK-7159)


## Motivation

W3C Web Authentication or WebAuthn for short is getting a lot of attention lately and it is becoming more and more 
important that Keycloak provides support for WebAuthn.

WebAuthn can provide stronger authentication than traditional authentication mechanisms like password and OTP. It is
also resistant to phishing attacks.

This is due to there not being any shared secret between the user and the server, but rather a public/private keypair
that is unique to the user and one web server.

WebAuthn has received attention from all major browser vendors and they already have or will soon have built-in support.

Another interesting aspect of WebAuthn is that it enables using different devices through a standard protocol. 


## Two-Factor vs Passwordless

WebAuthn can be used for Two-Factor authentication in combination with a password or it can be used for a passwordless
experience.

The Two-Factor experience is the most commonly requested today and it is also the easiest flow to achieve within 
Keycloak. As such Keycloak should focus on first adding support for Two-Factor with WebAuthn.


## Limitations in Keycloak

Keycloak has some limitations when it comes to two factor authentication. The login flows and the account
management console assumes a single two factor authenticator associated with an account, which out of the box is a 
software generated OTP through an application like Google Authenticator.

User should be able to register multiple two factor authenticators with their account, and select one as the default.
When prompted for two factor the user should be prompted with the default authenticator, but should have an option
to switch to any of the authenticators configured for their account.

Keycloak also does not currently deal well with enabling custom two factor authenticators to be added. First issue here
is the Account Console does not support customizing. Second issue is that the authentication flows does not allow
flexiblity allowing users to choose between different authenticators. Finally, it's not trivial for admins to configure
what two factor authentication mechanisms should be supported in the realm. This is currently done by creating custom
authentication flows, but ideally should be an option on the realm to enable/disable available two factor authenticators.


## Implementation stages

### Authenticator only

* [KEYCLOAK-9358](https://issues.jboss.org/browse/KEYCLOAK-9358) SPIKE Investigate W3C Web Authentication libraries
* [KEYCLOAK-9359](https://issues.jboss.org/browse/KEYCLOAK-9359) SPIKE Investigate how to test Web Authentication
* [KEYCLOAK-9360](https://issues.jboss.org/browse/KEYCLOAK-9360) Two factor authentication with W3C Web Authentication

The first step will be to implement an authenticator for Keycloak that can be used as a replacement for the current
OTP authenticator.

As we do not have support for managing two factor devices through the Account Console, initially the admin is required
to register a required action against a user account for the user to be prompted to setup WebAutn.

#### Libraries

This will include researching what Java WebAuthn libraries can be used. There are currently two of interest:

* [Yubico](https://github.com/Yubico/java-webauthn-server)
* [WebAuthn4j](https://github.com/webauthn4j/webauthn4j)

#### Testing

We also need to be able to do automated functional tests for the integration. This should ideally involve a real
browser and an emulated device so that we can verify registration flows and authentication flows, including JS parts of
WebAuthn libraries.


### Initiating registration from application

* [KEYCLOAK-9692](https://issues.jboss.org/browse/KEYCLOAK-9692) Initiate registration of Security Key from application 

First step to improve here will be to allow applications, including the Keycloak Account Console, to initiate the 
registration of the Security Key instead of requiring the admin to do it.

This will be done by adding support for Application Initiated Actions, which allows application to request a user to 
perform an action like registering the Security Key through a standard OAuth redirect flow.


### Allow users to configure two factor authenticators

* [KEYCLOAK-6197](https://issues.jboss.org/browse/KEYCLOAK-6197) New Account console

Through the Account Console users should be able to add and remove two factor authenticators supported by the realm. This
functionality will be added to the new Account Console, but will most likely not be added to the old console. 


### Allow user to have multiple two factor authenticators registered

* [KEYCLOAK-565](https://issues.jboss.org/browse/KEYCLOAK-565) Allow users to have multiple two factor authenticators

Users should be able to register multiple two factor authenticators to their account and choose which authenticator to
use during login.


### Allow admin to manage supported two factor authenticators

* [KEYCLOAK-9693](https://issues.jboss.org/browse/KEYCLOAK-9693)

An admin should be able to easily configure what two factor authenticators should be supported in a realm. This should
not require creating complex authentication flows, but should be a simple list where the admin can add/remove authenticators.


### Allow admin to manage users two factor authenticators

* [KEYCLOAK-9694](https://issues.jboss.org/browse/KEYCLOAK-9694)

An admin should be able to view two factor authenticators configured for a user. An admin should also be able to remove
a specific authenticator for the user.
