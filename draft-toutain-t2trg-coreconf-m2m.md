---
title: "CORECONF for Machine-to-Machine Communication"
abbrev: "coreconf-m2m"
category: info
ipr: trust200902

docname: draft-toutain-t2trg-coreconf-m2m-01
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
  RFC9179:   # YANG Grouping for Geographic Locations
  RFC8949:   # CBOR
  RFC7396:   # JSON Merge Patch
  RFC8428:   # SenML
  RFC9232:   # Network Telemetry
  RFC8639:   # YANG Subscribed Notifications
  I-D.ietf-core-sid:
  I-D.ietf-core-yang-cbor:
  I-D.gudi-t2trg-senml-as-coreconf:
  I-D.birkholz-yang-core-telemetry:
  OMA-LwM2M:
    title: "Lightweight Machine to Machine Technical Specification: Core"
    target: https://www.openmobilealliance.org/release/LightweightM2M/V1_2-20201110-A/OMA-TS-LightweightM2M_Core-V1_2-20201110-A.pdf
    org: Open Mobile Alliance (OMA)
    date: 2020
  SOSA:
    title: "Semantic Sensor Network Ontology"
    target: https://www.w3.org/TR/vocab-ssn/
    org: W3C/OGC
    date: 2017
  SAREF:
    title: "SAREF: the Smart Applications REFerence ontology"
    target: https://saref.etsi.org/
    org: ETSI
    date: 2020

--- abstract

The document addresses the specific challenges of M2M interactions where both
endpoints may be constrained nodes, and explores the use of CORECONF primitives.

This document describes the use of CORECONF (CoAP Management Interface) for
Machine-to-Machine (M2M) communication in constrained IoT environments. It
defines a YANG data model enabling remote management and configuration of
constrained devices using CoAP, CBOR, and YANG SID identifiers. The
serialization in CBOR of this data model limits the payload size. It documents
also how the YANG data model can interact with common IoT ontologies such as
SOSA or SAREF.


--- middle

# Introduction

This document proposes a YANG data model designed for constrained devices and
low-power networks. By combining YANG's strong typing with CBOR's compact binary
serialization and CoAP's lightweight transport, the model enables efficient
Machine-to-Machine (M2M) data exchange while remaining within the bandwidth and
energy budgets of constrained environments. SCHC header compression {{RFC8724}}
may further be used to reduce the overhead of IPv6, UDP, and CoAP headers on the
most constrained links.

Some data models and protocols already exist for M2M communication in IoT
environments, but none is fully adapted to the constraints of low-power networks
in terms of message size, energy consumption, and interaction patterns. This
section reviews the main existing approaches and explains why a new model is
needed.

SenML {{RFC8428}} has become a widely adopted format for Machine-to-Machine
(M2M) data exchange in IoT environments, enabling constrained devices to report
sensor measurements and time series in JSON or CBOR. However, SenML is primarily
a data serialization format: it structures payloads but does not enforce strong
type checking, schema validation, or support for configuration and remote
operations.

SenML is also part of the LwM2M framework {{OMA-LwM2M}}, which defines a broader
device management protocol built on CoAP and SenML for operator-to-device
interactions. However, LwM2M relies on periodic reporting and registration
messages that impose a non-trivial overhead, particularly on Low-Power Wide-Area
Networks (LPWANs) {{RFC8376}} where bandwidth and energy budgets are severely
constrained.

In some ways, SenML may be described by a YANG Data Model
{{I-D.gudi-t2trg-senml-as-coreconf}}, but the integration in the YANG ecosystem
remains limited.

The CORECONF protocol stack  using YANG {{RFC7950}} for Data Modeling,  CoAP
{{RFC7252}} for data transport, and CBOR {{RFC8949}} and YANG SID identifiers
{{I-D.ietf-core-sid}} for the compact data serialization provides the richer
foundation that SenML lacks: a strongly typed data model, schema validation, and
support for full CRUD operations and actions. However, CORECONF has so far been
designed for operator-to-device management, leaving peer-to-peer M2M
communication — where both endpoints may themselves be constrained nodes —
largely unaddressed.

Some YANG Data Models have been defined for telemetry. {{RFC9232}} introduces
Network Telemetry used to collect vast amounts of data to supervise a network.
{{RFC8639}} allows subscribing to a datastore filtered through XPath and
receiving notifications. {{I-D.birkholz-yang-core-telemetry}} proposes to extend
telemetry to CORECONF, but using a traditional approach.

This document adopts a different approach. The goal is to define a YANG Data
Model that will benefit from CBOR serialization to optimize the bandwidth to
extend CORECONF for M2M use cases over low-power links. This document focuses on
transducer management: resource discovery, value polling, statistical
computation, threshold alerts, and time-series history notifications. This is an
early-stage work; future revisions will explore other categories of measurements
and interaction patterns.

## Use Cases

The targeted use cases are remote sensors installed in the field and connected
with LPWAN or Satellite connectivity. SCHC {{RFC8724}} {{RFC8824}} is used to
compress headers such as IPv6, UDP and CoAP for CORECONF traffic. The payload
results from the serialization of the datastore in CBOR.

The model covers the following use cases:

* resource discovery: sensors or actuators, regrouped under the name
  transducers, are discovered with their characteristics (units, precision)

* simple query: each transducer can be individually queried or set.

* statistical computation: the sensor can compute some statistical values, such
  as mean, variance, min and max. They can be reset.

* alert notification: when a value reaches a threshold (minimum and maximum) a
  notification message is sent

* time series: values are collected by the device and sent when a limit is
  reached (number of samples, duration, message size)


## Data Representation

CBOR is designed to be concise to represent numerical information since it is
directly coded in binary and not represented in ASCII. CBOR also uses binary
representation to encode structures such as Maps and Arrays. The length of a
numerical value depends on its value; for instance, numbers between -24 and 23
are coded on a single byte, values between -255 and 255 on two bytes,...

Nevertheless, some representations may be less efficient numerically or less
precise. CBOR defines 3 IEEE 754 encodings on 3, 5, or 9 bytes. The smallest
representation introduces a close to 1% error. CBOR also provides a decimal
fraction type (tag 4) encoding a value as a `[exponent, mantissa]` pair, which
avoids floating-point rounding. However, this representation requires the
exponent (the divider) to be repeated alongside every individual quantity,
adding overhead for each encoded value.

The assumption leading to this YANG module is to avoid floating-point numbers
for their size or precision and rely on integers with a precision parameter
indicating, if positive, the number of digits after the decimal point, or the
power of 10 if negative.

The module also introduces the notion of time series to record several
measurements during a period of time and send them in a single message using a
notification. Time series values may further be compressed depending on the
nature of the data. This version proposes a compression based on delta encoding:
instead of transmitting absolute values, each sample is encoded as the
difference from the previous one, which significantly reduces the CBOR payload
size for slowly-varying measurements.

The device can also send an alert when a measured value reaches a threshold,
allowing the receiver to react promptly without waiting for a scheduled report.

Since the targeted networks are constrained in bandwidth, measurements are expected
to be reported infrequently, so the second is chosen as the time unit. To limit the
size of the timestamps, they are expressed as the number of seconds elapsed since
the equipment was started, rather than as an absolute date and time. If the
equipment is able to do so, it can indicate its startup epoch to reconstitute
an absolute time.



## Requirements Language

{::comment}
Standard boilerplate for normative language.
{:/comment}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Terminology

The following terminology is used in this document:

CORECONF: : CoAP Management Interface, as defined in {{I-D.ietf-core-comi}}.

M2M: : Machine-to-Machine communication, referring to direct data exchanges
between devices without human intervention.

SID: : YANG Schema Item iDentifier, a compact numeric identifier for YANG data
nodes, as defined in {{I-D.ietf-core-sid}}.

Constrained Device: : A device with limited processing, memory, and energy
resources, as characterized in {{?RFC7228}}.

Device: : A piece of equipment containing one or more transducers.

Transducer: : an interface between the analog and digital world. Transducers are
sensors reporting values or actuators having an action on the physical world.

Quantity: : values manipulated by transducers.

Value: : other information stored in the datastore including quantities

{::comment}
TODO: Add further terminology as needed.
{:/comment}

# YANG Modules needed for coreconf-m2m

coreconf-m2m YANG module is a framework: it defines the generic structure of a CORECONF
server hosting one or several transducers, a generic name for a sensor or
an actuator, but it does not define the transducers themselves. Each
transducer is identified by a unique identity, and if several transducers
report the same kind of measurement, several instances of the same
identity have to be defined.

A device manufacturer using coreconf-m2m MUST therefore define, or reuse,
a YANG module describing the transducers embedded in its device, as
identities extending the base "transducer-type" identity defined in
coreconf-m2m (see {{identities-data-model}}).

## Identities Data Model

The identity data model defines all the transceivers that may be used inside the
hosts, this can be a generic data model or a model specific to the hosts. This
module indicates default parameters used for that transceiver (units, precision,
nature). These values are mandatory and may be overridden by the coreconf-m2m.

A manufacturer module defines the concrete transducer-type identities for a
given product family; coreconf-m2m only defines the base "transducer-type"
identity that these modules extend. The ATMOS41 weather station example
used throughout this document is defined this way, in a separate "atmos"
module that imports coreconf-m2m and extends its base identity with
atmospheric and weather-related transducer types.

coreconf-m2m defines the template for identities extensions. {{fig-identity-template}}
gives an example of an identity definition, where "ccm2m" refers to the
coreconf-m2m module.

~~~~
  identity air-temperature {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "Cel";
    ccm2m:default-precision "2";
    description "Air temperature measurement (°C).";
  }
