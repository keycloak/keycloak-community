# Design Proposals

For larger features we expect a proposal document to be written. There is no strict requirements of the content or
layout of the proposal today, but it should at least cover motivation, use-cases and how the feature will be used.

## Participating

To propose a new feature to Keycloak start writing a design proposal. It is a free format document and there are no 
currently no strict format. 

It should have the following heading:

    # Title

    * **Status**: {STATUS}
    * **JIRA**: [KEYCLOAK-????](https://issues.jboss.org/browse/KEYCLOAK-????)
    
Status can be one of (in order of maturity):

* Notes - Research notes / PoC / WIP
* Draft #N - Initial proposal should be "Draft #1". For every major update to the proposal the version should be increased 
* Final #N - Final means the implementation is completed and the proposal matches the implementation

The idea is not to follow a waterfall model, but rather to share and collaborate. That means a proposal can be written
prior to or during implementation. In most cases you will want to share some ideas early, then update it as you start
on the implementation.

Do consider including:

* Motivation
* Use-cases
* How the feature will be used (configuration, APIs, UIs, etc.)
* Milestones
* Implementation details
* Resources

To participate in a current proposal you can:

* Submit a PR with proposed changes
* Create a GitHub issue with the title "Proposal Title - Description"
* Review and comment on PR for new proposals, and updates to proposals