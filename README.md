# 3DS-FIDO: ProVerif Models Including Revocation Confidentiality

These models follow Sections III, IV, and V-B and Figures 5–9 of **3DSFIDO**. Both `.pv` files include registration, card binding, transactions, and revocation, and each can be executed independently. The manuscript itself has not been modified.

## 1. Why Two Models?

| File | Adversary knowledge and capabilities | Verification objectives |
|---|---|---|
| `01_authentication.pv` | PANs, UIDs, public keys, logical FIDO and EMV responses, and revocation commitments are public. The adversary can intercept, modify, fabricate, or replay FIDO responses during binding and transactions, and EMV responses during binding. | Registration, binding, and transaction correspondences; message origin; challenge agreement; counter validation; and key secrecy. |
| `02_revocation_confidentiality.pv` | Registration, binding, and transaction data use assumed secure channels. PAN, UID, and CID are initially unknown to the external adversary. Verification keys and revocation pairs `(rpId, commitment)` are public. | Whether an external observer can derive PAN, UID, CID, seed, msk, or private keys. |

The public response interfaces in the first model deliberately grant more adversarial control than correctly established TLS permits. They do not imply that these data should be exposed in deployment. Registration still relies on an authenticated banking session, an honest authenticator, and a trusted FIDO Client. The second model assumes correct TLS and trusted App, Client, and local card-reading environments.

A model that initially gives the adversary a PAN cannot also establish secrecy of that PAN. The two models therefore use different initial knowledge and observation boundaries. The second establishes that disclosing revocation payloads does not make the protected values derivable under its assumptions; it does not establish unconditional PAN secrecy in every environment.

## 2. Mapping to the Manuscript

| Manuscript location | Model implementation |
|---|---|
| Section III, Figure 5, Equations (1)–(4) | Registration in `Authenticator` and `IssuerRPACS`. UID remains fixed within the account context; CID is fresh; `sk = deriveSK(msk,CID,UID,rpId)`; the commitment is `commitment(seed,rpId)`; and the initial counter is 0. PAN is not an input to key derivation. The master secret msk and revocation seed are distinct secrets. |
| Figure 6, Equations (5)–(8) | The authenticator increments its counter and reconstructs its private key. The issuer retains `(m,sigma)`, computes `mu = hash(encodeResponse(m,sigma))` and `trunc32(hmacUN(kw,mu))`, validates the card PAN and UN, and then verifies the retained FIDO signature and context. |
| EMV card authentication | The issuer signs SSAD before the card role starts. The card holds its own private key and the issued SSAD, not the issuer's private key. SDAD signs a fresh NC and the current UN. Correct certificate validation linking the verification key to the PAN is assumed. |
| Figure 7: transactions | Transactions use the established PAN, UID, CID, and public-key association. The RP generates a fresh challenge; the authenticator increments its counter; and the RP checks the challenge, RP identity, UID, CID, signature, increasing counter, and active status. `tid` represents local transaction context and is not added to the signed message. |
| Section IV, Figure 9 | `RevocationApp` retrieves PAN and commitment records for two issuers from its local seed-indexed backup. It authenticates one bound card locally and matches its PAN before submitting the issuer-specific commitments. `RevocationService` publishes and routes the payloads. Each issuer matches its commitment, updates the credential state, and replies. |
| Cleanup after revocation | The App stops using the record after receiving confirmations from both issuers. This abstracts the record's lifecycle rather than proving secure memory erasure. |

The App's seed and backup records are assumed to be provisioned through a trusted local mechanism. Because the manuscript does not fully specify a backup or import protocol, the models do not introduce an additional network protocol for this purpose. Local card authentication during revocation uses the modeled ODA signature relationships, rather than a complete EMV APDU implementation.

## 3. State, Sessions, and Model Scope

