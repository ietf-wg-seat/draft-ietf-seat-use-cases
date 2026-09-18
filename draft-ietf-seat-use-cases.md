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
    I-D.draft-reddy-rats-key-binding:
    I-D.jiang-seat-dynamic-attestation:
    I-D.ayerbe-trip-protocol:
    RFC9190:
    RFC3579:
    CoCo-Trustee:
     title: Trustee
     target: https://github.com/confidential-containers/trustee

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

This section defines protocol-focused profiles for composing RA with (D)TLS.
Application examples provide deployment context and can map to several profiles.

The server-as-Attester and client-as-Attester profiles define the two base attestation directions.
The remaining profiles describe composition, lifecycle, or topology considerations that can be overlaid on either direction.
Base profiles use the fields below; overlay profiles specify only the properties they add or modify.

Across all profiles, the Verifier appraises Evidence and produces an Attestation Result (AR), while the Relying Party makes the authorization or release decision based on that AR.
The Relying Party rejects an unacceptable result.
Missing, stale, unverifiable, or unbound Evidence, Verifier unavailability, and timeouts make the decision indeterminate.
When policy requires attestation, an unacceptable Attestation Result or an indeterminate decision blocks the operation.
Any non-attested fallback is separate and explicit.

Across all profiles, the Relying Party's policy defines when an Attestation Result is usable for a protected decision and whether it may be reused for a later decision.
Fresh assurance is required when the result expires, when the Relying Party identifies a relevant state change, or when a new connection invalidates a required connection binding.
Verifier appraisal can supply applicable validity constraints.

Systems may combine base and overlay profiles.
When profiles are combined, the attestation exchanges and their associated Evidence, Verifiers, trust anchors, appraisal policies, timing, lifetimes, and authorization decisions are independent unless a deployment explicitly requires them to be shared or coordinated.

Profiles state the required assurance but do not define the values or mechanism used to bind attestation to a connection.
Those details belong to the protocol solution.

## Server as Attester Before a Protected Operation

**Scenario and protected decision or asset:** A TLS client needs assurance about a TLS server before releasing sensitive data, accepting a result, or sending a high-impact command.

**TLS and RATS roles:** The server is the Attester, the client is the Relying Party, and the Verifier can be separate.

**Existing TLS authentication:** TLS authenticates the server's network identity; RA adds information about its Target Environment.

**Attestation topology and trust boundaries:** The TLS endpoint is part of the appraised Target Environment.
The intermediary-aware profile also applies when an intermediary terminates TLS.

**Attestation trigger:** Evidence is appraised before the protected operation that requires current assurance.

**Relying Party authorization or release decision:** The Relying Party applies local policy to the Attestation Result and authenticated TLS identity.

**Required security outcome:** The acceptable attested state applies to the intended server endpoint and connection carrying the protected operation.

**Assumptions, limitations, and non-goals:** Attestation neither authorizes the application action nor guarantees the server's future state or behavior.

**Application examples:**

* **High-Assurance Command Execution**

  An operator sends a critical command to a remote system, such as an industrial controller.
  The system provides fresh Evidence for appraisal before the command is sent.

* **Data Clean Rooms**

  Data providers contribute sensitive data to a confidential workload for joint analysis, while data consumers receive only aggregated results.
  Before sending data or accepting results, they attest that the workload runs authorized code in a Trusted Execution Environment (TEE).

* **Securing Control and Management Planes**

  Before an administrator uses a network device's management interface, the client appraises Evidence about the endpoint's state.
  This avoids exposing credentials or policy to a compromised interface.

* **Attestation of Certificate Private Key**

  A TLS endpoint authenticates with an end-entity certificate whose private key is claimed to be protected by a secure element.
  TLS proves possession of the key, but not where or how it is stored and used.

  {{I-D.draft-reddy-rats-key-binding}} describes the use-case, and the required checks and properties.
  {{I-D.draft-ietf-rats-pkix-key-attestation}} partially addresses this use case by attesting the module and key at certificate issuance, but does not describe their state at connection establishment or later.

## Client as Attester Before Service Admission

**Scenario and protected decision or asset:** A service needs assurance about a TLS client before granting access or releasing a protected asset.

**TLS and RATS roles:** The client is the Attester and the server is the Relying Party.
A separate Verifier can appraise the Evidence.

**Existing TLS authentication:** Client attestation works with or without TLS client authentication.
Without it, the client can attest anonymously.

**Attestation topology and trust boundaries:** The Target Environment is the client component being appraised.
The service and Verifier can belong to different administrative domains and use different trust anchors.

**Attestation trigger:** Evidence is appraised before the protected decision.

**Relying Party authorization or release decision:** The Relying Party applies local policy to the Attestation Result and any authenticated client identity.

**Required security outcome:** The Relying Party associates the acceptable state with the client receiving access or the asset.
If client identity is used, both inputs refer to the same endpoint.

**Assumptions, limitations, and non-goals:** SEAT does not define acceptable measurements or how the client obtains its identity credential.
Attestation supports an authorization decision based on the client’s appraised state at the time of the protected decision, but does not assure that the client will remain in that state, remain uncompromised, or use any granted access only as expected afterward.

**Application examples:**

* **Attested Workload and Device Provisioning**

  A workload or device obtains the secrets, configuration, credentials, or policy needed to become operational only after its state is appraised.
  This includes runtime secret provisioning for a confidential workload and onboarding an IoT device without a pre-provisioned PKI identity.

  For example, a Confidential Container can obtain configuration data or disk keys from a Key Broker Service or Trustee {{CoCo-Trustee}} after appraisal of its TEE claims and software measurements.
  The workload is the Attester; the Trustee is the Relying Party and can also be the Verifier.

  It also includes injecting a Web PKI credential into a short-lived TEE container: a cluster utility can obtain the certificate and private key before loading them into the container, while the Relying Party obtains current assurance that the intended Target Environment uses the provisioned credential.
  Credential issuance remains outside SEAT.

* **Attestation of Network Functions**

  Before admitting a network device or function to the management plane, the orchestrator appraises Evidence about its state.
  This gates admission and policy delivery; it does not attest other devices or the traffic path.

* **Enterprise Network Access Control**

  An endpoint seeks access through an 802.1X Ethernet port or WPA3-Enterprise access point.
  The Network Access Server is the Relying Party and relays the authentication exchange to a RADIUS/AAA server {{RFC3579}}, which can act as the Verifier.
  General network access remains blocked until authorization completes.
  EAP-TLS {{RFC9190}} is one TLS-based authentication profile for this deployment.

## Mutual Attestation Before Sensitive Exchange

This profile overlays the server-as-Attester and client-as-Attester profiles when both TLS peers need assurance before releasing protected data.
The two directions remain independent as described above.
Mutual attestation does not require mutual TLS authentication.

**Required security outcome:** In each direction, the acceptable attested state is associated with the peer endpoint that will receive protected data.

**Operational and failure behavior:** A failure in one direction prevents that party's protected release.
A successful attestation in only one direction is not mutual attestation.

**Assumptions, limitations, and non-goals:** Mutual attestation does not prove a joint computation correct.
The application defines its transaction semantics.

**Application examples:**

* **Secure Multi-Party Computation (MPC)**

  Parties compute a function without sharing their local data.
  The aggregator and clients mutually attest that they run the expected MPC software in trusted environments.

* **Platform-to-Platform Workload Migration**

  Workloads can migrate between platforms to maintain service availability.
  Migration agents authorize and transfer the workload while enforcing policies that prevent migration to a platform with lower security guarantees.

  The destination migration agent attests to its source peer; deployments can also require source attestation.
  Intel TDX Migration Trust Domains (MigTDs) {{MigTD}} use an attested TLS connection between the source and destination.

## Re-Evaluation on Long-Lived or Resumed Connections

**Scenario and protected decision or asset:** A peer needs a refresh of its assurance before a protected operation on a long-lived or resumed connection.

**TLS and RATS roles:** Either endpoint can be the Attester or Relying Party; both can re-attest.

