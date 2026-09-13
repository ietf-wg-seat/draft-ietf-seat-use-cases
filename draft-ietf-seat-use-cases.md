---
title: "Security Goals and Use Cases for Integrating Remote Attestation with Secure Channel Protocols"
abbrev: "SEAT Use Cases"
category: info

docname: draft-ietf-seat-use-cases-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: Security
workgroup: SEAT Working Group
keyword:
  - remote attestation
  - TLS
  - confidential computing
  - IoT
  - RATS
venue:
group: SEAT
type: Working Group
mail: seat@ietf.org

author:
  - fullname: Ionuț Mihalcea
    organization: Arm
    email: ionut.mihalcea@arm.com
    role: editor

normative:

informative:
    RFC9334: rats-arch
    RFC9397: teep-arch
    RFC4949:
    RFC3552: int-threat-model
    RFC9846: tls
    I-D.draft-ccc-wimse-twi-extensions: wimse-twi
    I-D.draft-ietf-rats-eat-measured-component: rats-measured
    ID-Crisis:
      title: "Identity Crisis in Confidential Computing: Formal Analysis of Attested TLS"
      date: November 2025,
      target: https://www.researchgate.net/publication/398839141_Identity_Crisis_in_Confidential_Computing_Formal_Analysis_of_Attested_TLS
      author:
        - ins: M. U. Sardar
        - ins: M. Moustafa
        - ins: T. Aura
    AI-agents:
     title: "AI agents that matter"
     date: 1 July 2024,
     target: https://arxiv.org/abs/2407.01502
     author:
     - ins: S. Kapoor
     - ins: B. Stroebl
     - ins: Z. S. Siegel
     - ins: N. Nadgir
     - ins: A. Narayanan
    MigTD:
     title: Intel TDX Migration TD
     target: https://github.com/intel/MigTD
    I-D.ietf-tls-rfc8446bis:
    I-D.ietf-tls-rfc9147bis:
    I-D.ietf-tls-extended-key-update:
    CVE-2026-33697:
     title: CVE-2026-33697
     target: https://www.cve.org/CVERecord?id=CVE-2026-33697
    I-D.aylward-aiga-2:
    I-D.draft-ietf-rats-pkix-key-attestation:
    I-D.jiang-seat-dynamic-attestation:

--- abstract

This document outlines desirable security goals and use cases for integrating remote
attestation (RA) capabilities with secure channel establishment protocols (e.g., TLS and DTLS).
Peer authentication in such protocols establishes trust in a peer's network identifiers but
provides no assurance regarding the integrity of its underlying software and
hardware stack. Remote attestation addresses this gap by enabling a peer to
provide verifiable evidence about the current state of the Target Environment. This document specifies a set of essential
security goals the protocol solution must have, including cryptographic binding to
the secure connection, evidence freshness, and flexibility to support different
attestation models. It then explores relevant use cases, such as confidential
data collaboration and secure secrets provisioning, to motivate the
need for this integration.  This document is intended
to serve as an input to the design
of protocol solutions within the SEAT working group.

--- middle

# Introduction

## Establishing Trust in Secure Communications

Secure channel protocols, such as Transport Layer Security (TLS),
primarily establish trust in a peer's identity. This is typically achieved
through mechanisms like a Public Key Infrastructure (PKI), where a trusted
Certification Authority (CA) vouches for the binding between a public key and an
identifier (e.g., a hostname).

However, this model has one key limitation: entity authentication provides no assurance about the peer's state, such as the integrity of its software stack at boot time and during runtime.
A compromised endpoint, for instance, can still present a valid X.509
certificate and be considered "trusted" by a client. This gap allows compromised
endpoints to maintain network access and the trust of their peers, posing a
significant security risk in many environments.

## The Role of Remote Attestation

Remote Attestation (RA), as described in the RATS architecture {{RFC9334}}, is a
mechanism designed to fill this gap. RA allows an entity (the "Attester") to
produce verifiable "Evidence" about its current runtime state. This Evidence covers the Attester's TCB and can thus include measurements of:

- firmware
- operating system
- application code
- the configuration of its hardware and software security features (e.g., secure boot status and memory
isolation).

