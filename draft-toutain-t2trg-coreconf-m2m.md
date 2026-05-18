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
  RFC8724:   # SCHC
  RFC8824:
  RFC9254:   # YANG-CBOR
  RFC9363:   # YANG-SID (SCHC YANG module)
  I-D.ietf-core-comi:

informative:
  RFC8376:   # LPWAN overview
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

The targeted usage are remote sensors installed in the field and connected with LPWAN or Satellite connectivity. SCHC {{RFC8724}} {{RFC8824}} is used to compress headers such as IPv6, UDP and CoAP for CORECONF traffic. Payload results of the serialization of the datastore in CBOR.

The model covers the following use cases:

* resource discovery: sensors or actuators, regrouped under the name transducers, are discovered with their characteristics (units, prevision)

* simple query: each transducer can be individually query or set.

* statistic computation: the sensor can compute some statistical values, such as mean, variance, min and max. They can be reset.

* alert notification: when a value reach a threshold (minimum and maximum) a notification message is sent

* time series: values are collected by the device and sent when a limit is reach (number of sample, duration, message size)


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

Device:
: An piece of equipment containing one or more transducers.

Transducer:
: an interface between the analogical and digital world. Transducers are sensors reporting value or actuators having an action on the physical world.

Quantity:
: values manipulated by transducers.

Value:
: other information stored in the datastore including quantities

{::comment}
TODO: Add further terminology as needed.
{:/comment}

# CORECONF Overview in the M2M Context

## CoAP Methods Mapping

CORECONF uses mainly two methods:

* FETCH used to get any values in the YANG Data Model and to initiate notifications,
* iPATH used to modify quantities and notification parameters.


# The coreconf-m2m YANG Module

## Module Overview

The module is divided into two parts. The first part defines a set of transducers identityref. For convenience they are joined to the Data Model, but since they are specific to a device, they should be present is another YANG module. 

