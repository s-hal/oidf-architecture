# OpenID Federation Architecture: Three Trust Models

This document describes three architectural models for building a national-scale
OpenID Federation that incorporates existing federations and supports
inter-federation interoperability. Each model outlines the structure of trust
anchors (TAs), intermediaries, and trust relationships, along with an analysis
of pros and cons.

* **Hierarchical Trust Anchors (HTA):** A centralized model with a single
  National Trust Anchor, delegating to federation intermediaries.
* **Nested Trust Anchors (NTA):** A hierarchical-but-distributed model in which
  federation-local Trust Anchors point to a common national root.
* **Disconnected Trust Anchors (DTA):** A decentralized model in which
  independent Trust Anchors are made interoperable via a trusted key-publishing
  entity (TBE).

## Model 1: Hierarchical Trust Anchors (HTA)

### Structure

* A single National TA is the root of the trust hierarchy.
* Existing federations  act as intermediaries.
* Federation Operators, acting as intermediaries, issue subordinate statements
  for their members (OPs, RPs).

### Policy Behavior

* In HTA, the policy is defined at the National TA and applied top-down.
* Policy claims in metadata are merged starting at the TA and processed through
  each level until reaching the leaf node.
* Only one policy chain is followed.

### Trust Chain Example

```text
       +-------------+
       | National TA |
       +-------------+
              |        
      +---------------+
      |               |
+-----------+   +-----------+
| Fed A Int |   | Fed B Int |
+-----------+   +-----------+
      |               |
+-----------+   +-----------+
|  OP / RP  |   |  OP / RP  |
+-----------+   +-----------+
```

### Resolver Behavior

* Entities acting in the Resolver role are configured with only the National
  TA's key.
* Trust chains are always constructed to terminate at the National TA.
* The resolver validates each trust chain by verifying every subordinate
  statement signature from the National TA down to the leaf entity.

### Trust Mark Handling

* Trust Mark Issuers must be listed in the National Trust Anchor’s
  `trust_mark_issuers` or `trust_mark_owners` claim.
* The Trust Anchor may list an issuer directly or list a Trust Mark Owner that
  delegates issuance via a trust mark delegation. Resolvers must
  verify that the trust mark issuer is within the National TA's trust tree.
* Intermediaries may appear as issuers only if the National TA includes them
  explicitly in one of these claims.

### Pros

* Centralized governance ensures uniform policy enforcement, while
  intermediaries retain operational control over local entities.
* Trust chains are clear, deterministic, and always anchored at a single Trust
  Anchor, simplifying validation logic.
* Metadata policy and constraints are applied top-down from the National TA
  through each subordinate level.
* Fully compatible with standard OpenID Federation validation mechanisms and
  tooling.

### Cons

* Trust Mark Issuers must be explicitly listed by the National Trust Anchor to
  be valid. Intermediates cannot authorize issuers unless delegated through the
  TA’s `trust_mark_issuers` or `trust_mark_owners` claims.
* Centralized onboarding and trust mark governance increase operational overhead
  and require structured coordination.
* All policy enforcement is top-down, limiting flexibility for
  federation-specific adaptations.

## Model 2: Nested Trust Anchors (NTA)

### Structure

* Each existing federation operates its own **Trust Anchor (TA)**.
* These TAs are not independent; each TA is also an intermediate and includes
  `authority_hints` pointing to a **superior TA**, typically the National TA.
* Chains are resolved recursively up to the national root.
* When two entities reside under the same federation-local TA and the validator
  is configured to trust that TA directly, validation and policy merging can be
  performed locally. When the entities are in different federations, resolution
  proceeds through the National TA and includes both local and national
  statements.

### Policy Behavior

* Policy merging occurs based on the starting point of trust chain resolution.
* If the validator starts at the National TA, policies are merged recursively
  down through each federation-local TA.
* If the validator starts at a federation-local TA, the policy chain will be
  resolved and applied locally within that autonomous federation, without
  invoking the national-level policy.
* To preserve this autonomy, the policy defined at the National TA should remain
  minimal, delegating the definition of most metadata constraints and
  conformance requirements to the federation-local Trust Anchors and Federation
  Operators (FOs). This facilitates adaptation to federation-specific
  regulatory, operational, or sectoral requirements while ensuring overall
  compliance with the federation's trust framework.
* Each entity in the chain contributes policy through its published entity
  statement. It is the same entity statement in both cases, whether the entity
  is acting as a Trust Anchor or as an intermediate. The policy it defines is
  identical and interpreted uniformly by validators.

### Trust Chain Example

