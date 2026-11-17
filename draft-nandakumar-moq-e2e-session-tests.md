---
title: "End-to-End Session Tests for Media over QUIC Transport"
abbrev: "MOQT E2E Session Tests"
category: info

docname: draft-nandakumar-moq-e2e-session-tests-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: false
v: 3
area: ""
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - interoperability
 - testing
 - relay
venue:
  group: "Media Over QUIC"
  type: ""
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"

author:

  -
    ins: S. Nandakumar
    name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com

normative:

  MoQTransport: I-D.draft-ietf-moq-transport-23

informative:

  MoQTransport22: I-D.draft-ietf-moq-transport-22
  MoQTransport18: I-D.draft-ietf-moq-transport-18
  WebTransport:
    title: "WebTransport"
    author:
      org: W3C
    target: https://www.w3.org/TR/webtransport/
  MOQ-WEB:
    title: "moq-web session-e2e test suite"
    target: https://github.com/Quicr/moq-web/tree/main/packages/session-e2e
  INTEROP-REPORT:
    title: "moq-web MOQT interop report"
    target: https://quicr.github.io/moq-web/interop/

--- abstract

This document describes a set of end-to-end tests that exercise a Media
over QUIC Transport (MOQT) relay through real client sessions. For each
test it gives a short description, the inputs, the message flow as a
call-flow diagram, and the expected outcome. The tests apply to MOQT
draft-18 and later; results for several public relays are published
in the moq-web interop report.
--- middle

# Introduction

Interoperability between MOQT {{MoQTransport}} implementations is best
established by running the same scenarios against many relays and
comparing what each one does. This document describes such a set of
scenarios, for draft-18 {{MoQTransport18}} onwards. Each test drives one or more client sessions through a relay
and checks only what is observable at those clients: the control
messages they receive and the Objects delivered to them.

The tests are implemented in the moq-web session-e2e suite {{MOQ-WEB}},
whose results for several public relays are published in
{{INTEROP-REPORT}}. This document is a description of the tests, not of
any implementation, and is intended to let other implementers reproduce
them.

# Conventions and Definitions

Pub:
: A client session that publishes Tracks through the relay.

Sub:
: A client session that subscribes to Tracks through the relay.

Relay:
: The MOQT relay under test.

Client:
: A single session in tests that need only one.

# Test Environment

Each client opens its own MOQT session to the relay over WebTransport
{{WebTransport}}, offering the MOQT version under test. Unless a test
says otherwise:

* Track Namespaces are unique per test run, of the form
  `moq-web-e2e/<profile>/<label>/<random>`, so concurrent runs against
  the same relay do not interfere.

* Object payloads identify the Object they belong to, so that every
  received Object can be checked byte for byte against the one sent.
  Tests 05 to 23 send generated chat payloads: the ASCII string
  `msg-<G>-<O>-`, padded to 64 bytes with a deterministic pattern.
  Test 15 sends `obj-<i>`, and tests 34 and later send the ASCII
  string `g<G>/o<O>` for the Object at Group G and Object ID O.

* Two delivery profiles are used: "stream", which sends Objects on
  subgroup streams, and "datagram", which sends them as Object
  datagrams. Tests that run under both are marked as such.

* A wait for a message or Object that does not arrive fails the test
  after the timeout given in the test.

* Applicability names the first MOQT draft the test runs on; it runs
  on every later draft too. A test that depends on a feature added in a
  later draft is skipped for earlier ones.

# Test Description Format

Each test is described with four parts:

Description:
: What the test exercises and why, in a few sentences.

Inputs:
: The parameters the test chooses: Tracks, counts, timings and options.

Call Flow:
: The messages exchanged, in order. Arrows show the direction of a
  message; the relay is in the middle column. A message the relay
  forwards appears once on each side. Lines starting with `*` are
  actions or checks at that endpoint, not messages. On draft-23 a
  FETCH is answered by REQUEST_OK; call flows show it as FETCH_OK, the
  draft-22 name.

A range written `G1/O1..G2/O2` runs from Object O1 of Group G1 to
Object O2 of Group G2, inclusive. An end Object of 0 means the whole
of Group G2: the FETCH carries no End Object.

Expected Outcome:
: The conditions the test asserts. The test passes only if all of them
  hold.

# Test Cases

## Session Setup {#t-01-setup}

Applicability:
: draft-18 and later

Description:
: A client opens a WebTransport session to the relay and completes the MOQT setup exchange. This proves that version negotiation and any authorization the relay requires work end to end. The setup path is the same for both delivery profiles.

Inputs:

* Delivery profiles: stream and datagram (run once each)
* MOQT version: the draft under test

Call Flow:

~~~ ascii-art
 Client                   Relay
    |                       |
    * open WebTransport
    |---- CLIENT_SETUP ---->|
    |<---- SERVER_SETUP ----|
    * session ready
~~~

Expected Outcome:

* CLIENT_SETUP and SERVER_SETUP complete.
* The session reaches the ready state.


## Publish Namespace {#t-02-announce}

Applicability:
: draft-18 and later

Description:
: A publisher sends PUBLISH_NAMESPACE for a fresh namespace. The relay must accept it.

Inputs:

* Delivery profiles: stream and datagram (run once each)
* Namespace: `moq-web-e2e/<profile>/announce-<profile>/<random>`

Call Flow:

~~~ ascii-art
   Pub                    Relay
    |                       |
    |- PUBLISH_NAMESPACE -->|
    |<----- REQUEST_OK -----|
~~~

Expected Outcome:

* The relay answers PUBLISH_NAMESPACE with REQUEST_OK.


## Publish {#t-03-publish}

Applicability:
: draft-18 and later

Description:
: A publisher sends PUBLISH for a Track that has no subscribers yet. The relay must accept it and the publisher must hold a Track Alias for the Track. The publisher does not wait for the relay to start forwarding.

Inputs:

* Delivery profiles: stream and datagram (run once each)
* Track: `<namespace>/chat`, publisher priority 128

Call Flow:

~~~ ascii-art
   Pub                    Relay
    |                       |
    |------ PUBLISH ------->|
    |<----- REQUEST_OK -----|
    * Track Alias known
~~~

Expected Outcome:

* The relay answers PUBLISH with REQUEST_OK.
* The Track Alias is a non-negative integer.


## Namespace Subscription {#t-04-subscribe-namespace}

Applicability:
: draft-18 and later

Description:
: A subscriber sends SUBSCRIBE_NAMESPACE for a prefix, then a publisher announces a namespace under that prefix. The relay must tell the subscriber about the new namespace with a NAMESPACE message on the SUBSCRIBE_NAMESPACE request stream.

Inputs:

* Delivery profiles: stream and datagram (run once each)
* Prefix: `moq-web-e2e/<profile>/nssub-<profile>`
* Namespace: the prefix plus one random field
* Wait between SUBSCRIBE_NAMESPACE and PUBLISH_NAMESPACE: 200 ms

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |                       |<-SUBSCRIBE_NAMESPACE--|
    |                       |----- REQUEST_OK ----->|
    |                       |                       * wait 200 ms
    |- PUBLISH_NAMESPACE -->|                       |
    |<----- REQUEST_OK -----|                       |
    |                       |----- NAMESPACE ------>|
    |                       |                       * namespace seen
~~~

Expected Outcome:

* The subscriber receives NAMESPACE for exactly the announced namespace within 10 s. Namespaces left over from earlier runs are ignored.


## Subscribe and Deliver {#t-05-subscribe}

Applicability:
: draft-18 and later

Description:
: A publisher publishes a Track and a subscriber subscribes to it. The publisher then sends a run of Objects and the subscriber checks each one it receives against what was sent. On streams every Object must arrive, in order. Datagrams are not retransmitted, so a small loss is tolerated.

Inputs:

* Delivery profiles: stream and datagram (run once each)
* Track: `<namespace>/chat`, default subscription filter
* Objects: 30 generated chat payloads of 64 bytes, 10 per Group (Groups 0 to 2)
* Collection time: up to 20 s (stream), 8 s (datagram)

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |------ PUBLISH ------->|                       |
    |<----- REQUEST_OK -----|                       |
    |                       |<----- SUBSCRIBE ------|
    |                       |--- SUBSCRIBE_OK ----->|
    |                       |                       * wait 250 ms
    |== Objects 0/0..2/9 ==>|== Objects 0/0..2/9 ==>|
    |                       |                       * check each
~~~

Expected Outcome:

* Stream: all 30 Objects arrive, in Group and Object order, and none arrives more than once.
* Datagram: at least 27 of the 30 Objects arrive (no more than 10% lost); the lost ones are reported.
* Every received Object is byte-identical to the Object sent.


## Pause and Resume {#t-06-subscribe-update}

Applicability:
: draft-18 and later

Description:
: A subscriber pauses its subscription with REQUEST_UPDATE, then resumes it. Objects sent after the resume must reach the subscriber, showing that the relay restarts delivery. Objects sent while paused are not required to be delivered.

Inputs:

* Delivery profiles: stream and datagram (run once each)
* Objects: 30 generated chat payloads, 15 before the pause and 15 after the resume
* Pause: REQUEST_UPDATE that stops forwarding (FORWARD 0 on draft-22, PAUSED on draft-23)
* Resume: REQUEST_UPDATE that restarts forwarding
* A relay may pass the pause on to the publisher. If it did, Pub waits up to 5 s for the relay to turn forwarding back on before sending the second 15 Objects, so they are not dropped at the publisher.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |------ PUBLISH ------->|                       |
    |<----- REQUEST_OK -----|                       |
    |                       |<----- SUBSCRIBE ------|
    |                       |--- SUBSCRIBE_OK ----->|
    |===== 15 Objects =====>|===== 15 Objects =====>|
    |                       |                       * wait 500 ms
    |                       |                       * some arrived
    |                       |                       * pause:
    |                       |<--- REQUEST_UPDATE ---|
    |                       |----- REQUEST_OK ----->|
    |<-- REQUEST_UPDATE ----|                       |
    |   (optional: pause    |                       |
    |   passed on to Pub)   |                       |
    |                       |                       * wait 250 ms
    |                       |                       * resume:
    |                       |<--- REQUEST_UPDATE ---|
    |                       |----- REQUEST_OK ----->|
    |<-- REQUEST_UPDATE ----|                       |
    |   forward 1 / PAUSED  |                       |
    |   cleared (if paused) |                       |
    * wait up to 5 s for it |                       |
    |== 15 more Objects ===>|== 15 more Objects ===>|
    |                       |                       * more arrive
