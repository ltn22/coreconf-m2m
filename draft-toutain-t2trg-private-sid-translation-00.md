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

# Security Considerations

TODO

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO
