---
title: "Private SID Translation for CORECONF"
abbrev: "private-sid-translation"
category: info
ipr: trust200902

docname: draft-toutain-t2trg-private-sid-translation-00
submissiontype: IRTF
number:
date:
consensus: false
v: 3

area: IRTF
workgroup: Thing-to-Thing Research Group (T2TRG)

keyword:
  - CORECONF
  - CoMI
  - YANG
  - SID
  - constrained devices

venue:
  group: T2TRG
  type: Research Group
  mail: t2trg@irtf.org
  github: "ltn22/coreconf-m2m"

author:
  -
    fullname: Laurent Toutain
    organization: IMT Atlantique
    email: laurent.toutain@imt-atlantique.fr

normative:
  RFC7252:   # CoAP
  RFC7950:   # YANG 1.1
  RFC9254:   # YANG-CBOR
  I-D.ietf-core-comi:
  I-D.ietf-core-sid:

informative:
  RFC8949:   # CBOR

--- abstract

This document describes a mechanism for translating privately assigned YANG SID
values to globally allocated SIDs, enabling constrained devices to use compact
local identifiers while remaining interoperable with standard CORECONF
implementations.

--- middle

# Introduction

YANG SID identifiers {{I-D.ietf-core-sid}} are designed to be globally unique,
ensuring that any node from any YANG Data Model can be unambiguously identified
without conflict across implementations and deployments. SIDs are compact
integers, far smaller than their ASCII path equivalents, and their delta
encoding in CBOR maps {{RFC9254}} further reduces the on-wire size.

However, on very constrained networks — such as LPWANs operating under strict
duty cycles or satellite links with tight bandwidth budgets — even a few extra
bytes per message can have a significant impact on energy consumption and
transmission cost. In these environments, the globally allocated SID values,
which may require two or more bytes in CBOR encoding, can still be considered
too large.

This document defines a mechanism for using small negative integers as private
SID aliases. Negative SIDs have no meaning in the global allocation scheme and
encode in a single byte in CBOR for values in the range -1 to -24. A device
can therefore maintain a local translation table mapping each official SID to a
private negative alias, use the compact alias in all on-air exchanges, and
perform the substitution to the canonical SID when interoperability with
standard CORECONF implementations is required.

This approach is intended for closed deployments where both endpoints share the
same translation table and full CORECONF interoperability is not a requirement.

## Requirements Language

{::comment}
Standard boilerplate for normative language.
{:/comment}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they appear in
all capitals, as shown here.

# Private SID Computation

Each YANG Data Model is assigned a contiguous range of globally allocated SIDs.
The private SID translator uses that allocation to derive a compact negative
alias for every node in the model according to the following formula:

~~~
privateSID = offset - (officialSID - firstSID)
~~~

where:

* `firstSID` is the lowest SID in the official range allocated to the model.
* `officialSID` is the globally allocated SID of the node to be translated.
* `offset` is a small negative integer chosen as the starting point of the
  private range (e.g., -1).

With `offset = -1`, the node whose `officialSID` equals `firstSID` maps to -1,
the next node maps to -2, and so on. The resulting private SIDs form a dense
sequence of negative integers starting from `offset`, each encoding in a single
byte in CBOR for values in the range -1 to -24 {{RFC8949}}.

The inverse translation is:

~~~
officialSID = firstSID - (privateSID - offset)
~~~

Both endpoints MUST be configured with identical values of `firstSID` and
`offset` for the translation to be consistent. The translation table is
therefore fully determined by these two parameters together with the official
SID range of the model.

# Case Studies

## SCHC Management

SCHC {{RFC8724}} compresses IPv6, UDP, and CoAP headers by applying rules stored
in Static Context shared between two entities. When the Static Context needs to be updated 
management messages must be exchanged
between these two entities.

In this deployment, the only YANG Data Model in use is the SCHC Rule  Data Model
{{RFC9363}}. Both endpoints are pre-configured with the same private SID
translation table covering exactly that model, and no other YANG module is
expected to be exchanged over the link. There is therefore no risk of SID
collision with another data model, and full CORECONF interoperability with
third-party implementations is not required.

This makes SCHC management an ideal candidate for private SID translation:
the set of nodes is fixed and known at deployment time, the two endpoints
share an implicit translation context, and the reduction in SID encoding size
directly lowers the overhead of management messages on the constrained link
that SCHC itself is compressing.

Note that the Data Model may be augmented, in that case, either these models
are also translated in another zone, or use the official SIDs.

The following figure shows an IPv6/UDP/CoAP compression rule with private SID
translation applied. All delta keys in CBOR maps become negative because the
private SID delta equals the negative of the official delta:
`p(child) - p(parent) = -(child - parent)`. Identityref values (field-id,
direction-indicator, matching-operator, comp-decomp-action) are also replaced
by their private SID aliases. Target values are encoded as h'hex'.

