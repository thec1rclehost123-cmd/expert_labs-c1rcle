# 00. Admin console: visual guide

Start here if you prefer pictures. Each diagram points to the detailed file that explains it. Colours are consistent:

- 🟩 **green** = works and is enforced
- 🟨 **yellow** = partial, manual, or policy still undecided
- 🟥 **red** = defect or risk (see `14-status-gaps-and-roadmap.md`)
- ⬜ **grey** = infrastructure or neutral step

How to view: GitHub, VS Code (Markdown Preview Mermaid Support), or paste a block into https://mermaid.live. A browser version is in `visual-guide.html` (needs internet for the Mermaid library).

Contents: 1 System map · 2 Navigation map · 3 Roles and tiers · 4 How the gateway decides · 5 Dual control · 6 Onboarding review · 7 Refunds · 8 Disputes and chargebacks · 9 Payouts and the release gate · 10 Manual adjustment and commission · 11 Reconciliation · 12 Support ticket · 13 What each action really does · 14 Settings: what changes what · 15 Console sign-in and its gaps · 16 Audit trail coverage · 17 Defect map · 18 Suggested fix order

---

## 1. System map
Files: `02-frontend-architecture.md`, `11-backend-api-reference.md`.
```mermaid
flowchart LR
  A["Admin user"] --> PG
  subgraph Console["Admin console (Next.js, Vercel)"]
    PG["27 pages<br/>TanStack Query"]
    BFF["BFF: /api/auth (login, logout, refresh, session)<br/>/api/health (readiness, version)"]
  end
  PG -->|"Bearer token, cross-origin"| GW
  PG --> BFF
  BFF -->|"cookie + CSRF, readiness token"| GW
  subgraph Render["Gateway (Render)"]
    GW["Fastify /api/v2/admin/*<br/>17 route files"] --> AUTH["AdminAuthorityService<br/>requireAdmin, authorize, record"]
    AUTH --> SVC["Services: onboarding, refunds, payouts,<br/>reconciliation, support, directory, settings"]
  end
  SVC --> FS[("Firestore<br/>v2_admins, proposals, audit,<br/>refunds, payouts, journals ...")]
  SVC --> RZ["Razorpay<br/>(refund execute, webhook facts)"]
  SVC --> ST[("Cloud Storage<br/>KYC documents")]
  SVC --> RS["Resend email"]
  classDef live fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef infra fill:#eeeeee,stroke:#888,color:#111
  class PG,BFF,GW,AUTH,SVC live
  class FS,RZ,ST,RS infra
```
The console is thin: it holds no database access and no policy. Everything is decided at the gateway.

## 2. Navigation map: 24 desks in 6 sections
Files: `01-product-model-and-roles.md`, `03-desks-and-screens.md`.
```mermaid
flowchart TB
  NAV["Admin console menu"] --> OV["Overview"]
  NAV --> AP["Approvals"]
  NAV --> CO["Commerce"]
  NAV --> CA["Care"]
  NAV --> DI["Directory"]
  NAV --> PL["Platform"]
  OV --> o1["Overview"] & o2["Analytics"] & o3["Lookup"]
  AP --> a1["Proposals"] & a2["Onboarding"] & a3["KYC review"]
  CO --> c1["Orders"] & c2["Tickets"] & c3["Promotions"] & c4["Promoters"] & c5["Commissions"]
  CA --> k1["Support"] & k2["Refunds"] & k3["Disputes"] & k4["Chargebacks"] & k5["Payouts"] & k6["Reconciliation"] & k7["Adjustments"]
  DI --> d1["Venues"] & d2["Events"] & d3["Users"] & d4["Hosts"]
  PL --> p1["Admins"] & p2["Audit"] & p3["Health"] & p4["Settings"]
  classDef sec fill:#e8f0fe,stroke:#4a6fa5,color:#111
  class OV,AP,CO,CA,DI,PL sec
```
Every desk reads live data. The menu is the same for every signed-in user: it is not filtered by admin role.