In this example, the module defines 12 transducers related to a weather station ATMOS41 (see https://metergroup.com/fr/products/atmos-41/). 

The second part defines the data structures:

* a container including a device description and a list of associated transducers. Each transducer is identified by a identityref and an instance number, allowing several transducers of the same type. Each transducer contains:

  * a quantity measured from the field or setup by the client.
  * a list of statistics.
  * a set of parameters used to control notifications

* an RPC to reset all statistics

* two notifications for each transducer. One to collect time-series and the other to inform when a quantity reach a minium or a maximum threshold.

{{fig-cf-m2m-tree}} gives an overview of the module YANG tree:

~~~~
module: coreconf-m2m
  +--ro state
  |  +--ro uptime?   uint64
  +--rw characteristics
  |  +--rw name?         string
  |  +--rw version?      string
  |  +--rw identifier?   string
  +--rw transducers
     +--rw transducer* [type id]
        +--rw type                       identityref
        +--rw id                         uint8
        +--rw unit?                      string
        +--rw nature?                    transducer-nature
        +--rw precision?                 int8
        +--ro quantity
        |  +--ro value?              int64
        |  +--ro timestamp?          uint64
        |  +--ro u-timestamp?        uint32
        |  +--ro timestamp-source?   enumeration
        |  +--ro statistics
        |     +--ro min?            int64
        |     +--ro max?            int64
        |     +--ro mean?           int64
        |     +--ro median?         int64
        |     +--ro stdev?          uint64
        |     +--ro sample-count?   uint64
        +--rw notification-parameters
        |  +--rw history
        |  |  +--ro active?        boolean
        |  |  +--rw step?          uint32
        |  |  +--rw precision?     uint8
        |  |  +--rw max-samples?   uint32
        |  |  +--rw time-period?   uint32
        |  |  +--rw encoding?      encoding-type
        |  |  +--rw max-payload?   uint32
        |  +--rw sensor-alert
        |     +--ro active?       boolean
        |     +--rw t-min?        int32
        |     +--rw t-max?        int32
        |     +--rw hysteresis?   uint8
        |     +--rw dampening?    uint32
        +---x reset-stats

  rpcs:
    +---x reset-stats

  notifications:
    +---n history
    |  +--ro last?          boolean
    |  +--ro time-series* [type id]
    |     +--ro type        identityref
    |     +--ro id          uint8
    |     +--ro values*     int64
    |     +--ro internal
    |        +--ro last-update?     uint64
    |        +--ro start-time?      uint64
    |        +--ro messages-sent?   uint64
    +---n sensor-alert
       +--ro target* [type id]
          +--ro type     identityref
          +--ro id       uint8
          +--ro value?   int64
~~~~
{: #fig-cf-m2m-tree title="coreconf-m2m module tree" artwork-align="center"}


## state and characteristics sub-tree

These two sub-tress contains global parameters generic to the device, such as uptime.
They are still under definition.

It can be noted, that some parameters such as battery level may be stored in the transducer
tree, this way notification may be applied.

## transducer sub-tree

Transducer sub-tree contains the list of transducers (i.e. sensors, actuators) maintained by the device.
A transducer is identified by to elements:
* a "type" identityref giving the nature of the transducer,
* an "id" giving the instance number allowing several transducers of the same type.

At this level, three other leaves are defined:
* "unit" is a string giving the nature of the quantity. This value may be taken from SenML maintained by IANA. 
The use of a string, instead of identityref is intentional. Units are short names. YANG Identity may be larger and less flexible.
* "nature" reveal if a a transducer is a sensor, where quantity can be read from or an actuator where quantity can be written. 
* "precision": is the number of digits after the dot. So quantity values are integers and multiplied by 10^precision. It can be noted that precision can also be negative for very large values.

These five leaves form the first level of the transducers tree. A query with a depth of 0 returning only the list of transducers, there unit and precision is used to discover the device resources.

If the depth is set to 1, the system recovers transducers and their associated value, "notification-parameter" will be empty since there is no leaf at this level, only sub-trees. The rissk with a query with depth set to 1 is to get large messages, especially if timestamping is done by the device. 


### quantity

The "quantity" sub-tree pushed the leaves in a deeper level, to avoid to be retrieved during the resource discovery phase. This limits the traffic on constrained networks. Quantity contains:
* the value adjusted with the precision to be an integer,
* the timestamp in second and micro-second,
* the entity in charge of the timestamp which can be the device itself or the receiver.

Getting the transducer tree with a depth of 1 allows to discover all the transducers and their associated values.

Under "quantity", the sub-level "statistics" includes major statistic for a specific transducer. Locally computed statistics. Statistics may be erased,for each transducer by the call of the "reset-stat" action, or globally for all transducers with the "reset-stats" RPC.


### notification parameters

The model supports two kind of notifications:

* "sensor-alert" will send notification when the measured quantity reach one or two limits minimal and maximal, or go back to a value between these two bounds. To avoid fluctuations two mechanisms are in place:
    * "hysteresis" defines a percentage, per default 5% around the limit, so if a maximum limit is set to 100, an alert message will be trigger when the quantity is higher that 105 and another alert will be sent when the quantity becomes lower that 95%. The value is sent in the notification message, so the client is able to know the state of the alert.
    * "dampening" limits the number of message sent. 
    
* "history" build time series:
    * "step" parameter defines at which interval sample are taken. 
    * "precision" allow to differ from the quantity in transducer. By default the precision is the one associated to the transducer.
    * "encoding" indicates how information is stored in the time serie:
      * "direct": all the values are stores with the precision.
      * "delta": the first value is the reference and the follwing ones are the difference with the previous. This allow to limit the size of the message since CBOR encodes small numbers more efficiently.
    * A notification is set when a number of measurement is reached, either:
      * the number of samples in the time series,
      * "max-payload" is based on the size of the time serie. Since small numbers are taking less space than large numbers in CBOR, a time serie may contain a different number of samples.
      * the "time-period" after which the collect should be sent.

## starting notifications

If one client waiting to initiate a notification, may send first an iPath to setup notification parameters. 
To start the notification, the client do a FETCH on a specific transducers. If several clients are observing the same transducer they will receive the same notification.

# CORECONF traffic

The following examples shows some CoAP messages between a client and a device (the CoAP server). 
In the example, the device is an ATMOS41 weather station, able to measure 12 parameters. An identity is associated to all these parameters and of course a unique SID:

~~~~
  identity solar-radiation {
    base transducer-type;
    description "Solar radiation measurement (W/m2).";
  }

  identity precipitation {
    base transducer-type;
    description "Precipitation measurement (mm).";
  }

  identity air-temperature {
    base transducer-type;
    description "Air temperature measurement (°C).";
  }
~~~~


## resource discovery

The client ignores the transducers managed by the device. It sends a message to get "/transducers/transducer" with a depth of 0.

~~~~
Constrained Application Protocol, Non-Confirmable, FETCH, MID:47709
    Token: 8ed4
    Opt Name: #1: Uri-Path: c
    Opt Name: #2: Content-Format: Unknown Type 141
    Opt Name: #3: Uri-Query: d=0
    Opt Name: #4: Accept: Unknown Type 142

 Payload: 5 bytes
  1A 000186DF # unsigned(100063) : /transducers/transducer                 ....
~~~~

The device answers with the list of transducers. 

~~~~ 
Constrained Application Protocol, Non-Confirmable, 2.05 Content, MID:39441
    Token: 8ed4
    Opt Name: #1: Content-Format: Unknown Type 142

 Payload: 220 bytes
  {100063: 
    [{33: 100008, 1: 0, 17: 1, 34: "W/m2"}, 
      {33: 100006, 1: 0, 17: 3, 34: "mm"}, 
      {33: 100009, 1: 0, 17: 0, 34: ""}, 
      {33: 100002, 1: 0, 17: 1, 34: "km"}, 
      {33: 100013, 1: 0, 17: 1, 34: "deg"}, 
      {33: 100015, 1: 0, 17: 2, 34: "m/s"}, 
      {33: 100014, 1: 0, 17: 2, 34: "m/s"}, 
      {33: 100010, 1: 0, 17: 1, 34: "deg"}, 
      {33: 100001, 1: 0, 17: 1, 34: "degC"}, 
      {33: 100012, 1: 0, 17: 3, 34: "kPa"}, 
      {33: 100003, 1: 0, 17: 2, 34: "kPa"}, 
      {33: 100007, 1: 0, 17: 1, 34: "%RH"}]} 

~~~~

The client, may translate the transducer's identityref to the name defined in the YANG module. For instance:

~~~
    1  solar-radiation              W/m2     [type='solar-radiation'][id='0']
    2  precipitation                mm       [type='precipitation'][id='0']
    3  strike-count                          [type='strike-count'][id='0']
    4  average-distance             km       [type='average-distance'][id='0']
    5  wind-direction               deg      [type='wind-direction'][id='0']
    6  wind-speed                   m/s      [type='wind-speed'][id='0']
    7  wind-gust                    m/s      [type='wind-gust'][id='0']
    8  tilt                         deg      [type='tilt'][id='0']
    9  air-temperature              degC     [type='air-temperature'][id='0']
   10  vapor-pressure               kPa      [type='vapor-pressure'][id='0']
   11  barometric-pressure          kPa      [type='barometric-pressure'][id='0']
   12  relative-humidity            %RH      [type='relative-humidity'][id='0']
~~~


## Querying a quantity

A specific quantity may be requested through a FETCH.

~~~~
Constrained Application Protocol, Non-Confirmable, FETCH, MID:47710
    Token: 8ed5
    Opt Name: #1: Uri-Path: c
    Opt Name: #2: Content-Format: Unknown Type 141
    Opt Name: #3: Accept: Unknown Type 142
  Payload: Length: 12 Bytes
  [100092, 100001, 0]


Constrained Application Protocol, Non-Confirmable, 2.05 Content, MID:39442
    Token: 8ed5
    Opt Name: #1: Content-Format: Unknown Type 142
  Payload:  Length: 8
  {100092: 112}
~~~~

Or the full statistic table for a specific transducer:

~~~~
Constrained Application Protocol, Non-Confirmable, FETCH, MID:47711
    Token: 8ed6
    Opt Name: #1: Uri-Path: c
    Opt Name: #2: Content-Format: Unknown Type 141
    Opt Name: #3: Accept: Unknown Type 142
  Payload:  Length: 12
    [100082, 100001, 0]

Constrained Application Protocol, Non-Confirmable, 2.05 Content, MID:39443
    Token: 8ed6
    Opt Name: #1: Content-Format: Unknown Type 142
  Payload: Length: 24
    {100082: {4: 79, 1: 119, 2: 94, 3: 103, 6: 12, 5: 53}}
~~~~

The client can display:

~~~~
  [9] Statistiques — air-temperature:
    min:     7.9 degC
    max:     11.9 degC
    mean:    9.4 degC
    median:  10.3 degC
    σ:       1.2 degC
    n:       53
~~~~

##  Notification


Constrained Application Protocol, Non-Confirmable, iPATCH, MID:47712
    Token: 8ed7
    Opt Name: #1: Uri-Path: c
    Opt Name: #2: Content-Format: Unknown Type 142
  Payload:  Length: 28
    {[100066, 100008, 0]: {100066: {6: 5000, 4: 3, 2: 1}}}                            .....


Constrained Application Protocol, Non-Confirmable, 2.04 Changed, MID:39444
    Token: 8ed7


Constrained Application Protocol, Non-Confirmable, FETCH, MID:47713
    Token: 8ed8
    Opt Name: #1: Observe: 0
    Opt Name: #2: Uri-Path: s
    Opt Name: #3: Content-Format: Unknown Type 141
    Opt Name: #4: Accept: Unknown Type 142
  Payload:  Length: 12
    [100044, 100008, 0]


Constrained Application Protocol, Non-Confirmable, 2.05 Content, MID:39445
    Token: 8ed8
    Opt Name: #1: Observe: 0
    Opt Name: #2: Content-Format: Unknown Type 142
  Payload:  Length: 1
    {}


Constrained Application Protocol, Non-Confirmable, 2.05 Content, MID:39446
    Token: 8ed8
    Opt Name: #1: Observe: 1
    Opt Name: #2: Content-Format: Unknown Type 142
  Payload:  Length: 43
    {100042: {2: [{6: 100008, 1: 0, 7: [1043, 82, 82, 362, 316, 318, 226, 94, -147, -16]}]}}

  [1] solar-radiation: 104.3 W/m2  (16:50:29)
  [1] solar-radiation: 112.5 W/m2  (16:52:29)
  [1] solar-radiation: 120.7 W/m2  (16:54:29)
  [1] solar-radiation: 156.9 W/m2  (16:56:29)
  [1] solar-radiation: 188.5 W/m2  (16:58:29)
  [1] solar-radiation: 220.3 W/m2  (17:00:29)
  [1] solar-radiation: 242.9 W/m2  (17:02:29)
  [1] solar-radiation: 252.3 W/m2  (17:04:29)
  [1] solar-radiation: 237.6 W/m2  (17:06:29)
  [1] solar-radiation: 236.0 W/m2  (17:08:29)



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
