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
        |  +--ro value?              int64
        |  +--ro timestamp?          uint64
        |  +--ro timestamp-source?   enumeration
        +--ro statistics
           +--ro min?            int64
           +--ro max?            int64
           +--ro mean?           int64
           +--ro median?         int64
           +--ro stdev?          uint64
           +--ro sample-count?   uint64
~~~~
{: #fig-transducer-tree title="transducer sub-tree (type, quantity and statistics)" artwork-align="center"}

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

"statistics" is a sibling of "quantity" and holds the main statistics
locally computed for a specific transducer, accumulated over a window of
samples rather than tied to the current measurement. Statistics can be
reset for a single transducer via the "reset-stats" action, or for all
transducers at once via the "reset-stats" RPC.


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

##  Notifications

As seen in the previous section, each transducer has a
"notification-parameters" branch holding its notification settings. A
client MAY FETCH this branch to retrieve its content and check the current
status of the notification (e.g., the "active" flag).

A client wishing to start a notification MAY first send an iPATCH to
configure its parameters. Parameters changed while a notification is
already active take effect immediately. These parameters are global to
the transducer: a modification impacts every notification currently
active for that transducer, regardless of which client configured it.

Notification is started by sending a FETCH+Observe request on the notification
stream resource (`/s`), with a body identifying the SID of the transducer to
observe. Multiple clients may observe the same transducer simultaneously; each
receives an independent copy of every notification.

The notification ends when either:

* the client explicitly sends a termination message (a GET/FETCH with the
  Observe option set to 1, i.e. deregister, using the same token as the
  original observation), or
* the server sends Confirmable notifications and does not receive an
  acknowledgment, or
* an ICMPv6 unreachable message is received by the
  server in response to the IPv6/UDP/CoAP packet carrying the
  notification's token.

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

## SID Allocations

The coreconf-m2m model is intended to be assigned in the IETF experimental
SID space, with a model entry point of 62000. {{annex-sid}} gives the
complete SID mapping for this model.

The atmos module, however, cannot be assigned SIDs in the IETF or another
SDO space, unless this commercial module is itself standardized. For
commercial modules, we assume that a registrar allocates SIDs and
implements the constraints defined in {{I-D.ietf-core-comi}}. In this
example, we assume that the registrar owns a 10 million SID mega-range.
{{annex-atmos-yang}} gives the atmos.yang model, and {{annex-atmos-sid}}
gives the corresponding SID mapping, composed of the identities
representing the transducers.

# CORECONF Traffic

This section illustrates different traffic patterns used in coreconf-m2m.

## Resource Discovery

The client does not know the transducers managed by the device. It sends a FETCH
on "/bootstrap" (SID 62002) as shown in
{{fig-resource-discovery}}.

~~~~
  CoAP Request:
  Non-Confirmable, FETCH, MID:39919
    Token: 2aaf
    Opt #1: Uri-Path: c
    Opt #2: Content-Format: 141 (application/yang-fetch+cbor)
    Opt #3: Accept: 142 (application/yang-data+cbor;id=sid)

  Payload: 3 bytes
    19 F2 32  # unsigned(62002) : /bootstrap

  CoAP Response:
  Non-Confirmable, 2.05 Content, MID:57745
    Token: 2aaf
    Opt #1: Content-Format: 142 (application/yang-data+cbor;id=sid)

  Payload: 118 bytes
    {62002:
      {7: 1788334280, 8: 30172, 6: 120,
       1: [{3: 10000010}, {3: 10000008}, {3: 10000011}, {3: 10000002},
           {3: 10000014}, {3: 10000016}, {3: 10000015}, {3: 10000012},
           {3: 10000001}, {3: 10000013}, {3: 10000003}, {3: 10000009},
           {3: 10000006}, {3: 10000004}]}}
~~~~
{: #fig-resource-discovery title="Resource discovery: FETCH request and response" artwork-align="left"}

In the request, the payload `19 F2 32` is the CBOR encoding of the unsigned
integer 62002, which is the SID of "/bootstrap". In the response, the outer
key 62002 identifies the bootstrap branch, and the inner keys are delta
SIDs relative to 62002, as defined in {{RFC9254}}: 7 (reference-epoch), 8
(uptime), 6 (minimal-step), and 1 (the "inventory" list). Each entry in
the inventory list is itself a map whose key 3 is a delta SID relative to
the list SID (62003), i.e. "type", carrying the identity SID of one
transducer known to the device; no override is present here, meaning every
transducer uses the default-unit, default-precision and default-category
declared on its identity.

The client MUST have access to the corresponding YANG file to discover the
transducers' default parameters, indicating the unit, the precision and
the nature of the transducer.

~~~~
 coreconf-m2m:bootstrap
    reference-epoch  1788334280  (2026-09-02 09:31:20)   ← origin of every timestamp in the model
    uptime           30172 s
    minimal-step     120 s   ← floor for history/step

  coreconf-m2m:bootstrap/inventory — SID → module → YANG defaults resolution
        SID  Identity                           Module         Unit      Prec.  Category
  ────────────────────────────────────────────────────────────────────────────────────────────
   10000010  atmos:solar-radiation              atmos          W/m2          0  sensor
   10000008  atmos:precipitation                atmos          mm            3  sensor
   10000011  atmos:strike-count                 atmos          count         0  sensor
   10000002  atmos:average-distance             atmos          km            0  sensor
   10000014  atmos:wind-direction               atmos          deg           0  sensor
   10000016  atmos:wind-speed                   atmos          m/s           2  sensor
   10000015  atmos:wind-gust                    atmos          m/s           2  sensor
   10000012  atmos:tilt                         atmos          deg           1  sensor
   10000001  atmos:air-temperature              atmos          Cel           1  sensor
   10000013  atmos:vapor-pressure               atmos          kPa           2  sensor
   10000003  atmos:barometric-pressure          atmos          kPa           2  sensor
   10000009  atmos:relative-humidity            atmos          %RH           1  sensor
   10000006  atmos:humidity-sensor-temperature  atmos          Cel           1  sensor
   10000004  atmos:compass-heading              atmos          deg           0  sensor
  ────────────────────────────────────────────────────────────────────────────────────────────
  * override sent by the device; without a star, default value read from the YANG module.
    The inventory completed this way stays local: bootstrap is config false and is never sent back.
Connected. 14 sensor(s) discovered, minimal step 120 s.

    #  Type                         Unit     Prec.  Filter
  ──────────────────────────────────────────────────────────────────────────────
    1  solar-radiation              W/m2         0  [type='atmos:solar-radiation']
    2  precipitation                mm           3  [type='atmos:precipitation']
    3  strike-count                 count        0  [type='atmos:strike-count']
    4  average-distance             km           0  [type='atmos:average-distance']
    5  wind-direction               deg          0  [type='atmos:wind-direction']
    6  wind-speed                   m/s          2  [type='atmos:wind-speed']
    7  wind-gust                    m/s          2  [type='atmos:wind-gust']
    8  tilt                         deg          1  [type='atmos:tilt']
    9  air-temperature              Cel          1  [type='atmos:air-temperature']
   10  vapor-pressure               kPa          2  [type='atmos:vapor-pressure']
   11  barometric-pressure          kPa          2  [type='atmos:barometric-pressure']
   12  relative-humidity            %RH          1  [type='atmos:relative-humidity']
   13  humidity-sensor-temperature  Cel          1  [type='atmos:humidity-sensor-temperature']
   14  compass-heading              deg          0  [type='atmos:compass-heading']

~~~~
{: #fig-transducer-list title="Decoded transducer inventory from resource discovery response" artwork-align="left"}


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

The FETCH body `[100082, 100001, 0]` requests the statistics sub-tree (SID
100082) for air-temperature (100001), instance 0. In the response, the
inner map keys are delta SIDs relative to 100082, each encoding the
difference between consecutive SIDs to minimize CBOR size. The six values
correspond to the statistics leaves: min, max, mean, median, stdev, and
sample-count, all scaled by the transducer precision.

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
        +--ro statistics
        |  +--ro min?            int64
        |  +--ro max?            int64
        |  +--ro mean?           int64
        |  +--ro median?         int64
        |  +--ro stdev?          uint64
        |  +--ro sample-count?   uint64
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

  revision 2026-09-01 {
    description
      "Move container statistics out of container quantity, making it a
       sibling of quantity under list transducer. Statistics accumulate
       over a window and are not part of the current measurement, so
       nesting them under quantity conflated two different lifetimes and
       cost one extra level of delta encoding on every statistics access.
       This reverts the nesting introduced in revision 2026-03-29.";
  }

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

# SID File (CSV) {#annex-sid}

The following table lists the SID assignments for the coreconf-m2m module
(assignment range: 62000-62399, revision 2026-09-01). The transducer-type
identities defined by manufacturer modules such as atmos are assigned
separately and are not yet included in this table.

~~~~
SID,Namespace,Identifier
62000,module,coreconf-m2m
62001,identity,transducer-type
62002,data,/coreconf-m2m:bootstrap
62003,data,/coreconf-m2m:bootstrap/inventory
62004,data,/coreconf-m2m:bootstrap/inventory/category-override
62005,data,/coreconf-m2m:bootstrap/inventory/precision-override
62006,data,/coreconf-m2m:bootstrap/inventory/type
62007,data,/coreconf-m2m:bootstrap/inventory/unit-override
62008,data,/coreconf-m2m:bootstrap/minimal-step
62009,data,/coreconf-m2m:bootstrap/reference-epoch
62010,data,/coreconf-m2m:bootstrap/uptime
62011,data,/coreconf-m2m:characteristics
62012,data,/coreconf-m2m:characteristics/description
62013,data,/coreconf-m2m:characteristics/geo-location
62014,data,/coreconf-m2m:characteristics/geo-location/height
62015,data,/coreconf-m2m:characteristics/geo-location/latitude
62016,data,/coreconf-m2m:characteristics/geo-location/longitude
62017,data,/coreconf-m2m:characteristics/geo-location/reference-frame
62018,data,/coreconf-m2m:characteristics/geo-location/reference-frame/alternate-system
62019,data,/coreconf-m2m:characteristics/geo-location/reference-frame/astronomical-body
62020,data,/coreconf-m2m:characteristics/geo-location/reference-frame/geodetic-system
62021,data,/coreconf-m2m:characteristics/geo-location/reference-frame/geodetic-system/coord-accuracy
62022,data,/coreconf-m2m:characteristics/geo-location/reference-frame/geodetic-system/geodetic-datum
62023,data,/coreconf-m2m:characteristics/geo-location/reference-frame/geodetic-system/height-accuracy
62024,data,/coreconf-m2m:characteristics/geo-location/timestamp
62025,data,/coreconf-m2m:characteristics/geo-location/valid-until
62026,data,/coreconf-m2m:characteristics/geo-location/velocity
62027,data,/coreconf-m2m:characteristics/geo-location/velocity/v-east
62028,data,/coreconf-m2m:characteristics/geo-location/velocity/v-north
62029,data,/coreconf-m2m:characteristics/geo-location/velocity/v-up
62030,data,/coreconf-m2m:characteristics/geo-location/x
62031,data,/coreconf-m2m:characteristics/geo-location/y
62032,data,/coreconf-m2m:characteristics/geo-location/z
62033,data,/coreconf-m2m:characteristics/hosted-by
62034,data,/coreconf-m2m:characteristics/identifier
62035,data,/coreconf-m2m:characteristics/installation-time
62036,data,/coreconf-m2m:characteristics/manufacturer
62037,data,/coreconf-m2m:characteristics/model
62038,data,/coreconf-m2m:characteristics/name
62039,data,/coreconf-m2m:characteristics/version
62040,data,/coreconf-m2m:history
62041,data,/coreconf-m2m:history/last
62042,data,/coreconf-m2m:history/time-series
62043,data,/coreconf-m2m:history/time-series/internal
62044,data,/coreconf-m2m:history/time-series/internal/last-update
62045,data,/coreconf-m2m:history/time-series/internal/messages-sent
62046,data,/coreconf-m2m:history/time-series/internal/start-time
62047,data,/coreconf-m2m:history/time-series/type
62048,data,/coreconf-m2m:history/time-series/values
62049,data,/coreconf-m2m:reset-stats
62050,data,/coreconf-m2m:reset-stats/input
62051,data,/coreconf-m2m:reset-stats/output
62052,data,/coreconf-m2m:sensor-alert
62053,data,/coreconf-m2m:sensor-alert/target
62054,data,/coreconf-m2m:sensor-alert/target/type
62055,data,/coreconf-m2m:sensor-alert/target/value
62056,data,/coreconf-m2m:transducers
62057,data,/coreconf-m2m:transducers/transducer
62058,data,/coreconf-m2m:transducers/transducer/notification-parameters
62059,data,/coreconf-m2m:transducers/transducer/notification-parameters/check-interval
62060,data,/coreconf-m2m:transducers/transducer/notification-parameters/history
62061,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/active
62062,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/encoding
62063,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/max-payload
62064,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/max-samples
62065,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/precision
62066,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/step
62067,data,/coreconf-m2m:transducers/transducer/notification-parameters/history/time-period
62068,data,/coreconf-m2m:transducers/transducer/notification-parameters/sensor-alert
62069,data,/coreconf-m2m:transducers/transducer/notification-parameters/sensor-alert/active
62070,data,/coreconf-m2m:transducers/transducer/notification-parameters/sensor-alert/dampening
62071,data,/coreconf-m2m:transducers/transducer/notification-parameters/sensor-alert/hysteresis
62072,data,/coreconf-m2m:transducers/transducer/notification-parameters/sensor-alert/t-max
62073,data,/coreconf-m2m:transducers/transducer/notification-parameters/sensor-alert/t-min
62074,data,/coreconf-m2m:transducers/transducer/quantity
62075,data,/coreconf-m2m:transducers/transducer/quantity/timestamp
62076,data,/coreconf-m2m:transducers/transducer/quantity/timestamp-source
62077,data,/coreconf-m2m:transducers/transducer/quantity/value
62078,data,/coreconf-m2m:transducers/transducer/reset-stats
62079,data,/coreconf-m2m:transducers/transducer/reset-stats/input
62080,data,/coreconf-m2m:transducers/transducer/reset-stats/output
62081,data,/coreconf-m2m:transducers/transducer/statistics
62082,data,/coreconf-m2m:transducers/transducer/statistics/max
62083,data,/coreconf-m2m:transducers/transducer/statistics/mean
62084,data,/coreconf-m2m:transducers/transducer/statistics/median
62085,data,/coreconf-m2m:transducers/transducer/statistics/min
62086,data,/coreconf-m2m:transducers/transducer/statistics/sample-count
62087,data,/coreconf-m2m:transducers/transducer/statistics/stdev
62088,data,/coreconf-m2m:transducers/transducer/type
~~~~
{: #fig-sid-csv title="SID assignments for coreconf-m2m (CSV format)" artwork-align="left"}

# atmos YANG Module {#annex-atmos-yang}

~~~~
module atmos {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:atmos";
  prefix atmos;

  import coreconf-m2m {
    prefix ccm2m;
  }

  organization "METER Group, Inc.";
  contact
    "Technical Support
     https://www.metergroup.com/";

  description
    "Atmospheric/weather transducer-type identities for use with the
     coreconf-m2m generic M2M CoMI data model.";

  revision 2026-08-24 {
    description
      "Initial revision. Extracted all concrete transducer-type identities
       from coreconf-m2m: atmospheric/weather (solar-radiation,
       precipitation, air-temperature, relative-humidity,
       barometric-pressure, vapor-pressure, wind-speed, wind-direction,
       wind-gust, north-wind-speed, east-wind-speed), lightning
       (strike-count, average-distance), orientation (tilt, x-orientation,
       y-orientation, compass-heading), and humidity-sensor-temperature.
       coreconf-m2m now only defines the base transducer-type identity;
       atmos is the product-specific module extending it.
       Applied the ccm2m:default-unit and ccm2m:default-precision
       extensions (defined in coreconf-m2m) to each identity, to advertise
       the recommended SenML (RFC 8428) unit and decimal precision for a
       transducer of that type.";
  }

  /* ------------------------------------------------------------------ */
  /* Environmental                                                        */
  /* ------------------------------------------------------------------ */

  identity solar-radiation {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "W/m2";
    ccm2m:default-precision "0";
    description "Solar radiation measurement (W/m2).";
  }

  identity precipitation {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "mm";
    ccm2m:default-precision "1";
    description "Precipitation measurement (mm).";
  }

  identity air-temperature {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "Cel";
    ccm2m:default-precision "2";
    description "Air temperature measurement (°C).";
  }

  identity relative-humidity {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "%RH";
    ccm2m:default-precision "1";
    description "Relative humidity measurement (%).";
  }

  identity barometric-pressure {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "Pa";
    ccm2m:default-precision "0";
    description "Barometric pressure measurement (kPa).";
  }

  identity vapor-pressure {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "Pa";
    ccm2m:default-precision "0";
    description "Vapor pressure measurement (kPa).";
  }

  /* ------------------------------------------------------------------ */
  /* Wind                                                                 */
  /* ------------------------------------------------------------------ */

  identity wind-speed {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "m/s";
    ccm2m:default-precision "1";
    description "Horizontal wind speed measurement (m/s).";
  }

  identity wind-direction {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "deg";
    ccm2m:default-precision "0";
    description "Wind direction measurement (degrees).";
  }

  identity wind-gust {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "m/s";
    ccm2m:default-precision "1";
    description "Wind gust measurement (m/s).";
  }

  identity north-wind-speed {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "m/s";
    ccm2m:default-precision "1";
    description "North wind speed component (m/s).";
  }

  identity east-wind-speed {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "m/s";
    ccm2m:default-precision "1";
    description "East wind speed component (m/s).";
  }

  /* ------------------------------------------------------------------ */
  /* Lightning                                                            */
  /* ------------------------------------------------------------------ */

  identity strike-count {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "count";
    ccm2m:default-precision "0";
    description "Lightning strike count.";
  }

  identity average-distance {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "km";
    ccm2m:default-precision "1";
    description "Average lightning distance (km).";
  }

  /* ------------------------------------------------------------------ */
  /* Orientation                                                          */
  /* ------------------------------------------------------------------ */

  identity tilt {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "deg";
    ccm2m:default-precision "1";
    description "Sensor tilt measurement (degrees).";
  }

  identity x-orientation {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    description "X-axis orientation (raw accelerometer data).";
  }

  identity y-orientation {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    description "Y-axis orientation (raw accelerometer data).";
  }

  identity compass-heading {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "deg";
    ccm2m:default-precision "0";
    description "Compass heading clockwise from north reference (degrees).";
  }

  /* ------------------------------------------------------------------ */
  /* Internal                                                             */
  /* ------------------------------------------------------------------ */

  identity humidity-sensor-temperature {
    base ccm2m:transducer-type;
    ccm2m:default-category "sensor";
    ccm2m:default-unit "Cel";
    ccm2m:default-precision "2";
    description "Internal temperature of the humidity sensor (°C).";
  }
}
~~~~
{: #fig-atmos-yang title="atmos.yang module"}

# atmos SID File (CSV) {#annex-atmos-sid}

The following table illustrates the SID assignments for the atmos module,
assuming a registrar-owned range with an illustrative entry point of
10000000 (out of a 10 million SID mega-range assumed for this example).
These SIDs are commercial/vendor assignments, not IETF or SDO ones, and
are provided here for illustration only.

~~~~
SID,Namespace,Identifier
10000000,module,atmos
10000001,identity,air-temperature
10000002,identity,average-distance
10000003,identity,barometric-pressure
10000004,identity,compass-heading
10000005,identity,east-wind-speed
10000006,identity,humidity-sensor-temperature
10000007,identity,north-wind-speed
10000008,identity,precipitation
10000009,identity,relative-humidity
10000010,identity,solar-radiation
10000011,identity,strike-count
10000012,identity,tilt
10000013,identity,vapor-pressure
10000014,identity,wind-direction
10000015,identity,wind-gust
10000016,identity,wind-speed
10000017,identity,x-orientation
10000018,identity,y-orientation
~~~~
{: #fig-atmos-sid-csv title="SID assignments for atmos (CSV format, illustrative)" artwork-align="left"}

# Acknowledgments
{:numbered="false"}
This work has been supported by the SCHC Chair from IMT Atlantique and Afnic.

