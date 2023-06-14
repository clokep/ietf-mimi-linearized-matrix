# Implementation notes for Matrix developers

This document is primarily intended for Matrix developers like the Spec Core Team (SCT): folks who already
likely know about the inner workings of Matrix and want to know how Linearized Matrix is different from
spec.matrix.org's definition of Matrix.

If you're not already familiar with Matrix, or are trying to implement Linearized Matrix from scratch, please
review the I-D instead.

**This is a living document.** Please track changes aggressively.

## Architecture

Linearized Matrix (LM) operates at the per-room level, creating a cluster of servers which do not support DAG
operations natively. Routing within LM is hub-and-spoke, where a hub server converts partial events from
"participant" servers into real events that work with DAG servers. DAG-capable servers in the room *do not*
need to route their events through the hub server.

A "participant server" is a non-DAG capable, non-hub, homeserver in the room.

This is the send path for participant->world:

```
+-------------+          +-----+                            +-----------+ +-----------+
| Participant |          | Hub |                            | Synapse1  | | Conduit1  |
+-------------+          +-----+                            +-----------+ +-----------+
       |                    |                                     |             |
       | Partial event      |                                     |             |
       |------------------->|                                     |             |
       |                    |                                     |             |
       |                    | Add fields to form normal PDU       |             |
       |                    |------------------------------       |             |
       |                    |                             |       |             |
       |                    |<-----------------------------       |             |
       |                    |                                     |             |
       |                    | Internally send PDU                 |             |
       |                    |--------------------                 |             |
       |                    |                   |                 |             |
       |                    |<-------------------                 |             |
       |                    |                                     |             |
       |    Send PDU (echo) |                                     |             |
       |<-------------------|                                     |             |
       |                    |                                     |             |
       |                    | Send PDU                            |             |
       |                    |------------------------------------>|             |
       |                    |                                     |             |
       |                    | Send PDU                            |             |
       |                    |-------------------------------------------------->|
       |                    |                                     |             |
```

Synapse1 and Conduit1 are considered "DAG-capable" servers.

If Synapse1 were to want to send an event, it does so just as it does today, though avoids making contact to the
participant server directly (as the participant will reject the request due to not being the hub):

```
+-------------+   +-----+                             +-----------+               +-----------+
| Participant |   | Hub |                             | Synapse1  |               | Conduit1  |
+-------------+   +-----+                             +-----------+               +-----------+
       |             |                                      |                           |
       |             |                                      | Internally send PDU       |
       |             |                                      |--------------------       |
       |             |                                      |                   |       |
       |             |                                      |<-------------------       |
       |             |                                      |                           |
       |             |                                      | Send PDU                  |
       |             |                                      |-------------------------->|
       |             |                                      |                           |
       |             |                             Send PDU |                           |
       |             |<-------------------------------------|                           |
       |             |                                      |                           |
       |             | Run linearization on DAG event       |                           |
       |             |-------------------------------       |                           |
       |             |                              |       |                           |
       |             |<------------------------------       |                           |
       |             |                                      |                           |
       |    Send PDU |                                      |                           |
       |<------------|                                      |                           |
       |             |                                      |                           |
```

On the hub server, the room is an **append-only** singly linked list. Events are appended in order of receipt. As
part of linearization the hub may need to accept/generate "filler" events to handle cases where state res kicks a
user out of the room or invalidates prior events, for example.

In short, LM will treat accepted data as forever accepted. This is to ensure maximum compatibility with MLS, a
requirement for encryption within MIMI. LM can support Olm/Megolm as well, but how it does so is out of scope for
MIMI.

Participant servers are not required to do anything beyond trusting the hub server to send events to it, but are
encouraged to track room state locally to validate the hub is behaving correctly.

## Room Version

