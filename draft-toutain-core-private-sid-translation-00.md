---
title: "Private SID Translation for CORECONF"
abbrev: "private-sid-translation"
category: info
ipr: trust200902

docname: draft-toutain-core-private-sid-translation-00
submissiontype: IETF
number:
date:
consensus: true
v: 3

area: ART
workgroup: Constrained RESTful Environments (CoRE)

keyword:
  - CORECONF
  - CoMI
  - YANG
  - SID
  - constrained devices

venue:
  group: CoRE
  type: Working Group
  mail: core@ietf.org
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
  RFC9595:   # YANG SID
  RFC2119:
  RFC8174:
  I-D.ietf-core-comi:
  I-D.ietf-core-sid:

informative:
  RFC8949:   # CBOR
  RFC8724:   # SCHC
  RFC9363:   # SCHC Rule Data Model

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
privateSID = (offset - 1) - (officialSID - firstSID)
~~~

where:

* `firstSID` is the lowest SID in the official range allocated to the model.
* `officialSID` is the globally allocated SID of the node to be translated.
* `offset` is a non-positive integer that shifts the private range to avoid
  overlapping with other translated models (e.g., 0 for the first model).

With `offset = 0`, the node whose `officialSID` equals `firstSID` maps to -1,
the next node maps to -2, and so on. The resulting private SIDs form a dense
sequence of negative integers starting from -1, each encoding in a single
byte in CBOR for values in the range -1 to -24 {{RFC8949}}.

The inverse translation is:

~~~
officialSID = firstSID - (privateSID - offset + 1)
~~~

Both endpoints MUST be configured with identical values of `firstSID` and
`offset` for the translation to be consistent. The translation table is
therefore fully determined by these two parameters together with the official
SID range of the model.

## Offset

The `offset` parameter allows multiple YANG Data Models to be translated
into non-overlapping regions of the private SID space.

The first module SHOULD use `offset = 0`, so that its first node maps to
private SID -1 and the translated range starts at the most compact encoding
possible.

When a second module must also be translated, its `offset` MUST be set so
that its range does not overlap with the first module. If the first module
covers N SIDs (i.e., its official SID range has N entries), then the second
module MUST use:

~~~
offset = -N
~~~

More generally, each additional module uses an offset equal to the negative
of the cumulative number of SIDs already allocated to previously translated
modules. This ensures that all private SIDs remain distinct and that each
endpoint can unambiguously determine which module a received private SID
belongs to.

# Processing and Interoperability {#processing-and-interoperability}

Private SID translation is a wire-encoding optimization applied exclusively
when transmitting or receiving datastore elements over a constrained link.
It MUST NOT be used as an internal representation for data processing.

Any validation, comparison, augmentation, or other processing of YANG
data MUST be performed using the official SID values. An implementation
MUST translate incoming private SIDs back to their official counterparts
before passing data to any processing layer, and MUST translate outgoing
data to private SIDs only at the point of serialization.

If a YANG Data Model has not been configured for private SID translation,
its SIDs are transmitted as-is, using their official values. Translation
is therefore purely opt-in per model: untranslated models and translated
models can coexist within the same message, each identified by whether
the SID falls in the negative (private) or positive (official) range.

# Case Studies

## SCHC Management

SCHC {{RFC8724}} compresses IPv6, UDP, and CoAP headers by applying rules stored
in a Static Context shared between two entities. When the Static Context needs
to be updated, management messages must be exchanged between these two entities.

In this deployment, the only YANG Data Model in use is the SCHC Rule Data Model
{{RFC9363}}. Both endpoints are pre-configured with the same private SID
translation table covering exactly that model, and no other YANG module is
expected to be exchanged over the link. There is therefore no risk of SID
collision with another data model, and full CORECONF interoperability with
third-party implementations is not required.

The SCHC SID allocation {{RFC9363}} has already been manually designed to
optimize delta encoding: nodes are ordered so that consecutive siblings stay
within a delta range of 23, which allows each CBOR map key to be encoded in a
single byte.

However, SCHC rules make extensive use of identityref leaves (field-id,
direction-indicator, matching-operator, comp-decomp-action) to allow easy
extensibility. Unlike map keys, identityref values are encoded as absolute SIDs,
not as deltas. Even though the SCHC SIDs are relatively small (around 2550-2950),
each identityref value still requires 3 bytes of CBOR encoding. A typical rule
entry contains four such identityrefs, contributing 12 bytes per field entry
in overhead.

Private SID translation addresses both costs: delta keys become small negative
integers (one byte each), and identityref values mapped into the -1 to -24 range
also encode in a single byte, reducing the per-entry identityref overhead from
12 bytes to 4.

This makes SCHC management an ideal candidate for private SID translation: the
set of nodes is fixed and known at deployment time, the two endpoints share an
implicit translation context, and the reduction in encoding size directly lowers
the overhead of management messages on the constrained link that SCHC itself is
compressing.

Note that if the Data Model is augmented, the additional nodes can either be
covered by a separate private SID zone or left with their official SIDs.

An illustration of this translation applied to an IPv6/UDP/CoAP compression
rule is provided in {{sec-example-schc}}.

## PEN 

Private Enterprise Numbers (PENs) are IANA-assigned identifiers used by
organizations to define their own private SID ranges, following the formula
described in {{RFC9595}}. Since PENs can reach values up to 2^32, the resulting
SID values can be very large and require up to 9 bytes of CBOR encoding.

Even if delta encoding is used to limit the impact of large SID values,
the base SID still appears in full at least once per message. The cost
may therefore be significant for short messages such as FETCH or iPATCH
requests.

## Other SDOs

SID ranges are allocated globally by IANA following {{RFC9595}}. The IETF
receives the first million values (1 to 999,999). Within this range, most
IETF YANG Data Models are assigned SIDs in the low thousands, requiring at
most 3 bytes of CBOR encoding per absolute value, and benefiting from
single-byte delta keys within a well-ordered model.

Other Standards Development Organizations (SDOs) such as ETSI, 3GPP, or
oneM2M receive ranges starting in the millions. Their SID values immediately
require 5 bytes of CBOR encoding for absolute references, and even delta keys
spanning tens of thousands of positions may cost 3 to 5 bytes. This
represents a significant overhead for any deployment over constrained
networks — precisely the environments where those SDOs are often active
(e.g., NB-IoT, LoRaWAN, or Zigbee-based management).

Private SID translation levels this asymmetry: regardless of where a SID
range falls in the global allocation, the entire model can be remapped to
a dense set of small negative integers, giving any SDO the same on-wire
efficiency that the IETF enjoys by virtue of its early allocation.

# SID Allocation Strategy

The ordering of nodes within the official SID range directly determines which
private SIDs they receive. Since the formula maps the node at `firstSID` to
`offset - 1` (e.g., -1 when offset=0), `firstSID+1` to `offset - 2` (e.g., -2), and so on,
nodes that appear most frequently in on-wire messages SHOULD be allocated
the lowest offsets from `firstSID` so that they receive the smallest private
SIDs and benefit from single-byte CBOR encoding (values -1 to -24).

For a YANG Data Model containing identityref leaves, the values of those
identities are the elements that appear most often in serialized instances.
The SID allocation SHOULD therefore place the most-used identity values
at the beginning of the range, ahead of structural nodes and less-frequent
identities.

## Example: SCHC Rule Data Model {#sec-example-schc}

In the SCHC Rule Data Model {{RFC9363}}, every compression rule entry
contains at least one Matching Operator (MO), one Compression/Decompression
Action (CDA), and a Field Length function. These three categories of
identityref appear in every field descriptor of every rule and are therefore
the most bandwidth-sensitive values.

The allocation for the SCHC Rule Data Model uses `firstSID` = 2551 and
`offset` = 0. The first 24 entries (private SIDs -1 to -24) cover MO, CDA,
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

The following figure shows an IPv6/UDP/CoAP compression rule with private SID
translation applied. All delta keys in CBOR maps become negative because the
private SID delta equals the negative of the official delta:
`p(child) - p(parent) = -(child - parent)`. Identityref values (field-id,
direction-indicator, matching-operator, comp-decomp-action) are also replaced
by their private SID aliases. Target values are encoded as h'hex'.

The following example shows the translation from a SCHC rule into the private SID address space. The start of the rule in JSON si given for clarity. The full Set or Rules contains 5 compression rules. The file before translation is 3994 byte long and after translation 3057, so a compression rate of 23 %. 
 

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

{2700: {23: [
  {23: [
    {1: 0,  2: 2575, 5: 4,  8: 1, 7: 2562, 12: 2551, 16: 2555,
                              9: [{1: 0, 2: h'06'}]},
    {1: 1,  2: 2576, 5: 8,  8: 1, 7: 2562, 12: 2552, 16: 2556,
                              9: [{1: 0, 2: h'00'}]},
    {1: 2,  2: 2579, 5: 20, 8: 1, 7: 2562, 12: 2552, 16: 2556,
                              9: [{1: 0, 2: h'00'}]},
    {1: 3,  2: 2580, 5: 16, 8: 1, 7: 2562, 12: 2552, 16: 2559},
    {1: 4,  2: 2581, 5: 8,  8: 1, 7: 2562, 12: 2551, 16: 2555,
                              9: [{1: 0, 2: h'11'}]},
    {1: 5,  2: 2582, 5: 8,  8: 1, 7: 2562, 12: 2552, 16: 2555,
                              9: [{1: 0, 2: h'ff'}]},
    {1: 6,  2: 2584, 5: 64, 8: 1, 7: 2562, 12: 2551, 16: 2555,
                              9: [{1: 0, 2: h'2001066073015c4c'}]},
    {1: 7,  2: 2583, 5: 64, 8: 1, 7: 2562, 12: 2551, 16: 2555,
                              9: [{1: 0, 2: h'0000000000000005'}]},
    {1: 8,  2: 2586, 5: 64, 8: 1, 7: 2562, 12: 2552, 16: 2556},
    {1: 9,  2: 2585, 5: 64, 8: 1, 7: 2562, 12: 2552, 16: 2556},
    {1: 10, 2: 2590, 5: 16, 8: 1, 7: 2562, 12: 2551, 16: 2555,
                              9: [{1: 0, 2: h'1633'}]},
    {1: 11, 2: 2591, 5: 16, 8: 1, 7: 2562, 12: 2552, 16: 2556},
    {1: 12, 2: 2592, 5: 16, 8: 1, 7: 2562, 12: 2552, 16: 2559,
                              9: [{1: 0, 2: h'00'}]},
    {1: 13, 2: 2593, 5: 16, 8: 1, 7: 2562, 12: 2552, 16: 2559,
                              9: [{1: 0, 2: h'00'}]},
    {1: 14, 2: 2600, 5: 2,  8: 1, 7: 2562, 12: 2551, 16: 2555,
                              9: [{1: 0, 2: h'01'}]},
    {1: 15, 2: 2601, 5: 2,  8: 1, 7: 2562, 12: 2551, 16: 2555,
                              9: [{1: 0, 2: h'01'}]},
    {1: 16, 2: 2602, 5: 4,  8: 1, 7: 2562, 12: 2552, 16: 2556,
                              9: [{1: 0, 2: h'00'}]},
    {1: 17, 2: 2603, 5: 8,  8: 1, 7: 2562, 12: 2553, 16: 2557,
                              9: [{1: 0, 2: h'05'},
                                  {1: 1, 2: h'07'},
                                  {1: 2, 2: h'45'}]},
    {1: 18, 2: 2606, 5: 16, 8: 1, 7: 2562, 12: 2552, 16: 2556,
                              9: [{1: 0, 2: h'00'}]},
    {1: 19, 2: 2607, 5: CBORTag(45, 2566), 6: 16,
             8: 1, 7: 2562, 12: 2552, 16: 2556},
    {1: 20, 3: 2571, 4: 11, 5: CBORTag(45, 2568),
             8: 1, 7: 2563, 12: 2553, 16: 2557,
                              9: [{1: 0, 2: h'63'},
                                  {1: 1, 2: h'73'}]},
    {1: 21, 3: 2571, 4: 12, 5: CBORTag(45, 2568),
             8: 1, 7: 2563, 12: 2553, 16: 2557,
                              9: [{1: 0, 2: h'8d'},
                                  {1: 1, 2: h'8e'}]},
    {1: 22, 3: 2571, 4: 12, 5: CBORTag(45, 2568),
             8: 1, 7: 2564, 12: 2551, 16: 2555,
                              9: [{1: 0, 2: h'8e'}]},
    {1: 23, 3: 2571, 4: 15, 5: CBORTag(45, 2568),
             8: 1, 7: 2563, 12: 2551, 16: 2555,
                              9: [{1: 0, 2: h'643d30'}]},
    {1: 24, 3: 2571, 4: 17, 5: CBORTag(45, 2568),
             8: 1, 7: 2563, 12: 2551, 16: 2555,
                              9: [{1: 0, 2: h'8e'}]}],
  2: 0, 1: 5, 3: 2655},

{-151: {-23: [
  {-23: [
    {-1: 0,  -2: -26, -5: 4,  -8: 1, -7: -13, -12: -2, -16: -6,
                               -9: [{-1: 0, -2: h'06'}]},
    {-1: 1,  -2: -27, -5: 8,  -8: 1, -7: -13, -12: -3, -16: -7,
                               -9: [{-1: 0, -2: h'00'}]},
    {-1: 2,  -2: -30, -5: 20, -8: 1, -7: -13, -12: -3, -16: -7,
                               -9: [{-1: 0, -2: h'00'}]},
    {-1: 3,  -2: -31, -5: 16, -8: 1, -7: -13, -12: -3, -16: -10},
    {-1: 4,  -2: -32, -5: 8,  -8: 1, -7: -13, -12: -2, -16: -6,
                               -9: [{-1: 0, -2: h'11'}]},
    {-1: 5,  -2: -33, -5: 8,  -8: 1, -7: -13, -12: -3, -16: -6,
                               -9: [{-1: 0, -2: h'ff'}]},
    {-1: 6,  -2: -35, -5: 64, -8: 1, -7: -13, -12: -2, -16: -6,
                               -9: [{-1: 0, -2: h'2001066073015c4c'}]},
    {-1: 7,  -2: -34, -5: 64, -8: 1, -7: -13, -12: -2, -16: -6,
                               -9: [{-1: 0, -2: h'0000000000000005'}]},
    {-1: 8,  -2: -37, -5: 64, -8: 1, -7: -13, -12: -3, -16: -7},
    {-1: 9,  -2: -36, -5: 64, -8: 1, -7: -13, -12: -3, -16: -7},
    {-1: 10, -2: -41, -5: 16, -8: 1, -7: -13, -12: -2, -16: -6,
                               -9: [{-1: 0, -2: h'1633'}]},
    {-1: 11, -2: -42, -5: 16, -8: 1, -7: -13, -12: -3, -16: -7},
    {-1: 12, -2: -43, -5: 16, -8: 1, -7: -13, -12: -3, -16: -10,
                               -9: [{-1: 0, -2: h'00'}]},
    {-1: 13, -2: -44, -5: 16, -8: 1, -7: -13, -12: -3, -16: -10,
                               -9: [{-1: 0, -2: h'00'}]},
    {-1: 14, -2: -51, -5: 2,  -8: 1, -7: -13, -12: -2, -16: -6,
                               -9: [{-1: 0, -2: h'01'}]},
    {-1: 15, -2: -52, -5: 2,  -8: 1, -7: -13, -12: -2, -16: -6,
                               -9: [{-1: 0, -2: h'01'}]},
    {-1: 16, -2: -53, -5: 4,  -8: 1, -7: -13, -12: -3, -16: -7,
                               -9: [{-1: 0, -2: h'00'}]},
    {-1: 17, -2: -54, -5: 8,  -8: 1, -7: -13, -12: -4, -16: -8,
                               -9: [{-1: 0, -2: h'05'},
                                    {-1: 1, -2: h'07'},
                                    {-1: 2, -2: h'45'}]},
    {-1: 18, -2: -57, -5: 16, -8: 1, -7: -13, -12: -3, -16: -7,
                               -9: [{-1: 0, -2: h'00'}]},
    {-1: 19, -2: -58, -5: CBORTag(45, -17), -6: 16,
              -8: 1, -7: -13, -12: -3, -16: -7},
    {-1: 20, -3: -22, -4: 11, -5: CBORTag(45, -19),
              -8: 1, -7: -14, -12: -4, -16: -8,
                               -9: [{-1: 0, -2: h'63'},
                                    {-1: 1, -2: h'73'}]},
    {-1: 21, -3: -22, -4: 12, -5: CBORTag(45, -19),
              -8: 1, -7: -14, -12: -4, -16: -8,
                               -9: [{-1: 0, -2: h'8d'},
                                    {-1: 1, -2: h'8e'}]},
    {-1: 22, -3: -22, -4: 12, -5: CBORTag(45, -19),
              -8: 1, -7: -15, -12: -2, -16: -6,
                               -9: [{-1: 0, -2: h'8e'}]},
    {-1: 23, -3: -22, -4: 15, -5: CBORTag(45, -19),
              -8: 1, -7: -14, -12: -2, -16: -6,
                               -9: [{-1: 0, -2: h'643d30'}]},
    {-1: 24, -3: -22, -4: 17, -5: CBORTag(45, -19),
              -8: 1, -7: -14, -12: -2, -16: -6,
                               -9: [{-1: 0, -2: h'8e'}]}],
  -2: 0, -1: 5, -3: -106},

~~~~
{: #fig-schc-rule-private title="IPv6/UDP/CoAP rule with private SID translation" artwork-align="left"}


# SCHC Considerations

When SCHC {{RFC8724}} is used to compress CORECONF messages that carry
private SIDs, the compression rules themselves need to account for the
translation mapping. This document defines a new Compression/Decompression
Action (CDA) to support this use case.

## sid-translation CDA

The `sid-translation` CDA is applied to fields that carry SID values
(map keys and identityref values) in a CORECONF CBOR message. It replaces
the SID value in the field with its private alias, or restores the original
SID upon decompression, according to the translation parameters configured
in the rule.

The CDA takes the following Function Arguments, which MAY be repeated to
cover multiple YANG Data Models translated in the same session:

`first-sid`:
: The lowest SID value in the official range allocated to a given YANG
  Data Model (the `firstSID` parameter as defined in this document).

`sid-range`:
: The number of consecutive SIDs covered by the translation range for
  that model, starting from `first-sid`.

`offset`:
: The offset parameter for that model's private SID translation, as
  defined in this document.  Combined with `first-sid`, it fully determines
  the bijective mapping between official and private SIDs.

Each (`first-sid`, `sid-range`, `offset`) triplet describes the translation
for one YANG Data Model. When multiple models are translated in the same
session, one triplet per model is included in the Function Arguments list,
ordered by ascending `first-sid`.

During compression, the residue sent for a field carrying an official SID
that falls within a configured range is the corresponding private SID.
For SID values outside all configured ranges, the official SID is sent
unchanged. During decompression, the inverse formula is applied to recover
the official SID.

The following example shows a SCHC rule entry that applies `sid-translation`
to the CORECONF payload field, covering the SCHC Rule Data Model SID range
(first-sid=2550, sid-range=400) with no additional offset (offset=0):

~~~~
/--------+-----+----+----+----+--------+------------------------\
|  FID   | FL  | FP | DI | TV |   MO   |          CDA           |
+========+=====+====+====+====+========+========================+
| CoAP   | var |  1 | bi |    | ignore | sid-translation        |
| Payload|     |    |    |    |        | (2550, 400, 0)         |
\--------+-----+----+----+----+--------+------------------------/
~~~~
{: #fig-sid-translation-rule title="SCHC rule entry for CORECONF SID translation" artwork-align="left"}

# Security Considerations

TODO

# IANA Considerations

This document requests that IANA update the "YANG Schema Item iDentifier
(SID)" registry established by {{RFC9595}} to define the semantics of
negative SID values.

Specifically, this document requests the reservation of the range -1 to
-1000 for use as private SIDs, as defined in this document. These values
MUST NOT be used as globally allocated SIDs. They are intended exclusively
for local, bilateral use between two endpoints that have agreed on a
private SID translation configuration, as described in {{processing-and-interoperability}}.

The following table summarizes the requested addition to the SID registry:

| Range       | Registration Procedure | Description                  | Reference     |
|------------:|:-----------------------|:-----------------------------|:--------------|
| -1 to -1000 | Reserved               | Private SID (local use only) | This document |
{: title="Addition to the YANG SID Registry"}

--- back

# Acknowledgments
{:numbered="false"}

TODO
