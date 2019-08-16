# User Event SPI

* **Status**: Draft v0.1
* **Author**: [vmuzikar](https://github.com/vmuzikar)

## Motivation

Current Event SPI design has several limitations. E.g. it's not defined when an event is fired exactly, there are no pre-
and post-events. There's also no way of pre-events intercepting for approve/reject functionality.

## Use cases

* A user is removed from Keycloak. All related records in an external system are removed before the user is permanently
  deleted (e.g. eshop orders).
* A user self-registers in Keycloak. Their registration must be approved by an external system (either human or automated)
  before the new account is actually fully created.
* A user self-edits their email address. The change must be approved first by an external system before it's actually
  applied.

## Design proposal

To overcome the current Event SPI limitations, a new "domain-specific SPI" will be implemented – User Event SPI.

Key changes over current generic Event SPI:
* Each event is fired twice – as a pre- and post-event.
* Pre-events are "interactive" – listener can approve, reject or delegate (for later decision by an external system)
  each pre-event. 
* The same User Event SPI will be used everywhere – admin console and REST endpoints, user registration form,
  account console and REST endpoints, etc.
* The event types will be much more detailed, not just usual CRUD. There'll be separated events for user self-registration,
  creation through Admin REST API, Account Console, etc.

### Terminology

User Action (in this context) is a process which fires the User Event. This can be e.g. a user creation through Admin REST
API, user self-registration, user field modification, assigning a user role, etc.

Pre-event is an event that is fired after all other verifications passed (authorization, request validation, etc.), just
before persisting any changes.

Post-event is an event that is fired after a User Action has finished completely and all changes are persisted.

### Interactive pre-events

If a listener supports events interception, it needs to explicitly approve or reject each pre-event. This can be done
either synchronously (the approve/reject decision is made right away by the listener), or the listener can signal that
the decision is delegated for asynchronous processing.

**When an event is approved** the SPI doesn't interfere with the User Action in any way.

**When an event is rejected:**
* The SPI throws an exception.
* Based on this exception the User Action is terminated and some meaningful error message is sent.

**When an event is delegated:**
* The SPI stores any necessary data (representations, etc.) required later in case this event is approved.
* The SPI throws an exception.
* Based on this exception the User Action is intercepted and some meaningful error message is sent.
* A one-time token is created. The token is used later by an external system to approve or reject this event.
* The listener informs the external system that approval is required.
* Later the external system approves or rejects the event. The SPI is then responsible for "resuming" the User Action
  in case it's approved.

#### Integration with Red Hat Process Automation Manager (former BPMS)

The external system the events are delegated to can be BPMS. The implementation for this will be in form of a Quickstart
Guide. There'll be an example listener that'll be able to start a process in BPMS.

There'll also be an example BPMS process with User Task to approve or reject incoming events.

## See also

[Approvals System Design Proposal](approvals-system.md) on which is the User Event SPI originally based on.