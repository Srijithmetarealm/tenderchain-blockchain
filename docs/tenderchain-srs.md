# TenderChain — Software Requirements Specification (SRS)

| | |
|---|---|
| System | TenderChain — Public Tender & Procurement (Module 26) |
| Components | `pallet_tender_chain` (Substrate runtime pallet, index 8) and the TenderChain web portal (React + TypeScript) |
| Version | 1.0 |
| Date | 2026-10-20 |
| Companion | `tenderchain-business-logic.md` (rules + 8 worked examples), `tenderchain-integration.md` (event schema) |

---

## 1. Introduction

### 1.1 Purpose

This document specifies the functional and non-functional requirements of
TenderChain, a blockchain system that runs the full public-procurement
lifecycle — from tender notice to signed contract — with integrity guarantees
enforced by the chain rather than by policy.

### 1.2 Scope

In scope:

- Creating, publishing, running and closing tenders of five kinds (RFQ, RFT, EOI, Panel, JobTask).
- Sealed (commit–reveal) and open bidding.
- Q&A, addenda, evaluation, award, standstill, challenge, contract execution, cancellation.
- Bid bonds, eligibility checks, standing-offer panels and call-off orders.
- A web portal that drives every on-chain action.

Out of scope (provided by other modules through integration seams):
document storage (Module 2 DNC), escrow custody (Module 8), reputation scoring
(Module 10), email (Module 13), identity credentials (Module 15), multisig
governance (Module 16), delivery management (Module 25).

### 1.3 Definitions

| Term | Meaning |
|---|---|
| Entity | The procuring organisation that owns the tender |
| Officer | The staff account administering a tender |
| Gate | A block-number deadline processed automatically by the chain |
| Commitment | `blake2_256(bidder ‖ documents_hash ‖ price_schedule ‖ salt)` lodged in place of a readable bid |
| Reveal | Disclosure of the bid preimage during the opening window |
| Standstill | Period after award during which challenges can be lodged |
| Call-off | An order placed against a standing-offer panel |
| Deadline wheel | Storage map of gates keyed by block, processed in `on_initialize` |
| Hash256 | 32-byte blake2-256 hash |

### 1.4 References

- Module 26 work assignment (spec §1–§9)
- `pallets/tender-chain/src/lib.rs`, `types.rs`
- `runtime/src/configs/mod.rs`

---

## 2. Overall description

### 2.1 Product perspective

TenderChain is a FRAME pallet in a Substrate runtime plus a browser portal
using `@polkadot/api`. All confidential content lives off-chain and is
referenced by hash; the chain stores structure, proofs and public metadata.

```
Browser portal ──(polkadot.js extrinsics/queries)──▶ Substrate node
                                                     └─ pallet_tender_chain
                                                          ├─ Bonds        → Module 8
                                                          ├─ Eligibility  → Modules 15/10
                                                          ├─ Delivery     → Module 25
                                                          ├─ Notices      → Module 13
                                                          ├─ Documents    → Module 2
                                                          ├─ Reputation   → Module 10
                                                          └─ Award/Challenge/Policy origins → Module 16
```

### 2.2 User classes

| Class | Description | Privilege |
|---|---|---|
| Procuring entity | Organisation account (multisig in production) | Officer rights + cancellation |
| Procurement officer | Runs the tender | Lifecycle administration |
| Supplier | Any eligible account | Questions, bids, reveals, challenges |
| Evaluator | Appointed scorer | Conflict declaration, scoring |
| Award authority | Governed origin | Award |
| Challenge resolver | Governed origin | Uphold / dismiss challenges |
| Policy authority | Governed origin | Procurement policy |
| Public / auditor | Anyone | Read all state and events |

### 2.3 Operating environment

- Substrate node, 6-second block time.
- Portal served by nginx from `/var/www/tenderchain`; modern browser with the Polkadot.js extension or dev accounts.

### 2.4 Constraints

- Every id is a domain-separated hash, never a sequential counter.
- All collections are bounded (see §5.3).
- Award, challenge resolution and policy must be governed origins, never a single signed key (currently `Root` as stand-in).
- No extrinsic may mutate criteria, weights, gates or eligibility after publication.

### 2.5 Assumptions and dependencies

- Integration modules listed in §1.2 are currently permissive stand-ins (everyone eligible, all documents accepted, bonds reserved on the bidder's own account, no mail, no delivery project).
- Bidders keep their salt and bid content off-chain until reveal.

---

## 3. Functional requirements

Priority: **M** = must, **S** = should. Each requirement maps to an extrinsic or hook.

### 3.1 Tender creation and drafting

| ID | Requirement | Pri |
|---|---|---|
| FR-01 | An officer shall create a tender in `Draft` state specifying entity, kind, bid mode, title, summary, notice hash, criteria hash, weights, five gates, standstill period, required credentials, minimum reputation, bond terms, blind-questions flag and optional EOI shortlist source. (`create_tender`) | M |
| FR-02 | The system shall reject creation if weights do not total exactly 100% (`WeightsInvalid`), contain a duplicate criterion (`DuplicateCriterion`), or exceed 20 criteria. | M |
| FR-03 | The system shall reject gates unless `publishAt < questionsCloseAt < submissionCloseAt ≤ openingAt < openingEndAt` (`GateOrderInvalid`). | M |
| FR-04 | The system shall allow `BidMode::Open` only for RFQ tenders (`OpenBidNotPermitted`). | M |
| FR-05 | The system shall reject an empty title, a title over 128 bytes or a summary over 512 bytes, without truncation. | M |
| FR-06 | If `shortlist_from` is set it shall reference an existing EOI tender (`NotAnEoi`). | M |
| FR-07 | The officer shall be able to replace criteria and weights only while in `Draft` (`amend_criteria`; otherwise `CriteriaLocked`). | M |
| FR-08 | The officer or entity shall be able to delete a tender that was never published (`delete_tender`; otherwise `TenderNotDeletable`). | S |
| FR-09 | The tender id shall be `blake2_256("tenderchain/tender" ‖ officer ‖ entity ‖ nonce ‖ created_at)`. | M |

### 3.2 Publication

| ID | Requirement | Pri |
|---|---|---|
| FR-10 | The officer shall publish a draft no earlier than `publishAt` (`PublishTooEarly`) and before `questionsCloseAt`. (`publish_tender`) | M |
| FR-11 | Publication shall freeze criteria, weights, gates, eligibility, bond terms, shortlist source and a snapshot of the current procurement policy. | M |
| FR-12 | Publication shall validate the policy against the actual publication block (`SubmissionPeriodTooShort`, `StandstillTooShort`). | M |
| FR-13 | Publication shall schedule `QuestionsClose`, `SubmissionClose` and `OpeningEnd` on the deadline wheel and emit `TenderPublished` with the close block. | M |

### 3.3 Questions and addenda

| ID | Requirement | Pri |
|---|---|---|
| FR-14 | Any account shall be able to ask a question while in `QaWindow` and before `questionsCloseAt` (`ask_question`; otherwise `QuestionsClosed`). | M |
| FR-15 | If `blind_questions` is set, the system shall store only `blake2_256(asker ‖ salt)` as author and never the account. | M |
| FR-16 | The officer shall answer a question once; answers are public and immutable (`AlreadyAnswered`). | M |
| FR-17 | The officer shall publish addenda in `QaWindow`/`Submission`, optionally extending the close; the close shall never move earlier (`CloseCannotShorten`). | M |
| FR-18 | An extension shall shift `openingAt` and `openingEndAt` by the same amount. | M |
| FR-19 | Addenda shall respect the policy snapshot: response window (`AddendumNeedsExtension`) and extension cap (`CloseExtensionTooLong`). | M |
| FR-20 | Limits: 256 questions, 32 addenda per tender. | M |

### 3.4 Bidding

| ID | Requirement | Pri |
|---|---|---|
| FR-21 | A supplier shall lodge a sealed commitment in `QaWindow` or `Submission` before `submissionCloseAt` (`commit_bid`). | M |
| FR-22 | A supplier shall submit an open bid (documents + price schedule, readable immediately) on open-mode RFQs (`submit_open_bid`); it is recorded as validly revealed. | M |
| FR-23 | The system shall reject a second bid from the same supplier (`CommitmentExists`), a bid from an appointed evaluator (`EvaluatorIsBidder`), a non-shortlisted bidder on a shortlist-restricted tender (`NotShortlisted`), and an ineligible bidder (`NotEligible`). | M |
| FR-24 | The system shall lock the bond amount on submission (`BondRequired` if insufficient funds). | M |
| FR-25 | A supplier shall withdraw before close; the bond shall be returned or forfeited per `forfeit_on_withdrawal`. (`withdraw_commitment`) | M |
| FR-26 | Maximum 128 bidders per tender, 64 price lines per bid. | M |

### 3.5 Closing, opening and reveal

| ID | Requirement | Pri |
|---|---|---|
| FR-27 | At `questionsCloseAt` the chain shall move `QaWindow → Submission` automatically and emit `SubmissionOpened`. | M |
| FR-28 | At `submissionCloseAt` the chain shall move to `Closed` automatically and emit `TenderClosed`. | M |
| FR-29 | The officer shall open a closed tender at or after `openingAt`, only if at least `MinRevealWindow` (100 blocks) remain before `openingEndAt` (`open_tender`; `OpeningNotReached`, `RevealWindowTooShort`). The participant count becomes public. | M |
| FR-30 | A sealed bidder shall reveal once during `Opening`; a mismatch shall void the bid but keep it on record with `RevealMismatch`. (`reveal_bid`) | M |
| FR-31 | At `openingEndAt` the chain shall move to `Evaluation`; if opened, it forfeits bonds of non-revealed/mismatched bids when `forfeit_on_non_reveal`; if never opened, it returns all bonds. | M |

### 3.6 Evaluation

| ID | Requirement | Pri |
|---|---|---|
| FR-32 | The officer shall appoint evaluators (max 16) with a credential reference; the officer, entity and bidders cannot be appointed. | M |
| FR-33 | An evaluator shall lodge a conflict-of-interest declaration before activation and cannot change it once active. | M |
| FR-34 | The officer shall activate an evaluator only after a declaration exists (`ConflictNotDeclared`). | M |
| FR-35 | An active evaluator shall submit one scoresheet per valid bid in `Evaluation`, covering exactly the locked criteria once each, each score 0–100. Resubmission replaces the sheet. | M |
| FR-36 | The system shall emit `ScoreVarianceFlagged` when two evaluators' scores on a criterion differ by more than 30. | M |
| FR-37 | Ranking shall be `Σ(average score per criterion × weight%)`, highest first, excluding invalid and unrevealed bids. | M |

### 3.7 Award, standstill and challenge

| ID | Requirement | Pri |
|---|---|---|
| FR-38 | The award authority shall award one or more distinct bidders with valid reveals, given ≥ 3 active evaluators, no open challenge, and a non-EOI tender. (`award`) | M |
| FR-39 | The award shall publish the full ranking, rationale hash and `standstill_end = now + standstill_period`, and schedule `StandstillEnd`. | M |
| FR-40 | A losing bidder shall lodge a challenge with readable grounds (≤ 2048 bytes, non-empty) and optional evidence until `standstill_end`; the tender moves to `Challenged`. Max 32 challenges. | M |
| FR-41 | The resolver shall uphold (tender → `Evaluation`) or dismiss (tender → `Awarded` when no challenges remain open) with a readable ruling. | M |
| FR-42 | For EOI tenders the officer shall publish a shortlist of distinct validly-revealed suppliers instead of awarding; tender → `Shortlisted`. | M |

### 3.8 Execution, panels, cancellation, policy

| ID | Requirement | Pri |
|---|---|---|
| FR-43 | The officer shall execute the award after `standstill_end` with no open challenge, giving a contract hash. (`execute_award`) | M |
| FR-44 | Execution shall return all remaining bonds, instantiate a delivery project per awardee, record reputation facts and move to `Contracted`. | M |
| FR-45 | For Panel tenders, execution shall admit every awardee to the panel pool. | M |
| FR-46 | The officer shall place call-off orders only with panel members (max 256 per panel), stored publicly. | M |
| FR-47 | Only the entity shall cancel a non-terminal tender with a reason hash; all bonds are returned and remaining gates ignored. | M |
| FR-48 | The policy authority shall set the procurement policy; it applies only to tenders published afterwards. | M |

### 3.9 Portal requirements

| ID | Requirement | Pri |
|---|---|---|
| FR-49 | The portal shall provide a 5-step create wizard: Basics, Criteria & weights, Gates, Eligibility & bond, Review. | M |
| FR-50 | The Gates step shall accept each gate as a time from now (minutes / hours / days), convert it to blocks using the chain's average block time, and display the resulting block. | M |
| FR-51 | The wizard shall validate gate order, weight total, `MinRevealWindow` and `MinStandstillPeriod` client-side and explain every blocking issue. | M |
| FR-52 | The portal shall display live chain constants and the current block. | S |
| FR-53 | The portal shall compute commitments locally and never send salt or bid content before reveal. | M |
| FR-54 | Every portal action shall map to exactly one extrinsic and show the chain's error in readable form. | S |

---

## 4. Lifecycle state requirements

| State | Entered by | Allowed actions |
|---|---|---|
| Draft | `create_tender` | amend criteria, publish, delete, cancel, appoint evaluators |
| QaWindow | `publish_tender` | ask/answer, addendum, bid, withdraw, cancel |
| Submission | gate `QuestionsClose` | answer, addendum, bid, withdraw, cancel |
| Closed | gate `SubmissionClose` | open, cancel |
| Opening | `open_tender` | reveal, cancel |
| Evaluation | gate `OpeningEnd` / upheld challenge | declare, activate, score, award, shortlist (EOI), cancel |
| Awarded | `award` / dismissed challenge | challenge, execute (after standstill), cancel |
| Challenged | `lodge_challenge` | further challenges, resolve, cancel |
| Shortlisted | `publish_shortlist` | terminal |
| Contracted | `execute_award` | call-offs (Panel) — terminal |
| Cancelled | `cancel_tender` | terminal (delete if never published) |

---

## 5. Non-functional requirements

### 5.1 Integrity and security

| ID | Requirement |
|---|---|
| NFR-01 | No sealed bid content shall be readable on chain before reveal. |
| NFR-02 | The bidder's account shall be part of the commitment preimage so commitments cannot be replayed by others. |
| NFR-03 | No officer-controlled action shall be able to move a deadline earlier or close a phase. |
| NFR-04 | Award, challenge resolution and policy shall require governed origins. |
| NFR-05 | Blinded question authors shall not be derivable from any stored value or id. |
| NFR-06 | Bonds shall never be forfeited when revealing was impossible (tender not opened) or on cancellation. |
| NFR-07 | Forfeits shall move funds directly from reserve, so a partial failure never reports a bond as forfeited when it was not. |

### 5.2 Transparency and auditability

| ID | Requirement |
|---|---|
| NFR-08 | Every lifecycle transition shall emit an event; a tender shall be reconstructible from events alone. |
| NFR-09 | Invalid reveals, challenge grounds, rulings and cancellation reasons shall remain permanently on the public record. |
| NFR-10 | Each tender shall store `created_at` for ordering, since ids carry no order. |

### 5.3 Performance and bounded resources

| ID | Requirement |
|---|---|
| NFR-11 | All storage collections shall be bounded (bidders 128, evaluators 16, criteria 20, questions 256, addenda 32, challenges 32, price lines 64, call-offs 256, credentials 16). |
| NFR-12 | `on_initialize` shall process at most 16 gate transitions per block, deferring the rest up to 16 blocks ahead; a gate that cannot be placed shall emit `GateDropped`. |
| NFR-13 | At most 64 deadlines per block may be scheduled (`DeadlineWheelFull`). |
| NFR-14 | Every extrinsic shall have a benchmarked weight. |

### 5.4 Reliability and maintainability

| ID | Requirement |
|---|---|
| NFR-15 | Each integration module shall be connected by changing one runtime `Config` line, with no pallet change. |
| NFR-16 | Notification failure shall never block the lifecycle. |
| NFR-17 | The pallet shall be covered by unit tests (`tests.rs`) and benchmarks (`benchmarking.rs`). |
| NFR-18 | Portal deployment shall be atomic (staged directory swap) with one-command rollback. |

### 5.5 Usability

| ID | Requirement |
|---|---|
| NFR-19 | The portal shall show all times in human units alongside block numbers. |
| NFR-20 | A disabled action shall always list why it is disabled. |
| NFR-21 | The portal shall work at phone width. |

---

## 6. External interfaces

### 6.1 Extrinsics (`api.tx.tenderChain.*`)

| # | Call | Origin |
|---|---|---|
| 0 | `create_tender` | Signed (officer) |
| 1 | `publish_tender` | Officer / entity |
| 2 | `ask_question` | Signed |
| 3 | `answer_question` | Officer / entity |
| 4 | `publish_addendum` | Officer / entity |
| 5 | `commit_bid` | Signed (bidder) |
| 6 | `withdraw_commitment` | Bidder |
| 7 | `open_tender` | Officer / entity |
| 8 | `reveal_bid` | Bidder |
| 9 | `appoint_evaluator` | Officer / entity |
| 10 | `declare_conflict` | Evaluator |
| 11 | `activate_evaluator` | Officer / entity |
| 12 | `submit_scores` | Evaluator |
| 13 | `award` | `AwardOrigin` |
| 14 | `lodge_challenge` | Losing bidder |
| 15 | `resolve_challenge` | `ChallengeResolverOrigin` |
| 16 | `execute_award` | Officer / entity |
| 17 | `cancel_tender` | Entity |
| 18 | `call_off` | Officer / entity |
| 19 | `publish_shortlist` | Officer / entity |
| 20 | `submit_open_bid` | Signed (bidder) |
| 21 | `amend_criteria` | Officer / entity |
| 22 | `set_policy` | `PolicyOrigin` |
| 23 | `delete_tender` | Officer / entity |

### 6.2 Storage (read interface)

`Tenders`, `Questions`, `QuestionCount`, `Addenda`, `Participants`,
`BidCommitments`, `Reveals`, `EvaluatorSet`, `Scores`, `Outcomes`,
`Challenges`, `Shortlist`, `ShortlistSource`, `PanelPool`, `PanelTender`,
`CallOffs`, `Policy`, `DeadlineWheel`.

### 6.3 Events

See `tenderchain-integration.md` §2 for the complete schema.

### 6.4 Integration traits

`BondManager`, `EligibilityProvider`, `DeliveryInstantiator`, `Notifier`,
`DocumentAnchor`, `ReputationSink` — see business-logic doc §8 for current
stand-ins.

---

## 7. Acceptance criteria

The system is accepted when each worked example in
`tenderchain-business-logic.md` §5 runs end to end on a dev node with the
stated outcomes:

| Test | Example | Pass condition |
|---|---|---|
| AT-01 | Ex 1 RFT happy path | Tender reaches `Contracted`; ranking 7800 / 7600 / 6200; all bonds returned |
| AT-02 | Ex 2 RFQ open bid | Bids readable at submission; open bid on RFT rejected |
| AT-03 | Ex 3 EOI → RFT | Non-shortlisted bidder gets `NotShortlisted` |
| AT-04 | Ex 4 Panel | 3 members admitted at execution; call-off to non-member rejected |
| AT-05 | Ex 5 JobTask | `DeliveryInstantiated` per awardee |
| AT-06 | Ex 6 Q&A / addendum | Blind author stored as hash; gates shift by extension; cap enforced |
| AT-07 | Ex 7 Challenge | Variance flagged; upheld → re-award; dismissed → execution |
| AT-08 | Ex 8 Bonds / cancel | Forfeits for withdrawal, mismatch, non-reveal; all returned on cancel; draft deleted |

---

## 8. Traceability

| Requirement group | Implementation | Tests |
|---|---|---|
| FR-01 – FR-48 | `pallets/tender-chain/src/lib.rs` | `pallets/tender-chain/src/tests.rs` |
| FR-49 – FR-54 | `frontend/tenderchain/src/pages/CreateTenderWizard.tsx` and pages | `frontend/tenderchain/TESTING.md`, `scripts/e2e.ts` |
| NFR-11 – NFR-14 | `runtime/src/configs/mod.rs`, `weights.rs` | `benchmarking.rs` |
