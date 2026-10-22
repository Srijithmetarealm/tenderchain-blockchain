# TenderChain — Business Logic & Worked Examples

How public procurement works on TenderChain (pallet `pallet_tender_chain`,
"Module 26"), explained as business rules and then walked through eight
example tenders that between them use every feature the chain offers.

Companion documents:

- `tenderchain-srs.md` — Software Requirements Specification
- `tenderchain-integration.md` — event schema and integration details
- `../../frontend/tenderchain/README.md` — the web portal
- `../../frontend/tenderchain/TESTING.md` — click-by-click test script

---

## 1. Why the chain, and what it guarantees

Public procurement usually goes wrong in a few familiar ways: criteria are
rewritten after bids are seen, bids leak before opening, deadlines move because
an official moved them, nobody can tell who scored what, and awards cannot be
contested. On TenderChain those failures are **not possible through the API**,
not merely against policy:

| Guarantee | How it is enforced |
|---|---|
| **Rules are locked before any bid exists** | `publish_tender` freezes criteria, weights, gates, eligibility, bond terms and the procurement policy. No call can change them afterwards. |
| **Sealed bids are truly sealed** | The chain holds only `blake2_256(bidder ‖ documents ‖ prices ‖ salt)` until opening. Nothing readable exists to leak. |
| **The chain keeps the clock** | Deadlines are processed automatically at the start of each block (the *deadline wheel*). No official can close early or late. |
| **Evaluation is attributed** | Each scoresheet is signed by a named evaluator, who must first declare conflicts of interest. Large disagreements are flagged automatically. |
| **Awards can be contested** | Every award opens a standstill window. Any losing bidder can lodge a challenge, which suspends the contract until it is ruled on. |
| **Nothing disappears silently** | Cancellation reasons, challenge grounds, rulings and invalid reveals all stay on the public record permanently. |

---

## 2. Who does what (roles)

| Role | Who it is | What they can do |
|---|---|---|
| **Procuring entity** | The government body / organisation buying (ideally a multisig account) | Everything the officer can do, plus **cancel** a tender |
| **Procurement officer** | Staff member running the tender | Create, amend draft criteria, publish, answer questions, issue addenda, open bids, appoint/activate evaluators, publish EOI shortlists, execute the award, place call-offs, delete unpublished drafts |
| **Supplier / bidder** | Any account meeting eligibility | Ask questions, commit or submit bids, withdraw, reveal, challenge an award |
| **Evaluator** | Appointed by the officer | Declare conflicts, then score bids |
| **Award authority** | Governed origin (currently `Root`; Multisig in production) | Make the award |
| **Challenge resolver** | Governed origin (currently `Root`) | Uphold or dismiss challenges |
| **Policy authority** | Governed origin (currently `Root`) | Set the deployment-wide procurement policy |

**Separation rules** (enforced on chain):

- An evaluator may not bid on the same tender, and a bidder may not be appointed evaluator (`EvaluatorIsBidder`).
- The officer and the entity may never evaluate their own tender (`OfficerCannotEvaluate`).
- An awardee cannot challenge their own award (`AwardeeCannotChallenge`).
- Only accounts that actually bid can challenge (`NotAParticipant`).

---

## 3. The five kinds of tender

| Kind | Use it for | Special behaviour |
|---|---|---|
| **RFQ** — Request for Quotation | Low-value, price-driven buys | The only kind allowed to use **Open** bidding (no sealing) |
| **RFT** — Request for Tender | Full competitive tender | Sealed bids, weighted criteria. Can be restricted to an EOI shortlist |
| **EOI** — Expression of Interest | Stage one of a two-stage procurement | Ends in a **shortlist**, not an award |
| **Panel** — Standing offer | Several suppliers on a pre-approved pool | Multiple awardees; officer places **call-off orders** afterwards |
| **JobTask** | Small work package | Award hands off to Module 25 (Work Task) to create a delivery project |

Two bid modes:

