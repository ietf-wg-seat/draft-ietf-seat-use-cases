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
    I-D.draft-ietf-rats-eat-measured-component: rats-measured
    MigTD:
     title: Intel TDX Migration TD
     target: https://github.com/intel/MigTD
    I-D.ietf-tls-rfc9147bis:
    I-D.ietf-tls-extended-key-update:
    I-D.ounsworth-rats-privacy-framework: rats-privacy
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

This document describes use cases for integrating remote attestation (RA) with
secure channel protocols, initially TLS and DTLS. Peer authentication establishes
a peer's identity; RA supplies Evidence for appraisal of its state. The document
identifies protected decisions and required assurance, describes the attacks and
failure modes that threaten them, and derives properties to guide protocol
designs in the SEAT working group. These properties address association of
attested state with a peer and connection, freshness, re-evaluation, negotiation,
and disclosure of attestation material.

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
produce verifiable "Evidence" about its current runtime state. This Evidence covers the Attester's security state and can thus include measurements of:

- firmware
- operating system
- application code
- the configuration of its hardware and software security features (e.g., secure boot status and memory
isolation).

A "Verifier" appraises this Evidence and produces an Attestation Result.
A "Relying Party" uses that result to decide whether the Attester's state is
acceptable for a protected operation. The Verifier and Relying Party roles can
be performed by the same entity.

Composing RA with a secure channel establishment protocol adds a second
dimension of trust - trustworthiness - to complement peer
authentication. This allows a peer to make authorization decisions based not
just on who the other party is, but also on what it is (e.g., an AMD
SEV-SNP-based server running in some known datacenter) and whether its state is
acceptable.

## Purpose and Scope

This document presents use cases for composing RA with secure channel protocols
in {{use-cases}}, describes relevant attacks and failure modes in
{{attacker-model}}, and derives integration properties in
{{integration-security-goals}}. The use cases include envisioned deployment
patterns contributed by SEAT WG participants. The initial focus is TLS 1.3
{{-tls}} and DTLS 1.3 {{I-D.ietf-tls-rfc9147bis}}.

The target audience for this document is the working group members, especially
those authoring protocol specifications intended to address the core purpose of
the SEAT WG. These protocol specifications should identify the use cases they
support and specify the exchanges and checks needed to establish their required
assurance.

The properties are agnostic to attestation technology. Protocol mechanisms and
the contents of appraisal policies ({{Section 8.5 of -rats-arch}}) are outside
this document's scope.

# Terminology

This document uses the terminology defined in the RATS Architecture {{RFC9334}},
including "Attester", "Relying Party", "Verifier", "Evidence", and "Attestation
Results".

This document also uses the following terms:

* Attesting and Target Environments: as described in {{Section 3.1 of -rats-arch}}.
* Attested workload: An instance of software executing within a Target Environment,
which can obtain remote attestation Evidence or Attestation Results about itself.
* Measurements: as defined in {{-rats-measured}}.
* AI agent: An AI agent is a software principal that performs closed-loop cycles
using an LLM or other model.
* Credential: as defined in {{RFC4949}}.
* Solution: used in this document to generically reference proposed protocol
designs which aim to fulfill the SEAT WG charter.

