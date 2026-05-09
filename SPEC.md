# OpenAgeProof Specification — Draft 0.10

*OpenAgeProof — an open protocol for privacy-preserving age assurance.*

**Status:** Draft. This document describes a protocol design that has not been implemented or audited. Do not deploy.

**Changes from 0.9:** Consistency pass to align all sections with prior changes. §7 issuer requirement #1 changed from "charge" to "validate" (matches §4.1.2). §6.3 reworded — blind issuance is MUST in §4.4, so §6.3 no longer says "when used." §10.3 fingerprint enforcement language reconciled with §4.2's purge-on-expiration policy. §10.5 removed reference to unspecified blinded re-signing protocol. §10.6 inlined the README content reference. §10.10 storage description updated to include token expiration date. §11 item 1 generalized to cover both charge and zero-dollar authorization. §13 README references removed. Footer version corrected.

**Normative keywords:** The terms MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY in this document are to be interpreted as in RFC 2119.

---

## 0. Position and prior art

OpenAgeProof is one proposal among several for privacy-preserving age assurance. It differs from the leading alternatives in the choice of anchor and the threat model it prioritizes. Readers evaluating OpenAgeProof should understand how it relates to existing work:

- **EU Age Verification Blueprint (2025–2026):** government-ID-anchored attestations issued via the EUDI Wallet framework, with zero-knowledge proof presentation. Cryptographically stronger than OpenAgeProof at presentation; requires government credential onboarding. OpenAgeProof's advantage is applicability to users without access to accredited digital identity infrastructure.
- **AgeAware / euCONSENT:** provider-agnostic token reuse network with double-blind architecture between issuer and relying party. Reduces repeat verification friction; does not eliminate the initial invasive check.
- **Privacy Pass (RFC 9576):** general-purpose anonymous token protocol. OpenAgeProof's blind-issuance variant (§4.4) may use Privacy Pass primitives.
- **Camenisch-Lysyanskaya anonymous credentials:** stronger cryptographic foundation for unlinkable attribute proofs. Bellovin (2025) analyzes deployment obstacles that OpenAgeProof's card-anchor approach partially addresses for card-holding populations.
- **Face-age estimation (Yoti, Privately, others):** estimation-based, not verification. Biometric data involvement raises distinct concerns (§10.9).

OpenAgeProof's distinctive properties in this landscape are:

1. **No credential-issuance record exists.** No party holds a record that associates a user's identity with the obtaining of an age assurance credential. See §10.8.
2. **The anchor credential is rotatable.** Credit cards can be cancelled and reissued after compromise; biometrics and government IDs cannot. See §10.9.
3. **No prior trust relationship with an identity provider is required.** A user with a credit card can obtain a token without being enrolled in any identity system.

These are trade-offs, not strict improvements. OpenAgeProof's age assurance is probabilistic (card-possession correlates with adulthood) rather than cryptographic (government attestation of date of birth). Implementers should choose based on threat model rather than on a single axis.

---

## 1. Terminology

- **Issuer** — a service that accepts a card charge and produces an OpenAgeProof token in exchange.
- **Holder** — the user who pays for and holds a token.
- **Relying Party (RP)** — a service that accepts OpenAgeProof tokens as evidence of age assurance.
- **Token** — the credential produced by the issuer and presented by the holder.
- **Card fingerprint** — an issuer-internal, non-reversible identifier derived from a payment method, used to enforce one-token-per-card.
- **Payment processor** — the entity (e.g., Stripe, Adyen, Braintree) that handles the actual card charge. The issuer does not directly handle PAN data.

### 1.1 Scope

OpenAgeProof is designed for contexts in which a service must verify that a user is an adult (18+ or the applicable age of majority) before granting access, and where the user prefers not to disclose their identity, date of birth, or biometric data to achieve this.

**In scope:**

- Adult content access gates mandated by laws such as the UK Online Safety Act, France SREN Law, and equivalent US state statutes (Texas HB 1181, Utah, Louisiana, etc.).
- Age-restricted purchases and services where jurisdictional law accepts credit card verification as a valid age assurance method.
- Any context where a service requires evidence of adult status and would otherwise rely on invasive document upload or biometric checks.

**Out of scope:**

- Identity verification (KYC, AML). OpenAgeProof does not identify the holder.
- Proof of unique personhood. A single individual with multiple cards can obtain multiple tokens.
- Minor-exclusion mandates (Australia's under-16 social media ban, Malaysia, Indonesia, France's proposed under-15 ban). These laws require ongoing detection of minors on platforms rather than one-time verification that a user is an adult. OpenAgeProof does not address this problem and should not be cited as a solution to it.
- Jurisdictions that require verified government identification, such as those where digital identity wallets are mandated for age verification.
- Age thresholds other than adulthood (e.g., 13+, 16+). Credit card underwriting provides evidence of adulthood; it does not support finer-grained age thresholds.

### 1.2 Jurisdictional applicability

The legal status of credit-card-based age assurance varies by jurisdiction. Implementers are responsible for determining whether OpenAgeProof meets local requirements. This section summarizes known compatibility as of the document date; implementers must verify current law before deployment.

**Strong fit (credit card verification is an accepted method, standalone):**

- **United Kingdom (Online Safety Act, Ofcom guidance):** credit card checks are on Ofcom's non-exhaustive list of methods "capable of being highly effective." Debit cards are specifically not accepted. OpenAgeProof's `funding === 'credit'` requirement aligns with enforcement practice.
- **United States — states following the Texas HB 1181 model:** statutes accept "a commercially reasonable method that relies on public or private transactional data to verify the age of an individual." Card charges qualify as transactional data. See §10.10 for the data retention constraint.

**Compatible as one component, not standalone:**

- **France (ARCOM standard under SREN Law):** requires double anonymity and at least two distinct proof methods. OpenAgeProof alone provides only one method (card possession). An OpenAgeProof-compliant issuer could be paired with a second method (e.g., facial age estimation) to form a standards-compliant stack, but OpenAgeProof is not standalone-compliant with ARCOM's technical standard.
- **EU generally:** once the EU Age Verification Blueprint is the dominant reference (ongoing rollout through 2026), OpenAgeProof is most plausibly positioned as an alternative method for users who cannot or will not use EUDI-Wallet-anchored attestations, rather than as a replacement.

**Not applicable:**

- Jurisdictions with under-16 or under-18 social media bans that require ongoing minor detection rather than adult verification. These include Australia, Malaysia, Indonesia, and similar frameworks.
- Jurisdictions that mandate government-issued digital identity credentials as the verification anchor.

## 2. Overview

The OpenAgeProof flow has three phases:

1. **Issuance.** The holder authorizes a small card charge through the issuer's payment processor. On successful charge, and provided the card has not previously been used with this issuer, the issuer returns a token.
2. **Presentation.** The holder presents the token to a relying party when the RP requests age assurance.
3. **Verification.** The relying party verifies the token locally using cached issuer public keys. The issuer is not contacted during verification. See §5.2.1 for why this is mandatory.

```
┌────────┐  (1) charge authorization   ┌──────────┐
│ Holder │ ─────────────────────────▶  │ Issuer   │
│        │ ◀─────────────────────────  │          │
│        │  (2) token                  └──────────┘
│        │                                   ▲
│        │                                   │ bulk fetch:
│        │                                   │ - public keys (§6.2)
│        │                                   │ (cached, refreshed periodically,
│        │                                   │  NOT per-token)
│        │                                   │
│        │  (3) token presentation    ┌──────┴──────┐
│        │ ─────────────────────────▶ │     RP      │
│        │                             │  verifies   │
│        │                             │  locally    │
└────────┘                             └─────────────┘
```

The issuer is contacted only at issuance (step 1) and, separately and non-per-token, for bulk distribution of public keys. The issuer does not participate in individual verifications.

## 3. Token format

A token is a signed structure containing:

| Field      | Type       | Description                                                      |
| ---------- | ---------- | ---------------------------------------------------------------- |
| `v`        | integer    | Protocol version. MUST be `1` for this draft.                    |
| `iss`      | string     | Issuer identifier (URI).                                         |
| `iat`      | integer    | Issuance time (Unix seconds).                                    |
| `exp`      | integer    | Expiration time (Unix seconds). Set per §4.3 (recommended 1 year from issuance, max 2 years). |
| `jti`      | bytes      | Unique token identifier. MUST be unpredictable (≥128 bits of entropy).|
| `scope`    | string     | Optional. Assurance level or market. See §3.1.                   |
| `sig`      | bytes      | Issuer signature over the above fields.                          |

Tokens MUST be serialized as compact JSON with fields in the order above, then signed. Signature algorithm is specified in §6.

### 3.1 Scope strings

The `scope` field carries a machine-readable description of what the token asserts. Implementations MUST support at least:

- `age-18` — issuer asserts card-issuer underwriting requires holder to be 18+
- `age-21` — issuer asserts card-issuer underwriting requires holder to be 21+ (rare; some US states have 21+ credit products)

Additional scopes MAY be defined. Relying parties MUST reject tokens whose scope they do not recognize.

### 3.2 What tokens MUST NOT contain

Tokens MUST NOT contain:

- Cardholder name, address, email, phone, or date of birth
- Card number (PAN), CVV, or expiration
- Any portion of the card fingerprint used internally by the issuer
- Device identifiers, IP addresses, or other identifying metadata
- Correlatable identifiers across different relying parties (unless the holder explicitly opts in to a linking scope)

## 4. Issuance protocol

### 4.1 Preconditions

Before issuing a token, the issuer MUST:

1. Validate the holder's card via the payment processor (see §4.1.2). This typically takes the form of a charge for the issuer's fee (see §4.1.3) but MAY use zero-dollar authorization where appropriate.
2. Reject the payment if it originated from a tokenizing wallet (see §4.1.4). Direct card entry is required.
3. Obtain from the payment processor a **stable, non-reversible card fingerprint** — for example, Stripe's `payment_method.card.fingerprint`. This fingerprint MUST be opaque to the issuer in the sense that it cannot be reversed to a PAN.
4. Check that the fingerprint has not previously produced a token with this issuer that is still active. If it has, issuance MUST fail with error `CARD_ALREADY_USED`.
5. Verify the card's product category matches the requested scope, per §4.1.1.

#### 4.1.1 BIN and card-type verification

Before signing a token with the `age-18` scope, the issuer MUST verify that the card's product category corresponds to an adult-only profile. This verification MUST use the payment processor's card metadata as the primary source (e.g., Stripe's `card.funding` and `card.brand` fields, or the equivalent from other processors). A local BIN database MAY be used as a fallback only when the processor does not expose sufficient metadata; such a database MUST be refreshed at least monthly.

An **adult-only profile** means a card product whose issuance by the card issuer requires the holder to be at least 18 (or the applicable age of majority in the issuing jurisdiction). This includes standard credit cards, charge cards, and adult debit cards tied to deposit accounts requiring adult account holders. This excludes:

- Prepaid cards
- Gift cards
- Teen/minor debit products, including family-banking products such as Greenlight, gohenry, Revolut <18, and Current Teen
- Virtual cards issued to minors through family banking apps
- Corporate or purchasing cards where no individual age attestation was performed

Issuers MUST reject cards falling into these categories for `age-18` scope tokens and return `CARD_INELIGIBLE_SCOPE`.

Some teen/minor debit products are issued under BINs that overlap with adult debit products from the same card issuer. BIN verification alone cannot reliably distinguish these cases. Issuers SHOULD maintain a deny-list of known minor-accessible products and update it as such products are identified. This is an active area of the protocol's threat model and is expected to improve as card networks publish more granular product metadata.

For the `age-21` scope, the issuer MUST additionally verify that the card product's underwriting threshold is at least 21. Most consumer cards do not meet this threshold; implementations supporting `age-21` should document which card products they accept.

Scopes are to be interpreted as lower bounds on the card's underwriting threshold. A card underwritten at 19 (e.g., a standard credit card issued in a Canadian province where the age of majority is 19) satisfies the `age-18` scope. A card underwritten at 18 does not satisfy the `age-21` scope.

#### 4.1.2 Card validation methods

The issuer needs to confirm two things about the card at issuance time: that it is a real, active card capable of being charged, and that it produces a stable fingerprint for one-token-per-card enforcement (§4.2). Two payment processor mechanisms can produce both:

- **Charge.** The issuer initiates an actual payment for a small amount. The processor returns a fingerprint and confirms the card is active. The cardholder is charged.
- **Zero-dollar authorization.** The issuer initiates a card validation request without an associated charge (e.g., Stripe's SetupIntent). The processor returns a fingerprint and confirms the card is active. The cardholder is not charged. Network fees apply (typically a few cents per validation), absorbed by the issuer.

Either mechanism produces the fingerprint required for protocol enforcement. The choice between them is operational, not protocol-level. RPs accepting tokens cannot tell which mechanism the issuer used.

Issuers SHOULD use a charge rather than zero-dollar authorization, for the reasons in §4.1.3. Zero-dollar authorization is an acceptable alternative for issuers operating under specific economic models (e.g., grant-funded, donation-supported) where charging users is undesirable.

#### 4.1.3 Fee structure

Issuers operate at a non-zero cost: per-validation network fees from the payment processor, infrastructure costs, and operational overhead. Fee structures determine how those costs are recovered.

The recommended structure is a **flat per-issuance fee** charged at issuance time, sufficient to cover the issuer's operational costs. Realistic minimum fees in 2026 are approximately $1.00–$2.00 USD, dominated by processor fixed fees (typically $0.25–$0.50 per transaction).

Other structures are permitted:

- **Per-card fee with free renewals.** Charge once on first use of a card; subsequent re-issuances on the same card (after the previous token expires) are free. Recognized by the issuer's existing fingerprint records — if the fingerprint is already in the database, the user has paid before. This model trades a small amount of additional retention (fingerprint rows kept past token expiration) for reduced user friction on renewal.
- **Free issuance.** Funded by donations, grants, or institutional support. Acceptable but requires the issuer to absorb processor fees per validation. Issuers MUST disclose their funding source if not charging users.
- **Subscription model.** A periodic fee covering multiple issuances. Less natural fit for the protocol's "one card, one active token" enforcement but possible.

The fee structure does not affect interoperability — RPs accepting tokens do not see how the issuer was funded. Each issuer chooses its own model and discloses it in service documentation.

Beyond cost recovery, charging serves additional purposes:

- **Legitimacy signaling.** A free service that asks for card information matches the pattern of card-harvesting fraud. A paid service does not.
- **Trust around card collection.** Users understand "I paid for this," which makes card use feel transactional rather than suspicious.
- **Modest abuse friction.** A few dollars discourages casual misuse, though it does not deter determined attackers with stolen cards.

These considerations push toward charging at the recommended fee level even when zero-dollar authorization is technically sufficient for card validation.

Issuers MAY charge more than cost recovery to build reserves or fund growth, but non-profit reference implementations SHOULD publish a cost breakdown to support auditability and user trust. Implementations that subsidize issuance below cost MUST disclose the subsidy source.

#### 4.1.4 Tokenizing wallet rejection

Issuers MUST reject payments that originate from tokenizing wallets such as Apple Pay, Google Pay, Samsung Pay, or other digital wallets that present a Device Primary Account Number (DPAN) or similar token rather than the underlying card.

In Stripe's API, this corresponds to rejecting any payment method where `card.wallet` is non-null. Equivalent fields exist in other payment processors' APIs.

The reason: tokenizing wallets produce different fingerprints for the same underlying card depending on which device or wallet was used to make the payment. A user with one card and three devices (phone, tablet, watch) could mint three tokens by paying via each. This breaks the protocol's "one card, one token" property — the foundational anti-Sybil guarantee that the protocol depends on.

Direct card entry produces a fingerprint that reflects the underlying card itself, ensuring that one card can produce only one active token regardless of how many devices the holder owns.

When rejecting a wallet payment, issuers SHOULD return error `WALLET_NOT_SUPPORTED` and direct the user to enter their card details directly. The error message SHOULD explain that the rejection is not a flaw in the user's card but a protocol requirement to ensure each card produces a single token.

This requirement may be relaxed in future versions of the protocol if payment processors expose stable card-level fingerprints for wallet payments. Until that capability is available and verified, direct card entry is required.

### 4.2 Fingerprint storage

The issuer MUST store, for each issued token:

- The card fingerprint (from the processor)
- The token's expiration date (per §4.3)

The issuer MUST NOT store:

- The PAN, CVV, or any other raw card data
- The unblinded `jti` (which the issuer cannot derive from blind issuance)
- The blinded value or the signature produced during issuance (transient inputs and outputs; not needed after the issuance flow completes)
- Any holder identifying information (the processor handles cardholder data; the issuer receives only the fingerprint)

The fingerprint is an opaque token from the processor's side; it cannot be used by an attacker without the corresponding card. Storing it enables one-token-per-card enforcement without creating a PII database. The token's expiration date is metadata about the protocol's operation, not about the user, and is stored to allow re-issuance after expiration.

This minimalism is deliberate. The issuer's database, after issuance, holds two values per token: an opaque fingerprint and a date. Neither identifies the user. Neither links the issuer's records to a specific token in the wild. A subpoena to the issuer reveals only that some card was used to obtain a token expiring on a particular date.

After a token's expiration date passes, the issuer MAY purge the corresponding row to allow re-issuance on the same card. Implementations SHOULD purge expired entries periodically. Implementations operating a "per-card with free renewals" fee model (§4.1.3) MAY retain expired entries to recognize repeat customers.

### 4.3 Token expiration

Issuers SHOULD set token expiration to 1 year from issuance. Issuers MAY set shorter expirations (no less than 6 months) for higher-security contexts or longer expirations (no more than 2 years) where operational constraints justify it. Issuers MUST publish their chosen lifespan in their service documentation.

Token expiration is independent of the card's expiration date. The card's role is to prove adulthood at issuance; once that proof is established, the card's continued validity does not affect the token. Constraining token lifespan based on card expiration would penalize users with older cards without providing meaningful additional protection — the lifespan cap below already bounds the relevant damage.

A 1-year lifespan bounds the impact of several events the protocol cannot detect at issuance time:

- **Card fraud**: if a card was used without the cardholder's authorization, the cardholder will typically discover and cancel the card within weeks. The fraudulently-obtained token remains valid for its remaining lifespan, but the damage is bounded.
- **Chargeback abuse**: a holder may dispute the issuance charge after obtaining a token. The token remains valid until expiration; the issuer absorbs only one year of operational cost per chargeback.
- **Shared card use**: a card legitimately issued to an adult but used by another party (e.g., a minor with a parent's card). The protocol cannot detect this at issuance; the lifespan bounds the duration of misuse.
- **Stolen tokens**: if a holder's stored token is exfiltrated through malware or device theft, the thief retains usability for at most the remaining lifespan.

The issuer stores the token's expiration date alongside the fingerprint (§4.2) so that re-issuance on the same card can be authorized once the previous token has expired. This is metadata about the protocol's operation, not about the user; it does not weaken the privacy properties in §10.8.

When the previous token expires, the same card MAY be used to obtain a new token. The issuer's record of the previous issuance is purged or replaced with the new one.

### 4.4 Blind issuance

Issuers MUST use a blind signature scheme. The recommended scheme is the Privately Verifiable Token issuance from IETF Privacy Pass (RFC 9576).

In blind issuance:

1. Holder generates a random token identifier locally and blinds it before sending to the issuer.
2. Issuer signs the blinded value after the charge succeeds.
3. Holder unblinds the signature, producing a token the issuer has never seen in unblinded form.

The issuer's database, after issuance, contains only the card fingerprint and the token's expiration date (per §4.2). The blinded value flows through the issuance protocol but is not retained — it has no operational purpose after the signature has been returned to the holder. The unblinded token (containing the actual `jti` that appears at relying parties) is mathematically unrecoverable from the issuer's records.

This property is what enables the subpoena-resistance claim in §10.8. Implementations that omit blind issuance do not provide this property and MUST NOT claim it.

## 5. Presentation and verification protocol

### 5.1 Presentation

The holder presents the token to the relying party through an implementation-defined channel (HTTP header, cookie, form field, QR code, etc.). The protocol does not mandate a transport.

For web contexts, a recommended transport is an HTTP header:

```
OpenAgeProof-Token: <base64url-encoded token>
```

### 5.2 Verification

On receipt of a token, the relying party MUST:

1. Parse the token and verify the structure matches §3.
2. Retrieve the issuer's public signing key for the `iss` value. Keys MAY be cached but caches MUST honor the issuer's published key rotation metadata.
3. Verify the signature over the token's fields.
4. Verify `iat` is in the past and `exp` is in the future, with an implementation-defined clock skew tolerance (SHOULD be ≤5 minutes).
5. Confirm the `scope` meets the relying party's requirement.

If all checks pass, the relying party accepts the token as valid age assurance for the scope indicated.

### 5.2.1 Local verification (normative)

All verification described in §5.2 MUST be performed locally by the relying party, using cached issuer signing keys (§6.2). The relying party MUST NOT contact the issuer on a per-token basis to verify a token.

Issuers MUST NOT offer a remote token-verification endpoint — that is, an endpoint that accepts a full token (or a `jti`) as input and returns a validity decision. Endpoints that serve key material (§6.2) are permitted because they are bulk, non-token-specific, and cacheable; they do not reveal individual verification events to the issuer.

**Rationale.** Any remote verification call made by a relying party reveals to the issuer, through standard HTTP request metadata (source IP, TLS handshake, request headers, reverse DNS), which relying party is performing the verification. Over many such calls, the issuer can accumulate a per-token log of which services a token is being presented to and when — effectively a browsing history for the token holder. This reintroduces the surveillance problem that OpenAgeProof is designed to prevent, regardless of any policy commitment by the issuer not to log such data.

The protocol's privacy guarantees are therefore structural, not policy-based: the issuer cannot observe presentations that never reach its servers. Prohibiting the remote verification endpoint is the mechanism that enforces this property at the protocol level rather than the trust level.

Relying parties that cannot perform local cryptographic verification (e.g., severely constrained embedded environments) are not supported in this version of the protocol. Remote verification is not a compliant fallback.

### 5.3 Revocation

OpenAgeProof does not provide per-token revocation. Tokens expire at the date specified in their `exp` claim (per §4.3) and become invalid at that point. There is no revocation list.

This is a deliberate design choice. Per-token revocation requires the issuer to maintain a `{card, jti}` mapping or equivalent linkage, which would defeat the subpoena-resistance property described in §10.8. Operational events that might suggest a need for revocation are handled differently:

- **Chargebacks:** absorbed into the issuance fee (§4.1.3). The disputed token remains valid for its remaining lifetime; the issuer accepts the financial loss.
- **Stolen or lost tokens:** the holder cannot revoke them. The token expires at its natural `exp` date. To prevent further abuse, the holder cancels the underlying card, which prevents new tokens being issued on that card (the cancelled card cannot pay).
- **Issuer key compromise:** handled by key rotation (§6.2), not per-token revocation. All tokens signed with a compromised key become invalid when the key is removed from the published key set.
- **Abuse at a specific RP:** the RP MAY maintain its own local block list of `jti` values it has seen behaving abusively. This is RP-local and does not require coordination with the issuer.

Implementers should understand that this design accepts a trade-off: stolen tokens remain usable until expiration. The trade-off is justified because per-token revocation would create the very `{card, jti}` linkage that blind issuance is designed to prevent.

### 5.4 Replay

Tokens MAY be single-use or multi-use depending on the relying party's requirements. For single-use:

- The RP MUST maintain a seen-`jti` set for the token's validity period.
- The RP rejects any token whose `jti` has been previously accepted.

For multi-use:

- The RP MAY accept the same token for repeated presentations by the same session.
- The RP SHOULD NOT broadcast the `jti` to other RPs.

### 5.5 Verification patterns

Relying parties implementing OpenAgeProof will choose one of several verification patterns based on their architecture. The protocol does not mandate a pattern; this section describes the common choices and their trade-offs. The choice is invisible to the issuer — none of the patterns require coordination with the issuer or change what the issuer stores.

**Stateless verification.** The RP verifies the token on each presentation. No state is maintained between requests beyond the seen-`jti` set required for replay prevention (§5.4). Suitable for content sites without user accounts. The protocol provides no protection against token sharing in this mode; an RP concerned about sharing must implement its own controls (rate limiting, IP-based heuristics, behavioral analysis). Privacy against the RP is bounded by what the RP logs alongside the token presentation — IP addresses, browser fingerprints, and request patterns can produce correlation even without explicit account binding.

**Per-session verification.** The RP verifies the token at session start. The session itself maintains state (typically a server-side session record or a signed cookie). Tokens are not re-checked within a session but must be re-presented for new sessions. Suitable for sites with login flows but without persistent user accounts. Reduces token presentation frequency without requiring durable account binding.

**Account-bound verification.** The RP records the token's `jti` against a user account at first verification. Subsequent logins use account state, not the token. The RP enforces uniqueness on `jti` within its user base, preventing one token from being claimed by multiple accounts. The token's `exp` is checked periodically; expired tokens require a new token to be obtained from any compliant issuer. Suitable for platforms with persistent accounts where regulatory compliance requires the age assurance to be associated with the account.

The account-bound pattern is the most resistant to token sharing — once a `jti` is claimed by Account A, it cannot be claimed by Account B at the same RP. Sharing across different RPs is still possible (a token claimed at RP A can be claimed at RP B), but sharing within an RP is prevented.

The privacy properties differ across patterns:

- In all three patterns, the issuer learns nothing about token usage. The issuer-side privacy properties in §10.8 hold regardless of RP pattern.
- The stateless pattern offers the least RP-side privacy in practice, because RPs implementing it must log enough to do replay prevention and abuse detection.
- The account-bound pattern is pseudonymous from the user's perspective: the RP knows "this account is held by an adult," not who the adult is, provided the user creates the account without supplying identifying information.

## 6. Cryptography

### 6.1 Signature algorithm

Implementations MUST support **Ed25519** for token signatures. Implementations MAY additionally support ECDSA P-256.

### 6.2 Key distribution

Issuers MUST publish their current signing public keys at a well-known URL:

```
<iss>/.well-known/openageproof-keys.json
```

Format:

```json
{
  "keys": [
    {
      "kid": "2026-01",
      "alg": "Ed25519",
      "public_key": "<base64url>",
      "not_before": 1735689600,
      "not_after": 1767225600
    }
  ]
}
```

Issuers SHOULD rotate keys at least annually. Relying parties MUST honor `not_before` and `not_after` when verifying signatures.

### 6.3 Blind signature scheme

The recommended blind signature scheme is the Privately Verifiable Token issuance from IETF RFC 9576 (Privacy Pass), which provides:

- Unlinkability between the issuance and presentation phases
- Issuer cannot recognize a token it signed
- Relying party can still verify the signature is from a trusted issuer

A concrete profile selection (specific PVT mode, key formats) is required for interoperability between independent implementations. This is noted in §13 as an open issue.

## 7. Issuer requirements

To be considered a compliant OpenAgeProof issuer, an implementation MUST:

1. Validate the holder's card via a PCI-compliant processor before issuing a token (per §4.1.2).
2. Enforce one active token per card fingerprint at any given time (re-issuance permitted after the prior token expires).
3. Not store PAN or cardholder PII beyond what is required for the charge itself (which is held by the processor, not the issuer).
4. Publish and rotate signing keys as described in §6.2.
5. Set token expiration per §4.3 (fixed lifetime, recommended 1 year, max 2 years).
6. Publish a privacy policy explicitly stating what is and is not collected.
7. Not operate a remote token-verification endpoint as prohibited in §5.2.1.
8. Implement blind issuance (§4.4).

Compliant issuers SHOULD:

1. Undergo third-party audit of the above requirements.
2. Publish open-source their server implementation.

## 8. Relying party requirements

To be considered a compliant OpenAgeProof relying party, an implementation MUST:

1. Verify tokens per §5.2.
2. Not log, store, or transmit token values beyond what is required for the verification and single-use replay check.
3. Not attempt to correlate tokens across sessions unless the holder has explicitly consented.
4. Clearly communicate to users when age assurance is required and which issuers are accepted.

## 9. Error codes

Errors returned during issuance or verification:

| Code                      | Meaning                                                  |
| ------------------------- | -------------------------------------------------------- |
| `CARD_CHARGE_FAILED`      | The payment processor declined the card.                 |
| `CARD_ALREADY_USED`       | This card has already been used with this issuer.        |
| `CARD_EXPIRED`            | The card is past its expiration date.                    |
| `CARD_INELIGIBLE_SCOPE`   | The card's underwriting does not match the requested scope. |
| `WALLET_NOT_SUPPORTED`    | The payment came from a tokenizing wallet (Apple Pay, Google Pay, etc.); direct card entry is required. |
| `TOKEN_EXPIRED`           | The token's `exp` has passed.                            |
| `TOKEN_REPLAY`            | The token has already been seen by this RP (single-use). |
| `TOKEN_SIGNATURE_INVALID` | The signature does not verify against the issuer's key.  |
| `ISSUER_UNKNOWN`          | The RP does not recognize the issuer.                    |

## 10. Security considerations

### 10.1 Issuer compromise

An issuer whose signing key is compromised can mint arbitrary tokens. Mitigation: key rotation (§6.2) and short key validity windows. When a key is removed from the published key set, all tokens signed with it become invalid at relying parties. There is no per-token revocation (§5.3); the only revocation mechanism is key rotation.

### 10.2 Card data exposure

The issuer never sees PAN data — the processor handles the charge and returns an opaque fingerprint. The PCI-DSS scope for the issuer is therefore limited to SAQ A territory (no direct PAN handling). Implementations MUST NOT accept raw PAN data server-side.

### 10.3 Chargebacks and stolen cards

A holder may dispute the issuance charge after obtaining a token, either legitimately (they don't recognize the charge, the card was stolen) or maliciously (to obtain a token at no cost).

Because OpenAgeProof has no per-token revocation mechanism (§5.3), the disputed token remains valid for its remaining lifetime. The issuer's mitigations are:

- **Pricing absorbs expected chargeback losses.** The charge amount (§4.1.3) is set to cover the operational cost of issuance plus an expected chargeback rate. Disputed tokens are accepted as a baseline operational cost, similar to how merchants generally absorb chargebacks on small transactions.
- **Fingerprint enforcement prevents reuse during the active token lifetime.** While a token issued from a card is active, the fingerprint cannot be used to mint another token from the same card (§4.2). A bad actor cannot use the same card to mint additional tokens during this window.
- **Stripe-side fraud detection.** The payment processor flags fraudulent and high-risk cards before issuance, blocking many bad-faith attempts at the payment layer.

For the **stolen card case**: the legitimate cardholder will typically notice the charge and either dispute it or cancel the card. The thief retains the previously-issued token until natural expiration. This is comparable to the behavior of any credential issued at a moment in time — the credential survives the lifecycle of its underlying authorization. The damage ceiling is bounded by the token's `exp` date.

Implementers MAY additionally choose to delay issuance briefly after charge (hold the token for several hours) to allow some chargeback events to occur before the token is delivered. This trades UX for a slight reduction in stolen-token exposure.

### 10.4 Correlation by timing

Even with blind issuance, an issuer that logs charge timestamps and an RP that logs presentation timestamps could, in principle, correlate users if the timestamps are close and the population is small. Mitigations: batch issuance, randomized delay between charge and token availability, large enough user base that timing doesn't uniquely identify.

### 10.5 Linkability across RPs

A token presented to multiple RPs is the same token. If two RPs collude and share received `jti` values, they can determine the user is the same person. Mitigations: holders MAY obtain multiple tokens for use with different RPs (subject to the one-active-token-per-card constraint, this requires multiple cards); future protocol versions may explore per-RP token derivation schemes.

### 10.6 Minor with access to parent's card

This is the dominant abuse case and OpenAgeProof does not prevent it. A card legitimately issued to an adult, held or used by a minor (e.g., a teenager using a parent's card), produces a valid token. The protocol cannot detect this at issuance time. Mitigation is non-technical and shared with every other age assurance method — no current verification mechanism reliably distinguishes "the cardholder is an adult" from "an adult cardholder is being represented by a minor."

### 10.7 Remote verification as a surveillance vector

A remote token-verification endpoint provided by the issuer — one that accepts a token and returns a validity decision — would allow the issuer to observe every presentation of every token. Because HTTP requests reveal their source through IP address, TLS metadata, and request headers, the issuer would be able to attribute each verification call to the relying party making it. Over time, this produces a per-token log of which services each holder uses and when — a browsing history that did not exist before the endpoint was introduced.

This is not a theoretical concern. Any standard web server logs incoming requests by default. Preventing such logging requires affirmative action by the issuer and cannot be verified by holders or relying parties from outside. The protocol therefore prohibits the remote verification endpoint entirely (§5.2.1) rather than relying on issuer policy.

The prohibition is also a defense against issuer compromise. An issuer whose operations are compromised by an attacker, a commercial acquirer, or a legal order cannot retroactively produce a verification log that was never generated. Data that does not exist cannot be breached, sold, or subpoenaed.

### 10.8 Absence of a credential-issuance record

Every widely deployed age assurance system — whether based on government ID, facial biometrics, credit reference data, or accredited digital identity wallets — depends on a trusted party holding a record that associates a user's identity with the obtaining of an age assurance credential. That party may be a government agency, an accredited identity provider, a commercial identity verification vendor, or a platform operator. In every case, the record exists and is findable by anyone who can compel or breach that party.

This record is an attack surface independent of the age assurance transaction itself. It is the thing that gets subpoenaed, breached, or re-used for purposes other than age assurance. It is also the thing that gets weaponized when the political environment shifts: a state that decides to police access to certain categories of content can query the credential-issuance record to identify who has ever obtained a credential to access that content.

OpenAgeProof is designed so that the issuer cannot produce a `{card, token}` mapping, even under legal compulsion.

The issuer's database contains only the card fingerprint (opaque, not reversible to a PAN) and the token's expiration date (per §4.2). It does not contain the blinded value, the signature produced at issuance, or any reference to the unblinded `jti` that appears in tokens at relying parties. The holder generated the `jti` locally, blinded it before sending it to the issuer, and unblinded the signature on receipt. The issuer never possessed the unblinded form, and does not retain the blinded form either.

A subpoena to the issuer asking "what token did this card receive?" cannot be answered. The database can confirm that a card was used to obtain a token expiring on a particular date, but cannot identify which token in the world is that card's. The expiration date is metadata about the protocol's operation; it is not user-identifying. This is enforced by cryptography, not policy.

**What this property does not protect against:**

- **Subpoena to the relying party:** RPs see tokens being presented and may log them alongside IP addresses, timestamps, or other identifying information. A subpoena to an RP can identify users via these side channels.
- **Subpoena to the payment processor:** Stripe (or equivalent) maintains records linking cards to PaymentIntents. These records identify which cards charged the issuer, even though they do not contain token data.
- **Network surveillance:** an adversary monitoring a holder's traffic can observe token presentations directly.
- **Compromised client code:** if the issuer or a third party serves malicious client-side code that exfiltrates the unblinded `jti` after issuance, the protocol's protections are circumvented. Users requiring strong guarantees should verify the client code they run, either through audit of open-source implementations or through independent client implementations from sources other than the issuer.

The protocol contributes the issuer's own database to a defense-in-depth privacy stack. It does not eliminate all paths to user identification. Users with state-level threat models should combine OpenAgeProof with network-level privacy tools (Tor, VPN), payment privacy practices, and careful client code verification.

Implementers adapting OpenAgeProof for contexts requiring stronger identity linkage (for regulatory reasons, for dispute resolution, etc.) should be aware that adding such linkage eliminates this property. Variants that retain identity information should be documented as such and should not be represented as preserving OpenAgeProof's privacy model.

### 10.9 Credential rotation and breach recovery

All deployed age assurance systems will eventually experience breaches. Identity verification providers have incurred documented breaches involving the exposure of government IDs, biometric templates, and verification selfies. The question is not whether breaches occur but what happens to users when they do.

Age assurance systems differ significantly in post-breach recovery:

- **Biometric-anchored systems** (facial age estimation, fingerprint-based verification): users cannot regenerate their biometrics. A leaked facial embedding permanently compromises the user's ability to use that biometric for future verification, and enables targeted spoofing of future systems that rely on the same biometric.
- **Government-ID-anchored systems** (document upload, mobile driver's license, EUDI Wallet): users can eventually obtain new government documents, but the process is slow (weeks to months), expensive, and in some jurisdictions rare enough that most users never do so. Certain attributes (date of birth, historical addresses) cannot be rotated at all.
- **Bank-account-anchored systems** (open banking verification): users can change banks, but account numbers are rarely rotated, and the breach exposes transaction history as well as identity.
- **Card-anchored systems (OpenAgeProof):** users can cancel and reissue a credit card within days at no cost. The banking system is structurally designed around the assumption that card credentials are compromised periodically and must be rotatable. A new card has a new fingerprint; the user can obtain a new OpenAgeProof token with no residual exposure from the breach.

OpenAgeProof is the only category of age assurance system in which the anchor credential is designed for routine rotation. This is not incidental — it is a consequence of the banking system's long-standing assumption that card credentials will be compromised, which predates the internet-era concerns that drive age assurance.

In cryptographic terms, the rotation property is analogous to forward secrecy at the session-key level: rotation of the underlying credential means that a future breach of historical records has limited ability to retroactively identify affected users, because the credentials referenced in those records may no longer be associated with the users who originally obtained them.

This property strengthens the argument in §10.8. Even if a future attacker, state actor, or litigant obtains an OpenAgeProof issuer's complete historical database, correlating those records with currently-held cards is difficult for the subset of users who have rotated their cards in the interim. No other age assurance system provides an equivalent recovery path.

### 10.10 Data retention under "no identifying information" statutes

Several jurisdictions (including Texas HB 1181 and statutes in at least a dozen other US states) prohibit the retention of "identifying information" by age verification providers after the verification is complete. Implementers deploying OpenAgeProof in these jurisdictions must evaluate whether the issuer's fingerprint storage (§4.2) complies with the applicable statute.

The case for compliance is as follows. The card fingerprint stored by the issuer is:

- Opaque: generated by the payment processor using its own internal schema; not reversible to a PAN without access to the processor's keys.
- Non-personal on its own: the fingerprint alone cannot be used to identify a natural person. A third party in possession of only the issuer's database cannot determine which person held a given fingerprint.
- Not linked to identifying attributes: the issuer stores only the fingerprint and the token's expiration date alongside it. No name, address, email, phone, date of birth, or other identifying data is recorded. The expiration date is operational metadata about the protocol, not the user.

Under these conditions, the fingerprint is arguably not "identifying information" in the sense contemplated by the statutes, which are directed at retention of data that directly identifies a natural person (driver's licenses, passports, etc.).

However, this is a legal interpretation and has not been tested in court. Implementers deploying in retention-sensitive jurisdictions SHOULD:

1. Obtain jurisdiction-specific legal advice before deployment.
2. Consider additional protections: hashing the fingerprint with a non-exportable secret, rotating the hashing secret periodically, or discarding fingerprints entirely after a configurable retention window (at the cost of degrading one-card-one-token enforcement across that window).
3. Document retention practices in the issuer's privacy policy in terms that would be readable by a regulator.

Implementations targeting jurisdictions with absolute retention prohibitions may need to operate in an ephemeral mode where the fingerprint is used for duplicate detection within a single session and not persisted beyond it. This weakens duplicate detection across longer time windows but may be required by local law.

## 11. Privacy considerations

The design assumes the following privacy properties, each of which depends on implementation correctness:

1. **Issuer learns:** that card X was validated (charged or zero-dollar authorized) and is eligible for the requested scope. Nothing after that. This property is structural (enforced by the prohibition on remote verification endpoints in §5.2.1) rather than dependent on issuer policy or promise.
2. **Relying party learns:** that a valid token was presented, nothing about the holder.
3. **Payment processor learns:** the charge occurred (standard payment flow), the issuer is the merchant of record.
4. **Card issuer learns:** a charge occurred at the OpenAgeProof issuer.

Items 3 and 4 are unavoidable given the use of the traditional payment system. They are equivalent to the privacy posture of any other paid online service the user might sign up for.

An issuer operating in a surveillance-hostile environment should publish a transparency report documenting requests received from law enforcement or other parties.

## 12. Versioning

This document describes version 1 of the protocol. The `v` field in tokens allows future versions to coexist. Breaking changes require incrementing `v` and publishing a new specification.

## 13. Open issues

The following design issues are unresolved in this draft:

- The scope vocabulary (§3.1) needs extension for non-age assertions if the protocol is generalized.
- The single-use replay check (§5.4) places state burden on the RP; a stateless alternative using time-bucketed nonces is worth exploring.
- The Privacy Pass blind-signature integration (§4.4, §6.3) is described abstractly and needs a concrete profile selection (e.g., specific PVT mode, key formats) before interoperable implementations are possible.
- A standardized BIN deny-list for teen banking products (§4.1.1) is community-maintained on an ad hoc basis. A formal mechanism for sharing and updating this list across issuers would improve consistency.

## 14. References

- RFC 9576 — Privacy Pass Privately Verifiable Tokens
- RFC 2119 — Key words for use in RFCs
- Chaum, D. (1983). "Blind signatures for untraceable payments." *Advances in Cryptology*.
- Ofcom. Guidance on highly effective age assurance under the UK Online Safety Act.
- ISO/IEC 18013-5 — Mobile driving licence (mDL).

---

*End of specification draft 0.10.*