~~~

Expected Outcome:

* Some Objects arrive before the pause.
* At least one Object arrives within 5 s of the resume.


## Unsubscribe {#t-07-unsubscribe}

Applicability:
: draft-18 and later

Description:
: A subscriber receives some Objects and then ends its subscription. On draft-22 and draft-23 this cancels the SUBSCRIBE request stream. Objects the publisher sends afterwards must not reach the subscriber.

Inputs:

* Delivery profiles: stream and datagram (run once each)
* Objects: 30 generated chat payloads, 15 before and 15 after the unsubscribe
* Settle time after unsubscribing: 500 ms; observation time: 1 s

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |------ PUBLISH ------->|                       |
    |<----- REQUEST_OK -----|                       |
    |                       |<----- SUBSCRIBE ------|
    |                       |--- SUBSCRIBE_OK ----->|
    |===== 15 Objects =====>|===== 15 Objects =====>|
    |                       |                       * wait 500 ms
    |                       |<-- cancel SUBSCRIBE --|
    |                       |                       * wait 500 ms
    |== 15 more Objects ===>|                       |
    |                       |                       * wait 1 s
~~~

Expected Outcome:

* Some Objects arrive before the unsubscribe.
* No Object arrives in the 1 s observation period after the settle time.


## Track Status {#t-08-track-status}

Applicability:
: draft-18 and later

Description:
: A second session queries the status of a Track another session has published. The relay must answer the TRACK_STATUS request rather than leave it pending.

Inputs:

* Delivery profiles: stream and datagram (run once each)
* Track: `<namespace>/chat`, published 250 ms before the query

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Req
    |                       |                       |
    |------ PUBLISH ------->|                       |
    |<----- REQUEST_OK -----|                       |
    |                       |<---- TRACK_STATUS ----|
    |                       |- REQUEST_OK / ERROR ->|
~~~

Expected Outcome:

* The relay answers with REQUEST_OK or REQUEST_ERROR; either is a pass.
* On REQUEST_OK, the Largest Location, when present, is a valid Group and Object.


## Fetch from Cache {#t-09-fetch}

Applicability:
: draft-18 and later

Description:
: A publisher sends Objects with no subscriber, so the relay can only cache them. A second session then FETCHes a range and checks the returned Objects byte for byte. A relay that refuses FETCH is reported, and the test does not fail.

Inputs:

* Delivery profiles: stream and datagram (run once each); a large-payload variant runs on stream only
* Default: 30 Objects of 64 bytes, 10 per Group; FETCH of 1/0..1/9 (10 Objects)
* Large payload: 300 Objects of 2048 bytes, 10 per Group; FETCH of 0/0..29/9 (300 Objects)
* Publisher Track property: max cache duration 60 s
* Wait before the FETCH: 500 ms

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |------ PUBLISH ------->|                       |
    |<----- REQUEST_OK -----|                       |
    |====== Objects =======>|                       |
    |                       * cached
    |                       |<--- FETCH 1/0..1/9 ---|
    |                       |------ FETCH_OK ------>|
    |                       |=fetch stream Objects=>|
    |                       |                       * check each
~~~

Expected Outcome:

* Every Object in the range arrives within 10 s.
* No Object outside the range arrives, and none arrives more than once.
* Every Object is byte-identical to the Object sent.
* If the relay answers the FETCH with REQUEST_ERROR, the test is reported as not supported rather than failed.


## Fetch Cancel {#t-10-fetch-cancel}

Applicability:
: draft-18 and later

Description:
: A subscriber starts a large FETCH and cancels it as soon as the first Object arrives. On draft-22 and draft-23 the cancel closes the FETCH request stream and stops the fetch data stream. Delivery must then stop. A relay that refuses FETCH is reported, and the test does not fail.

Inputs:

* Delivery profiles: stream and datagram (run once each)
* Objects: 600 of 2048 bytes, 10 per Group, cached by the relay (max cache duration 60 s)
* FETCH range: 0/0..59/9
* Settle time after the cancel: 500 ms; observation time: 1 s

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |------ PUBLISH ------->|                       |
    |<----- REQUEST_OK -----|                       |
    |==== 600 Objects =====>|                       |
    |                       |<-- FETCH 0/0..59/9 ---|
    |                       |------ FETCH_OK ------>|
    |                       |==== first Object ====>|
    |                       |<---- cancel FETCH ----|
    |                       |                       * wait 500 ms
    |                       |                       * watch 1 s
~~~

Expected Outcome:

* The first Object arrives within 5 s.
* No Object arrives in the 1 s observation period.
* Fewer than 600 Objects are delivered in total.


## Client GOAWAY {#t-11-goaway}

Applicability:
: draft-18 and later

Description:
: A client sends GOAWAY without a New Session URI. The relay is not required to respond; the test checks that a wire-valid GOAWAY is sent and that the session starts draining.

Inputs:

* Delivery profiles: stream and datagram (run once each)
* New Session URI: none

Call Flow:

~~~ ascii-art
 Client                   Relay
    |                       |
    |------- GOAWAY ------->|
    * session closing
~~~

Expected Outcome:

* After sending GOAWAY the session is in the closing state.


## Session Close Codes {#t-12-session-close}

Applicability:
: draft-18 and later

Description:
: A client closes its session with a given Session Termination code. The code must be carried in the WebTransport close. The relay may close the connection in response; that is allowed.

Inputs:

* Delivery profiles: stream and datagram (run once each)
* Case 1: code NO_ERROR (0x0), reason "test done"
* Case 2: code PROTOCOL_VIOLATION (0x3), reason "e2e-injected"

Call Flow:

~~~ ascii-art
 Client                   Relay
    |                       |
    |--- WT close(code) --->|
    * session ended
~~~

Expected Outcome:

* The close completes without an error.
* The session ends in the closed (none) state.


## Session Termination and Migration {#t-13-session-termination}

Applicability:
: draft-18 and later

Description:
: These cases check how the client handles the end of a session without needing the relay to end it. GOAWAY with a New Session URI is delivered to the client locally, as if the relay had sent it.

Inputs:

* Delivery profiles: stream and datagram (run once each)
* Case 1: the client closes its own session
* Case 2: GOAWAY with New Session URI `https://alt-relay.example/moq` and a 5 s timeout, delivered to the client locally
* Case 3: the client is asked to migrate to `https://foo.example/moq` without the worker it needs for migration

Call Flow:

~~~ ascii-art
 Client                   Relay
    |                       |
    * case 1: close(), no session-terminated
    |                       |
    * case 2: GOAWAY delivered locally;
    *         New Session URI held
    |                       |
    * case 3: migrate() refused
~~~

Expected Outcome:

* Case 1: no session-terminated event is raised for a close the client started.
* Case 2: the client records the New Session URI as its pending migration target and moves to the closing state.
* Case 3: the migration request is refused with an error that names the missing worker mode.

## SETUP Options {#t-14-setup-options}

Applicability:
: draft-18 and later

Three tests on the Setup Options carried in CLIENT_SETUP and SERVER_SETUP.

### Unknown Client Setup Options Are Ignored {#t-14-setup-unknown}

Description:
: The client advertises two Setup Options with unregistered keys. A
  peer is required to ignore unknown Setup Options ({{Section 6.3.2 of MoQTransport22}}), so the relay must still complete the setup exchange.

Inputs:
: * Setup Option 0x40 (even key): varint 0x1234.
  * Setup Option 0x41 (odd key): bytes "moq-web-e2e".

Call Flow:

~~~ ascii-art
 Client                           Relay
    |                               |
    |-- CLIENT_SETUP -------------->|
    * carries options 0x40 and 0x41
    |<-------------- SERVER_SETUP --|
    * session ready
    |                               |
~~~

Expected Outcome:
: * SERVER_SETUP is received and the session reaches the ready state.

### Relay Setup Options Are Well Formed {#t-14-setup-relay}

Description:
: The client records the Setup Options the relay advertises in
  SERVER_SETUP. A relay that advertises none passes.

Inputs:
: * None beyond a plain CLIENT_SETUP.

Call Flow:

~~~ ascii-art
 Client                           Relay
    |                               |
    |-- CLIENT_SETUP -------------->|
    |<-------------- SERVER_SETUP --|
    * list each option and its value kind
    |                               |
~~~

Expected Outcome:
: * Every option with an even key carries a varint.
  * Every option with an odd key carries a byte string.

### Client Setup Options Are Fixed After Setup {#t-14-setup-late}

Description:
: Once setup has run, the client must not change the Setup Options it
  advertised. This is a local check made on a live session; nothing is
  sent to the relay for it.

Inputs:
: * Setup Option 0x40, set after SERVER_SETUP.

Call Flow:

~~~ ascii-art
 Client                           Relay
    |                               |
    |-- CLIENT_SETUP -------------->|
    |<-------------- SERVER_SETUP --|
    * try to set option 0x40: refused locally, nothing sent
    |                               |
~~~

Expected Outcome:
: * The attempt to change Setup Options after setup fails locally.

## TRACK_STATUS Reports the Largest Object {#t-15-track-status}

Applicability:
: draft-18 and later

Description:
: A publisher sends three Objects, then a second session asks the
  relay for the Track status ({{Section 9.13 of MoQTransport22}}). The relay may
  answer from its own state or forward the request; only the answer is
  checked. A REQUEST_ERROR is tolerated.

Inputs:
: * Track: `<ns>/chat`, stream delivery.
  * Objects 0/0, 1/0 and 2/0, one per Group.
  * A 250 ms wait before TRACK_STATUS.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Req
    |                       |                       |
    |-- PUBLISH ----------->|                       |
    |<-------- REQUEST_OK --|                       |
    |== g0/o0 g1/o0 g2/o0 =>|                       |
    * wait 250 ms
    |                       |<------ TRACK_STATUS --|
    |                       |-- REQUEST_OK -------->|
    |                       |       LARGEST_OBJECT  *
    |                       |                       |
~~~

Expected Outcome:
: * The answer is REQUEST_OK with a Request ID, or REQUEST_ERROR (the
    test then ends).
  * If LARGEST_OBJECT ({{Section 9.20.17 of MoQTransport22}}) is present, its Group
    and Object are non-negative.

## NAMESPACE_DONE Ends a Namespace Batch {#t-16-namespace-done}

Applicability:
: draft-18 and later