~~~~
{
  "ietf-schc:schc": {
    "rule": [
      {
        "entry": [
          {
            "entry-index": 0,
            "field-id": "ietf-schc:fid-ipv6-version",
            "field-length": 4,
            "field-position": 1,
            "direction-indicator": "ietf-schc:di-bidirectional",
            "matching-operator": "ietf-schc:mo-equal",
            "comp-decomp-action": "ietf-schc:cda-not-sent",
            "target-value": [
              {
                "index": 0,
                "value": "Bg=="
              }
            ]
          },
          {
            "entry-index": 1,
            "field-id": "ietf-schc:fid-ipv6-trafficclass",
            "field-length": 8,
            "field-position": 1,
            "direction-indicator": "ietf-schc:di-bidirectional",
            "matching-operator": "ietf-schc:mo-ignore",
            "comp-decomp-action": "ietf-schc:cda-value-sent",
            "target-value": [
              {
                "index": 0,
                "value": "AA=="
              }
            ]
          },
          {
            "entry-index": 2,
            "field-id": "ietf-schc:fid-ipv6-flowlabel",
            "field-length": 20,
            "field-position": 1,
            "direction-indicator": "ietf-schc:di-bidirectional",
            "matching-operator": "ietf-schc:mo-ignore",
            "comp-decomp-action": "ietf-schc:cda-value-sent",
            "target-value": [
              {
                "index": 0,
                "value": "AA=="
              }
            ]
          },

{2700: {23: [{23: [{1: 0, 2: 2575, 5: 4, 8: 1, 7: 2562, 12: 2551, 16: 2555,
                                                      9: [{1: 0, 2: b'\x06'}]},
                   {1: 1, 2: 2576, 5: 8, 8: 1, 7: 2562, 12: 2552, 16: 2556,
                                                      9: [{1: 0, 2: b'\x00'}]},
                   {1: 2, 2: 2579, 5: 20, 8: 1, 7: 2562, 12: 2552, 16: 2556,
                                                      9: [{1: 0, 2: b'\x00'}]},
                   {1: 3, 2: 2580, 5: 16, 8: 1, 7: 2562, 12: 2552, 16: 2559},
                   {1: 4, 2: 2581, 5: 8, 8: 1, 7: 2562, 12: 2551, 16: 2555,
                                                      9: [{1: 0, 2: b'\x11'}]},
                   {1: 5, 2: 2582, 5: 8, 8: 1, 7: 2562, 12: 2552, 16: 2555,
                                                      9: [{1: 0, 2: b'\xff'}]},
                   {1: 6, 2: 2584, 5: 64, 8: 1, 7: 2562, 12: 2551, 16: 2555,
                                                      9: [{1: 0, 2: b' \x01\x06`s\x01\\L'}]},
                   {1: 7, 2: 2583, 5: 64, 8: 1, 7: 2562, 12: 2551, 16: 2555,
                                                      9: [{1: 0, 2: b'\x00\x00\x00\x00\x00\x00\x00\x05'}]},
                   {1: 8, 2: 2586, 5: 64, 8: 1, 7: 2562, 12: 2552, 16: 2556},
                   {1: 9, 2: 2585, 5: 64, 8: 1, 7: 2562, 12: 2552, 16: 2556},
                   {1: 10, 2: 2590, 5: 16, 8: 1, 7: 2562, 12: 2551, 16: 2555,
                                                      9: [{1: 0, 2: b'\x163'}]},
                   {1: 11, 2: 2591, 5: 16, 8: 1, 7: 2562, 12: 2552, 16: 2556},
                   {1: 12, 2: 2592, 5: 16, 8: 1, 7: 2562, 12: 2552, 16: 2559,
                                                      9: [{1: 0, 2: b'\x00'}]},
                   {1: 13, 2: 2593, 5: 16, 8: 1, 7: 2562, 12: 2552, 16: 2559,
                                                      9: [{1: 0, 2: b'\x00'}]},
                   {1: 14, 2: 2600, 5: 2, 8: 1, 7: 2562, 12: 2551, 16: 2555,
                                                      9: [{1: 0, 2: b'\x01'}]},
                   {1: 15, 2: 2601, 5: 2, 8: 1, 7: 2562, 12: 2551, 16: 2555,
                                                      9: [{1: 0, 2: b'\x01'}]},
                   {1: 16, 2: 2602, 5: 4, 8: 1, 7: 2562, 12: 2552, 16: 2556,
                                                      9: [{1: 0, 2: b'\x00'}]},
                   {1: 17, 2: 2603, 5: 8, 8: 1, 7: 2562, 12: 2553, 16: 2557,
                                                      9: [{1: 0, 2: b'\x05'}, {1: 1, 2: b'\x07'}, {1: 2, 2: b'E'}]},
                   {1: 18, 2: 2606, 5: 16, 8: 1, 7: 2562, 12: 2552, 16: 2556,
                                                      9: [{1: 0, 2: b'\x00'}]},
                   {1: 19, 2: 2607, 5: CBORTag(45, 2566), 6: 16, 8: 1, 7: 2562, 12: 2552, 16: 2556},
                   {1: 20, 3: 2571, 4: 11, 5: CBORTag(45, 2568), 8: 1, 7: 2563, 12: 2553, 16: 2557,
                                                      9: [{1: 0, 2: b'c'}, {1: 1, 2: b's'}]},
                   {1: 21, 3: 2571, 4: 12, 5: CBORTag(45, 2568), 8: 1, 7: 2563, 12: 2553, 16: 2557,
                                                      9: [{1: 0, 2: b'\x8d'}, {1: 1, 2: b'\x8e'}]},
                   {1: 22, 3: 2571, 4: 12, 5: CBORTag(45, 2568), 8: 1, 7: 2564, 12: 2551, 16: 2555,
                                                      9: [{1: 0, 2: b'\x8e'}]},
                   {1: 23, 3: 2571, 4: 15, 5: CBORTag(45, 2568), 8: 1, 7: 2563, 12: 2551, 16: 2555,
                                                      9: [{1: 0, 2: b'd=0'}]},
                   {1: 24, 3: 2571, 4: 17, 5: CBORTag(45, 2568), 8: 1, 7: 2563, 12: 2551, 16: 2555,
                                                      9: [{1: 0, 2: b'\x8e'}]}],
              2: 0, 1: 5, 3: 2655},
   
{-151: {-23: [{-23: [{-16: -6,
                      -12: -2,
                      -9: [{-2: b'\x06', -1: 0}],
                      -8: 1,
                      -7: -13,
                      -5: 4,
                      -2: -26,
                      -1: 0},
~~~~
{: #fig-schc-rule-private title="IPv6/UDP/CoAP rule with private SID translation" artwork-align="left"}

# SID Allocation Strategy

The ordering of nodes within the official SID range directly determines which
private SIDs they receive. Since the formula maps the node at `firstSID` to
`offset` (e.g., -1), `firstSID+1` to `offset-1` (e.g., -2), and so on,
nodes that appear most frequently in on-wire messages SHOULD be allocated
the lowest offsets from `firstSID` so that they receive the smallest private
SIDs and benefit from single-byte CBOR encoding (values -1 to -24).

For a YANG Data Model containing identityref leaves, the values of those
identities are the elements that appear most often in serialized instances.
The SID allocation SHOULD therefore place the most-used identity values
at the beginning of the range, ahead of structural nodes and less-frequent
identities.

## Example: SCHC Rule Data Model

In the SCHC Rule Data Model {{RFC9363}}, every compression rule entry
contains at least one Matching Operator (MO), one Compression/Decompression
Action (CDA), and a Field Length function. These three categories of
identityref appear in every field descriptor of every rule and are therefore
the most bandwidth-sensitive values.

The allocation for the SCHC Rule Data Model uses `firstSID` = 2551 and
`offset` = -1. The first 24 entries (private SIDs -1 to -24) cover MO, CDA,
Direction Indicator, and Field Length identities, all encoding in a single CBOR
byte. Field Identifiers (FID) start at rank 24 (private SID -25) and require
two bytes, which remains acceptable since a FID appears once per field
descriptor whereas MO, CDA, and FL appear in every entry.

~~~~
 SID  Private  Identity
----  -------  ------------------------------
2551       -1  mo-equal
2552       -2  mo-ignore
2553       -3  mo-match-mapping
2554       -4  mo-msb
2555       -5  cda-not-sent
2556       -6  cda-value-sent
2557       -7  cda-mapping-sent
2558       -8  cda-lsb
2559       -9  cda-compute
2560      -10  cda-deviid
2561      -11  cda-appiid
2562      -12  di-bidirectional
2563      -13  di-down
2564      -14  di-up
2565      -15  fl-length-bits
2566      -16  fl-length-bytes
2567      -17  fl-token-length
2568      -18  fl-variable
2569      -19  fl-variable-bits
2570      -20  (reserved)
2571      -21  space-id-coap
2572      -22  (reserved)
2573      -23  (reserved)
2574      -24  (reserved)        ← last single-byte encoding
----  -------  ------------------------------
2575      -25  fid-ipv6-version
2576      -26  fid-ipv6-trafficclass
2577      -27  fid-ipv6-trafficclass-ds
2578      -28  fid-ipv6-trafficclass-ecn
2579      -29  fid-ipv6-flowlabel
2580      -30  fid-ipv6-payload-length
2581      -31  fid-ipv6-nextheader
2582      -32  fid-ipv6-hoplimit
2583      -33  fid-ipv6-deviid
2584      -34  fid-ipv6-devprefix
2585      -35  fid-ipv6-appiid
2586      -36  fid-ipv6-appprefix
2590      -40  fid-udp-dev-port
2591      -41  fid-udp-app-port
2592      -42  fid-udp-length
2593      -43  fid-udp-checksum
 ...      ...  ...
~~~~
{: #fig-schc-allocation title="Private SID allocation for the SCHC Rule Data Model" artwork-align="left"}

# Security Considerations

TODO

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO
