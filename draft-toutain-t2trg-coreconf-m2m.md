---
title: "CORECONF for Machine-to-Machine Communication"
abbrev: "coreconf-m2m"
category: info
ipr: trust200902

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
  github: "ltn22/coreconf-m2m"

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
  RFC8376:   # LPWAN overview
  RFC8724:   # SCHC
  RFC9179:   # SCHC Generic Rule Set
  RFC8949:   # CBOR
  RFC7396:   # JSON Merge Patch
  RFC8428:   # SenML
  I-D.ietf-core-sid:
  I-D.ietf-core-yang-cbor:
  I-D.gudi-t2trg-senml-as-coreconf:
  I-D.birkholz-yang-core-telemetry:
  OMA-LwM2M:
    title: "Lightweight Machine to Machine Technical Specification: Core"
    target: https://www.openmobilealliance.org/release/LightweightM2M/V1_2-20201110-A/OMA-TS-LightweightM2M_Core-V1_2-20201110-A.pdf
    org: Open Mobile Alliance (OMA)
    date: 2020

--- abstract

The document addresses the specific challenges of M2M interactions where
both endpoints may be constrained nodes, and explores the use of CORECONF
primitives.

This document describes the use of CORECONF (CoAP Management Interface) for
Machine-to-Machine (M2M) communication in constrained IoT environments.
It defines a YANG data model enabling remote management and configuration
of constrained devices using CoAP, CBOR, and YANG SID identifiers. The
serialization in CBOR of this data model limits the payload size.


--- middle

# Introduction

SenML {{RFC8428}} has become a widely adopted format for Machine-to-Machine (M2M) data exchange in IoT environments, enabling constrained devices to report sensor measurements and time series in JSON or CBOR. However, SenML is primarily a data serialization format: it structures payloads but does not enforce strong type checking, schema validation, or support for configuration and remote operations. 

SenML is also part of the LwM2M framework {{OMA-LwM2M}}, which defines a broader device management protocol built on CoAP and SenML for operator-to-device interactions. However, LwM2M relies on periodic reporting and registration messages that impose a non-trivial overhead, particularly on Low-Power Wide-Area Networks (LPWANs) {{RFC8376}} where bandwidth and energy budgets are severely constrained.

In some ways, SenML  may be described by a YANG Data Model {{I-D.gudi-t2trg-senml-as-coreconf}}, but the limits
in term of integration in the YANG ecosystem is limited.

The CORECONF protocol stack  using YANG {{RFC7950}} for Data Modeling,  CoAP {{RFC7252}} for data transport, and CBOR {{RFC8949}} and YANG SID identifiers {{I-D.ietf-core-sid}} for the compact data serialization provides the richer foundation that SenML lacks: a strongly typed data model, schema validation, and support for full CRUD operations and actions. However, CORECONF has so far been designed for operator-to-device management, leaving peer-to-peer M2M communication — where both endpoints may themselves be constrained nodes — largely unaddressed.

Some YANG Data Model has been defined for telemetry. RFC 9232 introduces Network Telemetry used to collect vast amount of data to supervise a network. RFC 8639 allows to subscribe to a datastore filtered through XPath and receives notifications. {{I-D.birkholz-yang-core-telemetry}} proposes to extend telemetry to CORECONF, but using traditional approach.

This document adopts a different approach. The goal is to define a YANG Data Model that will benefits of CBOR serialization to optimize the bandwidth to extends CORECONF for M2M use cases over low-power links, covering geo-location reporting, action/RPC semantics, and YANG augmentation patterns.

## Motivation

CBOR is designed to be concise to represent numerical information since they are directly coded in binary and not represented in ASCII. CBOR uses also binary representation to encode the structures such as Maps and Arrays.
The length of a numerical value depends of its value, for instance numbers between -24 and 23 are coded on a single byte, values between -255 and 255 on two bytes,...

Nevertheless, some representations may be less efficient numerically or less precise. CBOR defines 3 IEEE 754 encoding on 3, 5 or 9 bytes. The smallest representation introduce a close to 1% error. 

The assumption leading to this YANG module is to avoid floating numbers for their size or precision and rely on integer with a precision parameter indication if positive, the number of digits after the dot or the power of 10 if negative. The module also introduces the notion of time series to record several measurement during a period of time and send them in a single message using a notification.



## Use Cases



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

# The coreconf-m2m YANG Module

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