Description:
: A subscriber sends SUBSCRIBE_NAMESPACE for a prefix with no
  publishers. The relay may close the initial batch with
  NAMESPACE_DONE ({{Section 9.17 of MoQTransport22}}) or keep the request open for
  later NAMESPACE messages; both are accepted.

Inputs:
: * Prefix: `moq-web-e2e/chat-stream/ns-done`.
  * A 500 ms wait for NAMESPACE and NAMESPACE_DONE.

Call Flow:

~~~ ascii-art
 Client                           Relay
    |                               |
    |-- SUBSCRIBE_NAMESPACE ------->|
    |<---------------- REQUEST_OK --|
    |<---------- NAMESPACE (0..n) --|
    |<------------ NAMESPACE_DONE --|
    * NAMESPACE_DONE optional; wait 500 ms
    |                               |
~~~

Expected Outcome:
: * SUBSCRIBE_NAMESPACE succeeds with a Request ID.
  * Whether NAMESPACE_DONE arrived is recorded, not asserted.

## SUBSCRIBE_TRACKS Is Accepted {#t-17-subscribe-tracks}

Applicability:
: draft-18 and later

Description:
: A subscriber sends SUBSCRIBE_TRACKS ({{Section 9.18 of MoQTransport22}})
  for a prefix under which a track is published, with the default
  forwarding state: FORWARD = 1 on draft-22, and no PAUSED on draft-23.
  Draft-22 and later have no subscription filter on SUBSCRIBE_TRACKS.
  The test checks that the relay accepts the request.

Inputs:
: * Published track: `<prefix>/<random>/chat`.
  * SUBSCRIBE_TRACKS prefix: `moq-web-e2e/chat-stream/subtracks`.
  * Forwarding: the default (FORWARD = 1, or no PAUSED); no filter.
  * A 250 ms wait after PUBLISH.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH ----------->|                       |
    |<-------- REQUEST_OK --|                       |
    * wait 250 ms
    |                       |<-- SUBSCRIBE_TRACKS --|
    |                       |   default forwarding  |
    |                       |-- REQUEST_OK -------->|
    |                       |                       |
~~~

Expected Outcome:
: * SUBSCRIBE_TRACKS succeeds and yields a subscription ID.
  * A REQUEST_ERROR is tolerated and ends the test.

## SUBSCRIBE with an Authorization Token {#t-18-auth-token}

Applicability:
: draft-18 and later

Description:
: A subscriber attaches an AUTHORIZATION_TOKEN parameter ({{Section 9.20.2 of MoQTransport22}}) to SUBSCRIBE. The token is not one the relay knows,
  so the relay may refuse it; the test checks that the parameter is
  parsed, either way.

Inputs:
: * Track: `<ns>/chat`.
  * Token type 0x63346d, 19 bytes of token value.
  * A 250 ms wait after PUBLISH.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH ----------->|                       |
    |<-------- REQUEST_OK --|                       |
    * wait 250 ms
    |                       |<- SUBSCRIBE (token) --|
    |                       |-- SUBSCRIBE_OK ------>|
    |                       |         or a refusal  *
    |                       |                       |
~~~

Expected Outcome:
: * SUBSCRIBE_OK with a subscription ID, or a SUBSCRIBE error refusing
    the token.
  * Any other failure, such as the session closing, fails the test.

## Subscription Filters {#t-19-filters}

Applicability:
: draft-18 and later

Two tests on SUBSCRIBE filters ({{Section 9.20.9 of MoQTransport22}}; LOCATION_FILTER on draft-22 and later).

### Each Filter Is Accepted {#t-19-filters-accept}

Description:
: A subscriber subscribes to a published track once with each filter
  type. Only acceptance is checked, not which Objects are delivered.

Inputs:
: * Track: `<ns>/chat`, one run per filter.
  * Filters: Next Group; Largest Object; Absolute Start at 0/0;
    Absolute Range from 0/0 to the end of Group 1.
  * A 250 ms wait after PUBLISH.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH ----------->|                       |
    |<-------- REQUEST_OK --|                       |
    * wait 250 ms
    |                       |< SUBSCRIBE (filter) --|
    |                       |-- SUBSCRIBE_OK ------>|
    |                       |                       |
~~~

Expected Outcome:
: * SUBSCRIBE_OK with a subscription ID.
  * A SUBSCRIBE error or REQUEST_ERROR is tolerated; a transport
    failure fails the test.

### Absolute Filters Deliver Exactly Their Range {#t-19-filters-deliver}

Description:
: A subscriber with an absolute filter receives every Object in the
  filter range and none after it.

Inputs:
: * Track: `<ns>/chat`, 30 Objects, 10 per Group (Groups 0 to 2).
  * Absolute Start at 0/0: expects Groups 0 to 2 (30 Objects).
  * Absolute Range 0/0 to end of Group 1: expects Groups 0 and 1 (20
    Objects).
  * Collection timeout 20 s, then a further 500 ms.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH ----------->|                       |
    |<-------- REQUEST_OK --|                       |
    |                       |< SUBSCRIBE (filter) --|
    |                       |-- SUBSCRIBE_OK ------>|
    * wait 250 ms
    |== g0..g2, 10 each ===>|                       |
    |                       |== Objects in range ==>|
    |                       |     wait 500 ms more  *
    |                       |                       |
~~~

Expected Outcome:
: * Every Object of the filter range arrives within 20 s.
  * No Object from a Group after the range arrives.

## EXPIRES and NEW_GROUP_REQUEST {#t-20-expires-ngr}

Applicability:
: draft-18 and later

Three tests that the relay accepts requests carrying EXPIRES ({{Section 9.20.16 of MoQTransport22}}) and NEW_GROUP_REQUEST ({{Section 9.20.19 of MoQTransport22}}). Delivery of a requested new Group is tested in {{t-44-new-group-request}}.

### SUBSCRIBE to a Track Published with EXPIRES {#t-20-expires}

Description:
: The publisher sets EXPIRES on its PUBLISH; a subscriber then
  subscribes.

Inputs:
: * Track: `<ns>/chat`, EXPIRES 30 s.
  * A 250 ms wait after PUBLISH.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH (EXPIRES) ->|                       |
    |<-------- REQUEST_OK --|                       |
    * wait 250 ms
    |                       |<--------- SUBSCRIBE --|
    |                       |-- SUBSCRIBE_OK ------>|
    |                       |                       |
~~~

Expected Outcome:
: * SUBSCRIBE_OK with a subscription ID.

### REQUEST_UPDATE with NEW_GROUP_REQUEST {#t-20-ngr}

Description:
: On a track published with DYNAMIC_GROUPS, the subscriber sends
  REQUEST_UPDATE carrying NEW_GROUP_REQUEST. The test does not wait
  for its REQUEST_OK.

Inputs:
: * Track: `<ns>/chat`, DYNAMIC_GROUPS = 1.
  * A 250 ms wait after PUBLISH.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH (DYN_GRP) ->|                       |
    |<-------- REQUEST_OK --|                       |
    * wait 250 ms
    |                       |<--------- SUBSCRIBE --|
    |                       |-- SUBSCRIBE_OK ------>|
    |                       |<---- REQUEST_UPDATE --|
    |                       |    NEW_GROUP_REQUEST  *
    |                       |                       |
~~~

Expected Outcome:
: * The subscription exists after SUBSCRIBE_OK.
  * Sending REQUEST_UPDATE with NEW_GROUP_REQUEST does not fail.

### SUBSCRIBE_TRACKS for a Namespace {#t-20-subscribe-tracks}

Description:
: A subscriber sends SUBSCRIBE_TRACKS for the exact namespace of a
  published track.

Inputs:
: * Track: `<ns>/chat`; SUBSCRIBE_TRACKS prefix `<ns>`.
  * A 250 ms wait after PUBLISH.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH ----------->|                       |
    |<-------- REQUEST_OK --|                       |
    * wait 250 ms
    |                       |<-- SUBSCRIBE_TRACKS --|
    |                       |-- REQUEST_OK -------->|
    |                       |         or a refusal  *
    |                       |                       |
~~~

Expected Outcome:
: * REQUEST_OK with a subscription ID, or a refusal or stream reset.
  * Any other failure, such as the session closing, fails the test.

## Publisher Resets a Subgroup Stream {#t-21-stream-reset}

Applicability:
: draft-18 and later

Description:
: While a subgroup stream is open, the publisher resets it with an
  error code ({{Section 11.3.2 of MoQTransport22}}). The test checks the publisher
  side; whether the relay passes the reset on to the subscriber is
  relay policy and is not checked.

Inputs:
: * Track: `<ns>/chat`, stream delivery.
  * 3 Objects on one subgroup stream, then a 100 ms wait.
  * Reset code TOO_FAR_BEHIND, reason "e2e-injected too-far-behind".

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH ----------->|                       |
    |<-------- REQUEST_OK --|                       |
    |                       |<--------- SUBSCRIBE --|
    |                       |-- SUBSCRIBE_OK ------>|
    * wait 250 ms
    |== 3 Objects =========>|                       |
    |                       |== Objects ===========>|
    * wait 100 ms
    |-- RESET_STREAM ------>|                       |
    * TOO_FAR_BEHIND; local stream-reset event
    |                       |                       |
~~~

Expected Outcome:
: * Exactly one stream reset is reported at the publisher, with code
    TOO_FAR_BEHIND, the given reason and the Track Alias of the track.

## Padding Does Not Disturb Delivery {#t-22-padding}

Applicability:
: draft-18 and later

Description:
: The publisher sends padding streams and padding datagrams ({{Section 11.5 of MoQTransport22}}) around two real Objects. The relay must accept the
  padding and keep delivering the real Objects.

Inputs:
: * Track: `<ns>/chat`, stream delivery.
  * Padding stream of 64 bytes and padding datagram of 32 bytes before
    the Objects.
  * 2 real Objects.
  * An empty padding stream and an empty padding datagram after them.
  * A 500 ms wait for delivery.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH ----------->|                       |
    |<-------- REQUEST_OK --|                       |
    |                       |<--------- SUBSCRIBE --|
    |                       |-- SUBSCRIBE_OK ------>|
    * wait 250 ms
    |-- pad stream 64 B --->|                       |
    |-- pad dgram 32 B ---->|                       |
    |== 2 Objects =========>|                       |
    |-- pad stream, dgram ->|                       |
    |                       |== 2 Objects =========>|
    |                       |          wait 500 ms  *
    |                       |                       |
~~~

Expected Outcome:
: * The subscriber receives at least the 2 real Objects.
  * No all-zero (padding) payload is delivered as an Object.
  * Neither session is closed by the relay while padding is sent.

## Subscribers with Different Priorities {#t-23-priority}

Applicability:
: draft-18 and later

Description:
: Two subscribers subscribe to the same track with different
  SUBSCRIBER_PRIORITY and GROUP_ORDER values ({{Section 5.1 of MoQTransport22}}).
  Both must receive delivery; arrival order is not checked, since
  scheduling is best effort.

Inputs:
: * Track: `<ns>/chat`, 4 Objects.
  * Sub1: priority 16, Group Order descending.
  * Sub2: priority 240, Group Order ascending.
  * Waits: 300 ms before publishing, 800 ms for delivery.

Call Flow:

~~~ ascii-art
   Pub                Relay               Sub1                Sub2
    |                   |                   |                   |
    |-- PUBLISH ------->|                   |                   |
    |<---- REQUEST_OK --|                   |                   |
    |                   |<----- SUBSCRIBE --|                   |
    |                   |-- SUBSCRIBE_OK -->|                   |
    |                   |                   * prio 16, desc
    |                   |<--------- SUBSCRIBE (prio 240, asc) --|
    |                   |-- SUBSCRIBE_OK ---------------------->|
    * wait 300 ms
    |== 4 Objects =====>|                   |                   |
    |                   |== Objects =======>|                   |
    |                   |== Objects ===========================>|
    * wait 800 ms
    |                   |                   |                   |
~~~

Expected Outcome:
: * Each subscriber receives at least one Object.

## Reserved Namespaces Are Refused Locally {#t-25-reserved}

Applicability:
: draft-18 and later

A Track Namespace whose first field begins with "." is reserved ({{Section 2.4.3 of MoQTransport22}}), and a client must not originate requests under it. Both tests run on a live session and check that the request is refused before anything is sent.

### PUBLISH Under a Reserved Namespace {#t-25-reserved-publish}

Description:
: The client tries to publish under a reserved namespace.

Inputs:
: * Track: `.session/chat`.

Call Flow:

~~~ ascii-art
 Client                           Relay
    |                               |
    |-- CLIENT_SETUP -------------->|
    |<-------------- SERVER_SETUP --|
    * publish .session/chat: refused locally, no PUBLISH sent
    |                               |
~~~

Expected Outcome:
: * The publish request fails locally, citing the reserved-namespace
    rule.
  * The request is rejected before it is sent.

### SUBSCRIBE Under a Reserved Namespace {#t-25-reserved-subscribe}

Description:
: The client tries to subscribe under a reserved namespace.

Inputs:
: * Track: `.reserved/chat`.

Call Flow:

~~~ ascii-art
 Client                           Relay
    |                               |
    |-- CLIENT_SETUP -------------->|
    |<-------------- SERVER_SETUP --|
    * subscribe .reserved/chat: refused locally, no SUBSCRIBE sent
    |                               |
~~~

Expected Outcome:
: * The subscribe request fails locally, citing the reserved-namespace
    rule.
  * The request is rejected before it is sent.

## Idle Connection Handling {#t-26-idle}

Applicability:
: draft-18 and later

Two tests on keeping an otherwise idle session alive and closing it when idle ({{Section 15.6.1 of MoQTransport22}}).

### Keepalive Padding Keeps the Session Up {#t-26-keepalive}

Description:
: With no other traffic, the client sends padding datagrams ({{Section 11.5 of MoQTransport22}}) as keepalives. The relay must drop them without
  affecting the session.

Inputs:
: * Keepalive interval 100 ms.
  * Idle period 400 ms.

Call Flow:

~~~ ascii-art
 Client                           Relay
    |                               |
    |-- CLIENT_SETUP -------------->|
    |<-------------- SERVER_SETUP --|
    |-- padding datagram ---------->|
    |-- padding datagram ---------->|
    * every 100 ms, for 400 ms
    |                               |
~~~

Expected Outcome:
: * The session is still ready after 400 ms.

### Local Idle Timeout Closes the Session {#t-26-idle-close}

Description:
: The client sets a short local idle timeout and sends nothing. It
  must close the session itself when the timer expires.

Inputs:
: * Idle timeout 250 ms.
  * Deadline 2 s.

Call Flow:

~~~ ascii-art
 Client                           Relay
    |                               |
    |-- CLIENT_SETUP -------------->|
    |<-------------- SERVER_SETUP --|
    * no traffic for 250 ms
    |-- session close ------------->|
    * NO_ERROR, "idle timeout"
    |                               |
~~~

Expected Outcome:
: * Within 2 s the session is terminated locally with NO_ERROR and an
    idle-timeout reason.

## MOQT_IMPLEMENTATION Setup Option {#t-27-implementation-string}

Applicability:
: draft-18 and later

Description:
: A client omits the MOQT_IMPLEMENTATION Setup Option by default and may
  opt in to sending one. Setup must complete either way: the relay
  ignores the option's value. A third check confirms the client refuses
  to set the string once setup has started.

### Setup without MOQT_IMPLEMENTATION {#t-27a}

Inputs:
: * MOQT_IMPLEMENTATION: not sent (the default)

Call Flow:

~~~ ascii-art
 Client                         Relay
    |                             |
    |-- CLIENT_SETUP (no impl) -->|
    |<------ SERVER_SETUP --------|
    * session ready               |
~~~

Expected Outcome:
: * SERVER_SETUP is received and the session is ready.

### Setup with MOQT_IMPLEMENTATION {#t-27b}

Inputs:
: * MOQT_IMPLEMENTATION: "moq-web e2e-27"

Call Flow:

~~~ ascii-art
 Client                         Relay
    |                             |
    * set implementation string   |
    |---- CLIENT_SETUP (impl) --->|
    |                             * ignores the option
    |<------ SERVER_SETUP --------|
    * session ready               |
~~~

Expected Outcome:
: * SERVER_SETUP is received and the session is ready.

### Implementation String after Setup {#t-27c}

Inputs:
: * MOQT_IMPLEMENTATION: "too-late", set after setup completes

Call Flow:

~~~ ascii-art
 Client                         Relay
    |                             |
    |------- CLIENT_SETUP ------->|
    |<------ SERVER_SETUP --------|
    * set implementation string   |
    * call throws; nothing sent   |
~~~

Expected Outcome:
: * Setting the string after setup fails locally and sends nothing.

## Grease Setup Options {#t-28-grease}

Applicability:
: draft-18 and later

Description:
: Grease values of the form 0x7f * N + 0x9D are reserved so that
  receivers are exercised on unknown values, which they must ignore.
  The client advertises two grease Setup Options in CLIENT_SETUP and
  expects setup to complete. A local check first confirms the grease
  pattern itself.

### Grease Codepoint Pattern {#t-28a}

Inputs:
: * Values 0x9D, 0x11C (grease) and 0x40 (not grease)

Call Flow:
: None; this check is local to the client.

Expected Outcome:
: * 0x9D and 0x11C match the grease pattern.
  * 0x40 does not.

### Setup with Grease Setup Options {#t-28b}

Inputs:
: * Setup Option 0x9D (odd key): a byte string
  * Setup Option 0x11C (even key): varint 0xDEAD

Call Flow:

~~~ ascii-art
 Client                         Relay
    |                             |
    |--- CLIENT_SETUP (grease) -->|
    |                             * ignores 0x9D, 0x11C
    |<------ SERVER_SETUP --------|
    * session ready               |
~~~

Expected Outcome:
: * SERVER_SETUP is received and the session is ready.

## Pause Scoped to One Subscription {#t-29-update-multi-track}

Applicability:
: draft-18 and later

Description:
: One publisher session publishes two Tracks, A and B, and one
  subscriber subscribes to both. Pausing A with REQUEST_UPDATE
  (Forward = 0) must affect only A: Objects on B keep arriving. This
  catches a publisher or relay that applies a pause to every
  subscription in the session.

Inputs:
: * Tracks A and B, each in its own namespace, stream delivery
  * 10 Objects on each Track before the pause, 20 after

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |---- PUBLISH A, B ---->|                       |
    |                       |<-- SUBSCRIBE A, B ----|
    |                       |-- SUBSCRIBE_OK x2 --->|
    |                       |                       * wait 250 ms
    |=== 10 Objects A, B ==>|                       |
    |                       |==== Objects A, B ====>|
    |                       |                       * wait 500 ms
    |                       |                       * got A and B
    |                       |<- REQUEST_UPDATE A ---|
    |                       |                       * Forward = 0
    |<- REQUEST_UPDATE A ---|                       |
    |                       |                       * wait 250 ms
    |=== 20 Objects A, B ==>|                       |
    |                       |====== Objects B =====>|
    |                       |      B arrives in 5 s *
~~~

Expected Outcome:
: * Objects on both A and B arrive before the pause.
  * After A is paused, new Objects on B arrive within 5 s.

## Room Fan-out of Namespaces {#t-30-room-fanout}

Applicability:
: draft-18 and later

Description:
: Four peers join a room: all of them subscribe to the room prefix
  with SUBSCRIBE_NAMESPACE, and then, one at a time, each announces its
  own namespace under it. Every peer must learn about every other peer,
  including peers that announced before it did. The diagram shows
  one pair; the test covers every ordered pair of the 4 peers.

Inputs:
: * 4 peers, room prefix `moq-web-e2e/chat-stream/room-<id>`
  * Peer namespace: `<room prefix>/<peer>`
  * 500 ms after every peer has subscribed, 500 ms between
    announcements, and 1000 ms to settle at the end

Call Flow:

~~~ ascii-art
  peerA                   Relay                   peerB
    |                       |                       |
    |- SUBSCRIBE_NAMESPACE >|                       |
    |                       |< SUBSCRIBE_NAMESPACE -|
    |- PUBLISH_NAMESPACE A >|                       |
    |                       |----- NAMESPACE A ---->|
    * wait 500 ms           |                       |
    |                       |< PUBLISH_NAMESPACE B -|
    |<---- NAMESPACE B -----|                       |
    |                       |                       * wait 1000 ms
~~~

Expected Outcome:
: * Each of the 4 peers receives NAMESPACE for each of the other 3.

## Room Fan-out of Tracks {#t-31-room-fanout-tracks}

Applicability:
: draft-18 and later

Description:
: Six peers join a room with the full sequence a conferencing client
  uses: SUBSCRIBE_NAMESPACE on the room prefix, PUBLISH of four Tracks,
  then PUBLISH_NAMESPACE of the peer's own namespace. On each NAMESPACE
  for another peer, a peer sends SUBSCRIBE_TRACKS for that peer's
  namespace and must then receive a PUBLISH for each of its Tracks. The
  diagram shows one pair; the test covers every ordered pair of the 6
  peers.

Inputs:
: * 6 peers, room prefix `moq-web-e2e/chat-stream/room-<id>`
  * Tracks per peer: timeline, catalog, video, audio
  * 300 ms between joins; 1200 ms to settle at the end

Call Flow:

~~~ ascii-art
  peerA                   Relay                   peerB
    |                       |                       |
    |- SUBSCRIBE_NAMESPACE >|                       |
    |                       |< SUBSCRIBE_NAMESPACE -|
    |----- PUBLISH x4 ----->|                       |
    |- PUBLISH_NAMESPACE A >|                       |
    |                       |----- NAMESPACE A ---->|
    |                       |< SUBSCRIBE_TRACKS A --|
    |                       |--- PUBLISH x4 (A) --->|
    |                       |<--- REQUEST_OK x4 ----|
    * wait 300 ms           |                       |
    |                       |<---- PUBLISH x4 ------|
    |                       |< PUBLISH_NAMESPACE B -|
    |<---- NAMESPACE B -----|                       |
    |- SUBSCRIBE_TRACKS B ->|                       |
    |<-- PUBLISH x4 (B) ----|                       |
    |---- REQUEST_OK x4 --->|                       |
~~~

Expected Outcome:
: * Each peer receives NAMESPACE for each of the other 5 peers.
  * Each peer receives a PUBLISH for all 4 Tracks of each of the other 5
    peers.

## Room Fan-out of Catalogs {#t-32-room-fanout-catalog}

Applicability:
: draft-18 and later

Description:
: Four peers join a room as in {{t-31-room-fanout-tracks}}, and each
  also publishes a catalog Track. Other peers subscribe to the catalog
  Track only later, through the PUBLISH the relay forwards after their
  SUBSCRIBE_TRACKS. A subscription starts at the next Object, and a
  relay need not hand out the latest Object it holds, so each peer
  re-sends its catalog as a new Group every 500 ms; a peer that
  subscribes later gets one of the re-sends. The diagram shows one pair; the test covers
  every ordered pair of the 4 peers.

Inputs:
: * 4 peers, room prefix `moq-web-e2e/chat-stream/room-<id>`
  * Tracks per peer: timeline, video (stream), audio (datagram),
    catalog
  * Catalog listing video and audio, re-sent as a new Group every
    500 ms
  * 400 ms between joins; up to 10 s for every catalog to arrive

Call Flow:

~~~ ascii-art
  peerA                   Relay                   peerB
    |                       |                       |
    |- SUBSCRIBE_NAMESPACE >|                       |
    |                       |< SUBSCRIBE_NAMESPACE -|
    |----- PUBLISH x4 ----->|                       |
    * incl. catalog Track   |                       |
    |=== catalog Object ===>|                       |
    * re-sent every 500 ms  |                       |
    |- PUBLISH_NAMESPACE A >|                       |
    |                       |----- NAMESPACE A ---->|
    |                       |< SUBSCRIBE_TRACKS A --|
    |                       |--- PUBLISH x4 (A) --->|
    |                       |<--- REQUEST_OK x4 ----|
    |                       |=== catalog Object ===>|
    |                       |                       * wait up to 10 s
    |                     catalog lists video+audio *
~~~

Expected Outcome:
: * Each of the 4 peers receives, from each of the other 3, a catalog
    that parses and lists Tracks named video and audio.

## PUBLISH_DONE Keeps the Session {#t-33-publish-done}

Applicability:
: draft-18 and later

Description:
: Ending a published Track sends PUBLISH_DONE as the last message on the
  PUBLISH request stream, which then closes. Sent there, it must not end
  the session: a second PUBLISH on the same session must still succeed.

Inputs:
: * Track `chat`, no subscriber, 3 Objects (one per Group)
  * Second Track `chat-2` in the same namespace

Call Flow:

~~~ ascii-art
   Pub                          Relay
    |                             |
    |------- PUBLISH chat ------->|
    |<------- REQUEST_OK ---------|
    |========= 3 Objects ========>|
    |----- PUBLISH_DONE, FIN ---->|
    |------ PUBLISH chat-2 ------>|
    |<------- REQUEST_OK ---------|
    * Track Alias assigned        |
~~~

Expected Outcome:
: * The second PUBLISH succeeds and gets a Track Alias, so the session
    is still open.

## Draft-23 Wire Changes {#t-34-draft23}

Applicability:
: draft-23

Description:
: Checks what draft-23 changed on the wire: setup over the draft-23
  control framing, FETCH answered by REQUEST_OK with the draft-23 Object
  ID rules on the fetch stream, REQUEST_ERROR with Error Details that
  leaves the session usable, and a grease Error Detail that the relay
  ignores.

### Setup over Draft-23 Framing {#t-34a}

Inputs:
: * WebTransport protocol offered: moqt-23

Call Flow:

~~~ ascii-art
 Client                         Relay
    |                             |
    |------- CLIENT_SETUP ------->|
    |<------ SERVER_SETUP --------|
    * draft-23 negotiated         |
~~~

Expected Outcome:
: * The negotiated version is draft-23 and the session is ready.

### FETCH Answered by REQUEST_OK {#t-34b}

Inputs:
: * Track `chat`, max cache duration 60 s, no subscriber
  * Groups 0 to 2, Object IDs 0 to 9 with 1, 4 and 7 skipped: 21 Objects,
    so Object ID Deltas on the fetch stream are non-zero
  * FETCH range 0/0 to 2/9

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |---- PUBLISH chat ---->|                       |
    |===== 21 Objects =====>|                       |
    * wait 500 ms           |                       |
    |                       |<-- FETCH 0/0..2/9 ----|
    |                       |----- REQUEST_OK ----->|
    |                       |==== fetch stream ====>|
    |                       |IDs match, bytes match *
~~~

Expected Outcome:
: * The FETCH is answered by REQUEST_OK, not FETCH_OK.
  * Exactly the 21 published Objects arrive, at their Group and Object
    IDs, in order and byte for byte, followed by the end of the fetch.
  * The session stays ready.
  * Tolerated: the relay refuses the FETCH; then only the last check
    applies.

### Refused SUBSCRIBE {#t-34c}

Inputs:
: * Track `nothing-here` in a namespace nobody publishes

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |                       |<----- SUBSCRIBE ------|
    |                       |---- REQUEST_ERROR --->|
    |                       |       waits up to 8 s *
    |                       |<- PUBLISH_NAMESPACE --|
    |                       |----- REQUEST_OK ----->|
    |                       |                       * still ready
~~~

Expected Outcome:
: * The REQUEST_ERROR decodes, with a numeric error code.
  * The session stays ready and serves the next request.
  * Tolerated: the relay accepts or holds the SUBSCRIBE open; this is
    reported, not failed.

### Grease Error Detail Ignored {#t-34d}

Inputs:
: * Publisher announces the namespace but publishes no Track
  * The publisher adds a grease Error Detail to every REQUEST_ERROR
  * FETCH range 0/0 to 1/0 for Track `no-objects`

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH_NAMESPACE ->|                       |
    |                       |<- FETCH no-objects ---|
    |<- FETCH no-objects ---|                       |
    |---- REQUEST_ERROR --->|                       |
    * grease Error Detail   |                       |
    |                       |---- REQUEST_ERROR --->|
    |                       |                       * wait up to 8 s
    * neither session closed|                       |
~~~

Expected Outcome:
: * Neither the publisher's nor the subscriber's session is closed.
  * Whether the relay forwards the FETCH to the publisher is not
    checked.

## PUBLISH Forwarding to Namespace Subscribers {#t-35-publish-forwarding}

Applicability:
: draft-18 and later

Description:
: SUBSCRIBE_NAMESPACE asks the relay for NAMESPACE messages only;
  SUBSCRIBE_TRACKS asks for a PUBLISH for every existing and future
  Track under a prefix. Five cases combine the two requests with the
  order of PUBLISH and subscription. In each case the publisher sends an
  Object every 100 ms and the subscriber is watched for 5 s. A plain
  SUBSCRIBE from a third session (Ctl), sent after the 5 s watch with
  filter Largest Object, checks that the publisher is reachable; if it
  gets no Objects, the case fails as inconclusive (publisher not
  reachable) rather than as a forwarding failure.

Inputs:
: * Prefix `moq-web-e2e/chat-stream/e2e-fwd/<id>`, Track `<prefix>/pub/t`
  * One Object every 100 ms, a new Group each
  * Case 1: SUBSCRIBE_NAMESPACE, then PUBLISH
  * Case 2: PUBLISH, then SUBSCRIBE_NAMESPACE (500 ms later)
  * Case 3: SUBSCRIBE_NAMESPACE, then PUBLISH and PUBLISH_NAMESPACE of
    `<prefix>/pub`
  * Case 4: SUBSCRIBE_TRACKS, then PUBLISH
  * Case 5: PUBLISH, then SUBSCRIBE_TRACKS (500 ms later)

Call Flow:

Case 4, subscription first. Case 5 sends the same messages with
  PUBLISH and the Objects first; the relay then sends the PUBLISH to Sub
  in answer to SUBSCRIBE_TRACKS.

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |                       |<- SUBSCRIBE_TRACKS ---|
    |                       |----- REQUEST_OK ----->|
    |                       |                       * wait 200 ms
    |---- PUBLISH pub/t --->|                       |
    |<---- REQUEST_OK ------|                       |
    |=== Object / 100 ms ==>|                       |
    |                       |---- PUBLISH pub/t --->|
    |                       |<---- REQUEST_OK ------|
    |                       |======= Objects ======>|
    |                       |    Objects within 2 s *
~~~

Case 3, namespace subscription. Cases 1 and 2 omit PUBLISH_NAMESPACE,
and NAMESPACE is not checked in them; a forwarded PUBLISH is reported
but not required in cases 1 to 3.

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |                       |< SUBSCRIBE_NAMESPACE -|
    |                       |----- REQUEST_OK ----->|
    |---- PUBLISH pub/t --->|                       |
    |-- PUBLISH_NAMESPACE ->|                       |
    * namespace pfx/pub     |                       |
    |=== Object / 100 ms ==>|                       |
    |                       |---- NAMESPACE pub --->|
    |                       |                       * watch 5 s
~~~

The control, in every case:

~~~ ascii-art
   Pub                    Relay                    Ctl
    |                       |                       |
    |                       |                       * after 5 s watch
    |                       |<-- SUBSCRIBE pub/t ---|
    |                       |   Largest Object      |
    |                       |--- SUBSCRIBE_OK ----->|
    |                       |======= Objects ======>|
    |                       |    Objects within 2 s *
~~~

Expected Outcome:
: * The control SUBSCRIBE receives Objects within 2 s; otherwise the
    case fails as inconclusive (publisher not reachable).
  * Cases 4 and 5: Sub receives a PUBLISH for the Track, and its Objects
    follow within 2 s of it.
  * Case 3: Sub receives NAMESPACE for the suffix `pub`.
  * No request errors or stream resets from the relay.

## Requests the Relay Cannot Serve {#t-36-request-errors}

Applicability:
: draft-22 and later

Description:
: Requests that a relay cannot serve must be refused with REQUEST_ERROR
  and the error code the draft names, within 5 s rather than left
  unanswered, and the session must stay usable. Three cases ask for a
  Track nobody publishes; two send a FETCH whose range the relay cannot
  satisfy ({{Section 3.2 of MoQTransport}}).

### SUBSCRIBE for an Unknown Track {#t-36a}

Inputs:
: * Track `nobody`, never published
  * Rendezvous timeout 1 s

Call Flow:

~~~ ascii-art
 Client                         Relay
    |                             |
    |----- SUBSCRIBE nobody ----->|
    |<------ REQUEST_ERROR -------|
    * within 5 s; still ready     |
~~~

Expected Outcome:
: * REQUEST_ERROR arrives within 5 s with code DOES_NOT_EXIST or TIMEOUT.
  * The session stays ready.

### FETCH for an Unknown Track {#t-36b}

Inputs:
: * Track `nobody`, never published
  * FETCH range 0/0 to 1/0

Call Flow:

~~~ ascii-art
 Client                         Relay
    |                             |
    |------- FETCH nobody ------->|
    |<------ REQUEST_ERROR -------|
    * within 5 s; still ready     |
~~~

Expected Outcome:
: * REQUEST_ERROR arrives within 5 s with code DOES_NOT_EXIST.
  * The session stays ready.

### TRACK_STATUS for an Unknown Track {#t-36c}

Inputs:
: * Track `nobody`, never published

Call Flow:

~~~ ascii-art
 Client                         Relay
    |                             |
    |---- TRACK_STATUS nobody --->|
    |<------ REQUEST_ERROR -------|
    * within 5 s; still ready     |
~~~

Expected Outcome:
: * REQUEST_ERROR arrives within 5 s with code DOES_NOT_EXIST.
  * The session stays ready.

### FETCH Starting after the Largest Object {#t-36d}

Inputs:
: * Track `t`, max cache duration 60 s
  * Groups 0 to 2, 3 Objects each
  * FETCH range 10/0 to 11/0

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |------ PUBLISH t ----->|                       |
    |                       |<---- SUBSCRIBE t -----|
    |                       |   Largest Object      |
    |                       |--- SUBSCRIBE_OK ----->|
    |                       |                       * wait 200 ms
    |====== 9 Objects =====>|                       |
    |                       |====== 9 Objects =====>|
    |                       |                       * wait 300 ms
    |                       |<- FETCH 10/0..11/0 ---|
    |                       |---- REQUEST_ERROR --->|
    |                       |                       * INVALID_RANGE
~~~

Expected Outcome:
: * REQUEST_ERROR arrives within 5 s with code INVALID_RANGE.
  * The session stays ready.

### FETCH for a Track with No Objects {#t-36e}

Inputs:
: * Track `t`, max cache duration 60 s
  * No Objects published
  * FETCH range 0/0 to 1/0

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |------ PUBLISH t ----->|                       |
    |                       |<---- SUBSCRIBE t -----|
    |                       |   Largest Object      |
    |                       |--- SUBSCRIBE_OK ----->|
    |                       |                       * wait 200 ms
    * sends nothing         |                       |
    |                       |                       * wait 300 ms
    |                       |<-- FETCH 0/0..1/0 ----|
    |                       |---- REQUEST_ERROR --->|
    |                       |                       * INVALID_RANGE
~~~

Expected Outcome:
: * REQUEST_ERROR arrives within 5 s with code INVALID_RANGE.
  * The session stays ready.

## Largest Location in SUBSCRIBE_OK {#t-37-subscribe-ok-largest}

Applicability:
: draft-22 and later

Description:
: SUBSCRIBE_OK reports the Track's Largest Location. A subscriber that
  arrives before any Object is published must see none; a second
  subscriber, arriving after Groups 0 to 2 of 3 Objects each, must see
  2/2, the last Object sent.

Inputs:
: * Track `t`, max cache duration 60 s
  * Groups 0 to 2, 3 Objects each
  * Two subscriber sessions, Early and Late, both filter Largest Object

Call Flow:

~~~ ascii-art
   Pub                Relay               Early               Late
    |                   |                   |                   |
    |---- PUBLISH t --->|                   |                   |
    |                   |<-- SUBSCRIBE t ---|                   |
    |                   |-- SUBSCRIBE_OK -->|                   |
    |                   |                   * no Largest        |
    |==== 9 Objects ===>|                   |                   |
    * wait 500 ms       |                   |                   |
    |                   |<------------ SUBSCRIBE t -------------|
    |                   |------------ SUBSCRIBE_OK ------------>|
    |                   |                   |       Largest 2/2 *
~~~

Expected Outcome:
: * The first SUBSCRIBE_OK carries no Largest Location.
  * The second carries Largest Location 2/2 and indicates content
    exists.

## PUBLISH_NAMESPACE Lifecycle {#t-38-publish-namespace-lifecycle}

Applicability:
: draft-22 and later

Description:
: A publisher announces a namespace, withdraws it, and announces it
  again. A subscriber with SUBSCRIBE_NAMESPACE on the parent prefix must
  see NAMESPACE, then NAMESPACE_DONE, then NAMESPACE again. The
  publisher withdraws by cancelling its PUBLISH_NAMESPACE request.

Inputs:
: * Prefix `moq-web-e2e/chat-stream/pubns-life/<id>`
  * Namespace `<prefix>/pub`
  * Each NAMESPACE or NAMESPACE_DONE awaited for up to 5 s

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |                       |< SUBSCRIBE_NAMESPACE -|
    |                       |----- REQUEST_OK ----->|
    |-- PUBLISH_NAMESPACE ->|                       |
    |                       |---- NAMESPACE pub --->|
    |--- cancel request --->|                       |
    |                       |- NAMESPACE_DONE pub ->|
    |-- PUBLISH_NAMESPACE ->|                       |
    |                       |---- NAMESPACE pub --->|
~~~

Expected Outcome:
: * Sub sees, for `pub` and in this order: NAMESPACE, NAMESPACE_DONE,
    NAMESPACE.
  * No request errors or stream resets from the relay.

## Namespace Prefix Update {#t-39-namespace-prefix-update}

Applicability:
: draft-22 and later

Description:
: A subscriber moves an existing SUBSCRIBE_NAMESPACE from one prefix to
  another with REQUEST_UPDATE carrying TRACK_NAMESPACE_PREFIX (draft-22
  Section 9.20.20). The relay must answer the update, stop matching the
  old prefix and start matching the new one.

Inputs:
: * Old prefix: `<ns>/a`; new prefix: `<ns>/b`.
  * Namespaces announced after the update: `<ns>/a/late`, `<ns>/b/late`.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |                       |<-- SUBSCRIBE_NAMESPACE|
    |                       |        <ns>/a         |
    |                       |-- REQUEST_OK -------->|
    |                       |                       * wait 300 ms
    |                       |<-- REQUEST_UPDATE ----|
    |                       |  TRACK_NAMESPACE_     |
    |                       |  PREFIX = <ns>/b      |
    |                       |------ REQUEST_OK ---->|
    |-- PUBLISH_NAMESPACE ->|                       |
    |     <ns>/a/late       |                       |
    |-- PUBLISH_NAMESPACE ->|                       |
    |     <ns>/b/late       |                       |
    |                       |-- NAMESPACE --------->|
    |                       |     <ns>/b/late       |
    |                       |                       * wait 500 ms
~~~

Expected Outcome:
: * REQUEST_OK arrives within 5 s.
  * NAMESPACE `<ns>/b/late` arrives within 5 s.
  * NAMESPACE `<ns>/a/late` does not arrive.
  * No unexpected error or session close occurs.

## Stream-Credit Flow Control {#t-40-stream-credit}

Applicability:
: draft-22 and later

Since MAX_REQUEST_ID was removed, each request holds a bidirectional
stream and each subgroup a unidirectional one, so the limits are the
QUIC stream credits the peer grants.

### Requests over the Open-Stream Limit {#t-40a-open-requests}

Description:
: One session sends far more SUBSCRIBEs than a typical stream limit at
  once. Those over the relay's limit must fail quickly and leave the
  session usable; once some subscriptions end, credit must come back so
  new requests succeed.

Inputs:
: * 170 tracks, published by several Pub sessions, 50 tracks each,
    each Pub under its own namespace.
  * 150 SUBSCRIBEs sent at once; 20 freed, then 20 new SUBSCRIBEs.

Call Flow:

~~~ ascii-art
   Pubs                   Relay                    Sub
    |                       |                       |
    |-- PUBLISH x170 ------>|                       |
    |                       |<-- SUBSCRIBE x150 ----|
    |                       |   (one bidi stream    |
    |                       |    each, at once)     |
    |                       |-- SUBSCRIBE_OK or --->|
    |                       |   REQUEST_ERROR or    |
    |                       |   stream refused      |
    |                       |                       * <= 20 s
    |                       |<- cancel SUBSCRIBE x20|
    |                       |                       * wait 1 s
    |                       |<-- SUBSCRIBE x20 -----|
    |                       |   (new tracks)        |
    |                       |-- SUBSCRIBE_OK x20 -->|
~~~

Expected Outcome:
: * All 170 PUBLISHes succeed; otherwise the burst cannot run.
  * Every one of the 150 SUBSCRIBEs is answered or fails within 20 s.
  * The session is still ready after the burst.
  * All 20 new SUBSCRIBEs succeed, each within 10 s.

### Short-Lived Streams {#t-40b-short-streams}

Description:
: A publisher sends many single-Object groups back to back, each on its
  own unidirectional stream, exceeding a typical initial limit of 100
  streams. Delivery succeeds only if both sides keep granting
  unidirectional stream credit.

Inputs:
: * Track `<ns>/t`, stream delivery.
  * 150 groups of one Object each.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH <ns>/t ---->|                       |
    |                       |<-- SUBSCRIBE <ns>/t --|
    |                       |-- SUBSCRIBE_OK ------>|
    |                       |                       * wait 300 ms
    |== g0..g149, one =====>|== g0..g149, one =====>|
    |   stream per group    |   stream per group    |
    |                       |                       * wait <= 20 s
~~~

Expected Outcome:
: * No send fails at the publisher.
  * All 150 groups arrive within 20 s.

## Unknown Message Parameter {#t-41-unknown-parameter}

Applicability:
: draft-22 and later

Description:
: Message Parameters are not greased; an endpoint that receives an
  undefined one must close the session, with INVALID_PARAMETER in
  draft-23 ({{Section 9.19 of MoQTransport}}) and PROTOCOL_VIOLATION
  before. The client sends a TRACK_STATUS carrying one parameter of
  undefined type 0x3C.

Inputs:
: * TRACK_STATUS for `<ns>/t`.
  * Extra Message Parameter 0x3C = 1, undefined in every draft.
  * The client first checks that its own decoder rejects the message
    for that parameter only.

Call Flow:

~~~ ascii-art
  Client                  Relay
    |                       |
    |-- CLIENT_SETUP ------>|
    |<-- SERVER_SETUP ------|
    * encode TRACK_STATUS   |
    * + parameter 0x3C = 1  |
    |-- TRACK_STATUS ------>|  (new bidi request stream)
    |   param 0x3C          |
    |<-- CONNECTION CLOSE --|
    |  INVALID_PARAMETER    |
    |  (0x305, draft-23) or |
    |  PROTOCOL_VIOLATION   |
    |  (0x3, draft-22)      |
~~~

Expected Outcome:
: * The relay closes the session within 5 s.
  * The close code is INVALID_PARAMETER (0x305) on draft-23, or
    PROTOCOL_VIOLATION (0x3) on draft-22.

## Fill with SUBSCRIBE {#t-42-fill}

Applicability:
: draft-22 and later

FILL_PARAMETERS ({{Section 3.4 of MoQTransport22}}) has the relay open a fill stream
with past groups next to the live subscription. In both tests a holder
session subscribes first, so the relay keeps groups 0..4 cached.

### Fill on SUBSCRIBE {#t-42a-fill-subscribe}

Description:
: A subscriber joining mid-track asks for the last 3 groups as fill. It
  must get those groups from the cache and the following groups live,
  and nothing older.

Inputs:
: * Track `<ns>/t`, stream delivery, MAX_CACHE_DURATION 60 s.
  * Cached: groups 0..4, 3 Objects each.
  * Fill: last 3 groups. Published afterwards: groups 5..6.

Call Flow:

~~~ ascii-art
   Pub                    Relay              Holder / Sub
    |                       |                       |
    |-- PUBLISH <ns>/t ---->|                       |
    |                       |<-- SUBSCRIBE (Holder)-|
    |== g0..g4, 3 obj =====>|== g0..g4 ============>| Holder
    |                       |                       * wait 500 ms
    |                       |<-- SUBSCRIBE (Sub) ---|
    |                       |   Largest Object,     |
    |                       |   FILL last 3 groups  |
    |                       |-- SUBSCRIBE_OK ------>|
    |                       |== fill: g2..g4 ======>| Sub
    |== g5..g6 ============>|== g5..g6 live =======>| Sub
~~~

Expected Outcome:
: * Every Object of groups 2..6 arrives at Sub, byte for byte.
  * No Object of groups 0..1 arrives at Sub.

### Fill on REQUEST_UPDATE {#t-42b-fill-update}

Description:
: A subscriber that joined without fill later asks for groups 1..2 with
  REQUEST_UPDATE. Exactly those groups must arrive on the fill stream.

Inputs:
: * Track `<ns>/t`, stream delivery, MAX_CACHE_DURATION 60 s.
  * Cached: groups 0..4, 3 Objects each.
  * Fill: start group 1, end group 2.

Call Flow:

~~~ ascii-art
   Pub                    Relay              Holder / Sub
    |                       |                       |
    |-- PUBLISH <ns>/t ---->|                       |
    |                       |<-- SUBSCRIBE (Holder)-|
    |== g0..g4, 3 obj =====>|== g0..g4 ============>| Holder
    |                       |                       * wait 500 ms
    |                       |<-- SUBSCRIBE (Sub) ---|
    |                       |   Largest Object      |
    |                       |-- SUBSCRIBE_OK ------>|
    |                       |<-- REQUEST_UPDATE ----|
    |                       |   FILL groups 1..2    |
    |                       |------ REQUEST_OK ---->|
    |                       |== fill: g1..g2 ======>| Sub
~~~

Expected Outcome:
: * The REQUEST_UPDATE is answered.
  * Every Object of groups 1 and 2 arrives at Sub.
  * No other Object arrives on the fill.

## FETCH from the Relay Cache {#t-43-fetch-cache}

Applicability:
: draft-22 and later

Description:
: A holder subscription keeps groups 0..2 in the relay's cache, then a
  second session fetches them in five shapes. Every Object is checked
  byte for byte. MOQT sets no cache size, so the test caches only 3
  groups, which the smallest relay cache seen keeps per track. A FETCH
  end location is inclusive ({{Section 9.11 of MoQTransport22}}), and a
  FETCH ends at the Largest Object, so Objects published after it are
  not part of the response ({{Section 3.2 of MoQTransport}}).

Inputs:
: * Track `<ns>/t`, stream delivery, MAX_CACHE_DURATION 60 s.
  * Cached: groups 0..2, 3 Objects each (Object IDs 0..2).
  * The five cases in the table below.

| Case | FETCH range | Variation | Expected Objects |
|---|---|---|---|
| whole | 1/0..2/0 | none | 1/0..2/2 (6) |
| partial | 0/1..1/1 | none | 0/1..1/1 (4) |
| descending | 0/0..1/0 | group order descending | 0/0..1/2 (6), groups 1, 0 |
| cache-only | 0/0..2/0 | Pub closes its session first | 0/0..2/2 (9), or DOES_NOT_EXIST |
| live-edge | 1/0..4/0 | Pub sends groups 3..4 after the FETCH | 1/0..2/2 (6) |
{: title="FETCH cases"}

The descending range stops short of the Largest Object, so it is a
range below the live edge in every relay.

Call Flow:

~~~ ascii-art
   Pub                    Relay              Holder / Sub
    |                       |                       |
    |-- PUBLISH <ns>/t ---->|                       |
    |                       |<-- SUBSCRIBE (Holder)-|
    |== g0..g2, 3 obj =====>|== g0..g2 ============>| Holder
    |                       |                       * wait 500 ms
    * [cache-only] close    |                       |
    |                       |<-- FETCH (Sub) -------|
    |                       |   range of the case   |
    * [live-edge] g3..g4    |                       |
    |== g3..g4 ============>|                       |
    |                       |-- FETCH_OK ---------->|
    |                       |== fetch stream ======>| Sub
    |                       |   Objects in range    |
~~~

Expected Outcome:
: * No FETCH error, except that cache-only may instead get
    REQUEST_ERROR DOES_NOT_EXIST with no Objects.
  * Every expected Object in the table arrives, byte for byte.
  * No Object outside the range arrives; in live-edge, none of groups
    3..4.
  * In descending, groups arrive in descending order.

## NEW_GROUP_REQUEST {#t-44-new-group-request}

Applicability:
: draft-22 and later

Description:
: A subscriber asks the publisher to start a new group. The publisher
  declares DYNAMIC_GROUPS, the relay must pass the NEW_GROUP_REQUEST to
  it, and the new group must reach the subscriber.

Inputs:
: * Track `<ns>/t`, stream delivery, DYNAMIC_GROUPS = 1.
  * Pub sends one Object every 100 ms in group 0 until it gets the
    request, then moves to group 1.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH <ns>/t ---->|                       |
    |   DYNAMIC_GROUPS=1    |                       |
    |== g0/o0, o1.. =======>|                       |
    |   every 100 ms        |<-- SUBSCRIBE <ns>/t --|
    |                       |-- SUBSCRIBE_OK ------>|
    |                       |   DYNAMIC_GROUPS      |
    |                       |== g0 objects ========>|
    |                       |                       * 3 Objects in
    |                       |<-- REQUEST_UPDATE ----|
    |                       |   NEW_GROUP_REQUEST   |
    |                       |------ REQUEST_OK ---->|
    |<-- REQUEST_UPDATE ----|                       |
    |   NEW_GROUP_REQUEST   |                       |
    * start group 1         |                       |
    |== g1/o0.. ===========>|== g1/o0.. ===========>|
~~~

Expected Outcome:
: * SUBSCRIBE_OK carries DYNAMIC_GROUPS.
  * The REQUEST_UPDATE is answered.
  * A REQUEST_UPDATE carrying NEW_GROUP_REQUEST reaches Pub.
  * A group 1 Object reaches Sub within 3 s of the request.

## Namespace Discovery Flow {#t-45-namespace-discovery}

Applicability:
: draft-22 and later

Description:
: The complete discover-then-subscribe flow. A subscriber learns a
  namespace through SUBSCRIBE_NAMESPACE, subscribes to a track in it by
  name, and later sees the track and the namespace end. A late
  subscriber must learn the existing namespace, and a sibling namespace
  whose last field only starts with the prefix's must not match, since
  namespaces match field by field.

Inputs:
: * Prefix `<p>`; namespace `<p>/pub`; track `t`.
  * Sibling namespace `<p>x/pub`.
  * Pub sends one Object every 100 ms.

Call Flow:

~~~ ascii-art
   Pub                    Relay              Sub / Late
    |                       |                       |
    |                       |<-- SUBSCRIBE_NAMESPACE|
    |                       |      <p> (Sub)        |
    |-- PUBLISH_NAMESPACE ->|                       |
    |     <p>/pub           |                       |
    |-- PUBLISH_NAMESPACE ->|                       |
    |     <p>x/pub          |                       |
    |-- PUBLISH <p>/pub/t ->|                       |
    |== Objects ===========>|-- NAMESPACE <p>/pub ->| Sub
    |                       |<-- SUBSCRIBE t -------| Sub
    |                       |-- SUBSCRIBE_OK ------>|
    |                       |== Objects ===========>| Sub
    |                       |<-- SUBSCRIBE_NAMESPACE|
    |                       |      <p> (Late)       |
    |                       |-- NAMESPACE <p>/pub ->| Late
    * stop, end track t     |                       |
    |-- PUBLISH_DONE ------>|-- PUBLISH_DONE ------>| Sub
    * cancel announcement   |                       |
    |-- (cancel PUBLISH_    |-- NAMESPACE_DONE ---->| Sub
    |    NAMESPACE) ------->|-- NAMESPACE_DONE ---->| Late
~~~

Expected Outcome:
: * Each step completes in order: NAMESPACE to Sub within 5 s, at
    least 3 Objects on the SUBSCRIBE within 3 s, then within 5 s each
    NAMESPACE to Late, PUBLISH_DONE to Sub, and
    NAMESPACE_DONE to both Sub and Late.
  * Neither Sub nor Late receives NAMESPACE for `<p>x/pub`.
  * No unexpected error or session close occurs.

## SUBSCRIBE_TRACKS Flow {#t-46-subscribe-tracks}

Applicability:
: draft-22 and later

Description:
: A subscriber asks for every track under a prefix with
  SUBSCRIBE_TRACKS. The relay must send a PUBLISH for each matching
  track, forward its Objects and PUBLISH_DONE, and stop after the
  request is cancelled. The publisher also sends SUBSCRIBE_TRACKS for
  the same prefix and must not get its own tracks back, since the
  request excludes "tracks published by the subscriber" ({{Section 3.6 of MoQTransport}}).

Inputs:
: * Prefix `<p>`; namespace `<p>/pub`.
  * Tracks a, b, c; track d is published after the cancel.
  * One Object per track every 100 ms.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |                       |<-- SUBSCRIBE_TRACKS --|
    |                       |      <p>              |
    |-- SUBSCRIBE_TRACKS -->|                       |
    |      <p>              |                       |
    |-- PUBLISH a, b, c --->|                       |
    |== Objects ===========>|-- PUBLISH a, b, c --->|
    |                       |== Objects a, b, c ===>|
    |   (no PUBLISH a..d    |                       |
    |    back to Pub)       |                       |
    * end track a           |                       |
    |-- PUBLISH_DONE a ---->|-- PUBLISH_DONE a ---->|
    |                       |<-- cancel SUBSCRIBE_  |
    |                       |    TRACKS ------------|
    |-- PUBLISH d --------->|                       |
    |                       |   (no PUBLISH d)      |
    |                       |                       * wait 2 s
~~~

Expected Outcome:
: * Sub receives a PUBLISH and Objects for each of a, b and c.
  * Sub receives PUBLISH_DONE for a.
  * Sub receives no PUBLISH for d after the cancel.
  * Pub receives no PUBLISH for any of its own tracks.
  * No unexpected error or session close occurs.

## GOAWAY Flows {#t-47-goaway}

Applicability:
: draft-22 and later

GOAWAY does not affect subscription state ({{Section 9.2 of MoQTransport}}): delivery continues until the sender ends its requests
and closes. A client cannot make the relay send GOAWAY, so both tests
send it from a client in the middle of delivery.

### Subscriber Sends GOAWAY {#t-47a-goaway-sub}

Description:
: The subscriber sends GOAWAY while Objects are flowing. Delivery must
  continue, and the subscriber then unsubscribes and closes cleanly.

Inputs:
: * Track `<ns>/t`; Pub sends one Object, in a new group, every 100 ms.
  * Drain period: 2 s.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH <ns>/t ---->|                       |
    |                       |<-- SUBSCRIBE <ns>/t --|
    |                       |-- SUBSCRIBE_OK ------>|
    |== 1 obj / 100 ms ====>|== Objects ===========>|
    |                       |                       * 3 Objects in
    |                       |<-- GOAWAY ------------|
    |== Objects ===========>|== Objects ===========>|
    |                       |                       * 2 s
    |                       |<-- cancel SUBSCRIBE --|
    |                       |<-- session close -----|
~~~

Expected Outcome:
: * At least 10 Objects arrive in the 2 s after GOAWAY.
  * No unexpected error or close code is seen.

### Publisher Sends GOAWAY {#t-47b-goaway-pub}

Description:
: The publisher sends GOAWAY while Objects are flowing. The subscriber
  must keep receiving, then get PUBLISH_DONE when the publisher ends the
  track, with its own session unaffected.

Inputs:
: * Track `<ns>/t`; Pub sends one Object, in a new group, every 100 ms.
  * Drain period: 2 s.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH <ns>/t ---->|                       |
    |                       |<-- SUBSCRIBE <ns>/t --|
    |                       |-- SUBSCRIBE_OK ------>|
    |== 1 obj / 100 ms ====>|== Objects ===========>|
    |                       |                       * 3 Objects in
    |-- GOAWAY ------------>|                       |
    |== Objects ===========>|== Objects ===========>|
    * 2 s, stop, end track  |                       |
    |-- PUBLISH_DONE ------>|-- PUBLISH_DONE ------>|
~~~

Expected Outcome:
: * At least 10 Objects arrive at Sub in the 2 s after GOAWAY.
  * PUBLISH_DONE arrives within 5 s of the track ending.
  * Sub's session is still ready.
  * No unexpected error or close code is seen.

## Track Properties {#t-48-track-properties}

Applicability:
: draft-22 and later

Description:
: Track Properties sent in the publisher's PUBLISH must reach
  subscribers unchanged, both in SUBSCRIBE_OK and in the answer to
  TRACK_STATUS ({{Section 3.7 of MoQTransport22}}).

Inputs:
: * Track `<ns>/t`.
  * Track Properties: subgroup delivery timeout 4000 ms, max cache
    duration 30000 ms, default publisher priority 64, default publisher
    group order descending, DYNAMIC_GROUPS.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH <ns>/t ---->|                       |
    |   Track Properties    |                       |
    |                       |<-- SUBSCRIBE <ns>/t --|
    |                       |-- SUBSCRIBE_OK ------>|
    |                       |   Track Properties    |
    |== 1 Object ==========>|== 1 Object ==========>|
    |                       |<-- TRACK_STATUS ------|
    |                       |-- REQUEST_OK -------->|
    |                       |   Track Properties    |
~~~

Expected Outcome:
: * SUBSCRIBE_OK carries all five Track Properties with the values the
    PUBLISH sent.
  * The TRACK_STATUS answer carries the same five values.

## Object Properties {#t-49-object-properties}

Applicability:
: draft-22 and later

Description:
: Every Object carries two Object Properties of types the relay does
  not know, one even (varint) and one odd (bytes). Both are below the
  Mandatory range, so the relay must forward them unchanged (draft-22
  Section 11.1).

Inputs:
: * Track `<ns>/t`, stream delivery.
  * 3 groups of 4 Objects.
  * Property 0x40 = group x 100 + object (varint).
  * Property 0x41 = `p<G>/<O>` (bytes).

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |-- PUBLISH <ns>/t ---->|                       |
    |                       |<-- SUBSCRIBE <ns>/t --|
    |                       |-- SUBSCRIBE_OK ------>|
    |                       |                       * wait 200 ms
    |== g0..g2, o0..o3 ====>|== g0..g2, o0..o3 ====>|
    |   props 0x40, 0x41    |   props unchanged     |
    |                       |                       * wait <= 5 s
~~~

Expected Outcome:
: * All 12 Objects arrive.
  * Each carries 0x40 = group x 100 + object and 0x41 =
    `p<G>/<O>`.

## Setup Options and Concurrent REQUEST_UPDATEs {#t-50-setup-options}

Applicability:
: draft-22 and later

Description:
: The Setup Options in the relay's SERVER_SETUP must be well formed,
  and the relay must honour MAX_REQUEST_UPDATES ({{Section 9.1.7 of MoQTransport22}}): the subscriber sends that many REQUEST_UPDATEs at once on one
  subscription, and each must be answered in order.

Inputs:
: * Track `<ns>/t`.
  * Updates sent at once: the relay's MAX_REQUEST_UPDATES, capped at 4;
    4 if the relay sets no limit.
  * The updates alternate pause and resume.

Call Flow:

~~~ ascii-art
   Pub                    Relay                    Sub
    |                       |                       |
    |                       |<-- CLIENT_SETUP ------|
    |                       |-- SERVER_SETUP ------>|
    |                       |   Setup Options       |
    |                       |                       * check options
    |-- PUBLISH <ns>/t ---->|                       |
    |                       |<-- SUBSCRIBE <ns>/t --|
    |                       |-- SUBSCRIBE_OK ------>|
    |                       |<-- REQUEST_UPDATE #0 -|
    |                       |<-- REQUEST_UPDATE #1 -|
    |                       |<-- ...  #N-1 ---------|
    |                       |------ REQUEST_OK ---->| #0
    |                       |------ REQUEST_OK ---->| #1
    |                       |-- ...               ->| #N-1
~~~

Expected Outcome:
: * Every Setup Option with an even key carries a varint, and every one
    with an odd key carries bytes.
  * Every REQUEST_UPDATE is answered within 5 s.
  * The REQUEST_OKs arrive in the order the updates were sent.

# Security Considerations

The tests send only synthetic data to relays whose operators make them
available for testing. A test that sends a malformed message (for
example, an unknown Message Parameter) expects the relay to close the
session, and does not attempt anything beyond that. Authorization
tokens used to reach a relay are not part of the test descriptions.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The test scenarios in this document come from the moq-web session-e2e
test suite {{MOQ-WEB}}, and the results that shaped them are published
in the moq-web interop report {{INTEROP-REPORT}}.

Thanks to the MOQ working group interop hackers who brought their
relays for interop testing. The tests have been run against the
following relays:

| Relay | Implementation | Provided by |
|---|---|---|
| Cloudflare moq-ng | moq-ng | Cloudflare |
| Meta | moxygen (mvfst) | Meta |
| openmoq | moqx (moxygen) | openmoq project |
| aiomoqt | aiomoqt | Marz Research |
| moqtail | moqtail | moqtail project |
| laps | libquicr | Quicr project (Cisco) |
| quichemoq | quichemoq | quichemoq project |
| stitcher-moq | moq-dev moq-relay | Paramount |
| xquic | xquic | xquic project (Alibaba) |
| lminiero | imquic | Lorenzo Miniero (Meetecho) |
{: title="Relays tested"}

# Disclaimer
{:numbered="false"}

This document was generated with the help of AI tools. The author
provided the test scenarios, the test descriptions, and the expected
inputs and outputs, and reviewed the result.