- The outer replication `!Device` permits unbounded independent authenticator instances. Each instance has credentials for two distinct RPs sharing the device's msk and seed.
- Each credential supports unbounded assertions and transactions. Each RP within an instance has one card and one credential. The results do not generalize to arbitrarily many issuers or credentials within a device instance.
- UID remains fixed for the account and is not regenerated for each assertion. Separate device instances represent independent account contexts; shared updates by multiple devices for the same account are not modeled.
- The authorized UID-to-PAN association is represented by the trusted login context and role parameters. CID uniqueness follows from fresh generation by an honest authenticator and trusted registration. A complete banking database, malicious enrollment, and general duplicate-CID handling are not separately verified.
- Authenticator and RP counters and credential states use consumable private-channel state tokens, avoiding the use of append-only ProVerif tables as mutable databases. State restoration and response transmission execute in parallel to avoid deadlock from synchronous private-channel communication.
- Each pending challenge has one response-acceptance process. The RP compares the received counter against the latest stored counter obtained from its state.
- The App can revoke after binding completes at both issuers; prior transactions are not required. A separate local `TransactionWitness` process observes two successive transactions for the same credential, with counter transitions 1→2 and 2→3. It does not control transactions or revocation. Its diagnostic notifications are not additional protocol message fields.
- `new x[]` and `noselect` are representation and proof-search hints, not security assumptions. The induction option associated with `noselect` may reduce proof precision; it does not assume an unproved property to be true. No user-supplied security axioms or assumptions declaring attacker non-knowledge are added.

## 4. Security Queries

| Query IDs | Meaning |
|---|---|
| A01–A02 | Each accepted registration corresponds to a unique credential-creation event and agrees with an RP-issued registration challenge. |
| A03–A04 | Each accepted binding corresponds to a unique matching FIDO assertion and a card response agreeing on PAN, card public key, and UN. |
| A05–A06 | Each accepted transaction corresponds to a unique assertion and agrees with a unique issuer transaction context on PAN, account, RP, CID, public key, and challenge. |
| A07 | A successfully verified logical FIDO message and signature originate from an honest signing event using the corresponding private key. |
| A08–A09 | An accepted transaction has a counter check for the corresponding CID and message, and the check satisfies `current > previous`. |
| S01–S08 | Secrecy of msk, seed, derived credential private keys, card private keys, the two issuers' signing private keys, and the two UN derivation keys. |
| P01–P03 | The initially private PAN, UID, and CID in the second model remain non-derivable by the external adversary. |
| R01–R05 | Reachability of registration, binding at counter 1, a transaction at counter 2, two transactions for the same credential at counters 2 and 3, and revocation completion involving both issuers. |

`inj-event` expresses injective event correspondence. A05–A06 address replay and session agreement for accepted assertions and challenges. A08–A09 establish local counter validation. These results should not be expanded into a theorem of global counter uniqueness under arbitrary rollback, cloning, or reset scenarios.

## 5. Assumptions and Analysis Boundaries

1. **Cloud compromise, malicious issuers, and denial-of-service attacks are outside scope.** The queries also do not establish revocation-request authorization, immediate revocation, cross-issuer atomicity, failure recovery, or complete concurrent safety after revocation. Implementing state guards does not itself prove these properties.
2. **Confidentiality is distinct from unlinkability.** A commitment is stable for the same RP and supports equality comparisons. Timing or batching may also create associations. The models contain no equivalence or unlinkability queries and do not establish that an issuer cannot link PAN, CID, and commitment records it already holds.
3. **A commitment is a public matching value.** The revocation payload excludes seed, PAN, UID, CID, and private keys. This does not mean that all authenticator-related identifying or comparable information is absent.
4. **Counter secrecy is not a verification objective.** Counters use `nat`, and the first model exposes signed counters. Security relies on validating their increase. UID and CID are not necessarily permanent secrets in general WebAuthn deployments; P02–P03 apply to the specified external observation boundary.
5. **A07 establishes signature origin and message integrity.** It does not independently establish legal or operational non-repudiation, cardholder intent, or correctness of biometric and user-verification implementations.
6. **Cryptography is idealized symbolically.** Signatures cannot be forged without the appropriate private key. KDF, HMAC, and hash operations have no inverse-recovery rules. Seed and msk are high-entropy secrets. `deriveSK` is public, not an algorithm hidden using `[private]`. Public message extraction from signatures is modeled, so signatures are not treated as encryption.
7. **Collisions in the 32-bit UN are not quantified.** `trunc32` does not represent a finite 32-bit output space. A04 therefore holds under this idealized abstraction; it does not establish that real UN values never collide or provide a concrete security bound.
8. **WebAuthn, CTAP, and EMV are modeled at the logical-message level.** Tuples represent logical responses, not byte-level implementations of `authData || clientDataHash`, COSE, CBOR, origin fields, UP/UV flags, or EMV certificate chains. Trusted Client behavior, correct RP checks, and local verification are explicit premises. The results are not a standards-compliance certification.
9. **PAN is a symbolic secret in the confidentiality model.** The analysis does not quantify PAN-format constraints, finite candidate sets, side channels, error-message leakage, traffic analysis, or external data correlation. An adversary that already knows the PAN belongs to the first model's knowledge setting; the second model's PAN-secrecy conclusion does not apply to that adversary.

## 6. Execution and Result Interpretation

The models were executed using **ProVerif 2.05**. Individual results and execution witnesses are recorded in `results/`, with summaries in `verification_summary.json` and `verification_results.tsv`.

| Model | Security queries: `true` | Reachability queries: expected `false` | Unresolved queries |
|---|---:|---:|---:|
| `01_authentication.pv` | 17 | 5 | 0 |
| `02_revocation_confidentiality.pv` | 11 | 5 | 0 |

Together, the models yield 28 successful security-query instances and 10 reachability witnesses. Some key-secrecy queries are evaluated in both settings; these are not 28 distinct, independent theorems.

Run each model from the repository directory:

```sh
proverif 01_authentication.pv
proverif 02_revocation_confidentiality.pv
```

Alternatively, specify the executable and regenerate the verification summaries:

```sh
python3 verify.py --proverif /absolute/path/to/proverif
```

For `query event(...)`, ProVerif reports whether the event is unreachable. The expected output for R01–R05 is therefore `RESULT not event(...) is false`: a successful execution witness has been found. This is not a security failure. Security queries are expected to return `is true`. A result of `cannot be proved` must be reported as unresolved, not counted as a successful proof.

### Additional Negative Controls

`tests/negative_controls.py` generates two deliberately modified models: one discloses seed and PAN during revocation, and the other omits transaction-signature verification. The tests select a concrete first-binding path at counter 1 to assist trace reconstruction.

The signature test additionally restricts the authenticator to its first two assertions. After a response that was already public, it explicitly publishes the attacker-constructible pair `(response, ok)` and checks whether a transaction at counter 2 can be accepted with the public constant `ok` used as a signature. Such an acceptance would violate A07 because `ok` cannot be constructed by `sign`. These settings introduce neither new secrets nor additional attacker computation capabilities and are not included in the two principal models.

These variants are diagnostic tests, not protocol implementations. The expected outcomes are `false` for the corresponding secrecy or unreachability queries, indicating disclosure or erroneous acceptance. Actual results are stored in `results/negative_controls_final/`.

| Negative control | Observed result | Interpretation |
|---|---|---|
| Explicitly disclose seed and PAN during revocation | Both secrecy queries return `false`, with concrete counterexamples. | Detection of these two information disclosures is confirmed. |
| Omit transaction-signature verification and attempt acceptance of `ok` as a signature | `cannot be proved` | ProVerif produces a symbolic derivation but does not reconstruct a concrete execution trace. This is not counted as successful attack detection, nor as evidence that the modified protocol is secure. |

The unresolved result above concerns only an additional deliberately modified test model. Results for the two principal models are reported in the main verification table.

## References

- [ProVerif](https://bblanche.gitlabpages.inria.fr/proverif/)
- [ProVerif User Manual](https://bblanche.gitlabpages.inria.fr/proverif/manual.pdf)
- [ProVerif 2.05 source archive used for verification](https://bblanche.gitlabpages.inria.fr/proverif/proverif2.05.tar.gz)

ProVerif and OCaml are not bundled with these files. The `.pv` files contain no dependencies on local filesystem paths.