A "Relying Party" can then use this Evidence, often with the help of
a trusted "Verifier", to appraise the Attester's trustworthiness.

Composing RA with a secure channel establishment protocol adds a second
dimension of trust - trustworthiness - to complement peer
authentication. This allows a peer to make authorization decisions based not
just on who the other party is, but also on what it is (e.g., an AMD
SEV-SNP-based server running in some known datacenter) and whether its state is
acceptable.

## Purpose and Scope

The purpose of this document is to establish a set of essential security goals
for composition of RA with secure channel protocols and to outline the key use
cases that can benefit from such a composition. Most of the use cases presented in this document are provided by industry contributors in the SEAT WG, who have plans to deploy this technology. The initial focus is on
TLS 1.3  {{I-D.ietf-tls-rfc8446bis}}
and its datagram-oriented variant, DTLS 1.3 {{I-D.ietf-tls-rfc9147bis}}.

This document is intended as an input to the design of protocol solutions within
the SEAT working group. It defines the "why" (the motivation) and the "what" (the requirements),
but not the "how" (the protocol design itself). The "how" part is out of scope of this document. A key goal of this
document is to define
requirements for a solution that is agnostic to any specific attestation
technology (e.g., Trusted Platform Modules (TPMs), Intel TDX, AMD SEV, Arm CCA).

Appraisal policies (cf. {{Section 8.5 of -rats-arch}}) are out of scope of this document.

# Terminology

This document uses the terminology defined in the RATS Architecture {{RFC9334}},
including "Attester", "Relying Party", "Verifier", "Evidence", and "Attestation
Results".

This document also uses the following terms:

* Trusted Computing Base (TCB) of a device: see {{RFC4949}}. Note that for this
draft, it includes respective configurations of hardware, firmware, and software.
* Confidential Workload: as defined in {{-wimse-twi}}.
* Measurements: as defined in {{-rats-measured}}.
* AI agent: An AI agent is a software principal (typically long-running) that performs
closed-loop "perceive -> plan -> act" cycles using an LLM or other model,
and invokes external tools/APIs that may read sensitive data or change
system/network state. Its configuration (e.g., model choice, tool enablement,
prompt template) can change independently of the binary/image and usually
more frequently than typical platform TCB updates {{AI-agents}}.