**Existing TLS authentication:** Re-evaluation adds current state information without replacing or retroactively strengthening TLS authentication.

**Attestation topology and trust boundaries:** Assurance does not carry over when an endpoint, workload, or intermediary changes merely because TLS state was retained.

**Attestation trigger:** Re-evaluation can be periodic or occur before a high-impact operation.

**Relying Party authorization or release decision:** The Relying Party applies the fresh Attestation Result to the later operation.

**State transition and lifetime:** Connection age does not establish freshness, and an earlier result does not prove that state persisted.

**Required security outcome:** Current assurance applies to the later operation and connection; earlier or unrelated Evidence cannot be substituted.

**Operational and failure behavior:** Indeterminate re-evaluation does not extend an old result.

**Assumptions, limitations, and non-goals:** Re-attestation cannot protect data released before an unacceptable state was detected or constrain future state changes.

**Application examples:**

* **Operation-Triggered Attestation for High-Impact Application Operations**
  {: #sec-operation-triggered }

  An application service, such as an AI agent, maintains a (D)TLS connection and later performs a high-impact action.
  Before that action, its peer obtains fresh, connection-bound Evidence of the current behavior-affecting posture.
  {{I-D.jiang-seat-dynamic-attestation}} describes this use case.

  The attestation can occur over the existing connection without requiring a new TLS handshake.

* **AI Governance and Accountability**

  A governance body appraises Evidence from an autonomous AI agent when deciding whether it may act.
  Runtime attestation follows the risk tiers in {{Section 2.2 of I-D.aylward-aiga-2}}.

* **Actor Identity Continuity via Longitudinal Trajectory Attestation**
  {: #sec-actor-identity-continuity }

  A TLS peer presents Evidence about the trajectory of an actor whose identity persists across platforms.
  The Relying Party attributes the trajectory to that identity and uses each policy-defined interval for a separate authorization decision.
  The actor is not tied to a particular machine.
  TRIP {{I-D.ayerbe-trip-protocol}} provides one example.

## Intermediary-Aware Deployment

**Scenario and protected decision or asset:** An intermediary terminates or mediates communication with an attested service.
The Relying Party needs to know which component and trust boundary it appraises.

**TLS and RATS roles:** The Attester can be an endpoint or intermediary.
This profile applies in either attestation direction.

**Existing TLS authentication:** TLS authenticates the identity presented by each connection's endpoint; an intermediary that terminates TLS remains visible as that endpoint even when presenting an origin's identity.

**Attestation topology and trust boundaries:** Either the intermediary is the attested service boundary, or a trusted terminator fronts an appraised origin.
The latter creates a channel discontinuity, so origin assurance requires a security association to the origin.
This document does not define that association.

**Attestation trigger:** Evidence is appraised before accepting the intermediary or origin as the service boundary.

**Relying Party authorization or release decision:** The Relying Party decides whether the observed termination and attestation topology is acceptable.
The result identifies the appraised component sufficiently for topology policy.

**State transition and lifetime:** A topology change can invalidate the result and trigger re-evaluation.

**Required security outcome:** The Relying Party can determine the appraised component and the TLS endpoint protecting its traffic.
A topology change does not transfer assurance between an origin and intermediary.

**Operational and failure behavior:** An unexpected or indeterminate topology cannot silently replace a policy-required attested path.

**Assumptions, limitations, and non-goals:** An authorized intermediary remains a trust boundary.
This profile neither mandates sidecars nor defines end-to-end association through an arbitrary TLS terminator.

**Application examples:**

* **Proxy-Fronted Attested Secure Channels**

  An endpoint reaches an application through an operational intermediary.
  The Relying Party can attest that intermediary or appraise an origin behind a trusted terminator.
  An untrusted terminator remains a trust boundary.

* **Service-Mesh Attestation**

  Service-mesh proxies terminate or mediate TLS for application workloads.
  A peer obtains fresh Evidence for the relevant proxy so policy can distinguish an expected attested proxy from an unacceptable one.

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
