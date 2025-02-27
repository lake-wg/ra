---
title: "Remote attestation over EDHOC"
abbrev: "RA-EDHOC"
category: std

docname: draft-ietf-lake-ra-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Lightweight Authenticated Key Exchange"
keyword: Internet-Draft
coding: utf-8
venue:
  group: "Lightweight Authenticated Key Exchange"
  type: "Working Group"
  mail: "lake@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/lake/"
  github: "lake-wg/ra"
  latest: "https://lake-wg.github.io/ra/draft-ietf-lake-ra.html"

author:
 -
    fullname: Yuxuan Song
    ins: Y. Song
    organization: Inria
    email: "yuxuan.song@inria.fr"
 -
    fullname: Göran Selander
    ins: G. Selander
    organization: Ericsson AB
    email: "goran.selander@ericsson.com"
normative:
    I-D.ietf-rats-eat:
    RFC9528:
informative:
    RFC9334:
    RFC8949:
    RFC8392:
    RFC9393:
    IANA.CWT.Claims: IANA.cwt
    IANA-CoAP-Content-Formats:
      title: CoAP Content-Formats
      target: https://www.iana.org/assignments/core-parameters
    I-D.ietf-lake-authz:
    IANA-COSE-Header-Parameters:
      title: COSE Header Parameters
      target: https://www.iana.org/cose/header-parameters

--- abstract

This document specifies how to perform remote attestation as part of the lightweight authenticated Diffie-Hellman key exchange protocol EDHOC (Ephemeral Diffie-Hellman Over COSE), based on the Remote ATtestation procedureS (RATS) architecture.

--- middle

# Introduction

<!--Discuss remote attestation and mention some use cases.-->
Remote attestation is a security process which verifies and confirms the integrity and trustworthiness of a remote device or system in the network.
This process helps establish a level of trust in the remote system before allowing the device to e.g. join the network or get access to some sensitive information and resources.
The use cases that require remote attestation include secure boot and firmware management, cloud computing, network access control, etc.

<!--Summarize RATS architecture {{RFC9334}} and main roles.-->
The IETF working group Remote ATtestation procedureS (RATS) has defined an architecture {{RFC9334}} for remote attestation.
The three main roles in the RATS architecture are the Attester, the Verifier and the Relying Party.
The Attester generates the evidence concerning its identity and integrity, which must be appraised by the Verifier for its validity.
Then, the Verifier produces the attestation result, which is consequently used by the Relying Party for the purposes of reliably applying application-specific actions.

<!--Discuss the two RATS models and say that this specification supports both models.-->
One type of interaction model defined in the RATS architecture is called the background-check model.
It resembles the procedure of how employers perform background checks to determine the prospective employee's trustworthiness, by contacting the respective organization that issues a report.
In this case, the employer acts as the Relying Party, the employee acts as the Attester and the organization acts as the Verifier.
The Attester conveys evidence directly to the Relying Party and the Relying Party forwards the evidence to the Verifier for appraisal.
Once the attestation result is computed by the Verifier, it is sent back to the Relying Party to decide what action to take based on the attestation result.
Another model is called passport model, where the Attester communicates directly with the Verifier.
The Attester presents the evidence to the Verifier and gets an attestation result from the Verifier.
Then the Attester conveys the attestation result to the Relying Party.
This specification employs both the RATS background-check model and the passport model.

This document specifies the protocol between the Attester and the Relying Party.
The details of the protocol between the Relying Party and the Verifier in the background-check model, and the protocol between the Attester and the Verifier in the passport model are out of the scope.
This communication may be secured through protocols such as EDHOC, TLS or other security protocols that support the secure transmission to and from the Verifier.

<!--Discuss EAT-->
One way of conveying attestation evidence or the attestation result is the Entity Attestation Token (EAT) {{I-D.ietf-rats-eat}}.
It provides an attested claims set which can be used to determine a level of trustworthiness.
This specification relies on the EAT as the format for attestation evidence and the attestation result.

<!--Summarize EDHOC {{RFC9528}}. Mention EAD fields of EDHOC.-->
Ephemeral Diffie-Hellman over COSE (EDHOC) {{RFC9528}} is a lightweight authenticated key exchange protocol for highly constrained networks.
In EDHOC, the two parties involved in the key exchange are referred to as the Initiator (I) and the Responder (R).
EDHOC supports the transport of external authorization data, through the dedicated EAD fields.
This specification delivers EAT through EDHOC.
Specifically, EAT is transported as an EAD item.
This specification also defines new EAD items needed to perform remote attesation over EDHOC in {{ead-bg}} and {{ead-pp}}.

<!--Discuss implementation aspects such as the internal attestation service running on the Attester.
Root of trust. Separation between secure and non-secure worlds.-->
For the generation of evidence, the Attester incorporates an internal attestation service, including a specific trusted element known as the "root of trust".
Root of trust serves as the starting point for establishing and validating the trustworthiness appraisals of other components on the system.
The measurements signed by the attestation service are referred to as the Evidence.
The signing is requested through an attestation API.
How the components are separated between the secure and non-secure worlds on a device is out of scope of this specification.