## 3. Roles and tiers: who may do what
Files: `01-product-model-and-roles.md`, `04-authority-dual-control-audit.md`.
```mermaid
flowchart TD
  ACT(["An admin action"]) --> TIER{"Which tier?"}
  TIER -->|"Tier 1: event pause, resume, force-complete,<br/>all support actions"| T1["Any active admin<br/>(super, admin, ops, finance, support)"]
  TIER -->|"Tier 2: onboarding decisions, refunds,<br/>payout run and complete, disputes, reconciliation,<br/>suspend or reinstate, ban or unban"| T2["super, admin, ops, finance only<br/>(support is refused)"]
  TIER -->|"Tier 3: provision admin, change role,<br/>commission, financial adjustment,<br/>payout freeze and release"| T3["super only<br/>AND a different super must approve"]
  TIER -->|"Platform settings update"| T0["No tier at all<br/>any active admin"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class T1,T2 ok
  class T3 warn
  class T0 bad
```

| Role | Reads | Tier 1 | Tier 2 | Tier 3 |
|---|---|---|---|---|
| super | everything | yes | yes | yes (with a second super) |
| admin | everything | yes | yes | no |
| ops | everything | yes | yes | no |
| finance | everything | yes | yes | no |
| support | everything | yes | **no** | no |

## 4. How the gateway decides on every admin request
Files: `04-authority-dual-control-audit.md`.
```mermaid
flowchart TD
  IN(["POST /api/v2/admin/..."]) --> SES{"Valid session?"}
  SES -->|no| U1["401"]
  SES -->|yes| RL{"Under 10 mutations per minute?"}
  RL -->|no| U2["429"]
  RL -->|yes| IDK{"Idempotency-Key header present?"}
  IDK -->|no| U3["422 validation error"]
  IDK -->|yes| ADM{"Record in v2_admins and active?"}
  ADM -->|no| U1
  ADM -->|yes| TR{"Role allowed for this action's tier?"}
  TR -->|no| U4["403 role cannot perform action"]
  TR -->|yes| HASH{"Seen this key before?"}
  HASH -->|"same body, done"| REP["Replay stored response"]
  HASH -->|"different body, running, or failed before"| U5["409"]
  HASH -->|new| DO["Run the service action"]
  DO --> AUD["Write audit row (separate step)"]
  AUD --> OUT(["Response"])
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  class U1,U2,U3,U4,U5 bad
  class DO,REP,OUT ok
```
A revoked admin gets the same 401 as a non-admin. The audit row is written after the change, not in the same transaction.

## 5. Dual control (two-person rule)
Files: `04-authority-dual-control-audit.md`, `12-domain-model-and-state-machines.md`.
```mermaid
flowchart TD
  R(["Super admin A raises a proposal<br/>action + reason + payload"]) --> P["Status: pending"]
  P --> Q{"Super admin B (not A) decides"}
  Q -->|reject| RJ["rejected (final)"]
  Q -->|approve| AP["Status: approved"]
  P -->|"A cancels"| CN["cancelled (final)"]
  AP --> EX["Any super runs the execute route<br/>(parameters come from the stored proposal)"]
  EX --> DONE["Action applied + audit row"]
  EX -.again, with a new idempotency key.-> DONE2["Applied again"]
  AP -.never expires.-> OLD["An old approval stays usable forever"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class DONE ok
  class DONE2,OLD bad
```
Safe design points: the approver cannot be the proposer, and execution reads the stored payload, not the request body. Gaps: no "executed" state, no expiry, and the executor can be the proposer or approver.

## 6. Onboarding review and KYC
Files: `05-flow-onboarding-kyc.md`.
```mermaid
flowchart TD
  Q(["Applications queue: submitted"]) --> OPEN["Open application"]
  OPEN --> DOC["For each document: get signed link, view"]
  DOC --> DV{"Document decision (Tier 2)"}
  DV -->|verify| V["document verified"]
  DV -->|"reject + reason"| X["document rejected"]
  V --> ALL{"All required documents verified?"}
  X --> CH["Request changes (note + email to applicant)"]
  ALL -->|no| CH
  ALL -->|yes| DEC{"Application decision (Tier 2)"}
  DEC -->|approve| ORG["Create organization with plan fee 15, 12 or 10 percent<br/>+ starter venue if venue type"]
  DEC -->|reject| RJ["rejected (final)"]
  DEC -->|"request changes"| CH
  CH --> BACK["Applicant edits and resubmits"]
  BACK --> Q
  ORG --> END(["Approved: partner can sign in"])
  DOC -.-> WARN["Signed links are not audited and any role can open them"]
  ORG -.-> WARN2["Creation is not atomic: retry with a new key could duplicate"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class ORG,END ok
  class WARN,WARN2 bad
```
Required documents: individual = id front, id back, selfie. Business = registration certificate + signatory id front, id back, selfie.

## 7. Refund: request to settled
Files: `06-flow-refunds-chargebacks-disputes.md`.
```mermaid
flowchart TD
  S(["Refund request raised (API only, no screen)"]) --> CHK{"Order paid and amount within balance?"}
  CHK -->|no| E["Rejected with error"]
  CHK -->|yes| LK["Order locked: refund_requested<br/>target tickets frozen"]
  LK --> AM{"Amount vs thresholds in settings"}
  AM -->|"below 500 rupees and no scanned ticket"| Z["0 approvers: status approved at once"]
  AM -->|"scanned ticket, or below 5,000 rupees"| O1["1 approver: status pending"]
  AM -->|"5,000 rupees or more"| O2["2 approvers: status pending"]
  O1 --> D{"Another admin decides"}
  O2 --> D
  D -->|reject + reason| RJ["rejected: order restored"]
  D -->|"all approvals in"| AP["approved"]
  Z --> AP
  AP --> EX["Admin presses Execute"]
  EX --> CL["Claim run (only one wins)"]
  CL --> PR["Call Razorpay with fixed key refund:id"]
  PR --> R{"Provider answer"}
  R -->|"definite failure"| FL["failed: tickets released, order restored. Retry allowed"]
  R -->|"unclear"| UN["Stays executing: wait for webhook or reconciliation"]
  R -->|processed| ST["Post journals, order refunded,<br/>tickets voided, settled"]
  FL --> EX
  Z -.-> WARN["Settings can lower thresholds to 0 (no tier gate)"]
  AP -.-> WARN2["No way to cancel an approved refund. Buyer is not notified"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class ST,AP ok
  class E,RJ,FL,WARN,WARN2 bad
  class UN warn
```
Open policy questions (finance and legal): who funds a refund of a split order, who bears the provider fee, tax on refunds, treatment if the partner was already paid.

## 8. Disputes and chargebacks are two different things
Files: `06-flow-refunds-chargebacks-disputes.md`.
```mermaid
flowchart LR
  subgraph PARTNER["Partner-raised dispute (desk: Disputes)"]
    P1["Partner disputes a ledger entry"] --> P2["open"]
    P2 --> P3{"Admin resolves (Tier 2)"}
    P3 -->|upheld| P4["Writes a refund entry in the legacy ledger<br/>(no journal v2, no order change, no provider call)"]
    P3 -->|denied| P5["No money movement"]
  end
  subgraph PROVIDER["Razorpay chargeback (desk: Chargebacks)"]
    C1["Buyer disputes with bank"] --> C2["Razorpay webhook"]
    C2 --> C3["Stored as provider fact<br/>journals: hold, loss, reversal"]
    C3 --> C4["Console lists it read-only<br/>no evidence upload, no actions"]
  end
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class P4,C4 warn
```
Undecided: who bears a lost chargeback and its fee, and whether payouts are held while a dispute is open.

## 9. Payouts and the production release gate
Files: `07-flow-payouts-and-adjustments.md`.
```mermaid
flowchart TD
  PR(["Partner requests payout (min 100 rupees)"]) --> G1{"PAYOUTS_ENABLED?"}
  G1 -->|"off (production default)"| X1["503 payouts disabled"]
  G1 -->|on| REQ["Payout: requested<br/>reservation journal posted"]
  REQ --> BR["Admin: Run batch (Tier 2)"]
  BR --> G2{"Gate on? reservation journal exists?"}
  G2 -->|"gate off"| X2["503"]
  G2 -->|"no reservation"| SK["Skipped with reason"]
  G2 -->|yes| PRC["processing"]
  PRC --> CM["Admin: Record transfer (Tier 2)<br/>reference, method, time, proof hash"]
  CM --> PD["paid + completion journal"]
  REQ -->|"Freeze (Tier 3, two supers)"| FR["frozen"]
  PRC -->|"Freeze (Tier 3)"| FR
  FR -->|"Release (Tier 3, gate on)"| BK["Back to previous status"]
  REQ --> FL["failed (no admin button)"]
  PRC --> FL
  CM -.-> W["Record transfer is NOT behind the gate:<br/>works even when payouts are off"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class PD,REQ ok
  class X1,X2,W bad
  class FR,SK,FL warn
```
Before the gate is turned on in production: lawyer-confirmed funds flow (RBI payment aggregator), CA tax treatment (GST, TDS 194-O, TCS 52), Razorpay written approval.

## 10. Manual adjustment and commission change (both need two supers)
Files: `07-flow-payouts-and-adjustments.md`.
```mermaid
flowchart TD
  subgraph ADJ["Financial adjustment"]
    A1["Adjustments page: organization id (typed), amount in paise,<br/>credit or debit, reason"] --> A2["Proposal: FINANCIAL_ADJUSTMENT pending"]
    A2 --> A3["Second super approves on Proposals page"]
    A3 --> A4["Operator pastes the approved proposal id into the Post form"]
    A4 --> A5{"Journal v2 is the active read model?"}
    A5 -->|no| A6["Refused"]
    A5 -->|yes| A7["Balanced journal posted once<br/>(key = proposal id) + audit"]
  end
  subgraph COM["Commission change"]
    C1["Commissions page: percent 0 to 100 + reason"] --> C2["Proposal: COMMISSION_ADJUST pending"]
    C2 --> C3["Second super approves"]
    C3 --> C4["Proposals page: Adjust commission"]
    C4 --> C5["Organization platform fee percent updated"]
  end
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class A7,C5 ok
  class A6 bad
```
Settings also holds a fee rate as a 0 to 1 fraction, but nothing reads it. Only the per-organization percent is used.

## 11. Reconciliation
Files: `08-flow-reconciliation.md`.
```mermaid
flowchart TD
  S(["Admin: Run reconciliation (Tier 2)"]) --> SC["Scan all orders, provider payments,<br/>tickets, payouts in memory"]
  SC --> CMP["Compare against Razorpay facts and journals"]
  CMP --> D["Discrepancies created (14 kinds)<br/>state: open, finance_review or support_required"]
  D --> L["Admin reviews list and evidence"]
  L --> CH{"Choose"}
  CH -->|"Recheck (evidence repair)"| RR["Re-runs the whole scan"]
  RR --> D
  CH -->|"Needs money correction"| PRP["Raise FINANCIAL_ADJUSTMENT proposal<br/>(manual, not linked in the UI)"]
  CH -->|Resolve + reason| RS["state: resolved<br/>(data is NOT repaired)"]
  RS -.next run, data still wrong.-> D
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class D ok
  class RS warn
```
Kinds include: payment without order, paid order without payment or ticket, ticket without paid order, missing or duplicate or unbalanced journal, amount or currency mismatch, refund or chargeback mismatch, stale pending payment, payout mismatch.

## 12. Support ticket lifecycle
Files: `09-flow-support-desk.md`.
```mermaid
stateDiagram-v2
  direction LR
  [*] --> open: guest or partner submits
  open --> in_progress: assign
  open --> waiting_on_customer: reply
  in_progress --> waiting_on_customer: reply
  waiting_on_customer --> in_progress: customer replies
  in_progress --> escalated: escalate
  in_progress --> resolved: resolve with reason
  escalated --> resolved
  resolved --> closed: close
  closed --> open: reopen
  note right of open: SLA clock starts. Urgent 1h response, 4h resolution. Medium 24h and 72h. Low 72h and 168h
```
Side actions at any time: internal note, priority change (resets SLA), link order, event, venue or user, merge a duplicate into another ticket, soft delete and restore. All support actions are Tier 1, so every admin role can do them.

## 13. What each action really does (enforcement map)
Files: `10-flow-directory-and-platform-actions.md`.
```mermaid
flowchart LR
  subgraph BTN["Admin buttons"]
    B1["Ban user"]
    B2["Suspend venue or organization"]
    B3["Maintenance mode"]
    B4["Platform fee rate"]
    B5["Pause or force-complete event"]
    B6["Suspend promoter"]
  end
  B1 --> R1["Recorded + audited"]
  B2 --> R1
  B3 --> R1
  B4 --> R1
  B5 --> R1
  B6 --> R1
  R1 --> E1["Hides from public discovery"]
  B2 --> E1
  B2 --> E2["Blocks new promoter connections and partnerships"]
  B5 --> E3["Pause: sales paused. Force-complete: event ends and leaves public view<br/>no refunds, no domain event emitted"]
  B6 --> E4["Assignments suspended"]
  B1 -.no enforcement found.-> N1["User can still sign in and buy"]
  B3 -.no reader found.-> N2["Nothing checks it"]
  B4 -.not read by checkout.-> N3["Ignored (per-organization percent is used)"]
  B2 -.-> N4["Checkout, partner writes and login not blocked"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef rec fill:#fff3b0,stroke:#b58900,color:#111
  class E1,E2,E3,E4 ok
  class N1,N2,N3,N4 bad
  class R1 rec
```

## 14. Platform settings: what each field really changes
Files: `10-flow-directory-and-platform-actions.md`.
```mermaid
flowchart TD
  ST["PUT /admin/settings/platform<br/>(any active admin, no tier, no dual control)"] --> F1["refundSingleApproverThresholdPaise"]
  ST --> F2["refundDualApproverThresholdPaise"]
  ST --> F3["accountingPolicy<br/>settlement delay, reserve, holds"]
  ST --> F4["maintenanceMode"]
  ST --> F5["platformFeeRate (0 to 1)"]
  ST --> F6["featureFlags"]
  F1 --> U1["Used: decides how many refund approvers"]
  F2 --> U1
  F3 --> U2["Used: checkout and payout rules"]
  F4 --> U3["Not read anywhere"]
  F5 --> U4["Not read anywhere"]
  F6 --> U5["Not audited in detail"]
  U1 -.set to 0.-> RISK["Every refund auto-approved"]
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class RISK,U3,U4 bad
  class U1,U2 ok
  class U5 warn
```

## 15. Console sign-in and its gaps
Files: `02-frontend-architecture.md`.
```mermaid
flowchart TD
  S(["Open any admin URL"]) --> SS{"Server finds a session?"}
  SS -->|no| RD["Shell shows 'Redirecting to sign in', goes to /login"]
  SS -->|yes| SH["Full console shell shown<br/>(for any signed-in account)"]
  RD --> LG["Login form"]
  LG --> API["Gateway login via BFF"]
  API --> TOK["Access token in memory, cookie set"]
  TOK --> SH
  SH --> CALL["Each desk calls the gateway"]
  CALL --> CHK{"Is this account an active admin?"}
  CHK -->|no| FAIL["Calls fail 401: generic error text, no 'not an admin' page"]
  CHK -->|yes| RL{"Role allowed this action?"}
  RL -->|no| F403["403: button was visible anyway"]
  RL -->|yes| OK["Works"]
  API -.temporary password.-> STUCK["Gateway blocks everything with 403<br/>console has no change-password page"]
  SH -.idle 30 min.-> OUT["Automatic sign-out"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class OK,TOK ok
  class FAIL,F403,STUCK bad
```

## 16. Audit trail: what is and is not recorded
Files: `04-authority-dual-control-audit.md`.
```mermaid
flowchart LR
  subgraph IN["Recorded"]
    I1["Proposal raise, approve, reject, cancel"]
    I2["Executed TIER3 actions"]
    I3["Refund request, approve, reject, execute"]
    I4["Onboarding decisions + document verify and reject"]
    I5["Payout run, complete, freeze, release"]
    I6["Suspend, ban, pause, force-complete<br/>(without a reason)"]
    I7["Settings change (partial fields)"]
    I8["Support actions"]
    I9["Users and audit CSV exports"]
  end
  subgraph OUT["Not recorded or incomplete"]
    O1["IP and user agent on approve, reject, cancel"]
    O2["Opening a KYC document link"]
    O3["Orders, tickets, venues, events, hosts, promotions,<br/>refunds, reconciliation exports"]
    O4["Refund failures and unclear outcomes"]
    O5["No-op repeats"]
    O6["Reason for suspend, ban, pause"]
    O7["Feature flags and most accounting policy fields"]
  end
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class I1,I2,I3,I4,I5,I8,I9 ok
  class I6,I7 ok
  class O1,O2,O3,O4,O5,O6,O7 bad
```
The audit row is written after the change, in a separate step. The console does not show the before and after values.

## 17. Defect map
Files: `14-status-gaps-and-roadmap.md`.
```mermaid
flowchart TD
  ROOT(["Admin console risks"]) --> SEC["Authority and security"]
  ROOT --> MON["Money"]
  ROOT --> ENF["Not enforced"]
  ROOT --> UX["Console UX"]
  SEC --> s1["Settings: no tier, can zero refund thresholds"]
  SEC --> s2["Approved proposals re-executable, no expiry"]
  SEC --> s3["Role never used in UI, no middleware"]
  SEC --> s4["Lookup shows emails to support role"]
  SEC --> s5["Health endpoints open to anyone reaching the app"]
  MON --> m1["Record transfer bypasses payouts gate"]
  MON --> m2["Zero-approver refund: one admin can do all"]
  MON --> m3["No cancel for approved refund, no fail-payout button"]
  MON --> m4["Policy undecided: refund and chargeback loss, tax"]
  ENF --> e1["Ban not enforced"]
  ENF --> e2["Maintenance mode not read"]
  ENF --> e3["Suspension only affects discovery and connections"]
  UX --> u1["Forced password change not handled"]
  UX --> u2["No pagination, 100 row cap, CSV capped too"]
  UX --> u3["No confirm on force-complete, refund approve, payout run"]
  UX --> u4["Generic errors hide the gateway reason"]
  UX --> u5["No screen to create refund or provision admin"]
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class s1,s2,s3,m1,m2,e1,e2,u1 bad
  class s4,s5,m3,m4,e3,u2,u3,u4,u5 warn
```

## 18. Suggested fix order
Files: `14-status-gaps-and-roadmap.md`.
```mermaid
flowchart LR
  F1["Step 1: Gate settings update<br/>(dual control) + validate thresholds"] --> F2["Step 2: Proposals: executed status,<br/>single use, expiry"]
  F2 --> F3["Step 3: Enforce or remove:<br/>ban, maintenance, suspension, fee rate"]
  F3 --> F4["Step 4: Put complete-payout behind the gate;<br/>add fail-payout and cancel-refund"]
  F4 --> F5["Step 5: Console: role in session, gate buttons,<br/>password-change page, confirms, pagination"]
  F5 --> F6["Step 6: Audit: IP on all actions, document links,<br/>exports, before and after view"]
  F6 --> F7["Step 7: Support list filters on the server"]
  F7 --> F8["Step 8: Reconciliation: schedule, de-duplicate, targeted retry"]
  F8 --> F9["Step 9: Decide policies, then consider enabling payouts"]
  classDef step fill:#e8f0fe,stroke:#4a6fa5,color:#111
  class F1,F2,F3,F4,F5,F6,F7,F8,F9 step
```