The majority of mechanics are based upon [MSC3820: Room version 11](https://github.com/matrix-org/matrix-spec-proposals/pull/3820).

Changes from v11 are:

* Event format: `hashes` has a new structure, and `hub_server` is a new top level property.
* Grammar: `I.` as a prefix is now reserved for use by the IETF specification process.
* Signing: an intermediary "Linearized PDU" (LPDU) event is incorporated into the overall event/PDU signatures.
* Clarification: size limits are checked under "checks performed upon receipt of a PDU".
* Auth rules: all rules relating to `join_authorised_via_users_server` and restricted rooms are removed.
  * This is not intentional, so just be careful in what you send in unstable room versions for now.
* Auth rules: `third_party_invite` is not checked on `m.room.member` events. 3rd party invites do not exist in LM.
* Auth rules: `notifications.*` is not checked as part of `m.room.power_levels` checks.
* Canonical JSON: we use [RFC 8785](https://datatracker.ietf.org/doc/html/rfc8785).
  * For the majority of cases, Matrix's existing canonical JSON implementation is *close enough*.
* Content hash: the algorithm has changed to account for LPDU hashes.
* Redactions: `hub_server` is preserved as a top-level field.

These are adopted as the unstable room version `org.matrix.i-d.ralston-mimi-linearized-matrix.02`.

Note that there's also a `org.matrix.i-d.ralston-mimi-linearized-matrix.00` room version out there, but is no longer
in use.

### Event Format / LPDU

Linearized PDUs (LPDUs) are partially-formed events generated by participant servers before sending them off to the
hub server, which adds fields to make them real events.

An LPDU has the same fields as a normal PDU, with the following changes:

* `auth_events` is *not* included, because the participant doesn't have reasonable visibility on "current state".
* `prev_events` is *not* included, because the participant doesn't track room history unless it wants to.
* `hashes` *only* has `lpdu` under it.
* `hub_server` is added to denote which hub server the participant is sending through.

This partial is then signed by the participant. Note that the `lpdu` hashes cover a content hash of the LPDU itself.

For example:

```json
{
  "room_id": "!abc:example.org",
  "type": "m.room.member",
  "state_key": "@alice:first.example.org",
  "sender": "@bob:second.example.org",
  "origin_server_ts": 1681340188825,
  "hub_server": "first.example.org",
  "content": {
    "membership": "invite"
  },
  "hashes": {
    "lpdu": {
      "sha256": "<unpadded base64>"
    }
  },
  "signatures": {
    "first.example.org": {
      "ed25519:1": "<unpadded base64 signature covering whole event>"
    }
  },
  "unsigned": {
    "arbitrary": "fields"
  }
}
```

The hub then appends the missing fields, hashes it, and signs it:


```json
{
  "room_id": "!abc:example.org",
  "type": "m.room.member",
  "state_key": "@alice:first.example.org",
  "sender": "@bob:second.example.org",
  "origin_server_ts": 1681340188825,
  "hub_server": "first.example.org",
  "content": {
    "membership": "invite"
  },
  "hashes": {
    "lpdu": {
      "sha256": "<unpadded base64>"
    },
    "sha256": "<unpadded base64>"
  },
  "signatures": {
    "first.example.org": {
      "ed25519:1": "<unpadded base64 signature covering whole event>"
    },
    "second.example.org": {
      "ed25519:1": "<unpadded base64 signature covering LPDU>"
    }
  },
  "auth_events": ["$first", "$second"],
  "prev_events": ["$parent"],
  "unsigned": {
    "arbitrary": "fields"
  }
}
```

`prev_events` for events created by the hub *should* always have exactly 1 event ID. If the server is dual-stack then
it may have multiple.

The presence of `hub_server` denotes the event was sent by a LM server. If not present, the event is assumed to be
sent by a DAG-capable server.

When `hub_server` is present, only the signature implied by the domain name of the `sender` is special (as it signs
the LPDU). All other signatures are expected to be validated as though they sign the fully-formed PDU, per normal.

### Content Hash

The only change is made to Step 1 of the content hashing algorithm. The full algorithm is:

1. Remove any existing `unsigned` and `signatures` fields.
   1. If calculating an LPDU's content hash, remove any existing `hashes` field as well.
   2. If *not* calculating an LPDU's content hash, remove any existing fields under `hashes` except
      for `lpdu`.
2. Encode the object using canonical JSON.
3. Hash the resulting bytes with SHA-256.
4. Encode the hash using unpadded base64.

## Request Authentication

There are largely clarifications to how request authentication works. Namely:

* A 401 M_FORBIDDEN error is returned when improperly authenticated.
* Only one of the sender's signing keys needs to be used, but senders should send as many as possible (if the server
  has multiple signing keys).
* A failure in any one `Authorization` header is treated as fatal for the whole request.
* `GET` requests, or those without a request body, are represented as `{}` in the signed JSON.
* `destination` in the `Authorization` header is formally required, though backwards compatible with Matrix today.

This is meant to be compatible with [MSC4029](https://github.com/matrix-org/matrix-spec-proposals/pull/4029), when
MSC4029 has real content in it.

## Linearization Algorithm

The specific mechanics of this algorithm are undefined. The rough idea is that when a DAG-capable server becomes
involved in the room that it get transfered hub status as well, if the hub isn't already pointing to a DAG-capable
server. That DAG-capable server then does local linearization based on the to-be-defined algorithm.

Characteristics of the algorithm are:
* Events are always appended to the HEAD of the array (for LM), or at least operates like that over the API surface.
* Because eventual consistency and state res can cause problems, "fix it" events might need to be sent. For example,
  "actually, Alice is banned now" events. Note that this will also need considering for state resets.

Identifying events which need linearizing is not yet decided, but it'll likely be some combination/either of the
`hub_server` and `prev_events` (when >1 value) properties.

Note that the implication here is that *all* DAG-capable servers would be expected to become dual stack servers,
supporting the semantics/details of LM servers. All DAG-capable servers can become hubs.

## Hub Transfers

Not yet decided on how this works. It's possible we want to support situations where there's multiple hubs in a room.

Rough theory is we can advertise that a server supports being a hub server (or is a DAG-capable server) somewhere,
and participants can choose their favourite, creating small clusters of LM servers in the room.

## HTTP API Changes

TBD.