# Conventions and Definitions
{::boilerplate bcp14-tagged}

The reader is assumed to be familiar with the terms and concepts defined in EDHOC {{RFC9528}} and RATS architecture {{RFC9334}}.

# Overview

This specification describes how to perform remote attestation over the EDHOC protocol, following the RATS architecture.
EDHOC provides the benefit of minimal message overhead and reduced round trips for a lightweight authentication.
More importantly, by integrating remote attestation with EDHOC, attestation can be run in parallel with authentication, improving the efficiency and maintaining lightweight properties.

Remote attestation protocol elements are carried within EDHOC's External Authorization Data (EAD) fields.
EDHOC {{RFC9528}} supports one or more EAD items in each EAD field.

In {{attestation-dimensions}}, this document defines three independent dimensions for performing remote attestation over EDHOC:

  1. Target (see {{targets}}) defining the entity that undergoes the attestation process (IoT device or network service).
  2. Model (see {{models}}) defining the attestation model in use based on the RATS architecture (background-check model or passport model).
  3. Message Flow (see {{flow}}) defining the EDHOC message flow in use (forward message flow or reverse message flow).

This document specifies the cases that are suited for constrained IoT environments.

# Assumptions

In the background-check model, one assumption is that the Verifier outputs a fresh nonce and that same nonce is passed on to the EDHOC session.
The Verifier is assumed to know how to verify at least one format of the evidence types provided by the Attester.
This specification assumes that the Relying Party also has knowledge about the Attester, so it can narrow down the evidence type selection and send to the Attester only one format of the evidence type.
The Attester should have an explicit relation with the Verifier, such as from device manufacturing, so that the Verifier can evaluate the Evidence that is produced by the Attester.

In the passport model, the credential identity of the Verifier is assumed to be stored at the Attester and the Relying Party, which means the Verifier is trusted by the Attester and the Relying Party to obtain the attestation result.
The use of timestamps for ensuring freshness in the passport model assumes synchronized time between the Attester and the Relying Party.
For detailed time considerations, refer to {{Appendix A of RFC9334}}.

EDITOR NOTE: add attestation public key stored in Verifier

# Remote Attestation in EDHOC {#attestation-dimensions}

This section specifies three independent dimensions that characterize the remote attestation process over EDHOC.

## Target {#targets}

Defines the entity that undergoes the attestation process.

### IoT Device Attestation (IoT)

The IoT device acts as the Attester.

### Network Service Attestation (Net)

The network service acts as the Attester.

This attestation pattern applies when constrained devices need to establish trust with a network gateway.
For example, before transmitting sensitive data to a network gateway, a constrained device requires attestation of the network gateway.

## Model {#models}

Defines the attestation models based on RATS architecture.
This specification supports both the RATS background-check model (see {{bg}}) and the passport model (see {{pp}}).
The corresponding EAD items for background-check model and the passport model are independent of each other.
The EAD items are specified in {{ead-bg}} for the background-check model and in {{ead-pp}} for the passport model.

### Background-check Model (BG) {#bg}

In the background-check model, the Attester sends the evidence to the Relying Party.
The Relying Party transfers the evidence to the Verifier and gets back the attestation result from the Verifier.

An EDHOC session is established between the Attester and the Relying Party.
A negotiation of the evidence type is required before the Attester sends the evidence.
All three parties must agree on a selected evidence type.

The Attester first sends a list of the proposed evidence types to the Relying Party.
The list is formatted as an Attestation proposal in an EDHOC EAD item.
The Relying Party relays the list to the Verifier and receives at least one supported evidence type from the Verifier in return.
If the Relying Party receives more than one evidence type, a single evidence type should be selected by the Relying Party based on its knowledge of the Attester.
The Relying Party then sends it back within the Attestation request to the Attester.

A nonce, at least 8-bytes long {{I-D.ietf-rats-eat}}), guarantees the freshness of the attestation session.
The nonce is generated by the Verifier and sent to the Relying Party.
The Relying Party puts the nonce and the selected evidence type together in a tuple to generate an Attestation request.

Once the Attester receives the Attestation request, it can call its attestation service to generate the evidence, with the nonce value as one of the inputs.

#### External Authorization Data (EAD) Items for background-check model {#ead-bg}

EAD items that are used for the background-check model are defined in this section.

##### Attestation_proposal {#attestation-proposal}

To propose a list of provided evidence types in background-check model, the Attester transports the Proposed_EvidenceType object.
It signals to the Relying Party the proposal to do remote attestation, as well as which types of the attestation claims the Attester supports.
The Proposed_EvidenceType is encoded in CBOR in the form of a sequence.

The EAD item for an attestation proposal is:

* ead_label = TBD1
* ead_value = Attestation_proposal, which is a CBOR byte string:

~~~~~~~~~~~~~~~~
Attestation_proposal = bstr .cbor Proposed_EvidenceType

Proposed_EvidenceType = [ + content-format ]

content-format = uint
~~~~~~~~~~~~~~~~

where

* Proposed_EvidenceType is an array that contains all the supported evidence types by the Attester.
* There MUST be at least one item in the array.
* content-format is an indicator of the format type (e.g., application/eat+cwt with an appropriate eat_profile parameter set), from {{IANA-CoAP-Content-Formats}}.

The sign of ead_label TBD1 MUST be negative to indicate that the EAD item is critical.
If the receiver cannot recognize the critical EAD item, or cannot process the information in the critical EAD item, then the receiver MUST send an EDHOC error message back as defined in {{Section 6 of RFC9528}}.

##### Attestation_request {#attestation-request}


As a response to the attestation proposal, the Relying Party signals to the Attester the supported and requested evidence type.
In case none of the evidence types is supported, the Relying Party rejects the first message_1 with an error indicating support for another evidence type.

The EAD item for an attestation request is:

* ead_label = TBD1
* ead_value = Attestation_request, which is a CBOR byte string:

~~~~~~~~~~~~~~~~
Attestation_request = bstr .cbor Selected_EvidenceType

Selected_EvidenceType = (
  content-format: uint,
  nonce: bstr .size 8..64
)
~~~~~~~~~~~~~~~~

where

* content-format is the selected evidence type by the Relying Party and supported by the Verifier.
* nonce is generated by the Verifier and forwarded by the Relying Party.

The sign of ead_label TBD2 MUST be negative to indicate that the EAD item is critical.
If the receiver cannot recognize the critical EAD item, or cannot process the information in the critical EAD item, then the receiver MUST send an EDHOC error message back as defined in {{Section 6 of RFC9528}}.

##### Evidence {#evidence}

The Attester calls its local attestation service to generate and return a serialized Entity Attestation Token (EAT) {{I-D.ietf-rats-eat}} as Evidence.
The Evidence is sent in EAD_3 within EDHOC message_3.

The EAD item is:

* ead_label = TBD1
* ead_value is a serialized EAT.

For remote attestation over EDHOC, The EAT MUST be formatted as a CBOR Web Token (CWT) containing attestation-oriented claims.
The complete set of attestation claims for the EAT is specified in {{I-D.ietf-rats-eat}}.

A minimal claims set MAY be implemented when the Attester operates under constrained message size requirements and/or limited computational resources.

~~~~~~~~~~~~~~~~
{
/eat-nonce/          10: bstr .size 8
/ueid/               256: bstr .size (7..33)
/measurements/       273: measurements-type
}
~~~~~~~~~~~~~~~~

The measurements claim MAY be formatted according to {{RFC9393}}.
When CoSWID {{RFC9393}} is used, the claim MUST be an evidence CoSWID rather than a payload CoSWID.
Formats other than CoSWID are permitted and MUST be identified by CoAP Content Format.


~~~~~~~~~~~~~~~~
measurements-type = [+ measurements-format]

measurements-format = [
      content-type: coap-content-format,
      content-format: bytes .cbor measurements-body-cbor
]
~~~~~~~~~~~~~~~~

##### trigger_bg {#trigger-bg}

The EAD item trigger_bg is used when the sender triggers the receiver to start a remote attestation in the background-check model.
The receiver MUST reply with an EAD item corresponding to the background-check model.
The ead_value MUST be empty, as the ead_label serves as the trigger.

The EAD item is:

* ead_label = TBD2
* ead_value = null

### Passport Model (PP) {#pp}

In the passport model, the Attester sends the evidence to the Verifier.
After the Attester receives the attestation result from the Verifier, the Attester sends the attestation result to the Relying Party.

An EDHOC session is established between the Attester and the Relying Party.
The Attester and the Relying Party should decide from which Verifier the Attester obtains the attestation result and transfers it to the Relying Party.
The Attester first sends a list of the Verifier identities that it can get the attestation result.
The Relying Party selects one trusted Verifier identity and sends it back as a Result request.

Regarding the freshness in passport model, the Attester could either establish a real-time connection with the selected Verifier, or use a pre-stored attestation result from the selected Verifier.
If the attestation result is not obtained via a real-time connection, it should include a time stamp and/or expiry time to indicate its validity.
Time synchronization is out of scope of this specification.

Once the Attester obtains the attestation result from the selected Verifier, it sends the attestation result to the Relying Party.

#### External Authorization Data (EAD) Items for passport model {#ead-pp}

EAD items that are used for the passport model are defined in this section.

##### Result_proposal {#result-proposal}

An attestation result proposal contains the identification of the credentials of the Verifiers to indicate Verifiers' indentities.
The identification of credentials relies on COSE header parameters {{IANA-COSE-Header-Parameters}}, with a header label and credential value.

The EAD item for the attestation result proposal is:

* ead_label = TBD3
* ead_value = Result_proposal, which is a CBOR byte string:

~~~~~~~~~~~~~~~~
Result_proposal = bstr .cbor Proposed_VerifierIdentity
Proposed_VerifierIdentity = [ + VerifierIdentity ]

VerifierIdentity = {
  label => values
}
~~~~~~~~~~~~~~~~

where

* Proposed_VerifierIdentity is defined as a list of one or more VerifierIdentity elements.
* Each VerifierIdentity within the list is a map defined in {{IANA-COSE-Header-Parameters}} that:
    * label = int / tstr
    * values = any


##### Result_request {#result-request}

As a response to the attestation result proposal, the Relying Party signals to the Attester the trusted Verifier.
In case none of the Verifiers can be trusted by the Relying Party, the session is aborted.
Relying Party generates a nonce to ensure the freshness of the attestation result from the Verifier.

The EAD item for an attestation result request is:

* ead_label = TBD3
* ead_value = Result_request, which is a CBOR byte string:

~~~~~~~~~~~~~~~~
Result_request = bstr .cbor Request_structure

Request_structure = {
  selected_verifier: VerfierIdentity
}
~~~~~~~~~~~~~~~~

##### Result {#result}

The attestation result is generated and signed by the Verifier as a serialized EAT {{I-D.ietf-rats-eat}}.
The Relying Party can decide what action to take with regards to the Attester based on the information elements in attetation result.

The EAD item is:

* ead_label = TBD3
* ead_value is a serialized EAT.

##### trigger_pp {#trigger-pp}

The EAD item trigger_pp is used when the sender triggers the receiver to start a remote attestation in the passport model.
The receiver MUST reply with an EAD item correspondign to the passport model.
The ead_value MUST be empty, as the ead_label serves as the trigger.

The EAD item is:

* ead_label = TBD4
* ead_value = null

## EDHOC Message Flow {#flow}

The EDHOC protocol defines two possible message flows, namely the EDHOC forward message flow and the EDHOC reverse message flow (see {{Appendix A.2.2 of RFC9528}}).
In this specification, both flows can be used to perform remote attestation.

### EDHOC Forward Message Flow (Fwd)

In forward message flow, EDHOC may run with the Initiator as a CoAP/HTTP client.
Remote attestation over EDHOC starts with a POST requests to the Uri-Path: "/.well-known/lake-ra".

### EDHOC Reverse Message Flow (Rev)

In the reverse message flow, the CoAP/HTTP client is the Responder and the server is the Initiator.
A new EDHOC session begins with an empty POST request to the server's resource for EDHOC.

# Instantiation of Remote Attestation Protocol {#attestation-combinations}

We use the format (Target, Model, Message Flow) to denote instantiations.
For example, (IoT, BG, Fwd) represents IoT device attestation using the background-check model with forward EDHOC message flow.

Although there are 8 cases (IoT/Net, BG/PP, Fwd/Rev), this document specifies the most relevant instantiations for constrained IoT environments.

## (IoT, BG, Fwd): IoT Device Attestation {#iot-attesation}

A common use case for (IoT, BG, Fwd) is to attest an IoT device to a network server.
For example, a simple illustratice case is doing remote attestation to verify that the latest version of firmware is running on the IoT device before the network server allows it to join the network (see {{firmware}}).

An overview of doing IoT device attestation in background-check model and EDHOC forward message flow is established in {{fig-iot-bg-fwd}}.
EDHOC Initiator plays the role of the RATS Attester (A).
EDHOC Responder plays the role of the RATS Relying Party (RP).
The Attester and the Relying Party communicate by transporting messages within EDHOC's External Authorization Data (EAD) fields.
An external entity, out of scope of this specification, plays the role of the RATS Verifier (V).
The EAD items specific to the background-check model are defined in {{ead-bg}}.

The Attester starts the attestation by sending an Attestation proposal in EDHOC message_1.
The Relying Party generates EAD_2 with the received evidence type and nonce from the Verifier, and sends it to the Attester.
The Attester sends the Evidence to the Relying Party in EAD_3.
The Verifier evaluates the Evidence and sends the Attestation result to the Relying Party.

~~~~~~~~~~~ aasvg
+-----------------+              +-----------------+
|   IoT device    |              | Network service |
+-----------------+              +-----------------+
| EDHOC Initiator |              | EDHOC Responder |
+-----------------+              +-----------------+            +----------+
|     Attester    |              |  Relying Party  |            | Verifier |
+--------+--------+              +--------+--------+            +-----+----+
         |                                |                           |
         |  EAD_1 = Attestation proposal  |                           |
         +------------------------------->|                           |
         |                                |                           |
         |                                | Attestation proposal, C_R |
         |                                +-------------------------->|
         |                                |<--------------------------+
         |                                |  EvidenceType(s), Nonce   |
         |                                |                           |
         |  EAD_2 = Attestation request   |                           |
         |<-------------------------------+                           |
         |                                |                           |
         |  EAD_3 = Evidence              |                           |
         +------------------------------->|                           |
         |                                |      Evidence, C_R        |
         |                                +-------------------------->|
         |                                |<--------------------------+
         |                                |    Attestation result     |
         |                                |                           |
         |                                |                           |
         |                                |                           |
         | <----------------------------> |                           |
         |         EDHOC session          |                           |