~~~~
{: #fig-identity-template title="Example of an identity definition using the coreconf-m2m extensions" artwork-align="left"}

Identity names and descriptions SHOULD be explicit, since this information
may be used by AI agents to formulate requests.


## Overview of the coreconf-m2m Module

The coreconf-m2m module is organized into three sub-modules:

* "characteristics" contains stable information describing the host. This
  information may be necessary for some ontologies such as {{SOSA}} and
  {{SAREF}}. These values may be modified during runtime,
* "bootstrap" contains the context needed by the client to interact with the
  server and its transducers. It mainly contains the time reference and the
  transducer list. These values cannot be changed by the client (config
  false); only a reboot, which also resets the communications, allows
  their modification, and
* "transducers" contains the runtime values of all the sensors and
  actuators maintained by the host. It also contains parameters to control
  notifications issued by the device.

* an RPC, now limited to resetting all statistics

* two notification types: one to collect time-series history, and one to alert
  when a quantity reaches a minimum or maximum threshold.

{{fig-cf-m2m-tree}} in {{annex-tree}} gives an overview of the module YANG tree.


### Bootstrap Sub-Tree

~~~~
module: coreconf-m2m
  +--ro bootstrap
     +--ro reference-epoch?   uint64
     +--ro uptime             uint64
     +--ro minimal-step?      uint32
     +--ro inventory* [type]
        +--ro type                  identityref
        +--ro unit-override?        string
        +--ro precision-override?   uint8
        +--ro category-override?    enumeration
~~~~
{: #fig-bootstrap-tree title="bootstrap sub-tree" artwork-align="center"}

The bootstrap sub-tree contains information determined at bootstrap:

* reference-epoch: the absolute time at bootstrap, if a reference time source
  (battery-powered clock, GPS, ...) is available on the device. This leaf is
  absent if no reference time is available.
* uptime: the number of seconds elapsed since bootstrap. A client can combine
  reference-epoch and uptime to estimate clock drift or propagation delay.
* minimal-step: the minimum number of seconds between two transceiver
  readings. For instance, if a node updates its reading every 2 minutes,
  querying it more frequently only increases battery drain without yielding
  fresher data.
* inventory: the list of transducers known to the device, identified by
  their "type" and their configuration overrides (see
  {{transducer-sub-tree}}). 

These values can only change on a reboot, which also resets the
communications and forces the client into a new bootstrap phase to
recover the data.


### Characteristics Sub-Tree

~~~~
module: coreconf-m2m
  +--rw characteristics
     +--rw geo-location
     |  +--rw reference-frame
     |  |  +--rw alternate-system?    string {alternate-systems}?
     |  |  +--rw astronomical-body?   string
     |  |  +--rw geodetic-system
     |  |     +--rw geodetic-datum?    string
     |  |     +--rw coord-accuracy?    decimal64
     |  |     +--rw height-accuracy?   decimal64
     |  +--rw (location)?
     |  |  +--:(ellipsoid)
     |  |  |  +--rw latitude?    decimal64
     |  |  |  +--rw longitude?   decimal64
     |  |  |  +--rw height?      decimal64
     |  |  +--:(cartesian)
     |  |     +--rw x?           decimal64
     |  |     +--rw y?           decimal64
     |  |     +--rw z?           decimal64
     |  +--rw velocity
     |  |  +--rw v-north?   decimal64
     |  |  +--rw v-east?    decimal64
     |  |  +--rw v-up?      decimal64
     |  +--rw timestamp?         yang:date-and-time
     |  +--rw valid-until?       yang:date-and-time
     +--rw name?                string
     +--rw version?             string
     +--rw identifier?          string
     +--rw description?         string
     +--rw manufacturer?        string
     +--rw model?               string
     +--rw hosted-by?           string
     +--rw installation-time?   uint64
~~~~
{: #fig-characteristics-tree title="characteristics sub-tree" artwork-align="center"}

This sub-tree contains stable information about the device that is not
expected to change across reboots:

* geo-location: the fixed geographic position of the device, set once at
  deployment and not expected to change during normal operation. The module
  reuses the "ietf-geo-location" YANG module {{RFC9179}}, of which only the
  "ellipsoid" choice is normally needed. Note that its coordinates are typed as
  "decimal64", encoded in CBOR as a tagged array containing an exponent and
  a mantissa. This encoding was not chosen for transducer values, in order
  to avoid repeating the exponent (called "precision" in this model) with
  every quantity. 
  
  If the device is mobile, its coordinates should instead
  be exposed through a transducer.
* name: a human-readable name for the device.
* version: the firmware or software version running on the device.
* identifier: a unique identifier for the device, such as a serial number
  or EUI.
* description: a free-text description of the device, its purpose, or its
  deployment context.
* manufacturer: the name of the device manufacturer.
* model: the commercial model name or number of the device.
* hosted-by: an identifier of the platform hosting this device, when
  applicable.
* installation-time: the absolute date and time (Unix epoch, in seconds)
  at which the device was installed or commissioned. Unlike
  bootstrap/reference-epoch, this is an absolute value, not a reference
  point for the device's internal clock.


### Transducer Sub-Tree

~~~~
module: coreconf-m2m
  +--rw transducers
     +--rw transducer* [type]
        +--rw type                       identityref
        +--ro quantity
           +--ro value?              int64
           +--ro timestamp?          uint64
           +--ro timestamp-source?   enumeration
           +--ro statistics
              +--ro min?            int64
              +--ro max?            int64
              +--ro mean?           int64
              +--ro median?         int64
              +--ro stdev?          uint64
              +--ro sample-count?   uint64
~~~~
{: #fig-transducer-tree title="transducer sub-tree (type and quantity)" artwork-align="center"}

The transducers sub-tree contains the list of transducers (i.e., sensors and
actuators) maintained by the device, listed in "/bootstrap/inventory". A
transducer is identified by its "type", the same identityref used in the
"/bootstrap/inventory" entry.

This branch holds two kinds of information: "quantity" carries the values
manipulated by the transducer, while "notification-parameters" configures
notifications through which the device can proactively inform the client.

Quantity contains:

* "value": the raw integer value, computed as
  `value = measurement * 10^precision`,
* "timestamp": the relative time of the measurement, in seconds,
* the entity in charge of the timestamp, which can be the device itself or the
  receiver.
* the "statistics" sub-level holds the main statistics
locally computed for a specific transducer. Statistics can be reset for a
single transducer via the "reset-stats" action, or for all transducers at
once via the "reset-stats" RPC.


~~~~
module: coreconf-m2m
  +--rw transducers
     +--rw transducer* [type]
        +--rw notification-parameters
           +--rw history
           |  +--ro active?        boolean
           |  +--rw step?          uint32
           |  +--rw precision?     uint8
           |  +--rw max-samples?   uint32
           |  +--rw time-period?   uint32
           |  +--rw encoding?      encoding-type
           |  +--rw max-payload?   uint32
           +--rw sensor-alert
           |  +--ro active?       boolean
           |  +--rw t-min?        int32
           |  +--rw t-max?        int32
           |  +--rw hysteresis?   uint8
           |  +--rw dampening?    uint32
           +--rw check-interval?   uint16
~~~~
{: #fig-notification-parameters-tree title="notification-parameters sub-tree" artwork-align="center"}

Notification Parameters supports two kinds of notifications:

* "sensor-alert" will send a notification when the measured quantity reaches one
  or two limits, minimal and maximal, or goes back to a value between these two
  bounds. To avoid fluctuations, two mechanisms are in place:
    * "hysteresis" defines a percentage, by default 5% around the limit, so if a
      maximum limit is set to 100, an alert message will be triggered when the
      quantity is higher than 105 and another alert will be sent when the
      quantity becomes lower than 95%. The value is sent in the notification
      message, so the client is able to know the state of the alert.
    * "dampening" limits the number of messages sent.

* "history" builds time series:
    * "step" parameter defines at which interval samples are taken.
    * "precision" allows overriding the quantity precision defined in the
      transducer. By default the precision is the one associated with the
      transducer for "quantity".
    * "encoding" indicates how information is stored in the time series:
      * "direct": all the values are stored with the precision.
      * "delta": the first value is the reference and the following ones are the
        difference with the previous. This allows limiting the size of the
        message since CBOR encodes small numbers more efficiently.
    * A notification is sent when a number of measurements is reached, either:
      * the number of samples in the time series reaches "max-samples",
      * "max-payload" is based on the size of the time series. Since small
        numbers take less space than large numbers in CBOR, a time series may
        contain a different number of samples.
      * the "time-period" after which the collection should be sent.

* "check-interval" controls how often a CON (confirmable) notification is
  sent instead of NON (non-confirmable), for both the history and
  sensor-alert streams of this transducer: 0 (default) disables it, so
  all notifications are NON; 1 makes every notification CON; N sends one
  CON every N notifications, all others being NON.

The read-only `active` flag in both notification types is set when one or more
clients observe a notification. Parameters are common to all observations of a
particular transducer.

## Starting Notifications

A client wishing to start a notification MAY first send an iPATCH to
configure its parameters. Parameters changed while a notification is
already active take effect immediately.

Notification is started by sending a FETCH+Observe request on the notification
stream resource (`/s`), with a body identifying the SID of the transducer to
observe. Multiple clients may observe the same transducer simultaneously; each
receives an independent copy of every notification.

# CORECONF Overview in the M2M Context

## CoAP Methods Mapping

CORECONF defines mappings for all CoAP methods, but this document uses only two:

* FETCH is used instead of GET to retrieve values from the YANG Data Model.
  Unlike GET, FETCH carries a body specifying the exact SIDs to retrieve,
  enabling precise and bandwidth-efficient queries. Combined with the CoAP
  Observe option, FETCH also serves to subscribe to notification streams.

* iPATCH is used instead of PUT or POST to modify quantities and notification
  parameters. It supports partial updates: only the specified nodes are
  modified, leaving others unchanged. Setting a node to an empty value with
  iPATCH is the preferred way to clear a parameter, making DELETE unnecessary
  for datastore modifications.

The recommended CoAP Content-Formats for all exchanges are:

* Content-Format 141 (`application/yang-fetch+cbor`) for FETCH request bodies,
  which carry the list of SIDs to retrieve.
* Content-Format 142 (`application/yang-data+cbor;id=sid`) for all response
  bodies and iPATCH payloads, where data nodes are identified by their SID.

Using these two content formats ensures maximum interoperability with CORECONF
implementations and keeps the payloads as compact as possible. Limiting
exchanges to a small number of well-known packet formats also benefits SCHC
compression {{RFC8724}}: the fewer distinct header patterns in use, the more
efficiently SCHC rules can compress the CoAP headers, reducing overhead on the
most constrained links.

FETCH and iPATCH requests MUST be sent as Non-Confirmable (NON) CoAP messages.
This leaves the application free to implement its own retransmission strategy
and timer management, which is essential on constrained networks where the
default CoAP confirmable retransmission behavior may be inappropriate or
wasteful.

For notification streams (Observe), Confirmable (CON) messages MAY be used. A
CON notification allows the server to detect that the observer is no longer
reachable when no ACK is received, and to cancel the Observe subscription
accordingly. A RST response from the client is also a valid way to signal that
the subscription should be terminated.

# CORECONF Traffic

The following examples show some CoAP messages between a client and a device
(the CoAP server). In the example, the device is an ATMOS41 weather station,
able to measure 12 parameters. An identity is associated with each parameter and
a unique SID, as illustrated in {{fig-identity-excerpt}}:

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
{: #fig-identity-excerpt title="Excerpt of YANG identity definitions for the ATMOS41 transducers (see Appendix for the complete module)" artwork-align="left"}


## Resource Discovery

The client does not know the transducers managed by the device. It sends a FETCH
on "/transducers/transducer" with a depth of 0, as shown in
{{fig-resource-discovery}}.

~~~~
  CoAP Request:
  Non-Confirmable, FETCH, MID:47709
    Token: 8ed4
    Opt #1: Uri-Path: c
    Opt #2: Content-Format: 141 (application/yang-fetch+cbor)
    Opt #3: Uri-Query: d=0
    Opt #4: Accept: 142 (application/yang-data+cbor;id=sid)

  Payload: 5 bytes
    1A 000186DF  # unsigned(100063) : /transducers/transducer

  CoAP Response:
  Non-Confirmable, 2.05 Content, MID:39441
    Token: 8ed4
    Opt #1: Content-Format: 142 (application/yang-data+cbor;id=sid)

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
{: #fig-resource-discovery title="Resource discovery: FETCH request and response" artwork-align="left"}

In the request, the payload `1A 000186DF` is the CBOR encoding of the unsigned
integer 100063, which is the SID of "/transducers/transducer". In the response,
the outer key 100063 identifies the list, and each entry is a CBOR map whose
keys are delta SIDs relative to the list SID, as defined in {{RFC9254}}. The
values encode the transducer type (as an identity SID), instance id, precision,
and unit string.

The client may translate the identityref values to the names defined in the YANG
module. {{fig-transducer-list}} shows the resulting transducer table:

~~~~
  1  solar-radiation      W/m2  [type='solar-radiation'][id='0']
  2  precipitation        mm    [type='precipitation'][id='0']
  3  strike-count               [type='strike-count'][id='0']
  4  average-distance     km    [type='average-distance'][id='0']
  5  wind-direction       deg   [type='wind-direction'][id='0']
  6  wind-speed           m/s   [type='wind-speed'][id='0']
  7  wind-gust            m/s   [type='wind-gust'][id='0']
  8  tilt                 deg   [type='tilt'][id='0']
  9  air-temperature      degC  [type='air-temperature'][id='0']
 10  vapor-pressure       kPa   [type='vapor-pressure'][id='0']
 11  barometric-pressure  kPa   [type='barometric-pressure'][id='0']
 12  relative-humidity    %RH   [type='relative-humidity'][id='0']
~~~~
{: #fig-transducer-list title="Decoded transducer list from resource discovery response" artwork-align="left"}


## Querying a quantity

A specific quantity may be requested through a FETCH. {{fig-query-value}} shows
the exchange for the current value of air-temperature.

~~~~
  CoAP Request:
  Non-Confirmable, FETCH, MID:47710
    Token: 8ed5
    Opt #1: Uri-Path: c
    Opt #2: Content-Format: 141 (application/yang-fetch+cbor)
    Opt #3: Accept: 142 (application/yang-data+cbor;id=sid)

  Payload: 12 bytes
    [100092, 100001, 0]

  CoAP Response:
  Non-Confirmable, 2.05 Content, MID:39442
    Token: 8ed5
    Opt #1: Content-Format: 142 (application/yang-data+cbor;id=sid)

  Payload: 8 bytes
    {100092: 112}
~~~~
{: #fig-query-value title="FETCH request and response for air-temperature current value" artwork-align="left"}

The FETCH body `[100092, 100001, 0]` is a CORECONF instance-identifier: the
first element is the SID of the requested leaf, followed by the list key values
identifying the transducer (type=100001 for air-temperature, id=0). The response
value 112, combined with precision=1, decodes to 11.2°C.

{{fig-query-stats}} shows the FETCH for the full statistics table of the same
transducer:

~~~~
  CoAP Request:
  Non-Confirmable, FETCH, MID:47711
    Token: 8ed6
    Opt #1: Uri-Path: c
    Opt #2: Content-Format: 141 (application/yang-fetch+cbor)
    Opt #3: Accept: 142 (application/yang-data+cbor;id=sid)

  Payload: 12 bytes
    [100082, 100001, 0]

  CoAP Response:
  Non-Confirmable, 2.05 Content, MID:39443
    Token: 8ed6
    Opt #1: Content-Format: 142 (application/yang-data+cbor;id=sid)

  Payload: 24 bytes
    {100082: {4: 79, 1: 119, 2: 94, 3: 103, 6: 12, 5: 53}}
~~~~
{: #fig-query-stats title="FETCH request and response for air-temperature statistics" artwork-align="left"}

The FETCH body `[100082, 100001, 0]` requests the quantity sub-tree (SID 100082)
for air-temperature (100001), instance 0. In the response, the inner map keys
are delta SIDs relative to 100082, each encoding the difference between
consecutive SIDs to minimize CBOR size. The six values correspond to the
statistics leaves: min, max, mean, median, stdev, and sample-count, all scaled
by the transducer precision.

The client decodes the response and displays the statistics as shown in
{{fig-stats-display}}:

~~~~
  [9] Statistics — air-temperature:
    min:     7.9 degC
    max:     11.9 degC
    mean:    9.4 degC
    median:  10.3 degC
    σ:       1.2 degC
    n:       53
~~~~
{: #fig-stats-display title="Decoded statistics for air-temperature" artwork-align="left"}

## Notification

The client first sends an iPATCH to configure the history notification
parameters for the solar-radiation transducer, as shown in
{{fig-notification-config}}.

~~~~
  CoAP Request:
  Non-Confirmable, iPATCH, MID:47712
    Token: 8ed7
    Opt #1: Uri-Path: c
    Opt #2: Content-Format: 142 (application/yang-data+cbor;id=sid)

  Payload: 28 bytes
    {[100066, 100008, 0]: {100066: {6: 5000, 4: 3, 2: 1}}}

  CoAP Response:
  Non-Confirmable, 2.04 Changed, MID:39444
    Token: 8ed7
~~~~
{: #fig-notification-config title="iPATCH to configure history notification parameters for solar-radiation" artwork-align="left"}

The iPATCH key `[100066, 100008, 0]` is an instance-identifier targeting the
notification-parameters node (SID 100066) for solar-radiation (SID 100008),
instance 0. The value `{100066: {6: 5000, 4: 3, 2: 1}}` sets three history
parameters using delta SIDs relative to 100066: step=5000 ms (one sample every 5
seconds), max-samples=3, and encoding=delta (value 1).

The client then initiates an Observe subscription with a FETCH on the
notification stream resource `/s`, as shown in {{fig-notification-observe}}.

~~~~
  CoAP Request:
  Non-Confirmable, FETCH, MID:47713
    Token: 8ed8
    Opt #1: Observe: 0
    Opt #2: Uri-Path: s
    Opt #3: Content-Format: 141 (application/yang-fetch+cbor)
    Opt #4: Accept: 142 (application/yang-data+cbor;id=sid)

  Payload: 12 bytes
    [100044, 100008, 0]

CoAP Response (subscription acknowledgment):
  Non-Confirmable, 2.05 Content, MID:39445
    Token: 8ed8
    Opt #1: Observe: 0
    Opt #2: Content-Format: 142 (application/yang-data+cbor;id=sid)

  Payload: 1 byte
    {}

CoAP Notification:
  Non-Confirmable, 2.05 Content, MID:39446
    Token: 8ed8
    Opt #1: Observe: 1
    Opt #2: Content-Format: 142 (application/yang-data+cbor;id=sid)

  Payload: 43 bytes
    {100042: {2: [{6: 100008, 1: 0, 7: [1043, 82, 82, 362, 316, 318, 226, 94, -147, -16]}]}}
~~~~
{: #fig-notification-observe title="FETCH+Observe subscription on `/s` and history notification for solar-radiation" artwork-align="left"}

The FETCH body `[100044, 100008, 0]` subscribes to the history time-series
stream (SID 100044) for solar-radiation. The server acknowledges with an empty
map `{}`. In the notification, the outer key 100042 identifies the history
notification; the inner structure uses delta SIDs to encode type, id, and the
values list. The values `[1043, 82, 82, ...]` use delta encoding: 1043 is the
absolute reference (104.3 W/m²), and each subsequent value is the difference
from the previous one, yielding a compact representation of slowly-varying
measurements.

The client decodes the delta-encoded time-series values and displays the result
as shown in {{fig-notification-decoded}}:

~~~~
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
~~~~
{: #fig-notification-decoded title="Decoded solar-radiation time-series from history notification" artwork-align="left"}



# SID Allocation

The SID range 100000–100399 is used as an example throughout this document.
Official values MUST be assigned by IANA prior to publication. The complete SID
file is provided in {{fig-sid-csv}}.

# Security Considerations

CORECONF operations over CoAP MUST be secured using either DTLS {{?RFC6347}} or
OSCORE {{?RFC8613}}. In M2M scenarios where a central manager is absent, the
trust model requires particular attention.

# IANA Considerations

## YANG Module Registration

This document registers the following YANG module in the "YANG Module Names"
registry {{RFC7950}}:

| Name          | Namespace                                         | Prefix | Reference                 |
|:-------------:|:-------------------------------------------------:|:------:|:-------------------------:|
| coreconf-m2m  | urn:ietf:params:xml:ns:yang:coreconf-m2m          | cm2m   | This document             |

## SID Range Allocation

This document requests the SID range starting at entry-point TBD with a size of
100, as defined in {{I-D.ietf-core-sid}}.

Transducer identites  MUST be sperated from this document and defined in
anjother YANG module.

--- back

# Module Tree {#annex-tree}

~~~~
module: coreconf-m2m
  +--ro bootstrap
  |  +--ro reference-epoch?   uint64
  |  +--ro uptime             uint64
  |  +--ro minimal-step?      uint32
  |  +--ro inventory* [type]
  |     +--ro type                  identityref
  |     +--ro unit-override?        string
  |     +--ro precision-override?   uint8
  |     +--ro category-override?    enumeration
  +--rw characteristics
  |  +--rw geo-location
  |  |  +--rw reference-frame
  |  |  |  +--rw alternate-system?    string {alternate-systems}?
  |  |  |  +--rw astronomical-body?   string
  |  |  |  +--rw geodetic-system
  |  |  |     +--rw geodetic-datum?    string
  |  |  |     +--rw coord-accuracy?    decimal64
  |  |  |     +--rw height-accuracy?   decimal64
  |  |  +--rw (location)?
  |  |  |  +--:(ellipsoid)
  |  |  |  |  +--rw latitude?    decimal64
  |  |  |  |  +--rw longitude?   decimal64
  |  |  |  |  +--rw height?      decimal64
  |  |  |  +--:(cartesian)
  |  |  |     +--rw x?           decimal64
  |  |  |     +--rw y?           decimal64
  |  |  |     +--rw z?           decimal64
  |  |  +--rw velocity
  |  |  |  +--rw v-north?   decimal64
  |  |  |  +--rw v-east?    decimal64
  |  |  |  +--rw v-up?      decimal64
  |  |  +--rw timestamp?         yang:date-and-time
  |  |  +--rw valid-until?       yang:date-and-time
  |  +--rw name?                string
  |  +--rw version?             string
  |  +--rw identifier?          string
  |  +--rw description?         string
  |  +--rw manufacturer?        string
  |  +--rw model?               string
  |  +--rw hosted-by?           string
  |  +--rw installation-time?   uint64
  +--rw transducers
     +--rw transducer* [type]
        +--rw type                       identityref
        +--ro quantity
        |  +--ro value?              int64
        |  +--ro timestamp?          uint64
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
        |  |  +--ro active?       boolean
        |  |  +--rw t-min?        int32
        |  |  +--rw t-max?        int32
        |  |  +--rw hysteresis?   uint8
        |  |  +--rw dampening?    uint32
        |  +--rw check-interval?   uint16
        +---x reset-stats

  rpcs:
    +---x reset-stats

  notifications:
    +---n history
    |  +--ro last?          boolean
    |  +--ro time-series* [type]
    |     +--ro type        identityref
    |     +--ro values*     int64
    |     +--ro internal
    |        +--ro last-update?     uint64
    |        +--ro start-time?      uint64
    |        +--ro messages-sent?   uint64
    +---n sensor-alert
       +--ro target* [type]
          +--ro type     identityref
          +--ro value?   int64
~~~~
{: #fig-cf-m2m-tree title="coreconf-m2m module tree" artwork-align="center"}

# Complete YANG Module {#annex-yang}

~~~~
module coreconf-m2m {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:coreconf-m2m";
  prefix ccm2m;

  import ietf-geo-location {
    prefix geo;
    reference "RFC 9179";
  }

  organization "IETF";
  contact
    "Internet Engineering Task Force (IETF)
     https://www.ietf.org/";

  description
    "YANG data model for a generic M2M CoMI weather station.
     This model defines the operational state data for sensor readings.";

  revision 2026-08-28 {
    description
      "Rename the 'states' container to 'bootstrap', since it holds
       operational data determined once at bootstrap time.
       Split the transducer list: unit-override, precision-override and
       category-override are moved out of /transducers/transducer into a
       new /bootstrap/inventory list (also keyed by type). This list
       describes the fixed configuration of each transducer instance,
       determined at bootstrap and not expected to change afterwards,
       separately from the transducer's runtime data (quantity,
       statistics, notification parameters) which stays in
       /transducers/transducer.
       Hoist check-interval from notification-parameters/history and
       notification-parameters/sensor-alert to notification-parameters:
       it was defined identically in both containers, and a single
       CON/NON cadence now applies to both notification streams.";
  }

  revision 2026-08-24 {
    description
      "Remove type from list keys; id alone now identifies a transducer
       instance, distinguishing multiple actuators of the same type. This
       reduces the key size.
       Add a reference-epoch leaf in the state container; other epochs in
       the model are relative to this one.
       Move all concrete transducer-type identities (solar-radiation,
       precipitation, air-temperature, relative-humidity,
       barometric-pressure, vapor-pressure, wind-speed, wind-direction,
       wind-gust, north-wind-speed, east-wind-speed, strike-count,
       average-distance, tilt, x-orientation, y-orientation,
       humidity-sensor-temperature, compass-heading) out of coreconf-m2m and
       into the new atmos module; coreconf-m2m now only defines the base
       transducer-type identity, which product-specific modules extend.
       Rename transducer/unit and transducer/precision to unit-override and
       precision-override (both now optional, no default): they only need
       to be set when they differ from the default-unit/default-precision
       extension declared on the transducer's type identity.
       Add default-unit and default-precision extensions, for use by
       product-specific modules (e.g. atmos) annotating the transducer-type
       identities they define.
       Add SOSA/SAREF-aligned leafs to the characteristics container
       (description, manufacturer, model, hosted-by, installation-time);
       move uptime back into states, as bootstrap operational data.
       Add default-category extension, for product-specific modules to mark
       each transducer-type identity as 'sensor', 'actuator' or
       'sensor-actuator', and a matching category-override leaf on the
       transducer list for per-instance overrides.
       Remove u-timestamp: all timestamps are now second-resolution only;
       sub-second precision would require a model update.
       Change history/step units from milliseconds to seconds, for the
       same reason.
       Add minimal-step leaf in states and a must constraint requiring
       history/step >= states/minimal-step.
       Move geo-location from states into characteristics: it is
       configuration data set once at deployment, not runtime state.
       Make states/uptime mandatory.
       Revert the id-based key: transducer, transducers-list/time-series
       and sensor-alert/target are again keyed on type alone, and the id
       leaf is removed from all three lists (only one transducer per
       type is supported).";
  }

  revision 2026-06-07 {
    description
      "Add humidity-sensor-temperature and compass-heading transducer-type identities.";
  }

  revision 2026-05-26 {
    description
      "Add check-interval leaf to history and sensor-alert notification parameters.";
  }

  revision 2026-03-29 {
    description
      "Move container statistics inside container quantity.";
  }

  revision 2026-03-25 {
    description
      "Add dampening leaf to sensor-alert notification parameters.";
  }

  revision 2026-03-23 {
    description
      "Make active leaf config false (reflects observe subscription state).
       Add must constraint t-min < t-max on sensor-alert.
       Add timestamp-source enum in quantity to indicate timestamp origin
       (source = timestamped by the sensor, receiver = timestamped on receipt).";
  }

  revision 2026-03-22 {
    description
      "Rename branch measurement to transducer
       (container, list, identity, notification).";
  }

  revision 2026-03-08 {
    description
      "Module renamed to coreconf-m2m for generic M2M CoMI use.";
  }

  revision 2026-03-02 {
    description
      "Major update for CoMI compliance:
       - Added multi-sensor support (id key).
       - Measurements are now configurable (writable).
       - Added standard deviation (stdev).
       - Added uptime to state.
       - Removed tilt from fixed state (now a measurement type).
       - Removed legacy RPCs (subscribe, get-stats) in favor of CoMI/Observe.
       - Cleaned up measurement lists and notifications.";
  }

  revision 2026-02-27 {
    description
      "Initial revision for testing.";
  }

  /* ------------------------------------------------------------------ */
  /* Extensions                                                           */
  /* ------------------------------------------------------------------ */

  extension default-unit {
    argument "senml-unit";
    description
      "Recommended default value for a transducer's unit-override leaf,
       expressed as a SenML (RFC 8428) unit string. Applied by
       product-specific modules to the transducer-type identities they
       define. Intended as guidance for clients populating
       /transducers/transducer/unit-override; not enforced by the schema.";
  }

  extension default-precision {
    argument "decimal-places";
    description
      "Recommended default value for a transducer's precision-override leaf
       (number of decimal places, i.e. real value = raw_value * 10^-precision)
       when using the default-unit. Applied by product-specific modules to
       the transducer-type identities they define. Intended as guidance for
       clients populating /transducers/transducer/precision-override; not
       enforced by the schema.";
  }

  extension default-category {
    argument "category";
    description
      "Recommended default value for a transducer's category-override leaf.
       Extension arguments are plain strings and cannot be constrained to
       an enumeration by the YANG language itself; the value SHOULD be one
       of the category-override enums: 'sensor' (the transducer-type
       identity represents an observed property), 'actuator' (it
       represents a property acted upon), or 'sensor-actuator' (both).
       Applied by
       product-specific modules to the transducer-type identities they
       define, since this role is a property of the measurement type, not
       of the device as a whole (a device can carry a mix of sensor- and
       actuator-typed transducers).
       For example, in the SOSA/SSN ontology 'sensor' corresponds to
       sosa:Sensor/sosa:ObservableProperty and 'actuator' to
       sosa:Actuator; in SAREF they correspond to the SensingFunction and
       ActuatingFunction classes respectively.";
  }

  /* ------------------------------------------------------------------ */
  /* Measurement-type identity hierarchy                                  */
  /* ------------------------------------------------------------------ */

  identity transducer-type {
    description
      "Base identity for all measurement types. Concrete transducer types
       are defined by product-specific modules, e.g. atmos.";
  }

  /* ------------------------------------------------------------------ */
  /* Typedefs                                                             */
  /* ------------------------------------------------------------------ */

  typedef encoding-type {
    type enumeration {
      enum direct {
        value 0;
        description "Value is encoded directly as an integer.";
      }
      enum delta {
        value 1;
        description "Value is encoded as a delta from the previous value.";
      }
    }
    description "Encoding used for the measurement values.";
  }

  /* ------------------------------------------------------------------ */
  /* Shared grouping used by notifications                                */
  /* ------------------------------------------------------------------ */

  grouping transducers-list {
    list time-series {
      key "type";
      description
        "List of measurements included in this notification.";

      leaf type {
        type identityref {
          base transducer-type;
        }
        description "The type of measurement (e.g., air-temperature).";
      }

      leaf-list values {
        type int64;
        ordered-by user;
        description
          "List of encoded measurement values.";
      }
    }
  }

  /* ------------------------------------------------------------------ */
  /* State data                                                           */
  /* ------------------------------------------------------------------ */

  container bootstrap {
    config false;
    description
      "Operational data determined once at bootstrap time.";

    leaf reference-epoch {
      type uint64;
      units "seconds";
      description
        "Epoch timestamp used as the reference point for the device's
         internal clock. Other epoch/timestamp values in this model are
         relative to this reference epoch.";
    }

    leaf uptime {
      type uint64;
      units "seconds";
      mandatory true;
      description "Time elapsed since the last boot.";
    }

    leaf minimal-step {
      type uint32;
      units "seconds";
      description
        "Minimum interval at which the underlying system refreshes
         transducer values, i.e. the smallest meaningful value for
         history/step. Determined by the hardware/firmware and not
         configurable (e.g. an atmospheric station may have a
         minimal-step of 120 seconds).";
    }

    list inventory {
      key "type";
      description
        "List of transducers known to the device, with their
         per-instance configuration overrides. Determined once at
         bootstrap and not expected to change afterwards.";

      leaf type {
        type identityref {
          base transducer-type;
        }
        description "The type of measurement.";
      }

      leaf unit-override {
        type string;
        description
          "Unit of measurement (e.g., 'Cel', 'm/s', '%'), overriding the
           default-unit extension declared on the transducer's type identity
           (if any). When absent, clients should use that identity's
           default-unit; if the identity declares no default-unit, the unit
           is undefined.";
      }

      leaf precision-override {
        type uint8;
        description
          "Number of decimal places for the measurement value
           (real value = raw_value * 10^-precision), overriding the
           default-precision extension declared on the transducer's type
           identity (if any). Applies to the current measurement value
           (polling/iPATCH), not to the time-series encoding in
           notifications. When absent, clients should use that identity's
           default-precision; if the identity declares no default-precision,
           the precision is undefined.";
      }

      leaf category-override {
        type enumeration {
          enum sensor {
            description "The transducer represents an observed property.";
          }
          enum actuator {
            description "The transducer represents a property acted upon.";
          }
          enum sensor-actuator {
            description
              "The transducer both observes and can be acted upon
               (e.g. a setpoint that also reports the measured value).";
          }
        }
        description
          "Role of this transducer instance, overriding the
           default-category extension declared on the transducer's type
           identity (if any). When absent, clients should use that
           identity's default-category; if the identity declares no
           default-category, the role is undefined.";
      }
    }
  }

  /* ------------------------------------------------------------------ */
  /* Device Characteristics                                               */
  /* ------------------------------------------------------------------ */

  container characteristics {
    description
      "Static descriptive information about the device.";

    uses geo:geo-location {
      description
        "Fixed geographic position of the weather station.
         Populated once at deployment; not expected to change
         during normal operation.";
      refine "geo-location/reference-frame/geodetic-system/geodetic-datum" {
        default "wgs-84";
      }
    }

    leaf name {
      type string;
      description "Human-readable name of the device.";
    }

    leaf version {
      type string;
      description "Firmware or software version of the device.";
    }

    leaf identifier {
      type string;
      description "Unique identifier of the device (e.g., serial number or EUI).";
    }

    leaf description {
      type string;
      description
        "Free-text description of the device, its purpose or its
         deployment context. For example, in SOSA/SSN this corresponds to
         rdfs:comment on the sosa:Sensor/sosa:Platform individual, and in
         SAREF to saref:hasDescription.";
    }

    leaf manufacturer {
      type string;
      description
        "Name of the device manufacturer. For example, in SAREF this
         corresponds to saref:hasManufacturer.";
    }

    leaf model {
      type string;
      description
        "Commercial model name/number of the device. For example, in
         SAREF this corresponds to saref:hasModel.";
    }

    leaf hosted-by {
      type string;
      description
        "Identifier of the platform hosting this device, when applicable.
         Left as an opaque string here; resolution to an actual platform
         resource is deployment-specific. For example, in SOSA this
         corresponds to the sosa:isHostedBy relation
         (device -> sosa:Platform).";
    }

    leaf installation-time {
      type uint64;
      units "seconds";
      description
        "Absolute Unix epoch timestamp (UTC) at which the device was
         installed/commissioned. Unlike bootstrap/reference-epoch, this is
         an absolute epoch value, not a relative reference point for the
         device's internal clock.";
    }
  }

  /* ------------------------------------------------------------------ */
  /* Measurements Configuration & Data                                    */
  /* ------------------------------------------------------------------ */

  container transducers {
    // config true (default);
    description
      "Current measurement values accessible via polling or configuration.";

    list transducer {
      key "type";
      description
        "List of available measurements.";

      leaf type {
        type identityref {
          base transducer-type;
        }
        description "The type of measurement.";
      }

      container quantity {
        config false;
        description "Operational state for this transducer.";

        leaf value {
          type int64;
          description
            "The current raw integer value of the transducer.";
        }

        leaf timestamp {
          type uint64;
          units "seconds";
          description "Epoch timestamp (seconds) of the last transducer update.";
        }

        leaf timestamp-source {
          type enumeration {
            enum source {
              value 0;
              description
                "The timestamp was generated by the sensor itself.";
            }
            enum receiver {
              value 1;
              description
                "The timestamp was applied by the receiver upon message arrival.";
            }
          }
          description
            "Indicates the origin of the timestamp associated with this measurement.";
        }

        container statistics {
          config false;
          description "Read-only accumulated statistics for this measurement.";

          leaf min {
            type int64;
            description "Minimum value observed since last reset.";
          }

          leaf max {
            type int64;
            description "Maximum value observed since last reset.";
          }

          leaf mean {
            type int64;
            description "Mean value observed since last reset.";
          }

          leaf median {
            type int64;
            description
              "Estimated median value observed since last reset.
               This median is computed incrementally from the current stored
               statistics and the newest sample, without keeping full history.";
          }

          leaf stdev {
            type uint64;
            description
              "Standard deviation of values observed since last reset.
               Scaled by same precision as value.";
          }

          leaf sample-count {
            type uint64;
            description "Number of samples used for statistics calculation.";
          }
        }
      }

      container notification-parameters {
        description
          "Configuration parameters controlling how this transducer
           is reported in CoMI notification streams.";

        container history {
          description "Parameters for the history time-series stream.";

          leaf active {
            type boolean;
            config false;
            description
              "Read-only. True when a FETCH+Observe subscription is active on /s
               for this transducer history stream. False when no observer is registered.";
          }

          leaf step {
            type uint32;
            units "seconds";
            must ". >= /bootstrap/minimal-step or not(/bootstrap/minimal-step)" {
              error-message
                "step must be greater than or equal to bootstrap/minimal-step.";
            }
            description
              "Time interval between samples in the notification stream.
               Must be at least bootstrap/minimal-step, the system's minimum
               refresh interval.";
          }

          leaf precision {
            type uint8;
            default 1;
            description
              "Number of decimal places for values encoded in time-series notifications.
               Real value = raw_value * 10^-precision.";
          }

          leaf max-samples {
            type uint32;
            description "Maximum number of samples kept in the time-series buffer.";
          }

          leaf time-period {
            type uint32;
            units "seconds";
            description "Duration of the time window covered by the history buffer.";
          }

          leaf encoding {
            type encoding-type;
            default "delta";
            description "Encoding used for the values list in notifications.";
          }

          leaf max-payload {
            type uint32;
            units "bytes";
            description
              "Maximum size in bytes of the encoded time-series payload in
               a single history notification.  When the encoded buffer would
               exceed this limit, the notification is sent early (before
               max-samples or time-period is reached).  0 means no size limit.";
          }

        }

        container sensor-alert {
          description "Parameters for threshold-based sensor-alert notifications.";

          must "not(t-min and t-max) or t-min < t-max" {
            error-message "t-min must be strictly less than t-max.";
          }

          leaf active {
            type boolean;
            config false;
            description
              "Read-only. True when a FETCH+Observe subscription is active on /s
               for this transducer sensor-alert stream. False when no observer is registered.";
          }

          leaf t-min {
            type int32;
            description
              "Minimum threshold value. A sensor-alert is raised when the
               transducer value falls below this value.";
          }

          leaf t-max {
            type int32;
            description
              "Maximum threshold value. A sensor-alert is raised when the
               transducer value exceeds this value.";
          }

          leaf hysteresis {
            type uint8 {
              range "0..100";
            }
            default 5;
            units "percent";
            description
              "Hysteresis applied to threshold crossings, expressed as a
               percentage of the threshold value.";
          }

          leaf dampening {
            type uint32;
            default 0;
            units "milliseconds";
            description
              "Minimum time that must elapse between two consecutive sensor-alert
               notifications for the same transducer.  A new alert is suppressed
               until the dampening period has expired since the last sent alert.
               0 (default) means no dampening.";
          }
        }

        leaf check-interval {
          type uint16;
          default 0;
          description
            "Controls how often a CON (confirmable) notification is sent
             instead of NON (non-confirmable), for both the history and
             sensor-alert streams of this transducer.
             0 = disabled, all notifications are NON.
             1 = every notification is CON.
             N = one CON every N notifications, all others are NON.";
        }
      }

      action reset-stats {
        description
          "Reset the accumulated statistics (min, max, mean, median, stdev)
           for this measurement. Counters restart from the next sample received.";
      }
    }
  }

  /* ------------------------------------------------------------------ */
  /* RPCs                                                                 */
  /* ------------------------------------------------------------------ */

  rpc reset-stats {
    description
      "Reset the accumulated statistics for all measurements.
       The counters restart from the next sample received after the call.";
  }

  /* ------------------------------------------------------------------ */
  /* Notifications                                                        */
  /* ------------------------------------------------------------------ */

  notification history {
    description
      "Measurement event sent to observers of the notification stream.
       To receive this, a CoLA/CoMI client must Observe the stream resource.";

    leaf last {
      type boolean;
      default "false";
      description "True if this is the last notification for the current series.";
    }

    uses transducers-list {
      augment "time-series" {
        container internal {
          description "Internal operational state for this time-series entry.";
          leaf last-update {
            type uint64;
            units "seconds";
            description "Epoch timestamp of the last sample appended to this time-series.";
          }
          leaf start-time {
            type uint64;
            units "seconds";
            description "Epoch timestamp of the first sample; set when the Observe subscription is created.";
          }
          leaf messages-sent {
            type uint64;
            description "Number of notification messages sent for this time-series since the Observe subscription was created.";
          }
        }
      }
    }
  }

  notification sensor-alert {
    description
      "Alert notification when a value exceeds a threshold.";

    list target {
      key "type";
      description "Measurement instances that triggered this alert.";

      leaf type {
        type identityref {
          base transducer-type;
        }
      }

      leaf value {
        type int64;
        description "The measurement value that triggered this alert.";
      }
    }
  }
}
~~~~
{: #fig-yang-module title="Complete coreconf-m2m YANG module" artwork-align="left"}

# SID File (CSV)

The following table lists the SID assignments for the coreconf-m2m module
(assignment range: 100000–100399, revision 2026-03-29).

~~~~
SID,Namespace,Identifier
100000,module,coreconf-m2m
100001,identity,air-temperature
100002,identity,average-distance
100003,identity,barometric-pressure
100004,identity,east-wind-speed
100005,identity,north-wind-speed
100006,identity,precipitation
100007,identity,relative-humidity
100008,identity,solar-radiation
100009,identity,strike-count
100010,identity,tilt
100011,identity,transducer-type
100012,identity,vapor-pressure
100013,identity,wind-direction
100014,identity,wind-gust
100015,identity,wind-speed
100016,identity,x-orientation
100017,identity,y-orientation
100018,data,/coreconf-m2m:characteristics
100019,data,/coreconf-m2m:characteristics/geo-location
100020,data,/coreconf-m2m:characteristics/geo-location/height
100021,data,/coreconf-m2m:characteristics/geo-location/latitude
100022,data,/coreconf-m2m:characteristics/geo-location/longitude
100023,data,/coreconf-m2m:characteristics/geo-location/reference-frame
100024,data,/coreconf-m2m:characteristics/geo-location/reference-frame/alternate-system
100025,data,/coreconf-m2m:characteristics/geo-location/reference-frame/astronomical-body
100026,data,/coreconf-m2m:characteristics/geo-location/reference-frame/geodetic-system
100027,data,/coreconf-m2m:characteristics/geo-location/reference-frame/geodetic-system/coord-accuracy
100028,data,/coreconf-m2m:characteristics/geo-location/reference-frame/geodetic-system/geodetic-datum
100029,data,/coreconf-m2m:characteristics/geo-location/reference-frame/geodetic-system/height-accuracy
100030,data,/coreconf-m2m:characteristics/geo-location/timestamp
100031,data,/coreconf-m2m:characteristics/geo-location/valid-until
100032,data,/coreconf-m2m:characteristics/geo-location/velocity
100033,data,/coreconf-m2m:characteristics/geo-location/velocity/v-east
100034,data,/coreconf-m2m:characteristics/geo-location/velocity/v-north
100035,data,/coreconf-m2m:characteristics/geo-location/velocity/v-up
100036,data,/coreconf-m2m:characteristics/geo-location/x
100037,data,/coreconf-m2m:characteristics/geo-location/y
100038,data,/coreconf-m2m:characteristics/geo-location/z
100039,data,/coreconf-m2m:characteristics/identifier
100040,data,/coreconf-m2m:characteristics/name
100041,data,/coreconf-m2m:characteristics/version
100042,data,/coreconf-m2m:history
100043,data,/coreconf-m2m:history/last
100044,data,/coreconf-m2m:history/time-series
100045,data,/coreconf-m2m:history/time-series/id
100046,data,/coreconf-m2m:history/time-series/internal
100047,data,/coreconf-m2m:history/time-series/internal/last-update
100048,data,/coreconf-m2m:history/time-series/internal/messages-sent
100049,data,/coreconf-m2m:history/time-series/internal/start-time
100050,data,/coreconf-m2m:history/time-series/type
100051,data,/coreconf-m2m:history/time-series/values
100052,data,/coreconf-m2m:reset-stats
100053,data,/coreconf-m2m:reset-stats/input
100054,data,/coreconf-m2m:reset-stats/output
100055,data,/coreconf-m2m:sensor-alert
100056,data,/coreconf-m2m:sensor-alert/target
100057,data,/coreconf-m2m:sensor-alert/target/id
100058,data,/coreconf-m2m:sensor-alert/target/type
100059,data,/coreconf-m2m:sensor-alert/target/value
100060,data,/coreconf-m2m:state
100061,data,/coreconf-m2m:state/uptime
100062,data,/coreconf-m2m:transducers
100063,data,/coreconf-m2m:transducers/transducer
100064,data,/coreconf-m2m:transducers/transducer/id
100065,data,/coreconf-m2m:transducers/transducer/nature
100066,data,/coreconf-m2m:transducers/transducer/notification-parameters
100067,data,/coreconf-m2m:transducers/transducer/notification-parameters/history
100068,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/active
100069,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/encoding
100070,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/max-payload
100071,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/max-samples
100072,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/precision
100073,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/step
100074,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/time-period
100075,data,/coreconf-m2m:transducers/transducer/notification-parameters/sensor-alert
100076,data,/coreconf-m2m:transducers/transducer/notification-parameters/sensor-alert/active
100077,data,/coreconf-m2m:transducers/transducer/notification-parameters/sensor-alert/dampening
100078,data,/coreconf-m2m:transducers/transducer/notification-parameters/sensor-alert/hysteresis
100079,data,/coreconf-m2m:transducers/transducer/notification-parameters/sensor-alert/t-max
100080,data,/coreconf-m2m:transducers/transducer/notification-parameters/sensor-alert/t-min
100081,data,/coreconf-m2m:transducers/transducer/precision
100082,data,/coreconf-m2m:transducers/transducer/quantity
100083,data,/coreconf-m2m:transducers/transducer/quantity/statistics
100084,data,/coreconf-m2m:transducers/transducer/quantity/statistics/max
100085,data,/coreconf-m2m:transducers/transducer/quantity/statistics/mean
100086,data,/coreconf-m2m:transducers/transducer/quantity/statistics/median
100087,data,/coreconf-m2m:transducers/transducer/quantity/statistics/min
100088,data,/coreconf-m2m:transducers/transducer/quantity/statistics/sample-count
100089,data,/coreconf-m2m:transducers/transducer/quantity/statistics/stdev
100090,data,/coreconf-m2m:transducers/transducer/quantity/timestamp
100091,data,/coreconf-m2m:transducers/transducer/quantity/timestamp-source
100092,data,/coreconf-m2m:transducers/transducer/quantity/u-timestamp
100093,data,/coreconf-m2m:transducers/transducer/quantity/value
100094,data,/coreconf-m2m:transducers/transducer/reset-stats
100095,data,/coreconf-m2m:transducers/transducer/reset-stats/input
100096,data,/coreconf-m2m:transducers/transducer/reset-stats/output
100097,data,/coreconf-m2m:transducers/transducer/type
100098,data,/coreconf-m2m:transducers/transducer/unit
~~~~
{: #fig-sid-csv title="SID assignments for coreconf-m2m (CSV format)" artwork-align="left"}

# Acknowledgments
{:numbered="false"}
This work has been supported by the SCHC Chair from IMT Atlantique and Afnic.