- **Sealed** (default, any kind) — commit a hash, reveal after opening.
- **Open** (RFQ only) — bid content is public the moment it is submitted; no reveal step.

---

## 4. The lifecycle

```
create_tender ─▶ Draft ──(amend_criteria / delete_tender allowed here)
                   │
publish_tender ─▶ QaWindow          questions + bids accepted
                   │  ⏱ questionsCloseAt (automatic)
                   ▼
                 Submission         bids accepted, no new questions
                   │  ⏱ submissionCloseAt (automatic)
                   ▼
                 Closed
                   │  open_tender (officer, at/after openingAt)
                   ▼
                 Opening            sealed bidders reveal
                   │  ⏱ openingEndAt (automatic) — non-reveal bonds settled
                   ▼
                 Evaluation         evaluators score
                   │
        ┌──────────┴───────────┐
   award (authority)     publish_shortlist (EOI only)
        ▼                      ▼
     Awarded ◀──────┐      Shortlisted  (terminal)
        │  lodge_challenge  │
        ▼                   │ dismissed
     Challenged ────────────┘
        │ upheld ──▶ back to Evaluation (re-score, re-award)
        │
        │ ⏱ standstill ends, no open challenge
        ▼
execute_award ─▶ Contracted  (terminal; bonds returned, delivery created)

cancel_tender (entity) ─▶ Cancelled  from any non-terminal state
```

### 4.1 The gates (deadlines)

Every tender has five block-number gates plus a standstill length. The portal
lets the officer enter them as **times** ("1 minute", "2 days"), converting at
the chain's 6-second block time (10 blocks = 1 minute, 14,400 blocks = 1 day).

| Gate | Meaning | Rule |
|---|---|---|
| `publishAt` | Earliest block the notice can go out | `publish_tender` fails before it (`PublishTooEarly`) |
| `questionsCloseAt` | End of Q&A | must be after `publishAt` |
| `submissionCloseAt` | Bids close | must be after `questionsCloseAt` |
| `openingAt` | Earliest block the officer can open | at or after `submissionCloseAt` |
| `openingEndAt` | End of the reveal window | after `openingAt`; when the officer clicks Open, at least **MinRevealWindow = 100 blocks (10 min)** must remain |
| `standstillPeriod` | Challenge window length after award | at least the policy's `min_standstill` |

Bids are accepted in both `QaWindow` and `Submission` — the Q&A gate only stops
new questions.

### 4.2 Chain constants (this runtime)

| Constant | Value | Effect |
|---|---|---|
| Block time | 6 s | Converts times ↔ blocks |
| `MinRevealWindow` | 100 blocks (10 min) | Minimum reveal time left when opened |
| `MinEvaluators` | 3 | Active evaluators needed before award |
| `MaxScore` | 100 | Score range per criterion is 0–100 |
| `ScoreVarianceThreshold` | 30 | Two evaluators more than 30 points apart on a criterion → flagged |
| `MaxBidders` | 128 | Per tender |
| `MaxEvaluators` | 16 | Per tender |
| `MaxWeights` (criteria) | 20 | Per tender |
| `MaxQuestions` / `MaxAddenda` / `MaxChallenges` | 256 / 32 / 32 | Per tender |
| `MaxPriceLines` | 64 | Per bid |
| `MaxCallOffs` | 256 | Per panel |
| Title / summary | 128 / 512 bytes | Rejected, never truncated |

### 4.3 Procurement policy

Set by governance via `set_policy`; each tender takes a **snapshot at
publication**, so later policy changes never affect a live tender.

| Field | Rule |
|---|---|
| `min_standstill` | Standstill period must be at least this |
| `min_submission_period` | `submissionCloseAt − publication block` must be at least this |
| `addendum_response_window` | After an addendum, bidders must still have at least this long before close; otherwise the addendum must extend the close |
| `max_close_extension` | Cap on how far addenda can push the close beyond the originally published close |

The default policy is fully permissive (all zero, no cap).

### 4.4 Scoring and ranking

1. Criteria weights are whole percentages and must total **exactly 100**.
2. Each active evaluator submits one scoresheet per bidder covering **every**
   criterion exactly once, each score 0–100. Resubmitting replaces your sheet.
3. For each bidder and criterion, the chain averages all evaluators' scores
   (integer division), multiplies by the weight, and sums:

   `total = Σ ( avg_score(criterion) × weight% )`  → range 0 – 10,000

4. Bidders are ranked highest total first. Voided and non-revealed bids are
   left out of the ranking.
5. The **full ranking is published** with the award. The award authority picks
   the awardee(s) — normally the top-ranked — and must give a rationale document.

### 4.5 Bid bonds

| Situation | Bond outcome |
|---|---|
| Bidder withdraws before close, `forfeit_on_withdrawal = false` | Returned |
| Bidder withdraws before close, `forfeit_on_withdrawal = true` | Forfeited to the entity |
| Sealed bid not revealed, or revealed with a mismatch, `forfeit_on_non_reveal = true` | Forfeited at `openingEndAt` |
| Same, `forfeit_on_non_reveal = false` | Returned at contract execution |
| Officer never opened the tender | **All bonds returned** — bidders cannot be punished for an officer's inaction |
| Tender cancelled | All bonds returned |
| Contract executed | All remaining bonds returned |

Forfeitures are reported to the reputation module as `BondForfeited`.

---

## 5. Worked examples

Times below assume the 6-second block time. "T+0" is the block the tender is
created. Account names are illustrative.

### Example 1 — Road resurfacing (RFT, sealed, full happy path)

**Shows:** create → publish → sealed commit → open → reveal → conflict
declarations → scoring → weighted ranking → award → standstill → contract.

| Field | Value |
|---|---|
| Entity / Officer | Colombo Municipal Council / Officer Nimal |
| Kind / Mode | RFT / Sealed |
| Criteria | Technical capability 40%, Price 40%, Past performance 20% |
| Gates | publish T+1 min, Q&A closes T+3 days, bids close T+21 days, opening T+21 days → T+23 days |
| Standstill | 10 days |
| Bond | 50,000 LKR, forfeit on non-reveal |

**Flow**

1. Officer creates the draft, then publishes at T+1 min. Criteria and gates are now frozen.
2. Three contractors — **Alpha Builders**, **Beta Roads**, **Gamma Infra** — each compute their commitment hash off-chain and call `commit_bid`. 50,000 LKR is reserved from each. The chain shows only that *a* bid exists from each.
3. At T+21 days the chain moves the tender to `Closed` by itself.
4. Officer calls `open_tender`; `OpeningStarted` reports 3 participants. Each bidder calls `reveal_bid` with documents, price schedule and salt; all three match.
5. Officer appoints three evaluators (Dr Perera, Eng Silva, Ms Fernando). Each lodges a conflict declaration; the officer activates them.
6. At T+23 days the chain moves to `Evaluation`. Evaluators score:

| Bidder | Technical (avg) | Price (avg) | Past perf. (avg) | Total |
|---|---|---|---|---|
| Alpha | 80 × 40 = 3,200 | 70 × 40 = 2,800 | 90 × 20 = 1,800 | **7,800** |
| Beta | 70 × 40 = 2,800 | 90 × 40 = 3,600 | 60 × 20 = 1,200 | **7,600** |
| Gamma | 60 × 40 = 2,400 | 60 × 40 = 2,400 | 70 × 20 = 1,400 | **6,200** |

7. The award authority awards **Alpha** with a rationale document. The full ranking is published; a 10-day standstill opens.
8. Nobody challenges. After the standstill, the officer calls `execute_award` with the signed contract hash. State → `Contracted`, all three bonds are returned, and a delivery project is created for Alpha.

### Example 2 — Office stationery (RFQ, open bidding)

**Shows:** open bid mode, price-weighted RFQ, no reveal step, short timetable.

| Field | Value |
|---|---|
| Kind / Mode | RFQ / **Open** |
| Criteria | Price 70%, Delivery time 30% |
| Gates | publish T+1 min, Q&A closes T+3 min, bids close T+9 min, opening T+9 → T+24 min |
| Bond | 0 |

**Flow**

1. Three stationers call `submit_open_bid` with their documents and price lines. Each bid is **readable immediately** and recorded as already revealed.
2. Bids close at T+9 min. The officer opens; there is nothing to reveal.
3. At T+24 min the tender enters `Evaluation`; three evaluators score and the lowest-priced/fastest supplier tops the ranking.
4. Award → standstill → execute, as in Example 1.

**Rule checks:** trying `BidMode::Open` on an RFT fails with `OpenBidNotPermitted`.
Calling `reveal_bid` on an open tender fails with `WrongBidMode`.

### Example 3 — Hospital IT system (EOI → shortlist → follow-on RFT)

**Shows:** two-stage procurement, shortlist restriction.

**Stage 1 — EOI**

| Field | Value |
|---|---|
| Kind | EOI, Sealed |
| Criteria | Experience 50%, Certifications 30%, Capacity 20% |

1. Six vendors express interest (commit + reveal).
2. In `Evaluation`, the officer calls `publish_shortlist` with **MedSoft**, **CareTech** and **HealthIQ**. Only vendors with a valid reveal can be listed, each once.
3. State → `Shortlisted` (terminal). An EOI can never be *awarded* (`EoiUsesShortlist`).

**Stage 2 — RFT**

1. The officer creates an RFT with `shortlist_from = <EOI tender id>`. This link is frozen at publication.
2. MedSoft, CareTech and HealthIQ bid normally. A fourth vendor who was not shortlisted tries `commit_bid` → **rejected** (`NotShortlisted`).
3. The RFT runs to award and contract like Example 1.

### Example 4 — Laptop supply panel (Panel / standing offer, call-offs)

**Shows:** multi-award, panel admission after standstill, call-off orders.

| Field | Value |
|---|---|
| Kind | Panel, Sealed |
| Criteria | Unit price 50%, Warranty 25%, Support SLA 25% |
| Standstill | 7 days |

1. Five suppliers bid; three evaluators score.
2. The award authority awards **three** suppliers at once — TechLanka, CompuWorld, ByteMart. One `Awarded` event is emitted per awardee. Duplicate awardees are rejected (`DuplicateAwardee`).
3. After the standstill, `execute_award` admits all three to the panel pool (`PanelMemberAdmitted`). Admission waits until execution so that an upheld challenge cannot leave wrongly admitted suppliers in the pool.
4. Over the following months the officer places orders:
   - `call_off(panel, TechLanka, order#1 – 40 laptops for Kandy office)`
   - `call_off(panel, ByteMart, order#2 – 15 laptops for Galle office)`
5. Each order is stored publicly in `CallOffs`. Ordering from a supplier who is not on the panel fails (`NotPanelMember`); only the tender's officer/entity can place orders.

### Example 5 — School roof repair (JobTask, quick work package)

**Shows:** lightweight tender, delivery hand-off to Module 25, eligibility rules.

| Field | Value |
|---|---|
| Kind | JobTask, Sealed |
| Eligibility | Credential `ICTAD-Grade-C5`, minimum reputation 40 |
| Bond | 5,000 LKR, forfeit on withdrawal |
| Timetable | ~25-minute test run (portal default) |

1. Only contractors holding the credential and reputation can bid (`NotEligible` otherwise). In this runtime the eligibility module is a stand-in that accepts everyone; once Identity/Reputation modules are connected, the check is enforced automatically.
2. Two contractors bid, reveal, and are scored.
3. On `execute_award`, the chain calls the Work Task module, which creates a delivery project; the reference appears in `DeliveryInstantiated`.
4. Reputation facts are recorded: `ContractWon` for the contractor, `ContractAwarded` for the entity.

### Example 6 — Bridge design consultancy (Q&A, blind questions, addendum)

**Shows:** blinded question authorship, public answers, addendum extending the close, procurement policy.

| Field | Value |
|---|---|
| Kind | RFT, Sealed, **blind questions on** |
| Policy in force | `min_submission_period` 14 days, `addendum_response_window` 5 days, `max_close_extension` 10 days |
| Gates | bids close T+20 days, opening T+20 → T+22 days |

1. A small firm asks "Is a geotechnical survey included?" via `ask_question`. Because blind questions are on, only `blake2_256(asker ‖ salt)` is stored — larger competitors cannot see who asked. The firm can later prove authorship to a probity observer by sharing the salt.
2. The officer answers. The answer is published to **everyone at once**; there is no private clarification. Once given, it cannot be rewritten (`AlreadyAnswered`).
3. On day 17 the officer issues an addendum adding the survey to scope. Only 3 days would remain, below the 5-day response window, so an addendum without extension fails (`AddendumNeedsExtension`).
4. The officer re-issues it extending the close to T+24 days. The chain shifts **opening and opening-end by the same 4 days**, so the reveal window keeps its length.
5. Trying to extend to T+31 days fails — it exceeds the 10-day cap over the published close (`CloseExtensionTooLong`). Trying to move the close *earlier* fails (`CloseCannotShorten`).
6. A question asked after `questionsCloseAt` fails (`QuestionsClosed`).

### Example 7 — Ambulance procurement (score variance, challenge upheld, re-award)

**Shows:** variance flag, challenge during standstill, upheld remittal, re-scoring, second award, challenge dismissed.

1. Three ambulance suppliers bid: **MediMotors**, **RescueAuto**, **LifeLine**.
2. On "Technical compliance", Evaluator A gives MediMotors 90 and Evaluator B gives 50 — a 40-point spread, above the threshold of 30. The chain emits `ScoreVarianceFlagged` for probity review.
3. MediMotors is ranked first and awarded; a 10-day standstill opens.
4. On day 4, **RescueAuto** lodges a challenge with written grounds ("MediMotors' vehicle fails the mandatory 4×4 requirement") and an evidence document. State → `Challenged`; execution is suspended (`execute_award` fails with `ChallengeOpen`).
5. The challenge resolver **upholds** it with a written ruling. The tender goes back to `Evaluation`. Grounds and ruling stay on chain permanently.
6. Evaluators resubmit corrected sheets; RescueAuto now ranks first. The authority makes a new award to RescueAuto (only possible once no challenge is open). A new standstill opens.
7. MediMotors challenges the new award; the resolver **dismisses** it. The tender returns to `Awarded`, and after the standstill the officer executes the contract with RescueAuto.

### Example 8 — Water pumps (bond forfeiture, invalid reveal, withdrawal, cancellation)

**Shows:** every bond outcome, invalid reveal kept on record, cancellation, draft deletion.

**Part A — bonds**

Terms: bond 100,000 LKR, forfeit on non-reveal **and** on withdrawal. Five bidders commit.

| Bidder | What happens | Result |
|---|---|---|
| PumpCo | Reveals correctly | Valid bid; bond returned at contract |
| AquaFlow | Withdraws before close | Bond **forfeited** to the entity (`BondForfeited`) |
| HydroMax | Reveals with a different price than committed | `RevealMismatch` + `BidRevealed{valid:false}`; bid voided but **kept on record**; bond forfeited at opening end |
| FlowTech | Never reveals | Bond forfeited at opening end |
| WaterWorks | Reveals correctly | Valid bid; bond returned at contract |

Only PumpCo and WaterWorks are ranked. Scoring a voided bid fails (`BidNotScoreable`).

If instead the officer had **never opened** the tender, the chain would return
every bond at `openingEndAt` — nobody could reveal, so nobody is punished.

**Part B — cancellation**

A second pump tender is published, but funding is withdrawn mid-submission.
The **entity** (not the officer — `NotEntity`) calls `cancel_tender` with a
reason document. All bonds are returned, the reason is permanently public,
remaining deadlines are ignored, and a `TenderCancelled` reputation fact is
recorded against the entity because the tender had been published.

**Part C — deleting a draft**

A third tender was created with a typo and never published. The officer calls
`delete_tender`; it is removed completely, since it never reached the public
record. Once published, a tender can **only** be cancelled, never deleted
(`TenderNotDeletable`).

---

## 6. Feature coverage matrix

| Feature | Ex1 | Ex2 | Ex3 | Ex4 | Ex5 | Ex6 | Ex7 | Ex8 |
|---|---|---|---|---|---|---|---|---|
| RFT | ✓ | | ✓ | | | ✓ | ✓ | ✓ |
| RFQ / open bidding | | ✓ | | | | | | |
| EOI + shortlist | | | ✓ | | | | | |
| Panel + call-offs | | | | ✓ | | | | |
| JobTask + delivery | | | | | ✓ | | | |
| Sealed commit / reveal | ✓ | | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Eligibility / credentials | | | ✓ | | ✓ | | | |
| Q&A, blind questions | | | | | | ✓ | | |
| Addendum + close extension | | | | | | ✓ | | |
| Procurement policy | | | | | | ✓ | | |
| Conflict declaration + activation | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Weighted ranking | ✓ | ✓ | | ✓ | ✓ | | ✓ | ✓ |
| Score variance flag | | | | | | | ✓ | |
| Multi-award | | | | ✓ | | | | |
| Challenge upheld / dismissed | | | | | | | ✓ | |
| Bond return / forfeit | ✓ | | | | ✓ | | | ✓ |
| Invalid reveal on record | | | | | | | | ✓ |
| Cancel / delete | | | | | | | | ✓ |
| Reputation facts | ✓ | | | | ✓ | | | ✓ |

---

## 7. Common errors and what they mean

| Error | Business meaning |
|---|---|
| `GateOrderInvalid` | Deadlines are out of order, or publishing after Q&A has already closed |
| `WeightsInvalid` | Criteria weights do not add up to 100% |
| `PublishTooEarly` | Publish block not reached yet |
| `SubmissionPeriodTooShort` / `StandstillTooShort` | Timetable breaks the procurement policy |
| `CriteriaLocked` | Tried to change criteria after publication |
| `SubmissionClosed` | Bid or withdrawal after the close |
| `CommitmentExists` | One bid per bidder per tender |
| `OpeningNotReached` / `RevealWindowTooShort` | Opened too early, or so late that under 10 minutes of reveal time would remain |
| `RevealWindowClosed` | Reveal outside the opening window |
| `ConflictNotDeclared` | Evaluator has not declared conflicts / not activated |
| `CriteriaMismatch` | Scoresheet does not cover exactly the locked criteria |
| `TooFewEvaluators` | Fewer than 3 active evaluators at award |
| `InvalidAwardee` | Awardee has no valid revealed bid |
| `ChallengeOpen` | Execution or re-award blocked by an unresolved challenge |
| `StandstillActive` | Execution attempted before the standstill ended |
| `ChallengeWindowClosed` | Challenge lodged after the standstill |
| `NotShortlisted` | Bidder not on the EOI shortlist for this RFT |
| `NotEligible` | Bidder lacks required credentials or reputation |

---

## 8. Integration status (what is a stand-in today)

| Module | Role in TenderChain | Current behaviour |
|---|---|---|
| 8 Escrow | Holds bid bonds | Bonds are reserved on the bidder's own account |
| 15 Identity / 10 Reputation | Eligibility checks | Everyone is eligible |
| 10 Reputation | Receives procurement facts | Facts are dropped |
| 25 Work Task | Delivery projects on award | No project created |
| 13 Email | Notices to bidders | Events only |
| 2 DNC | Document anchoring | Every hash accepted |
| 16 Multisig | Award / challenge / policy authority | `Root` |

Connecting each module is a one-line runtime configuration change; the
business rules above do not change.