~~~~~~~~~~~
{: #fig-iot-bg-fwd title="Overview of IoT device attestation in background-check model and EDHOC forward message flow. EDHOC is used between A and RP." artwork-align="center"}

This specification also supports a simplified attestation model for scenarios where a pre-existing relationship exists between the Attester and the Verifier.
In such cases, when the Verifier possesses prior knowledge of the evidence format (CBOR) and its expected profile of the EAT claims set (e.g., CoSWID), the evidence types negotiation MAY be omitted.
Under these circumstances, the attestation process is simplified as:

~~~~~~~~~~~ aasvg
+-----------------+              +-----------------+
|   IoT device    |              | Network service |
+-----------------+              +-----------------+
| EDHOC Initiator |              | EDHOC Responder |
+-----------------+              +-----------------+            +----------+
|     Attester    |              |  Relying Party  |            | Verifier |
+--------+--------+              +--------+--------+            +-----+----+
         |                                |                           |
         |  EAD_1 = trigger_bg            |                           |
         +------------------------------->|                           |
         |                                |                           |
         |                                |           C_R             |
         |                                +-------------------------->|
         |                                |<--------------------------+
         |                                |          Nonce            |
         |                                |                           |
         |  EAD_2 = Nonce                 |                           |
         |<-------------------------------+                           |
         |                                |                           |
         |  EAD_3 = Evidence              |                           |
         +------------------------------->|                           |
         |                                |      Evidence, C_R        |
         |                                +-------------------------->|
         |                                |<--------------------------+
         |                                |    Attestation result     |
         |                                |                           |
         |                                |                           |
         |                                |                           |
         | <----------------------------> |                           |
         |         EDHOC session          |                           |

~~~~~~~~~~~
{: #fig-iot-bg-fwd-s title="Simplified IoT device attestation in background-check model with EDHOC forward message flow. Pre-existing relationship between A and V." artwork-align="center"}


## (Net, PP, Fwd): Network Service Attestation {#network-attestation}

One use case for (Net, PP, Fwd) is when a network server needs to attest itself to a client (e.g., an IoT device).
For example, the client needs to send some sensitive data to the network server, which requires the network server to be attested first.
In (Net, PP, Fwd), the network server acts as an Attester and the client acts as a Relying Party.

An overview of the message flow is illustrated in {{fig-net-pp-fwd}}.
EDHOC Initiator plays the role of the RATS Relying Party.
EDHOC Responder plays the role of the RATS Attester.
An external entity, out of scope of this specification, plays the role of the RATS Verifier.
The EAD items specific to the passport model are defined in {{ead-pp}}.

The Relying Party asks the Attester to do a remote attestation by sending a trigger_pp (see {{trigger-pp}}) in EDHOC message_1.
The Attester replies to the Relying Party with a Result proposal in EAD_2.
Then the Relying Party selects a trusted Verifier identity and sends it as a Result request.
How the Attester negotiates with the selected Verifier to get the attestation result is out of scope of this specification.
A fourth EDHOC message is required to send the Result from the Attester to the Relying Party.


~~~~~~~~~~~ aasvg
+-----------------+          +-----------------+
|   IoT device    |          | Network service |
+-----------------+          +-----------------+
| EDHOC Initiator |          | EDHOC Responder |
+-----------------+          +-----------------+            +----------+
|  Relying Party  |          |     Attester    |            | Verifier |
+--------+--------+          +--------+--------+            +-----+----+
         |                            |                           |
         |  EAD_1 = trigger_PP        |                           |
         +--------------------------->|                           |
         |                            |                           |
         |  EAD_2 = Result proposal   |                           |
         |<---------------------------+                           |
         |                            |                           |
         |  EAD_3 = Result request    |                           |
         +--------------------------->|        (request)          |
         |                            +---  ---  ---  ---  --- -->|
         |                            |<---  ---  ---  --- ---  --+
         |                            |         Result            |
         |  EAD_4 = Result            |                           |
         |<---------------------------+                           |
         |                            |                           |
         |                            |                           |
         |                            |                           |
         | <------------------------> |                           |
         |       EDHOC session        |                           |
~~~~~~~~~~~
{: #fig-net-pp-fwd title="Overview of network service attestation in passport model and EDHOC forward message flow. EDHOC is used between RP and A. The dashed line illustrates a logical connection that does not need to occur in real time." artwork-align="center"}

# Mutual Attestation in EDHOC {#mutual-attestation}

Mutual attestation over EDHOC combines the cases where one of the EDHOC parties uses the IoT device attestation and the other uses the Network service attestation.
Performing mutual attestation to a single EDHOC message flow acheives a lightweight use with reduced message overhead.
Note that the message flow for the two parties in mutual attestation needs to be the same.

In this specification, we list the most relevant mutual attestation example for constrained IoT environments.

## (IoT, BG, Fwd) - (Net, PP, Fwd)

In this example, the mutual attestation is performed in EDHOC forward message flow, by one IoT device attestation in background-check model and another network service attestation in passport model.
The process is illustrated in {{fig-mutual-attestation-BP}}.
How the Network service connects with the Verifier_1 and potential Verifier_2 is out of scope in this specification.

The first remote attestation is initiated by the IoT device (A_1) in background-check model.
In parallel, the IoT device (A_1) requests the network service (A_2) to perform a remote attestation in passport model.
EAD_2 carries the EAD items Attestation request and Result proposal.
EAD_3 carries the EAD items Evidence and Result request.
EAD_4 carries the EAD item Result for the passport model.

~~~~~~~~~~~ aasvg
+-----------------+               +-----------------+
|   IoT device    |               | Network service |
+-----------------+               +-----------------+
| EDHOC Initiator |               | EDHOC Responder |
+-----------------+               +-----------------+            +------------+              +------------+
|    A_1/RP_2     |               |    RP_1/A_2     |            | Verifier_1 |              | Verifier_2 |
+--------+--------+               +--------+--------+            +------+-----+              +------+-----+
         |                                 |                            |                           |
         |  EAD_1 = Attestation proposal,  |                            |                           |
         |          trigger_PP             |                            |                           |
         +-------------------------------->|                            |                           |
         |                                 |                            |                           |
         |                                 |                            |                           |
         |                                 |  Attestation proposal, C_R |                           |
         |                                 +--------------------------->|                           |
         |                                 |<---------------------------+                           |
         |                                 |   EvidenceType(s), Nonce   |                           |
         |  EAD_2 = Attestation request,   |                            |                           |
         |          Result proposal        |                            |                           |
         |<--------------------------------+                            |                           |
         |                                 |                            |                           |
         |                                 |                            |                           |
         |  EAD_3 = Evidence,              |                            |                           |
         |          Result request         |                            |                           |
         +-------------------------------->|                            |                           |
         |                                 |                            |                           |
         |                                 |       Evidence, C_R        |                           |
         |                                 +--------------------------->|         (Request)         |
         |                                 +--- --- --- --- --- ---  ---+ ---  ---  --- --- --- --->|
         |                                 |                            |                           |
         |                                 |                            |                           |
         |                                 |    Attestation result      |                           |
         |                                 |<---------------------------+                           |
         |                                 |                            |                           |
         |                                 |<---  ---  ---  ---  ---  --+ ---  ---  --- ---  --- ---+
         |                                 |            Result          |                           |
         |  EAD_4 = Result                 |                            |                           |
         |<--------------------------------+                            |                           |
         |                                 |                            |                           |
         |                                 |                            |                           |
         |                                 |                            |                           |
         |                                 |                            |                           |
         | <-----------------------------> |                            |                           |
         |          EDHOC session          |                            |                           |

~~~~~~~~~~~
{: #fig-mutual-attestation-BP title="Overview of mutual attestation of (IoT, BG, Fwd) - (Net, PP, Fwd). EDHOC is used between A and RP. The dashed line illustrates a logical connection that does not need to occur in real time." artwork-align="center"}


# Error Handling {#error-handling}

This section specifies a new EDHOC error code and how it is used in the proposed protocol.

## EDHOC Error "Attestation failed"

This section specifies a new EDHOC error "Attestation failed".
The format of the error message follows the one in EDHOC protocol(see {{Section 6 of RFC9528}}).

~~~~~~~~~~~ aasvg
+----------+----------------+----------------------------------------+
| ERR_CODE | ERR_INFO Type  | Description                            |
+==========+================+========================================+
|     TBD5 | attestation    | Attestation failed                     |
+----------+----------------+----------------------------------------+
~~~~~~~~~~~
{: #fig-error title="EDHOC error code and error information for Attestation failed."}

Error code TBD5 indicates to the receiver that the remote attestation is failed after the evidence is sent.
This can occur in two cases:

1. The Verifier evaluates the attestation evidence and returns a negative result based on the Verifier's appraisal policy.

2. The Verifier provides a positive attestation result to the Relying Party, but the Relying Party can not establish a sufficient level of trust to proceed decision-specific actions based on its appraisal policy.

In case 1, the Verifier signals the error to the Relying Party, which then generates an EDHOC "Attestation failed" error and send it to the Attester.
In case 2, the Relying Party directly generates and sends the "Attestation failed" error to the Attester.
The application decides how to handle the error message.

# Security Considerations {#security-considerations}

This specification is performed over EDHOC {{RFC9528}} by using EDHOC's EAD fields.
The privacy considerations of EADs in EDHOC apply to this specification.

EAD_1 is not resistant to either active attackers or passive attackers, because neither the Initiator nor the Responder has been authenticated.

Although EAD_2 is encrypted, the Initiator has not been authenticated, rendering EAD_2 vulnerable against the active attackers.

The ead items in EAD_1 and EAD_2 MAY be very specific and potentially reveal sensitive information about the device.
The leaking of the data in EAD_1 and/or EAD_2 MAY risk to be used by the attackers for malicious purposes.
Data in EAD_3 and EAD_4 are protected between the Initiator and the Responder in EDHOC.

Mutual attestation carries a lower risk for EAD items when the Responder is the Attester.
For the mutual attestation at the EDHOC Responder, only the Attestation_proposal/Result_proposal in EAD_2 is not protected to active attackers.
Both the Attestation_request/Result_request in EAD_3 and the Evidence/Result in EAD_4 are protected.

# IANA Considerations

## EDHOC External Authorization Data Registry

IANA is requested to register the following entry in the "EDHOC External Authorization Data" registry under the group name "Ephemeral Diffie-Hellman Over Cose (EDHOC)".

* The ead_label = TBD1 corresponds to the ead_value Attestation_proposal in {{attestation-proposal}}, Attestation_request in {{attestation-request}} and Evidence in {{evidence}}.
* The ead_label = TBD2 corresponds to the EAT item trigger_bg as specified in {{trigger-bg}}.
* The ead_label = TBD3 corresponds to the ead_value Result_proposal in {{result-proposal}}, Result_request in {{result-request}} and the Result in {{result}}.
* The ead_label = TBD4 corresponds to the EAT item trigger_pp as specified in {{trigger-pp}}.

~~~~~~~~~~~ aasvg
+-----------+-------+------------------------+-------------------+
| Name      | Label | Description            | Reference         |
+===========+=======+========================+===================+
| TBD       | TBD1  | BG model               |                   |
|           |       | related information    | Section 5.2.1.1   |
+-----------+-------+------------------------+-------------------+
| TBD       | TBD2  | trigger to start       |                   |
|           |       | attestation in BG      | Section 5.2.1.1   |
+-----------+-------+------------------------+-------------------+
| TBD       | TBD3  | PP model               |                   |
|           |       | related information    | Section 5.2.2.1   |
+-----------+-------+------------------------+-------------------+
| TBD       | TBD4  | trigger to start       |                   |
|           |       | attestation in PP      | Section 5.2.2.1   |
+-----------+-------+------------------------+-------------------+
~~~~~~~~~~~
{: #fig-ead-labels title="EAD labels."}

--- back

# Example: Remote Attestation Flow

~~~~aasvg

.--------------------------.
|     EDHOC Initiator      |       .-------------------.
+--------------------------+       |  EDHOC Responder  |
| Attestation   | Attester |       +-------------------+   .----------.
| Service       |          |       |  Relying Party    |   | Verifier |
'--+----------------+------'       '---------+---------'   '-----+----'
   |                |                        |                   |
   |                |                        |                   |
   |                |EDHOC message_1         |                   |
   |                |  {...}                 |                   |
   |                |  EAD_1(                |                   |
   |                |    types(a,b,c)        |                   |
   |                |  )                     |                   |
   |                +----------------------->|                   |
   |                |                        |                   |
   |                |                        | types(a,b,c), C_R |
   |                |                        +------------------>|
   |                |                        |                   |
   |                |                        |                   |
   |                |                        | Body: {           |
   |                |                        |   nonce,          |
   |                | EDHOC message_2        |   types(a,b)      |
   |                |  {...}                 | }                 |
   |                |  EAD_2(                |<------------------+
   |                |    nonce,              |                   |
   |                |    type(a)             |                   |
   |                |  )                     |                   |
   |                |  Auth_CRED(Sig/MAC)    |                   |
   |                |<-----------------------+                   |
   |   Body:{       |                        |                   |
   |    nonce,      |                        |                   |
   |    type(a)     |                        |                   |
   |   }            |                        |                   |
   |<---------------+                        |                   |
   | Body:{         |                        |                   |
   |   Evidence     |                        |                   |
   | }              |                        |                   |
   +--------------->|                        |                   |
   |                | EDHOC message_3        |                   |
   |                |  {...}                 |                   |
   |                |  Evidence(EAT)         |                   |
   |                |  Auth_CRED(sig/MAC)    |                   |
   |                +----------------------->|                   |
   |                |                        |                   |
   |                |                        |                   |
   |                |                        |                   |
   |                |                        | Body: {           |
   |                |                        |  EAT, C_R}        |
   |                |                        +------------------>|
   |                |                        | Body: {           |
   |                |                        |  att-result: AR{} |
   |                |                        | }                 |
   |                |                        |<------------------+
   |                |                        +---.               |
   |                |                        |    | verify AR{}  |
   |                |                        |<--'               |
   |                |                        |                   |
   |                |    application data    |                   |
   |                |<---------------------->|                   |
   |                |                        |                   |
~~~~
{: #figure-iot-example title="Example of remote attestation."}

# Remote attestation in parallel with enrollment authorization

This section discusses the possibility of doing remote attestation in parallel with the enrollment authorization procedure defined in {{I-D.ietf-lake-authz}}.
In this case, the message count is much decreased.

The detailed procedure is TBD.

# Example: Firmware Version {#firmware}

The goal in this example is to verify that the firmware running on the device is the latest version, and is neither tampered or compromised.
A device acts as the Attester, currently in an untrusted state.
The Attester needs to generate the evidence to attest itself.
A gateway that can communicate with the Attester and can control its access to the network acts as the Relying Party.
The gateway will finally decide whether the device can join the network or not depending on the attestation result.
The attestation result is produced by the Verifier, which is a web server that can be seen as the manufacturer of the device.
Therefore it can appraise the evidence that is sent by the Attester.
The remote attestation session starts with the Attester sending EAD_1 in EDHOC message 1.

An example of the EAD_1 in EDHOC message_1 could be:

~~~~~~~~~~~~~~~~
[60,61,258]
~~~~~~~~~~~~~~~~

If the Verifier and the Relying Party can support at least one evidence type that is proposed by the Attester, the Relying Party will include in the EAD_2 field the same evidence type, alongside a nonce for message freshness.

~~~~~~~~~~~~~~~~
(258, h'a29f62a4c6cdaae5')
~~~~~~~~~~~~~~~~

The Evidence in EAD_3 field is an Entity Attestation Token (EAT) {{I-D.ietf-rats-eat}}, with the measurements claim formatted in CoSWID{{RFC9393}}.
The components of the Evidence should at least be:

~~~~~~~~~~~~~~~~
{
/eat-nonce/                 10: h'a29f62a4c6cdaae5',
/ueid/                      256: 'aaabbcc',
/measurements/              273: [
  /CoAP Content-Format ID/        [ 258,
  /evidence in CoSWID/              {
                                      0: 'tagID'             /tag-id/
                                      12: 0                  /tag-version/
                                      1: "DotBot firmware"   /software-name/
                                      2: {                   /entity/
                                          31: "Attester"     /entity-name/
                                          33: 1              /role, must be "tag-creator" which is 1/
                                          },
                                      3: {                   /evidence/
                                          17: [              /file/
                                               {
                                                24: "partition0-nrf52840dk.bin",   /fs-name/
                                                7: [                               /hash of file/
                                                    1,                             /alg SHA-256/
                                                    h'06294f6806b9c685eea795048579cfd02a0c025bc8b5abca42a19ea0ec23e81a'
                                                    ]                              /hash value/
                                                }
                                               ]
                                          }
                                    }
                                  ]
                                 ]
}
~~~~~~~~~~~~~~~~

The infomation above serves as the payload of the COSE object.
The complete resulting COSE object is:

~~~~~~~~~~~~~~~~~
18([
    /*protected header*/
    h'a10127',

    /*unprotected header*/
    {},

    /*payload*/
    h'A30A48A29F62A4C6CDAAE519010047616161626263631901118182190102A50045746
    16749440C00016F446F74426F74206669726D7761726502A2181F684174746573746572
    18210103A11181A218187819706172746974696F6E302D6E72663532383430646B2E626
    96E078201582006294F6806B9C685EEA795048579CFD02A0C025BC8B5ABCA42A19EA0EC
    23E81A',

    /*signature*/
    h'd4100901f4c3e51312c3110c6ddc8dcf7f68d8f5d3791c19133f2f0ac158c1f5ee6ed
    afe9d7c3d6eb3d2d197f82e733d375fdda9fb258b304961dfc38558950d'
])
~~~~~~~~~~~~~~~~~

which has the following base16 encoding:



~~~~~~~~~~~~~~~~~
D28443A10127A05890A30A48A29F62A4C6CDAAE51901004761616162626363190111818
2190102A5004574616749440C00016F446F74426F74206669726D7761726502A2181F68
417474657374657218210103A11181A218187819706172746974696F6E302D6E7266353
2383430646B2E62696E078201582006294F6806B9C685EEA795048579CFD02A0C025BC8
B5ABCA42A19EA0EC23E81A5840D4100901F4C3E51312C3110C6DDC8DCF7F68D8F5D3791
C19133F2F0AC158C1F5EE6EDAFE9D7C3D6EB3D2D197F82E733D375FDDA9FB258B304961
DFC38558950D
~~~~~~~~~~~~~~~~~

The Relying Party (co-located with the gateway) then treats the Evidence as opaque and sends it to the Verifier.
Once the Verifier sends back the Attestation Result, the Relying Party can be assured on the version of the firmware that the device is running.

# Open discussion: remote attestation over EDHOC/ over OSCORE

TBD

# Acknowledgments
{:numbered="false"}

The author would like to thank Thomas Fossati, Goran Selander, Malisa Vucinic, Ionut Mihalcea, Muhammad Usama Sardar, Michael Richardson and Geovane Fedrecheski for the provided ideas and feedback.
