# FAPI-SIG (Financial-grade API Security : Special Interest Group)

* **Status**: Draft #1

## Overview

FAPI-SIG is a group whose activity is mainly supporting [Financial-grade API (FAPI)](https://openid.net/wg/fapi/) and its related specifications to keycloak.

FAPI-SIG is open to everybody so that anyone can join it anytime. Nothing special need not to be done to join it. Who want to join it can only access to the communication channels shown below.  All of its activities and outputs are public so that anyone can access them.

FAPI-SIG mainly treats FAPI and its related specifications but not limited to. E.g., Ecosystems employing FAPI for their API Security like UK OpenBanking and Australia Consumer Data Right (CDR).

## Goals

Currently, proposed goals are as follows.

- [Read and Write API Security Profile (FAPI-RW)](https://openid.net/specs/openid-financial-api-part-2-ID2.html)
  - Implement and contribute necessary features
  - Pass FAPI-RW conformance tests (both FAPI-RW OP w/ MTLS and FAPI-RW OP w/ Private Key)
  - Get the certificates

- [Client Initiated Backchannel Authentication Profile (FAPI-CIBA)](https://openid.net/specs/openid-financial-api-ciba-ID1.html)
  - Implement and contribute necessary features
  - Pass FAPI-CIBA conformance tests (only both FAPI-CIBA OP poll w/ MTLS and FAPI-CIBA OP poll w/ Private Key)
  - Get the certificates

## Open Works

Currently, proposed open works are as follows.

- Integrating FAPI conformance tests run into keycloak’s CI/CD pipeline

- Implement [JWT Secured Authorization Response Mode for OAuth 2.0 (JARM)](https://openid.net/specs/openid-financial-api-jarm-ID1.html)

- Implement security profiles for Apps run on mobile devices
  - [RFC 8252 OAuth 2.0 for Native Apps](https://tools.ietf.org/html/rfc8252)
  - [OAuth 2.0 for Browser-Based Apps](https://tools.ietf.org/html/draft-ietf-oauth-browser-based-apps-06)

- Implement [FAPI-RW App2App](https://openid.net/2020/06/23/openid-foundation-announces-fapi-rw-app2app-certification-launched/)

## Communication Channels

Not only FAPI-SIG member but others can communicate with each other by the following ways.

- Mail : Google Group [keycloak developer mailing list](https://groups.google.com/forum/#!topic/keycloak-dev/Ck_1i5LHFrE)
- Chat : Zulip Chat stream ([#dev-sig-fapi](https://keycloak.zulipchat.com/#narrow/stream/248413-dev-sig-fapi))
- Meeting : Web meeting on a regular basis

## Working Repository

All of FAPI-SIG's activity outputs can be stored on [jsoss-sig/keycloak-fapi](https://github.com/jsoss-sig/keycloak-fapi/tree/master/FAPI-SIG) repository in github.

Who want to submit the output needs to send the pull-request to this repository.

