---
title: "CORECONF for Machine-to-Machine Communication"
abbrev: "coreconf-m2m"
category: info

docname: draft-ietf-t2trg-coreconf-m2m-00
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
  - CBOR
  - CoAP
  - M2M
  - IoT
  - constrained devices
  - SID

venue:
  group: T2TRG
  type: Research Group
  mail: t2trg@irtf.org
  arch: https://mailarchive.ietf.org/arch/browse/t2trg/
  github: "ltn22/coreconf-m2m"
  latest: "https://ltn22.github.io/coreconf-m2m/draft-ietf-t2trg-coreconf-m2m.html"

author:
  -
    fullname: Laurent Toutain
    organization: IMT Atlantique
    email: laurent.toutain@imt-atlantique.fr

normative:
  RFC7252:   # CoAP
  RFC7950:   # YANG 1.1
  RFC8610:   # CDDL
  RFC9254:   # YANG-CBOR
  RFC9363:   # YANG-SID (SCHC YANG module)
  I-D.ietf-core-comi:

informative:
  RFC8724:   # SCHC
  RFC9179:   # SCHC Generic Rule Set
  RFC8949:   # CBOR
  RFC7396:   # JSON Merge Patch
  I-D.ietf-core-sid:
  I-D.ietf-core-yang-cbor:

--- abstract

This document describes the use of CORECONF (CoAP Management Interface) for
Machine-to-Machine (M2M) communication in constrained IoT environments.
It defines a YANG data model enabling remote management and configuration
of constrained devices using CoAP, CBOR, and YANG SID identifiers.
The document addresses the specific challenges of M2M interactions where
both endpoints may be constrained nodes, and explores the use of CORECONF
primitives (GET, PUT, POST/action, DELETE) in an M2M context, including
geo-location, RPC/action semantics, and augmentation patterns.

--- middle

# Introduction

The CORECONF protocol stack — combining CoAP {{RFC7252}}, YANG {{RFC7950}},
CBOR {{RFC8949}}, and YANG SID identifiers {{I-D.ietf-core-sid}} — provides
a compact and efficient management interface for constrained devices.

While CORECONF has been primarily designed for device management (operator
to device), its application to Machine-to-Machine (M2M) scenarios raises
specific challenges:

- Both endpoints may be constrained (e.g., Class 1 or Class 2 devices)
- Communication may occur over Low-Power Wide-Area Networks (LPWAN)
- Data models must be compact and efficiently encoded
- Action/RPC semantics are needed for peer-to-peer interactions

This document explores how CORECONF primitives can be adapted and used
in M2M contexts, and proposes a YANG module (`coreconf-m2m`) addressing
these requirements.

## Motivation

{::comment}
TODO: Elaborate on the motivation for M2M CORECONF usage.
Describe use cases from LPWAN, LoRaWAN, SCHC context.
{:/comment}

[TODO: Add motivation section]

## Use Cases

{::comment}
TODO: List concrete M2M use cases.
Examples: sensor-to-actuator, peer configuration exchange,
distributed SCHC rule negotiation, geo-aware routing.
{:/comment}

[TODO: Add use cases]

## Requirements Language

{::comment}
Standard boilerplate for normative language.
{:/comment}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they appear in
all capitals, as shown here.

# Terminology

The following terminology is used in this document:

CORECONF:
: CoAP Management Interface, as defined in {{I-D.ietf-core-comi}}.

M2M:
: Machine-to-Machine communication, referring to direct data exchanges
  between devices without human intervention.

SID:
: YANG Schema Item iDentifier, a compact numeric identifier for YANG
  data nodes, as defined in {{I-D.ietf-core-sid}}.

Constrained Device:
: A device with limited processing, memory, and energy resources,
  as characterized in {{?RFC7228}}.

{::comment}
TODO: Add further terminology as needed.
{:/comment}

# CORECONF Overview in the M2M Context

## CoAP Methods Mapping

CORECONF maps YANG operations to CoAP methods as follows:

| CoAP Method | YANG Operation     | M2M Semantic              |
|:-----------:|:------------------:|:-------------------------:|
| GET         | data retrieval     | Read peer state           |
| PUT         | data replacement   | Set peer configuration    |
| POST        | action / RPC       | Trigger peer operation    |
| DELETE      | data deletion      | Reset / clear peer state  |
| iPATCH      | partial update     | Incremental config update |

## SID-Based Addressing

[TODO: Describe how SIDs are used to address YANG data nodes over CoAP URIs]

## CBOR Encoding

[TODO: Describe YANG-to-CBOR encoding per {{RFC9254}} in M2M context]

# The `coreconf-m2m` YANG Module

## Module Overview

The `coreconf-m2m` YANG module defines data structures enabling:

1. **Geo-location** — incorporation of RFC 9179-style geographic coordinates
   for location-aware M2M interactions
2. **Action/RPC semantics** — peer-triggered operations with input/output
   parameter encoding in CBOR
3. **Augment patterns** — extension of base device models for M2M-specific
   attributes

## Module Structure

~~~~ yang
module coreconf-m2m {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:coreconf-m2m";
  prefix "cm2m";

  // TODO: Add imports
  // import ietf-geo-location { ... }
  // import ietf-schc { ... }

  organization
    "IETF T2TRG Research Group";

  contact
    "WG Web:   <https://datatracker.ietf.org/rg/t2trg/>
     WG List:  <mailto:t2trg@irtf.org>

     Author:   Laurent Toutain
               <mailto:laurent.toutain@imt-atlantique.fr>";

  description
    "YANG module for CORECONF Machine-to-Machine communication.

     Copyright (c) 2026 IETF Trust and the persons identified as
     authors of the code. All rights reserved.";

  revision 2026-03-25 {
    description "Initial version.";
    reference "draft-ietf-t2trg-coreconf-m2m-00";
  }

  // ===== Geo-location grouping =====
  // TODO: Define or import geo-location grouping (cf. RFC 9179)

  // ===== M2M peer description =====
  // TODO: Define peer identity and capabilities container

  // ===== Actions / RPCs =====
  // TODO: Define action nodes for M2M operations

  // ===== Augmentations =====
  // TODO: Augment base SCHC or device models if needed

}
~~~~

## Geo-location Support

{::comment}
TODO: Describe integration of geographic location data (latitude, longitude,
altitude) following RFC 9179 patterns. Explain SID assignment for geo nodes.
{:/comment}

[TODO: Add geo-location section]

## Action and RPC Semantics

{::comment}
TODO: Describe how YANG actions are triggered via CoAP POST,
CBOR-encoded input/output, and the mapping to M2M interactions.
{:/comment}

[TODO: Add action/RPC section]

## Augment Patterns

{::comment}
TODO: Describe how coreconf-m2m augments existing YANG modules
(e.g., ietf-schc) to add M2M-specific data nodes.
{:/comment}

[TODO: Add augment section]

# SID Allocation

{::comment}
TODO: Request or define a SID range for the coreconf-m2m module.
Reference the YANG-SID allocation process per I-D.ietf-core-sid.
{:/comment}

[TODO: Request SID range allocation and provide .sid file]

The following SID file is associated with this module:

~~~~ json
{
  "ietf-sid-file:sid-file": {
    "module-name": "coreconf-m2m",
    "module-revision": "2026-03-25",
    "sid-range": [
      {
        "entry-point": "TBD1",
        "size": 100
      }
    ],
    "item": []
  }
}
~~~~

# CORECONF Operations for M2M

## Peer State Retrieval (GET)

[TODO: Example of GET request/response in M2M context with CBOR encoding]

## Peer Configuration (PUT / iPATCH)

[TODO: Example of PUT or iPATCH for partial configuration update]

## Triggering an Action (POST)

[TODO: Example of CoAP POST to trigger a YANG action on a peer device]

## Instance Identifier Handling

{::comment}
TODO: Describe YANG instance-identifier encoding in CBOR for M2M.
Reference Python implementation in pycoreconf.
{:/comment}

[TODO: Describe instance-identifier parsing and encoding]

# Security Considerations

{::comment}
TODO: Address security aspects specific to M2M CORECONF:
- DTLS/OSCORE for CoAP security
- Access control on YANG data nodes
- Trust model in M2M (no central manager)
- Key distribution for constrained devices
{:/comment}

[TODO: Security considerations]

CORECONF operations over CoAP MUST be secured using either DTLS {{?RFC6347}}
or OSCORE {{?RFC8613}}. In M2M scenarios where a central manager is absent,
the trust model requires particular attention.

# IANA Considerations

{::comment}
TODO: List IANA actions required:
- YANG module registration
- SID range allocation
- CoAP Content-Format if new media type needed
{:/comment}

## YANG Module Registration

This document registers the following YANG module in the "YANG Module Names"
registry {{RFC7950}}:

| Name          | Namespace                                         | Prefix | Reference                 |
|:-------------:|:-------------------------------------------------:|:------:|:-------------------------:|
| coreconf-m2m  | urn:ietf:params:xml:ns:yang:coreconf-m2m          | cm2m   | This document             |

## SID Range Allocation

[TODO: Request SID range from IANA per the process defined in {{I-D.ietf-core-sid}}]

--- back

# Complete YANG Module

{::comment}
TODO: Insert the complete, validated YANG module here once finalized.
Validate with pyang before submission.
{:/comment}

[TODO: Full YANG module listing]

# Example CBOR Encodings

{::comment}
TODO: Provide annotated examples of CBOR-encoded CORECONF requests/responses
using diagnostic notation (as per RFC 8949 Appendix G).
{:/comment}

[TODO: CBOR examples in diagnostic notation]

# Acknowledgments
{:numbered="false"}

The authors would like to thank the members of the T2TRG Research Group
and the CORE Working Group for their valuable feedback and discussions.

{::comment}
TODO: Add specific acknowledgments.
{:/comment}