```text
        +---------------+         
        |  National TA  |         
        +-------+-------+         
                |                 
     +----------+----------+      
     |                     |      
+----+----+          +-----+-----+
| Fed A TA |          | Fed B TA |
+----+----+          +-----+-----+
     |                     |      
+----+----+          +-----+-----+
|  OP / RP |          | OP / RP  |
+---------+          +-----------+
```

### Resolver Behavior

* Entities acting in the Resolver role may be configured with the National TA,
  federation-local TAs, or both.
* If the resolver trusts the local TA, resolution can be handled entirely within
  that federation.
* If the resolver starts with the National TA, it must recursively validate all
  intermediate entity statements, including any federation-local TAs, up to the
  configured anchor.

### Trust Mark Handling

* Trust marks may be issued by a federation-local TA or a delegated entity.
* For a trust mark to be valid across federation boundaries, the issuer must be
  registered with and reachable from the National TA.
* Federation-local TAs may choose to limit trust mark issuers to their domain or
  register them upward with the National TA.
* Leaf entities must include the `trust_marks` claim. Resolvers validate trust
  marks in the context of the validated trust chain, meaning the trust mark
  issuer must be explicitly trusted and reachable from the selected Trust
  Anchor. Trust marks are only considered valid if the issuer is part of the
  trust hierarchy defined by the trust chain used during resolution.

### Pros

* Federations retain full control of their own TA
* Federation-local TAs manage only their own keys. Only the National TA must
  track subordinate TA keys
* Enables layering of trust and constraints
* Maintains standard OIDF compliance with recursive validation
* Simplifies onboarding of independently operated federations

### Cons

* Requires all resolvers to support multi-level trust chains
* Trust Mark Issuers must be registered at both the federation-local TA and the
  National TA. This demands clear processes and system support

## Model 3: Disconnected Trust Anchors with External Key Distribution (DTA with TBE)

### Structure

* Each federation operates an **independent TA**.
* There is no cryptographic or logical link between the TAs.
* A **Trusted Bridging Entity (TBE)** serves as a non-standard out-of-band
  distribution point for the public keys of independent Trust Anchors. It
  enables key retrieval by relying parties or validators to support
  cross-federation trust in the absence of a shared trust hierarchy. This
  mechanism is external to the OpenID Federation 1.0 specification.
* Entities that perform trust chain resolution (e.g., RPs, OPs, intermediaries)
  must be designed to support retrieval of Trust Anchor keys from the TBE. This
  requires implementation-level support for fetching and validating key material
  from external sources as part of the trust chain validation process.

### Policy Behavior

* In DTA, there is no shared policy hierarchy.
* Each TA defines its own policy independently.
* Entities that consume federation metadata and perform trust chain resolution
  must determine which policy chains to accept. Policy merging is local to each
  trust chain, relative to the Trust Anchor chosen by the consuming entity.

### Trust Chain Resolution

* Entities that perform trust chain resolution must be able to build a trust
  chain from a foreign TA using its public key, which is retrieved from the TBE.
* Each entity that performs trust chain resolution is responsible for managing
  its own set of accepted Trust Anchors and corresponding key material, either
  statically or by retrieving keys from the TBE

### Trust Chain Example

```text
  +-------------------------+                    
  | Trusted Bridging Entity |                    
  +-------+---------+-------+                    
          |         |                            
      +---+         +---+                        
      |                 |                        
+-----+-----+     +-----+-----+                  
| Fed A TA  |     | Fed B TA  |                  
+-----+-----+     +-----+-----+                 
      |                 |                        
+-----+-----+     +-----+-----+                
|  OP / RP  |     |  OP / RP  |                
+-----------+     +-----------+                
```

### Resolver Behavior

* Resolvers must be aware of which Trust Anchors they accept and be able to
  retrieve their keys. This can be done via static configuration or dynamic
  retrieval from the TBE.
* Each Resolver independently determines which Trust Anchors to trust and under
  what conditions.
* Because there is no shared trust hierarchy, trust chains are resolved and
  validated independently for each accepted Trust Anchor, based on the
  Resolver's configured or supported trust set.

### Trust Mark Handling

* Trust marks are valid if the issuer is trusted locally by the entity
  performing trust chain resolution.
* This model permits the use of cross-federation trust marks, provided the trust
  mark issuer is recognized and accepted by the consuming entity as part of its
  configured or supported trust framework.

### Pros

* Fully decentralized. No need for unified governance or a root Trust Anchor
* Each federation operates with complete autonomy and independent policy control
* Enables dynamic, selective interoperability across federations. Coordination
  and inter-federation policy are limited to operational and admission policies
  defined by the TBE operator.
* Minimizes dependencies. No need to align the metadata policy or onboarding

  &#x20;procedures across domains

### Cons