# Attacker Model
{: #attacker-model }

This section defines the attacker capabilities and attack scenarios that a
solution integrating RA with a secure channel needs to consider. Attacks
in-scope for IETF protocols generally assume the Internet Threat Model of
{{-int-threat-model}}. This corresponds to the "network attacker" described
below. Some scenarios explicitly grant the attacker capabilities beyond that
baseline.

Unless stated otherwise, the Relying Party is not compromised and correctly
performs the checks required by the protocol and its appraisal policy. The
Attester is referenced in terms of its Attesting and Target Environments, as
described in {{Section 3.1 of -rats-arch}}. The Attesting Environment is the
entity which holds the Attestation key, which is used to sign Evidence for the
Target Environment. Part or all of the Target Environment may be executing
within a protected environment that provides certain security guarantees,
called a Trusted Execution Environment (TEE) (see {{-teep-arch}}).

The baseline security assumptions of the TLS v1.3 protocol apply. See
{{Appendix F of -tls}} for the relevant properties.

The TLS stack of the attester is assumed to be running within its Target
Environment.

## Attacker Capabilities

The attacks in this section use the following attacker profiles:

* **Network attacker:** Controls communication between the peers and can
  observe, inject, modify, drop, delay, reorder, and replay messages. This
  attacker does not control either endpoint or possess their cryptographic
  secrets. This closely follows the attacker model of {{-int-threat-model}}.

* **Malicious peer:** Controls a TLS endpoint and can deviate arbitrarily from
  the protocol. The attacker cannot generate Evidence which would be accepted by
  the Relying Party's appraisal policy. This can be because the endpoint is not
  running in a Target Environment, or because the Evidence for that Target
  Environment does not meet the requirements of the appraisal policy. The
  attacker does not control or compromise an Attesting Environment or its
  Attestation Key.

* **Ephemeral-key attacker:** Possesses an endpoint's ephemeral private key used
  for key establishment in a particular secure-channel handshake. Given the
  peer's public key share and the handshake transcript, this attacker can derive
  the connection's handshake and application traffic secrets when the key
  schedule does not also depend on independent secret keying material unknown to
  the attacker, such as a PSK. This capability does not, by itself, imply
  possession of an authentication key or Attestation Key.

* **Traffic-secret attacker:** Possesses either the handshake and/or the
  application traffic secrets for a secure connection. Each attack specifies
  which direction and epoch are relevant. Possession of a traffic secret does not,
  by itself, imply possession of an authentication key (see below) or Attestation
  Key.

* **Authentication-key attacker:** Possesses the private key used by a peer to
  authenticate itself in a secure connection (e.g., to sign the CertificateVerify
  message in the TLS 1.3 handshake).

* **Attestation-key attacker:** Can use an Attestation Key whose credential the
  Relying Party continues to accept.

* **Target-Environment attacker:** Can change the state of a Target Environment
  after Evidence about that environment has been generated. Post-attestation
  state changes can also occur through legitimate reconfiguration and therefore
  do not always imply an attack.

## Structure of Attack Descriptions

Each attack or failure mode below is described using the following fields:

* **Attacker capabilities:** Identifies the attacker profile or combination of
  profiles required by the scenario and states any additional capabilities.

* **Targeted assurance:** Identifies the security claim or property that the
  Relying Party expects to hold and that the scenario attempts to invalidate.

* **Preconditions and attack:** Describes the conditions required for the attack,
  the actions performed by the attacker, and the resulting incorrect conclusion
  or other security consequence. For a non-adversarial failure mode, this field
  is named **Preconditions and event** and describes the triggering event and
  its result instead.

* **Why validation succeeds:** Explains why the Evidence or secure-channel
  checks do not detect the attack, including which checks still pass despite
  the attack.

* **Mitigation strategy:** States the security property that a solution needs to
  preserve, without prescribing a particular protocol mechanism.

## Evidence Reuse
{: #evidence-reuse }

### Per-Connection Evidence Replay

**Attacker capabilities:** A malicious peer retains valid Evidence generated
earlier in the current TLS connection.

**Targeted assurance:** Evidence presented in response to a re-attestation
request reflects the current state of the Target Environment.

**Preconditions and attack:** Evidence is bound to the TLS connection but not
uniquely to an individual attestation exchange. After valid Evidence is accepted,
the state of the Target Environment changes and the Relying Party requests new
Evidence. The malicious peer retransmits the earlier Evidence in new TLS records,
causing the Relying Party to treat the earlier state as current.

**Why validation succeeds:** The Evidence remains authentic and its
connection-level binder still matches. TLS record replay protection does not
detect the attack because the old Evidence object is retransmitted as new
application data.

**Mitigation strategy:** Every Evidence response is fresh for a particular
attestation exchange and is bound to a unique challenge or equivalent freshness
mechanism. Evidence accepted for one attestation exchange cannot satisfy a later
request on the same TLS connection.

### Evidence Relay

**Attacker capabilities:** The attacker, acting as a malicious TLS peer, can
establish a TLS connection with a Relying Party and can obtain Evidence about a
separate Target Environment that contains the correct binder for the attacker's
connection as its challenge value. How the attacker obtains such Evidence is not
constrained by this model. The attacker does not need to control or collude with
the Attesting or Target Environment.

**Targeted assurance:** The Relying Party incorrectly attributes the claims
asserted by the Evidence to the TLS endpoint and assumes that the authentication
private key has the protection properties asserted by the Evidence.

**Preconditions and attack:** The attacker obtains the binder for its TLS
connection with the Relying Party. It then obtains authentic, fresh Evidence
about a separate Target Environment in which that binder appears as the
challenge value and presents the Evidence to the Relying Party. If the Evidence
is accepted, the Relying Party attributes the state of the separate Target
Environment to the attacker's endpoint and TLS connection.

**Why validation succeeds:** Proof of possession in CertificateVerify establishes
that the peer controls the private key. The Evidence is authentic and fresh, and
its challenge value matches the binder expected for the attacker's connection.
These checks establish that the Attesting Environment incorporated the correct
binder into the Evidence, but not that the evidenced Target Environment is
holds the authentication key or the TLS stack that can use it.

**Mitigation strategy:** Acceptance of Evidence establishes that the evidenced
Target Environment is associated with the peer participating in the TLS
connection being appraised. Merely including the correct connection binder as a
challenge value does not establish this association.

## Key Substitution

**Attacker capabilities:** A malicious peer controls an authentication private
key that was not generated or protected within the Target Environment and can
obtain valid Evidence about that environment.

**Targeted assurance:** The authentication private key benefits from the key
protection properties asserted by the Evidence.

**Preconditions and attack:** The Relying Party's appraisal policy does not cover
the lifecycle of the authentication key. The peer establishes the secure
connection by proving possession of its private key and presents valid Evidence
about a Target Environment that does not protect that key. The Relying Party
consequently treats a software-held, exportable, or otherwise unprotected key as
though it had the protection properties of the Target Environment. It can then
release secrets or authorize operations that it would deny to a peer using such a
key.

**Why validation succeeds:** Proof of possession establishes that the peer
controls the private key, and Evidence appraisal establishes properties of the
Target Environment. Neither check establishes that this particular private key
was generated, is stored, and is used strictly within that environment.

**Mitigation strategy:** The authentication public key is unambiguously bound to
Evidence asserting its relevant generation, storage, export, and use properties,
and those assertions are appraised together with the connection authentication.

## Evidence Privacy Loss

### Under Ephemeral-Key Compromise

**Attacker capabilities:** An ephemeral-key attacker possesses an endpoint's
ephemeral key-establishment secret for the handshake and observes the handshake
transcript, including the peer's public key share.

**Targeted assurance:** Evidence conveyed during or after the handshake remains
confidential from parties other than the connection endpoints.

**Preconditions and attack:** The attacker records the handshake and protected
records carrying Evidence. Using the compromised ephemeral private key, the
peer's public key share, and the recorded handshake messages, the attacker
computes the shared secret and derives the handshake and application traffic
secrets. It can then decrypt Evidence protected under those secrets, including
Evidence decrypted retrospectively from recorded traffic after the compromise.

**Why validation succeeds:** This is a secure-channel confidentiality failure,
not an Evidence-validation failure. The Evidence can remain authentic, fresh,
and correctly bound to the connection even though its contents are disclosed.

**Mitigation strategy:** Ephemeral key-establishment secrets are protected and
erased when no longer needed. A design that claims recovery from compromise of
such a secret provides Post-Compromise Security (PCS) by establishing new traffic
secrets using fresh key-establishment input independent of the compromised secret
before sending further Evidence.

### Under Traffic-Secret Compromise

**Attacker capabilities:** A traffic-secret attacker possesses the traffic secret
used to protect an attestation exchange on a (D)TLS connection. This capability
permits decryption of records protected by that secret, but does not by itself
imply control of either endpoint. See {{I-D.ietf-tls-extended-key-update}} for
additional discussion of the threat.

**Targeted assurance:** Evidence conveyed inside the secure connection remains
private from parties other than the connection endpoints.

**Preconditions and attack:** Attestation Evidence is sent under a compromised
handshake or application traffic secret. The attacker observes the protected
records and uses the secret to decrypt the Evidence, learning the platform and
software details it carries.

**Why validation succeeds:** This is a confidentiality failure rather than an
Evidence-validation failure. The Evidence can remain authentic and correctly
bound to the connection even though its contents are disclosed.

**Mitigation strategy:** Evidence is confidentiality-protected in transit. A
design that claims recovery from traffic-secret compromise does not send new
Evidence until it has established traffic secrets that are independent of the
compromised secrets.

## State Drift on Long-Lived and Resumed Connections

**Classification:** This is a stale-assurance failure mode. It can result from a
Target-Environment attacker, but it can also result from legitimate
reconfiguration, software update, or other state change.

**Targeted assurance:** Evidence appraised for a connection remains an adequate
basis for decisions made later in that connection or in a resumed connection.

**Preconditions and event:** The Relying Party accepts Evidence during an
initial handshake. The Target Environment subsequently changes without new
Evidence being requested and verified. Two cases are relevant:

* The original connection remains open and continues to carry sensitive data
  after the Evidence becomes stale.
* A later connection uses session resumption and omits a new attestation
  exchange, thereby inheriting an assurance established before the state change.

In either case, communication or authorized operations continue with a Target
Environment whose current state has not been appraised.

**Why validation succeeds:** The original Evidence remains authentic, but it no
longer describes the current state. The Relying Party has no newer Evidence on
which to base its decision.

**Mitigation strategy:** The validity of an appraisal is bounded by an explicit
lifetime or by security-relevant events. Continued and resumed connections do
not rely on an appraisal beyond that bound without obtaining and appraising new
Evidence.

## RA Negotiation Downgrade

**Attacker capabilities:** A network attacker can modify messages carrying the
peers' RA capabilities or selections. Alternatively, a malicious peer can
disregard the RP's request for attestation.

**Targeted assurance:** The use of RA and the selected Evidence format and
attestation model reflect the peers' authentic capabilities and configured
policies.

**Preconditions and attack:** The RA negotiation is not authenticated as part of
the secure-channel transcript or by equivalent protection, or an endpoint
permits unprotected fallback. The attacker removes an RA capability, causes
fallback to a channel without RA, or alters the selection to one that does not
satisfy a peer's configured policy. As a result, the connection completes without
required attestation or with attestation parameters that are unacceptable under
the policy that would have been applied to the authentic negotiation.

**Why validation succeeds:** The endpoint cannot distinguish the attacker-modified
negotiation from the peer's authentic offer or selection.

**Mitigation strategy:** The complete RA negotiation and its outcome are
integrity-protected and bound to the secure channel. An endpoint does not
silently fall back when its policy requires RA or particular attestation
properties.

## Compromise of Security-Critical Keys

### Authentication Key Compromise and Re-hosting

**Attacker capabilities:** An authentication-key attacker can also operate an
endorsed Target Environment that satisfies the Relying Party's appraisal policy.
The stolen authentication key is importable into that environment and has not
been revoked. This represents a Man-in-the-Middle attack as described in
{{Section 3.3.5 of -int-threat-model}}.

**Targeted assurance:** An authenticated peer identity remains associated with
an authorized Target Environment instance.

**Preconditions and attack:** The Relying Party accepts any endorsed environment
of an expected class rather than a particular authorized instance. The appraisal
policy also does not expect the key to have been created in the Target
Environment. The attacker imports the stolen authentication key into its own
acceptable environment, establishes a new connection using that key, and presents
fresh Evidence bound to both its connection and the corresponding authentication
public key. The Relying Party then authenticates the attacker as the holder of
the stolen identity and accepts the attacker's environment as an authorized host
for that identity.

**Why validation succeeds:** Proof of possession succeeds because the attacker
has the authentication private key. The connection binding and public-key
binding also succeed because the Evidence describes the attacker's current
connection and environment. Evidence appraisal accepts that environment.

**Mitigation strategy:** A deployment that requires continuity with a particular
platform instance binds authorization to that instance, rather than only to an
environment class, and provides a means to reject compromised authentication
credentials. Alternatively, the appraisal policy must inform the Relying Party
whether the authentication key was created in the Target Environment.

### Attestation Key Compromise

**Attacker capabilities:** An Attestation-key attacker can use an Attestation Key
whose credential remains accepted by the Relying Party.

**Targeted assurance:** Authenticated Evidence truthfully represents the claims
collected from the Target Environment.

**Preconditions and attack:** The compromised Attestation Key is sufficient to
authenticate the relevant Evidence claims. The attacker constructs Evidence
containing attacker-chosen claims and bindings and signs it with that key. The
Relying Party can consequently accept arbitrary asserted platform state, key
provenance, or connection associations as authentic.

**Why validation succeeds:** The Evidence signature, connection binding,
authentication-public-key binding, and asserted key provenance all verify
against attacker-chosen values. Those checks ultimately rely on the compromised
Attestation Key.

**Mitigation strategy:** The peers support rejection and recovery when an
Attestation Key is compromised.

# Integration Security Goals
{: #integration-security-goals }

This section provides a list of desirable security goals for designs that compose
RA with secure channel protocols. Proposed protocol specifications should
clearly state which of these security goals are fulfilled and explain how.

## Cryptographic Binding to Communication Channel

The Evidence or Attestation Result is cryptographically bound to the
specific secure connection (e.g., the (D)TLS connection). This prevents
**relay** attacks where an attacker presents valid, but unrelated
Evidence from a different connection or context. This binding is paramount for all
use cases because the absence of this binding can be exploited in high-severity
vulnerabilities, such as {{CVE-2026-33697}}.

## Compound Authentication
RA should complement endpoint authentication rather than replace it.
Combining the two security measures would ensure that the introduction of attestation increases security instead of replacing one security measure by another.
A formal representation of this requirement in the form of *composition* goal can be found in {{ID-Crisis}} for TLS 1.3 protocol.

## Cryptographic Binding to Machine Identifier

Evidence should be cryptographically bound to the identifier provided to the machine by the infrastructure provider to prevent **diversion** attacks {{ID-Crisis}}.

## Attestation Credential Freshness

The Relying Party is able to verify that the Evidence or Attestation Result it
receives was freshly generated by the Attester for the specific RA interaction.
State is
transient, and credentials from a previous RA interaction may no longer be valid.
See
{{Section 10 of -rats-arch}} for more details about freshness in the context of
RA. This is formalized for attestation nonce in  {{ID-Crisis}}.

## Negotiation and Capability Discovery

Peers have a secure mechanism to discover each other's support for RA, the
specific attestation formats they can produce or consume, and the attestation
models they support. This enables interoperability and allows for graceful
fallback for endpoints that do not support RA.
The negotiation of formats is required because several vendors -- like Intel,
AMD, Arm, and IBM -- have their own Evidence formats.
A conforming solution will have to support a mechanism to identify the content type and encoding of Evidence to facilitate interoperability.

## Attestation Model Flexibility

The solution supports both the Background Check and Passport models as defined
in the RATS architecture {{RFC9334}}. The Background Check model is essential
for use cases requiring maximum freshness, while the Passport model is better
suited for performance, scalability, and scenarios where the Verifier may be
offline or unreachable by the Relying Party.

## Interaction with Peer Authentication

The solution supports using RA in conjunction with traditional PKI-based
authentication (e.g., X.509 certificates). This provides two independent pillars
of trust: endpoint trustworthiness (from RA) and identity (from PKI).

## Runtime Attestation

Ideally, remote attestation should allow the Relying Party to assess that configuration change of the Attester is in accordance with a policy that the Relying Party accepts.
This enables more nuanced trust decisions based on how the Attester's state might change over time.
However, to our knowledge, current state-of-the-art systems do not achieve such a guarantee.
In such cases, frequent runtime attestation by the Verifier may reduce the exposure window, though the risk of a malicious configuration change occurring between the time Evidence is collected and the time of re-attestation, a Time-Of-Check-To-Time-Of-Use (TOCTOU) vulnerability cannot be entirely eliminated.

Evidence collected at certificate issuance or during the initial secure channel establishment reflects only the Target Environment’s state at that moment. It cannot guarantee that the Target Environment remains trustworthy for the lifetime of the certificate or even for the duration of the secure connection (e.g., the (D)TLS connection). As a result, such static Evidence is insufficient in environments where the Target Environment may change state after the connection is established and the connection is long-lived.

### Periodic vs. On-demand Attestation
It should be possible for the Relying Party to request new Evidence periodically or on-demand during the lifetime of the connection.
This may be necessary if the Target Environment has attributes that can change during the connection, thereby affecting its trustworthiness. Such changes cannot be detected using Evidence collected earlier.
For example, the Evidence may include dynamic parameters such as runtime configuration flags (e.g., FIPS mode), which indicate whether the device has entered or exited an approved mode, or measurements of critical system files.

## Privacy Preservation

The solution must not degrade the privacy of a standard secure connection (e.g., the (D)TLS connection). Evidence
can contain highly specific, unique information about a device's hardware and
software, which could be used as an advanced tracking mechanism, following a
user across different connections and services. The design must consider how to
minimize this leakage, especially when a third-party Verifier is involved in the
protocol exchange.

### Verifier Trust and Privacy
In the Background Check model, the Relying Party communicates with the Verifier at the time of appraisal.
This reveals the Attester's identity and connection timing to the Verifier.
This also reveals to the Verifier that the Relying Party is communicating with the specific Attester.
If the Verifier is a third party, it can observe which Attesters are being appraised and when, potentially exposing client identity and other correlation information.
Solutions should consider privacy-preserving attestation techniques being developed in the RATS working group, to minimize the data revealed to the Verifier.

## Performance and Efficiency

The introduction of remote attestation should not add prohibitive latency or overhead
to the connection establishment process. To be widely adopted, the solution must
be practical. While some overhead is unavoidable, multiple additional
round-trips or very large payloads in the initial handshake should be minimized.


# Use Cases

This section provides the concrete motivation for the WG's work by describing
specific use cases. For each case, the scenario, actors, and specific security
guarantees needed from RA are described.

## Secure Provisioning and High-Assurance Operations

Goal: Ensure the integrity of workloads and devices when bootstrapping their
PKI-based identity or receiving critical commands.

### Runtime Secret Provisioning

A confidential workload starts in a
generic state and needs to fetch secrets (e.g., API keys, database credentials,
encryption keys) to become operational.

* Requirement: The workload must attest its runtime state (TEE genuineness,
  software measurements) to a secrets management service. The service will only
  release the secrets after successful verification, ensuring they are delivered
  exclusively to a trustworthy environment. This use-case also covers secure
  device onboarding for IoT devices that lack a pre-provisioned PKI-based identity.

### High-Assurance Command Execution

An operator sends a critical command
to a remote system (e.g., an industrial controller, a financial transaction
processor).

* Requirement: The system must provide fresh Evidence to the
  operator to prove its integrity before the command is dispatched. This
  prevents commands from being executed on a compromised system.

## Confidential Data Collaboration

Goal: Enable multiple parties to collaborate on sensitive, combined datasets
without exposing raw data to each other or to the infrastructure operator.

### Data Clean Rooms

Multiple *data providers* contribute sensitive data to
a confidential workload for joint analysis. *Data consumers* receive aggregated
insights without ever accessing the raw, combined dataset.

* Requirement: Before sending data, each data provider must attest the
  confidential workload to verify it is running the authorized analysis code in
  a secure Trusted Execution Environment (TEE). Similarly, data consumers must
  attest the workload to trust the integrity of the results.

###Secure Multi-Party Computation (MPC)

Distributed parties
collaboratively compute a function (e.g., train a machine learning model)
without sharing their local data.

* Requirement: The central aggregator, as well as each participating client,
  must be able to mutually attest to ensure all parties are running the correct,
  untampered MPC algorithm in a trusted environment.

## Network Infrastructure Integrity

Goal: Verify the integrity of network devices that form the foundation of
communication.

### Attestation of Network Functions

A router, switch, or firewall joins
a network's management plane. A Virtualized Network Function (VNF) is
instantiated on a generic server.

* Requirement: The network orchestrator must verify the device's integrity
  (e.g., secure boot enabled, running signed OS and firmware) before allowing it
  to join the network and receive policy. This prevents a compromised router
  from misdirecting traffic or a malicious VNF from inspecting sensitive
  packets.

### Securing Control and Management Planes

An administrator connects to a
network device's management interface.

* Requirement: The administrator's client must verify the integrity of the
  management endpoint on the network device to ensure they are not connecting to
  a compromised interface that could steal credentials or manipulate the device.

## Operation-Triggered Attestation for High-Impact Application Operations
{: #sec-operation-triggered }

Goal: Ensure the integrity of application services at operation time,
when security posture may change after initial channel establishment.

Use case: **High-Assurance Operation Execution in Dynamic Application Services**:
An application service instance (e.g., AI agent) or confidential computing
environment (which could host an AI agent) maintains a (D)TLS connection with
a peer and must execute a high-impact action (e.g., payment initiation,
configuration change, privileged command).
See {{I-D.jiang-seat-dynamic-attestation}} for details.

* Requirement 1: Before executing a high-impact operation over the existing
connection, the peer must present fresh, connection-bound Evidence
reflecting the current behavior-affecting posture (e.g., enabled capabilities,
policy configuration, runtime permissions).

* Requirement 2: The mechanism should support lightweight, dynamic attestation
within the existing connection, without necessarily requiring a full new TLS
handshake, so that behavior-affecting posture changes are visible to relying
parties when required by local policy.

## Attestation of Certificate Private Key

A TLS endpoint authenticates itself using an end-entity certificate whose
corresponding private key is claimed to be protected by a secure element.
While standard TLS authentication verifies possession of the private
key, it provides no assurance about where or how that key is stored and used.

In this scenario, the peer acting as the Relying Party requires additional
assurance that the private key associated with the end-entity certificate used
to authenticate the TLS connection is generated, stored, and used within an
attested cryptographic module. In addition to verifying possession of the
private key via the TLS handshake, the Relying Party seeks
Evidence that the key is non-exportable, remains bound to the
cryptographic module, and that the module is operating in an expected
security configuration at the time the TLS connection is established.

Remote attestation is used to provide Evidence about the cryptographic module
where the private key used for TLS authentication is stored. The Evidence may
include claims about the security goals of the cryptographic module.
To prevent replay attacks, this Evidence has to be fresh and tied to the
current TLS connection. Replayed Evidence could otherwise be used to falsely
assert key security goals that no longer hold.

* Requirement: The Attester must be able to produce Evidence that demonstrates
  that the private key used for secure channel authentication:
  * is generated and stored within a specific cryptographic module or secure
    element,
  * is protected against export or software extraction
  * is attested using fresh Evidence that is bound to the current TLS connection.

The Relying Party uses this Evidence, potentially with the assistance of a
Verifier, to determine whether the key security goals satisfy its local
security policy.

The approach described in {{I-D.draft-ietf-rats-pkix-key-attestation}} addresses this
use case partially by providing attestation of the cryptographic module and associated
private key at certificate issuance time, reflecting their state when the
certificate is enrolled. This model does not provide guarantees about the
continued state of the module at connection establishment or during the lifetime of
the TLS connection.

## Platform-to-platform communication

Goal: Allow platforms to establish a trustworthy secure channel with each other.

Use case: Migration of workloads (confidential workloads in particular) between
different platforms. Migration is occasionally required in order to maintain
uptime for the hosted services across periods of scheduled downtime for the
hosting platform. Having remote attestation-enforced policies for such migration
events provides guarantees that the services will not be exposed to lower
security guarantees when migrating. Migration is typically performed by trusted,
low-level components (migration agents) on both source and destination
platforms, which perform the authorization checks and handle the data migration.

* Requirement: The migration agent on the destination platform typically acts
  as Attester, proving its state for its peer on the source platform (where the
  workload initially resides).

* Example: Intel TDX offers migration capabilities via its Migration Trust Domain (MigTD)
  {{MigTD}}. Peer MigTDs on the initiating and target platforms set up an
  attested TLS connection to perform the migration over.

## AI Governance and Accountability

Goal: Design framework for governing autonomous AI agents.

Use case: See {{I-D.aylward-aiga-2}} for details. Contrary to {{sec-operation-triggered}}, the entity verifying the Evidence in this case is the governance body and for the purposes of ensuring that no unethical or harmful action is performed.

* Requirement: Runtime attestation based on agent risk tiers defined in {{Section 2.2 of I-D.aylward-aiga-2}}

# Security Considerations

This whole document is about security. The adversary considered by this document
and the attack vectors that motivate its security goals are described in
{{attacker-model}}.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

We would like to thank Muhammad Usama Sardar, Thomas Fossati, Tirumaleswar Reddy, Yuning Jiang, and Meiling Chen for their work on establishing this document and enabling its adoption.

We would like to thank Eric Rescorla for his detailed review.