# Use Cases
{: #use-cases }

The server-as-Attester and client-as-Attester profiles define the two base attestation directions.
The remaining profiles add composition, lifecycle, or topology considerations. Application examples can map to several profiles.

The Verifier appraises Evidence and produces an Attestation Result (AR).
The Relying Party uses the AR and any required authenticated identity for its protected decision.
An unacceptable result is rejected; inability to complete required appraisal, validation, association, or freshness checks leaves the decision indeterminate.
When policy requires attestation, either outcome blocks the operation.
Any non-attested fallback is separate and explicit.
{: #uc-common-decisions }

Relying Party policy determines when a result is usable and whether it can be reused.
Fresh assurance is needed when the result expires, a relevant state change is identified, or a new connection invalidates a required binding.
Verifier appraisal can supply applicable validity constraints.
{: #uc-common-lifetime }

Combined profiles can use different Verifiers, trust anchors, policies, timing, and lifetimes.
Sharing or coordinating them is a deployment choice.
{: #uc-common-composition }

## Server as Attester Before a Protected Operation
{: #uc-server }

**Scenario and protected decision or asset:** A TLS client needs assurance about a TLS server before releasing sensitive data, accepting a result, or sending a high-impact command.

**TLS and RATS roles:** The server is the Attester, the client is the Relying Party, and the Verifier can be separate.

**Existing TLS authentication:** TLS authenticates the server's network identity; RA adds information about its Target Environment.

**Attestation scope:** For endpoint attestation, the Target Environment contains the TLS endpoint.
The key-component example below has a narrower scope.
{{uc-intermediary}} applies when an intermediary terminates TLS.

**Required security outcome:** The assurance about the server endpoint or its authentication key applies to the connection carrying the protected operation.

**Limitations:** Attestation does not guarantee the server's future state or behavior.

**Application examples:**

* **High-Assurance Command Execution**

  An operator sends a critical command to a remote system, such as an industrial controller.
  The system provides fresh Evidence for appraisal before the command is sent.

* **Data Clean Rooms**

  Data providers contribute sensitive data to an attested workload for joint analysis, while data consumers receive only aggregated results.
  Before sending data or accepting results, they attest that the workload runs authorized code in a Trusted Execution Environment (TEE).

* **Securing Control and Management Planes**

  Before an administrator uses a network device's management interface, the client appraises Evidence about the endpoint's state.
  This avoids exposing network credentials or policy to a compromised interface.

* **Attestation of Certificate Private Key**
  {: #uc-certificate-key }

  A TLS endpoint authenticates with an end-entity certificate whose private key is claimed to be protected by a secure element.
  This case appraises the key-holding component, which need not run the TLS stack.
  The required association is with the authentication key ({{property-authentication-key}}); it does not establish the state of the whole TLS endpoint.

  {{I-D.draft-reddy-rats-key-binding}} describes the key claims and checks.
  {{I-D.draft-ietf-rats-pkix-key-attestation}} addresses attestation at certificate issuance; this use case needs assurance applicable to the connection.

## Client as Attester Before Service Admission
{: #uc-client }

**Scenario and protected decision or asset:** A service needs assurance about a TLS client before granting access or releasing a protected asset.

**TLS and RATS roles:** The client is the Attester and the TLS server is the Relying Party.
A separate Verifier can appraise the Evidence, and a separate component can enforce admission, as in the network-access example.

**Existing TLS authentication:** Client attestation works with or without TLS client authentication.

**Attestation topology and trust boundaries:** The Target Environment is the client component being appraised.
The service and Verifier can belong to different administrative domains and use different trust anchors.

**Required security outcome:** The Relying Party associates the acceptable state with the client receiving access or the asset.
If client identity is used, both inputs refer to the same endpoint.

**Limitations:** Attestation does not guarantee the client's future state or use of granted access.
Acceptable measurements and credential issuance are outside this profile.

**Application examples:**

* **Attested Workload and Device Provisioning**
  {: #uc-provisioning }

  A workload or device obtains the secrets, configuration, credentials, or policy needed to become operational only after its state is appraised.
  This includes runtime secret provisioning for an attested workload and onboarding an IoT device without a pre-provisioned PKI identity.

  For example, a Confidential Container can obtain configuration data or disk keys from a Key Broker Service or Trustee {{CoCo-Trustee}} after appraisal of its TEE claims and software measurements.
  The workload is the Attester; the Trustee is the Relying Party and can also be the Verifier.

  It also includes injecting a Web PKI credential into a short-lived TEE container: a cluster utility can obtain the certificate and private key before loading them into the container, while the Relying Party obtains current assurance that the intended Target Environment uses the provisioned credential.
  Credential issuance remains outside SEAT.

* **Attestation of Network Functions**

  Before admitting a network device or function to the management plane, the orchestrator appraises Evidence about its state.
  This gates admission and policy delivery; it does not attest other devices or the traffic path.

* **Enterprise Network Access Control**

  An endpoint seeks access through an 802.1X Ethernet port or WPA3-Enterprise access point.
  With EAP-TLS {{RFC9190}}, the Network Access Server relays the exchange to an EAP/AAA server {{RFC3579}}.
  That server is the TLS server and Relying Party; it can also perform the Verifier role.
  The Network Access Server enforces the resulting admission decision.

  Attestation associates the appraised client with the EAP-TLS exchange.
  Delivery and enforcement of network authorization are functions of the access system.
  The admitted traffic does not use the EAP-TLS connection.

## Mutual Attestation Before Sensitive Exchange
{: #uc-mutual }

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

  The destination and source migration agents attest to each other.
  Intel TDX Migration Trust Domains (MigTDs) {{MigTD}} use an attested TLS connection between the source and destination.

## Re-Evaluation on Long-Lived or Resumed Connections
{: #uc-reevaluation }

**Scenario and protected decision or asset:** A peer needs current assurance before a later operation on a long-lived or resumed connection.

**TLS and RATS roles:** Either endpoint can be the Attester or Relying Party; both can re-attest.

**Attestation trigger:** Re-evaluation can be periodic or occur before a high-impact operation.

**Required security outcome:** The assurance applies to the later operation and connection; earlier or unrelated Evidence cannot be substituted.
Retaining TLS state alone does not preserve assurance across a change of endpoint, workload, or intermediary.

**Operational and failure behavior:** An indeterminate re-evaluation does not satisfy a request for new assurance or extend an old result.
Operations requiring that assurance wait for the necessary checks, including on resumption.

**Limitations:** Re-attestation cannot protect data already released or constrain future state changes.

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
{: #uc-intermediary }

**Scenario and protected decision or asset:** An intermediary terminates or mediates communication with an attested service.
The Relying Party needs assurance about the intermediary or an origin behind it.

**TLS and RATS roles:** The Attester can be an endpoint or intermediary, in either attestation direction.

**Existing TLS authentication:** TLS authenticates the terminating peer using its presented identity.
That identity alone need not distinguish a proxy from the origin.

**Attestation scope:** The intermediary can be the appraised service boundary.
Assurance about an origin behind a separate terminator requires an additional association to that origin.

**Required security outcome:** The appraisal applies to the intended component.
Assurance about an origin does not establish assurance about its terminator, or conversely.

**Operational and failure behavior:** An unexpected or indeterminate topology cannot silently replace a policy-required attested path.

**Limitations:** A TLS terminator remains a trust boundary.

**Application examples:**

* **Proxy-Fronted Attested Secure Channels**

  An endpoint reaches an application through an operational intermediary.
  The Relying Party can attest that intermediary or appraise an origin behind a trusted terminator.
  An untrusted terminator remains a trust boundary.

* **Service-Mesh Attestation**

  Service-mesh proxies terminate or mediate TLS for application workloads.
  A peer obtains fresh Evidence for the relevant proxy so policy can distinguish an expected attested proxy from an unacceptable one.

These profiles identify the decisions that require attestation and the assurance
needed to support them. The following attacker model describes attacks and
failure modes that can invalidate that assurance. Together, they motivate the
integration properties in {{integration-security-goals}}.

# Attacker Model
{: #attacker-model }

This section models the attacks that may affect the assurances provided by
solutions. To allow preservation of these assurances, mitigation avenues are
also given. Solutions may devise alternative mitigations, but should thoroughly
document how these enforce the desired assurance.

The baseline attacker is the Internet Threat Model of {{-int-threat-model}},
represented by the network attacker below. Other scenarios grant additional
capabilities or describe inherited security limits. A solution considers the
scenarios relevant to its supported use cases and claimed assurance.

Unless stated otherwise, the Relying Party is uncompromised and performs the
required protocol and policy checks. The TLS security assumptions in
{{Appendix F of -tls}} apply.

A Target Environment may represent a Trusted Execution Environment (TEE)
{{-teep-arch}}. For endpoint attestation, the model assumes that the Attester's
TLS stack runs within the appraised Target Environment. The key-component case
in {{uc-certificate-key}} instead concerns the authentication key.

Origin attestation behind a separate terminator requires an additional
association ({{uc-intermediary}}).

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
  after Evidence about that environment has been generated.

## Structure of Attack Descriptions

Each scenario identifies:

* **Attacker capabilities:** The attacker profiles and any additional capabilities.
* **Targeted assurance:** The claim the scenario invalidates.
* **Preconditions and attack:** The conditions, actions, and consequences.
  Non-adversarial failure modes use **Preconditions and event**.
* **Why validation succeeds:** The checks that still pass.
* **Mitigation strategy:** The required outcome or an inherited limit or
  deployment dependency, without selecting a protocol mechanism.
* **Why optional:** The circumstances under which an optional scenario applies.

## Evidence Reuse
{: #evidence-reuse }

### Per-Connection Evidence Replay
{: #attack-evidence-replay }

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

**Mitigation strategy:** The response is bound to the requested attestation
exchange using a mechanism that meets its freshness condition. A request for
newly generated Evidence cannot be satisfied by a response prepared before the
request, even on the same connection ({{property-exchange-freshness}}).

### Evidence Relay
{: #attack-evidence-relay }

**Attacker capabilities:** The attacker, acting as a malicious TLS peer, can
establish a TLS connection with a Relying Party and can obtain Evidence that
contains the correct binder for the attacker's connection as its challenge value,
and which represents a Target Environment. How the attacker obtains such Evidence
is not constrained by this model. The attacker does not need to control or
collude with the Attesting or Target Environment.

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
holding the authentication key or the TLS stack that can use it.

**Mitigation strategy:** Acceptance of Evidence establishes that the evidenced
Target Environment is associated with the peer participating in the TLS
connection being appraised. Merely including the correct connection binder as a
challenge value does not establish this association.

## Key Substitution
{: #attack-key-substitution }

**Attacker capabilities:** A malicious peer controls an authentication private
key that was not generated or protected within a TEE, but can obtain valid
Evidence claiming the key is protected.

**Targeted assurance:** The authentication private key benefits from the key
protection properties asserted by the Evidence.

**Preconditions and attack:** The Relying Party's appraisal policy does not cover
the lifecycle of the authentication key. The peer imports the key pair into a
TEE, establishes the secure connection by proving possession of its private key
and presents valid Evidence about the Target Environment protecting a copy of the
key. The Relying Party consequently treats a software-held, exportable, or
otherwise unprotected key as though it had the protection properties of the Target
Environment. It can then release secrets or authorize operations that it would
deny to a peer using such a key.

**Why validation succeeds:** Proof of possession establishes that the peer
controls the private key, and Evidence appraisal establishes properties of the
Target Environment. Neither check establishes that this particular private key
was generated, and is used strictly within that environment.

**Mitigation strategy:** The authentication public key is unambiguously bound to
Evidence asserting its relevant generation, storage, export, and use properties,
and those assertions are appraised together with the connection authentication.

## Evidence Privacy Loss

### Under Ephemeral-Key Compromise
{: #attack-ephemeral-key-privacy }

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

**Mitigation strategy:** This is a limit of transport protection. A recovery
claim requires fresh key-establishment input independent of the compromised
secret before further Evidence is sent.

### Under Traffic-Secret Compromise
{: #attack-traffic-secret-privacy }

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

**Mitigation strategy:** This is a limit of transport protection. A recovery
claim requires independent traffic secrets before further Evidence is sent.

## State Drift on Long-Lived and Resumed Connections
{: #attack-state-drift }

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
{: #attack-ra-downgrade }

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

### Authentication Key Compromise and Re-hosting (OPTIONAL)
{: #attack-key-rehosting }

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

**Mitigation strategy:** Instance-specific authorization or a key-protection
policy that rejects the imported key can prevent this attack. The integration
preserves the associations needed by those policies
({{property-authentication-key}}). The contents or shape of those policies is
outside the scope of solution documents.

**Why optional:** For class-based appraisal (i.e., where the Target Environment
is not restricted to a set of known platforms or workload instances), this attack
is out of scope as the attacker's posture (acceptable Target Environment that
proves ownership of some network identity) is what the RP expects to see.

### Attestation Key Compromise
{: #attack-attestation-key-compromise }

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

**Mitigation strategy:** Compromised-key rejection and recovery depend on the
attestation trust infrastructure. Valid connection and key bindings do not make
forged claims trustworthy.

# Properties of RA and TLS Integration
{: #integration-security-goals }

The use cases in {{use-cases}} and attacks in {{attacker-model}} motivate the
properties below. A solution identifies the profiles and attestation models it
supports, and specifies the protocol behavior and checks that establish the
applicable properties. These include the bindings, exchange outcomes, and
lifecycle transitions needed to use attestation in a protected decision.

The properties apply whether the Attester conveys Evidence or an Attestation
Result. Solutions preserve the association between the appraisal result, the
appraised subject, and the relevant connection and exchange. The common decision
rules in {{uc-common-decisions}} apply throughout.

## Attestation Status Before a Protected Decision
{: #property-protected-decision }

For {{uc-server}}, {{uc-client}}, and later decisions under {{uc-reevaluation}},
a solution specifies:

* successful, rejected, and incomplete exchange outcomes;
* how the applicable Attestation Result or the outcomes of association and
  freshness checks are made available to the Relying Party;

Completion of TLS alone does not establish successful attestation.

Failure to complete a check required for the decision leaves it indeterminate.
This can result from missing or unusable material, a required Verifier being
unavailable, or a timeout. A usable Attestation Result need not require Evidence
delivery to the Relying Party or contact with a live Verifier. Result reuse
follows {{property-appraisal-lifetime}}. Neither rejection nor an indeterminate
decision permits an operation requiring attestation; any fallback to operations
without attestation is explicit and policy-permitted, as described in
{{property-negotiation}}.

## Association of Attested State with the TLS Endpoint and Connection
{: #property-endpoint-association }

For endpoint attestation in {{uc-server}} and {{uc-client}}, a solution
specifies how its checks establish a cryptographically supported association
between the appraised Target Environment and the peer endpoint of the relevant
TLS connection. Authentic, fresh Evidence from an unrelated environment cannot
establish that association merely by containing the expected connection binder
({{attack-evidence-relay}}).

When protected data or operations use that connection, the association applies
to the connection carrying them. In admission use cases such as network access,
it applies to the TLS exchange supporting the admission decision; enforcement
of the resulting authorization is orthogonal to establishment of the association.
Failure to establish the required association leaves the decision indeterminate.

A solution supporting session resumption specifies how the association with the
appraised Target Environment is retained or re-established for the resumed
connection. Successful TLS resumption alone does not establish that the peer is
still the appraised environment or that its state remains acceptable. The
solution specifies the checks needed to reuse prior assurance and when fresh
attestation is required, subject to {{property-appraisal-lifetime}}.

Attestation of a key-holding component uses {{property-authentication-key}} and
does not by itself establish the state of the TLS endpoint. Origin attestation
behind a separate terminator uses {{property-intermediary}}.

## Association with Authenticated Peer Identity
{: #property-peer-identity }

When a protected decision relies on TLS peer identity, a solution specifies how
the authenticated identity and appraised state are associated with the same peer
endpoint and connection ({{uc-server}} and {{uc-client}}). It preserves the
configured TLS authentication; attestation does not satisfy a separate
requirement for identity authentication.

Solutions supporting client attestation should permit it with and without TLS
client authentication.

## Association with Authentication-Key Properties
{: #property-authentication-key }

For {{uc-certificate-key}}, a solution specifies how the appraised public key is
matched to the key used by TLS authentication and which authentication modes it
supports. Holding a copy of a key in an appraised environment does not establish
its generation, storage, export, or use properties ({{attack-key-substitution}});
those require the applicable attestation claims and appraisal.

The solution specifies what key association is retained or re-established on
resumption. A resumed connection can authenticate with a PSK without exercising
the original certificate private key. Unsupported modes cannot satisfy a
decision requiring that key assurance. Key-protection claims and policies,
including permitted import in {{uc-provisioning}}, are supplied by the
attestation mechanisms and deployment.

## Freshness for an Attestation Exchange
{: #property-exchange-freshness }

A solution specifies how Evidence or an Attestation Result conveyed in an
attestation exchange satisfies its freshness
requirements ({{attack-evidence-replay}} and {{uc-reevaluation}}). If an exchange
requires Evidence generated after its initiation, neither earlier Evidence nor
an Attestation Result based on earlier Evidence satisfies that requirement. A
unique but predictable challenge is insufficient to establish this ordering;
the nonce-based method in {{Section 10.2 of -rats-arch}} uses an unpredictable
nonce.

Solutions specify the context and acceptance rules for their chosen freshness
mechanism. A valid connection binding alone may not establish freshness.
Policy-permitted reuse follows {{property-appraisal-lifetime}}. Exchange
freshness does not itself establish that the measurements are fresh; see
{{Section 10 of -rats-arch}}.

## Appraisal Lifetime and Re-Evaluation
{: #property-appraisal-lifetime }

For decisions that reuse a result, a solution makes its applicable validity
constraints available. Solutions supporting
{{uc-reevaluation}} specify how re-attestation is requested on an existing
connection and how its outcome updates attestation status. An incomplete
re-evaluation does not extend an old result or satisfy a request requiring new
assurance ({{attack-state-drift}}).

A solution supporting resumption specifies what attestation context is retained,
invalidated, or re-established and which checks precede its use on the resumed
connection. Retaining TLS state alone does not transfer assurance to a changed
endpoint, workload, or intermediary, or extend a result's validity.

If early data is supported, the solution specifies whether attestation-dependent
operations can use it and what assurance is available before release or
processing. The early-data rules in {{Appendix F.5 of -tls}} apply.
Result lifetimes and periodic re-evaluation intervals remain policy choices.

## Protected Negotiation and Policy-Respecting Fallback
{: #property-negotiation }

A solution specifies its RA capability and parameter negotiation, including any
format or attestation-model selection, and how the complete negotiation and
outcome are integrity-protected and bound to the connection
({{attack-ra-downgrade}}). Peers validate that protection before relying on an
unambiguous selection.

The solution specifies behavior when required attestation is omitted,
unavailable, or unacceptable. Such outcomes cannot satisfy a policy requiring RA
or particular attestation properties. Any non-attested fallback is an explicit,
policy-permitted choice. Negotiation cannot compel a peer to attest or prevent
denial of service.

## Independent Attestation Directions
{: #property-directions }

For {{uc-mutual}} or other combinations of the base profiles, a solution
specifies how exchanges, results, and negotiated state are associated with their
Attester, Relying Party, and attestation direction. Success in one direction
cannot satisfy the other direction's requirements. Each direction's protected
release depends on its own successful appraisal and decision.

Mutual attestation may not require mutual TLS authentication.

## Explicit Endpoint and Origin Boundaries
{: #property-intermediary }

A solution supporting {{uc-intermediary}} identifies the appraised component and
the scope of its assurance. Attestation of the TLS terminator uses
{{property-endpoint-association}}. A solution claiming assurance about an origin
behind a separate terminator specifies or references the additional association
to that origin, its validation, and the trust placed in the terminator.

Assurance about an origin and assurance about its terminator are not
interchangeable. A solution that cannot establish the required association
cannot claim origin assurance. Support for arbitrary termination topologies is
not required.

## Confidentiality and Disclosure of Attestation Material
{: #property-attestation-confidentiality }

A solution conveys Evidence and Attestation Results under confidentiality
protection and specifies the protection used, when they are sent, who can read
them, and what recipient authentication or attestation has completed before
disclosure. This covers initial and post-handshake exchanges, terminators, and
separate Verifiers. It also identifies disclosures from negotiation data.

The solution states the limits of its protection, including disclosure to
unauthenticated peers and active attackers before later validation fails.
Neither transport encryption nor recipient authentication prevents a recipient
from redistributing Claims.

If restricted disclosure is supported, the solution specifies or references the
mechanisms and recipient-key trust assumptions
({{Sections 4.2, 4.3, 4.4 and 4.5 of -rats-privacy}}). Where Evidence is opaque to
the Relying Party, it specifies how required connection and key bindings are
checked by the Verifier and confirmed in authenticated Attestation Results.

Any recovery claim under {{attack-ephemeral-key-privacy}} or
{{attack-traffic-secret-privacy}} specifies the protection mechanism, the
attackers it covers, and which checks precede release under independent secrets.
Detection after release does not protect material already disclosed.

# Security Considerations

The properties in {{integration-security-goals}} depend on the security
assumptions of TLS and the attestation mechanisms used. The security
considerations in {{Appendix F of -tls}} and the RATS security and privacy
considerations in {{Sections 11 and 12 of -rats-arch}} apply.

Bindings cannot establish the truth of claims forged under an accepted,
compromised Attestation Key ({{attack-attestation-key-compromise}}). Solutions
honor rejection of unusable credentials or results and apply
{{property-protected-decision}}. Invalidation affecting an existing result also
affects its reuse under {{property-appraisal-lifetime}}.

The key-exposure scenarios in {{attack-ephemeral-key-privacy}} and
{{attack-traffic-secret-privacy}} describe inherited confidentiality limits.
Recovery claims require the protection described in
{{property-attestation-confidentiality}}; attestation alone does not repair
compromised TLS secrets.

Attestation material can disclose platform, workload, or user identifiers.
A separate Verifier in a Background Check exchange can also learn which Relying
Party requests appraisal and when. Solutions account for disclosures introduced
by message placement, negotiation, and Verifier interactions as described in
{{property-attestation-confidentiality}}. Omitting TLS client authentication
does not make identifying Evidence anonymous.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

We would like to thank Muhammad Usama Sardar, Thomas Fossati, Tirumaleswar Reddy, Yuning Jiang, and Meiling Chen for their work on establishing this document and enabling its adoption.

We would like to thank Camilo Ayerbe Posada, Chengxin Huang, Eric Rescorla, Hannes Tschofenig, Haowen Song, Henk Birkholz, Ira McDonald, Mark Novak, Markus Rudy, Michael Richardson, Nathanael Ritz, Paul Wouters, Serhii Nikolaichuk, Songbo Bu, Steve Luo, Waqas Nawaz, Yaron Sheffer, Yaroslav Rosomakho, Ayoub Benaissa, Iman Schrock, Katapulta Pultalowska, Ned Smith, Sophie Schmieg for their reviews and feedback.