* The TBE is not part of the OpenID Federation specification. It introduces an
  external and non-standard dependency for key distribution.
* Trust Anchor keys must be managed and trusted individually by each entity
  performing trust chain resolution, either through static configuration or
  explicit support for dynamic retrieval.
* No signed trust relationships exist between Trust Anchors. Validation occurs
  only within each locally trusted chain.
* Local policy divergence and inconsistent trust evaluations are more likely due
  to the absence of a shared policy authority.

## Key Management and Rollover Considerations

Key rollover is a critical operational aspect of all trust models, but presents
different challenges depending on the architecture:

* **HTA**: Rollover is centralized. A change in the National TA’s key requires
  reconfiguration or re-validation only at the leaf nodes or resolvers that
  trust it directly.

* **NTA**: Rollover must be coordinated between the National TA and each
  federation-local TA. If a federation-local TA changes its key, the National TA
  must update its entity statements accordingly. The same applies in reverse.

* **DTA**: Key rollover is decentralized. Every relying party, identity
  provider, or intermediary that accepts one or more independent TAs must track
  and refresh each TA’s keys independently. If a TA rotates its key, all
  consuming entities must update their trust configurations or fetch the new
  keys from the TBE. This places a high operational burden on all participants
  and may introduce trust gaps or outages if key changes are not promptly
  propagated.

Because there is no chain of signed entity statements connecting TAs in DTA, the
integrity and freshness of key material is entirely dependent on local
configuration and key discovery mechanisms like the TBE.

## Evaluation Scenarios

This section illustrates how the three trust models behave under common
cross-federation interoperability conditions.

### Example 1: Assurance Level Trust Mark Recognition

An RP in Federation A wants to accept users from Federation B, but only if their
OP presents a verified AL-High trust mark, issued by a common accreditation
body.

* **HTA**: supported if the Trust Mark Issuer is authorized by the National TA
  and part of its signed trust hierarchy.
* **NTA**: supported if the Trust Mark Issuer is registered with and reachable
  from the National TA. Federations can limit or elevate issuer visibility.
* **DTA**: supported only if the RP trusts the Trust Anchor responsible for the
  Trust Mark Issuer and can validate a trust chain from that TA. There is no
  cross-TA inheritance unless the RP is explicitly configured to support
  multiple anchors.

### Example 2: Trust Anchor Key Rollover

Federation B replaces the signing key of its Trust Anchor (TA-B) using the
standard overlap procedure: TA-B first publishes a statement that contains
both the old and new keys, then later removes the old key. An RP in Federation A
continues to use an OP in Federation B during the rollover.

* **HTA**: seamless. A single National TA distributes TA-B’s statement with both
  keys, so every RP transparently accepts signatures made with either key.
* **NTA**: seamless if the National TA republishes TA-B’s updated statement
  before the old key is retired. RPs inside Federation B work immediately;
  cross-federation traffic stays up as soon as the National TA refreshes its
  subordinate statement.
* **DTA**: inconsistent. Each RP must fetch TA-B’s new key out-of-band,
  typically by querying the Trusted Bridging Entity (TBE) key registry or
  manually importing an updated JWKS. RPs that refresh in time keep working;
  those that do not reject the OP after the old key is retired.

### Example 3: **Cross-Federation Policy Coordination**

A new legal requirement mandates that all OPs handling sensitive personal data
must advertise `userinfo_signed_response_alg = RS256` in their metadata. The
goal is for RPs across federations to rely on this constraint and reject OPs
that do not comply.

* **HTA**: The National Trust Anchor defines the constraint
  (`userinfo_signed_response_alg = RS256`) in its federation-wide policy and
  distributes it top-down through all subordinate entity statements. Because
  every trust chain terminates at the National TA, all OPs are required to
  conform. Every RP receives metadata that reflects this enforced policy.
  **Policy coordination is guaranteed and uniform.**
* **NTA**: The National TA includes the constraint in its entity statements,
  which affects **cross-federation** chains. OPs that intend to interact outside
  their local federation must conform to the national policy. However,
  federation-local Trust Anchors can define **additional or different policies**
  for local traffic. This model supports coordination via the National TA, while
  still allowing federations to tailor policy for internal use.
  **Interoperability is enforced nationally, while flexibility remains
  locally.**
* **DTA**: There is no shared policy source. Each Trust Anchor defines its own
  metadata policy independently. RPs validate OP metadata relative to the policy
  of the TA they choose to trust. Some RPs may enforce RS256, others may not.
  Even if a common recommendation exists, enforcement depends entirely on
  individual TA configuration and RP behavior. **No coordinated enforcement.
  Consistency cannot be assumed.**

## Summary

TBD