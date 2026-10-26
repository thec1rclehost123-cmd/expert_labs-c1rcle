# 00. Payments and finance: visual guide

Start here if you prefer pictures. Each diagram points to the file that explains it. Colours:

- 🟩 **green** = works and is enforced
- 🟨 **yellow** = partial, manual, or a decision is pending
- 🟥 **red** = defect or risk (see `09-status-prs-and-open-decisions.md`)
- ⬜ **grey** = infrastructure or neutral step

How to view: GitHub, VS Code (Markdown Preview Mermaid Support), or https://mermaid.live. A browser version is in `visual-guide.html` (needs internet for the Mermaid library).

Contents: 1 Where the money goes · 2 Price the guest pays · 3 Checkout sequence · 4 Hold and order states · 5 Confirming a payment · 6 Webhook handling · 7 Settlement split · 8 Two ledgers · 9 Journal postings · 10 How balances are computed · 11 Paying partners · 12 Refund paths · 13 Chargebacks · 14 What runs automatically · 15 Failure cases · 16 What is merged where · 17 Defect map · 18 Fix order

---

## 1. Where the money goes
Files: `01-money-model-and-pricing.md`, `04-ledger-and-journal.md`.
```mermaid
flowchart LR
  G["Guest pays<br/>ticket + fees + GST on fees"] --> RZ["Razorpay<br/>(collects, deducts its own fee)"]
  RZ --> PC["provider_clearing<br/>(platform's books)"]
  PC --> FEES["Guest fees kept by platform:<br/>5 percent platform fee + 2.5 percent payment fee"]
  PC --> GST["GST on those fees<br/>liability account"]
  PC --> NET["Net ticket revenue"]
  NET --> PL["Platform commission<br/>15, 12 or 10 percent by plan"]
  NET --> VS["Venue share<br/>0 to 50 percent per partnership"]
  NET --> PR["Promoter commission<br/>from assignment or volume tier"]
  NET --> HO["Host: the remainder"]
  VS --> PAY["Payable to partners"]
  PR --> PAY
  HO --> PAY
  PAY -->|"payout request, manual bank transfer"| BANK["Partner bank account"]
  PAY -.gate off in production.-> BLK["Payouts blocked until legal, tax and provider approvals"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  classDef infra fill:#eeeeee,stroke:#888,color:#111
  class G,PC,NET,HO ok
  class PAY,BANK,BLK warn
  class RZ infra
```

## 2. The price the guest pays (example)
Files: `01-money-model-and-pricing.md`.
```mermaid
flowchart TD
  T["Ticket 1,500.00"] --> D{"Promo code?"}
  D -->|"yes: capped to eligible subtotal"| DS["Discounted subtotal"]
  D -->|no| DS
  DS --> PF["Platform fee 5 percent = 75.00"]
  DS --> PY["Payment fee 2.5 percent = 37.50"]
  PF --> GS["GST 18 percent of the two fees = 20.25<br/>(hard-coded rate, not on the ticket price)"]
  PY --> GS
  DS --> TOT["Grand total = 1,500.00 + 75.00 + 37.50 + 20.25 = 1,632.75"]
  PF --> TOT
  PY --> TOT
  GS --> TOT
  TOT --> CHK{"Total between 0 and 1 rupee?"}
  CHK -->|yes| REJ["Rejected: online payments must be at least 1 rupee"]
  CHK -->|"0 or 1+"| OK["Quote accepted"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class OK,TOT ok
  class REJ bad
  class GS warn
```

## 3. Checkout, step by step
Files: `02-checkout-and-payment-flow.md`.
```mermaid
sequenceDiagram
  participant U as Guest
  participant G as Gateway
  participant R as Razorpay
  U->>G: quote (read-only price)
  U->>G: hold (Idempotency-Key)
  G->>G: transaction takes the next per-event number, then checks capacity
  G-->>U: hold for 10 minutes
  U->>G: payment attempt (Idempotency-Key)
  G->>R: create order (once per hold)
  G-->>U: order id, amount, public key
  U->>R: pay
  R-->>U: payment id + signature
  U->>G: verify (signature checked against the order stored on the hold)
  R->>G: webhook payment.captured (HMAC)
  G->>G: confirmPayment (same code for both paths, idempotent on payment id)
  G-->>U: order and tickets
```

## 4. Hold and order states
Files: `03-order-hold-inventory.md`.
```mermaid
stateDiagram-v2
  direction LR
  state "Hold" as H {
    [*] --> active
    active --> converted: payment confirmed
    active --> released: expired, lost the race, abandoned
    released --> active: late payment revives it
  }
```
```mermaid
stateDiagram-v2
  direction LR
  [*] --> paid: online checkout and free RSVP create orders already paid
  paid --> refund_requested: admin starts a refund
  refund_requested --> paid: refund failed or partial
  refund_requested --> refunded: full refund settled
  note right of paid: Only door ticket sales walk pending, awaiting_payment, paid
```

## 5. Confirming a payment (decision tree)
Files: `02-checkout-and-payment-flow.md`.
```mermaid
flowchart TD
  S(["Payment arrives (redirect or webhook)"]) --> E{"Order already exists for this payment id?"}
  E -->|yes| SAME{"Same user, event and hold?"}
  SAME -->|yes| FUL["Finish fulfilment (safe to repeat)"]
  SAME -->|no| R1["Reject: payment belongs to a different hold"]
  E -->|no| B{"Hold bound to this Razorpay order?<br/>amount, currency, captured all match the provider?"}
  B -->|no| R2["Reject, webhook is retried by Razorpay"]
  B -->|yes| T{"Hold still valid?"}
  T -->|yes| NEW["Create order as paid"]
  T -->|"expired or released"| LATE{"Seats still free after revival?"}
  LATE -->|yes| NEW
  LATE -->|no| AR["Auto-refund in full, hold released, 409 to the buyer"]
  NEW --> P{"Promo limit still ok?"}
  P -->|no| AR2["Auto-refund in full, 409"]
  P -->|yes| FUL
  FUL --> TK["Tickets issued, ledger and journal posted"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class FUL,TK,NEW ok
  class R1,R2,AR,AR2 bad
```
Not covered: a second payment against a hold that is already converted is captured with no order and no refund.

## 6. Webhook handling
Files: `02-checkout-and-payment-flow.md`, `05-razorpay-and-provider-facts.md`.
```mermaid
flowchart TD
  W(["POST /webhooks/payments/razorpay"]) --> SEC{"Webhook secret configured?"}
  SEC -->|no| E503["503 (fails closed)"]
  SEC -->|yes| SIG{"HMAC of the raw body matches?"}
  SIG -->|no| E400["400"]
  SIG -->|yes| EV{"Event type"}
  EV -->|"payment.captured"| COR{"Hold and order ids found in notes?"}
  COR -->|yes| CP["Record capture fact, confirmPayment as system actor"]
  COR -->|no| ORF["Record capture fact, log ids, answer 200<br/>(no auto refund, no alert)"]
  EV -->|"refund.processed, payment.dispute.*"| FACT["Record provider fact, post chargeback journals"]
  EV -->|"payment.failed, order.paid, payment.authorized, refund.created or failed"| IGN["Acknowledged and ignored"]
  CP -->|"error"| RETRY["Non-2xx so Razorpay retries"]
  FACT -->|"dispute without a correlated order"| LOOP["Error: retry loop until support acts"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class CP,FACT ok
  class E503,E400,LOOP bad
  class ORF,IGN,RETRY warn
```
No rate limit on this route and no replay window.

## 7. Settlement split
Files: `01-money-model-and-pricing.md`.
```mermaid
flowchart TD
  N["Net ticket revenue (discounted subtotal)<br/>guest fees and GST are NOT in this base"] --> A["Platform commission = round(net x plan rate)<br/>basic 15, silver 12, diamond 10"]
  N --> B["Venue share = round(net x rate)<br/>partnership rate 0 to 50, null means 0"]
  N --> C{"Promoter attributed?"}
  C -->|"yes"| C1["Promoter = frozen commission from the order<br/>or volume tier (10, 12, 15, 18, 20 percent), rounded down"]
  C -->|no| C2["Promoter = 0"]
  A --> H["Host = net minus the other three"]
  B --> H
  C1 --> H
  C2 --> H
  C1 -.guard.-> GD["Fails later if promoter exceeds net minus platform minus venue,<br/>even though the quote check was weaker"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class N,A,B,H,C1,C2 ok
  class GD bad
```
Example: net 1,500 with plan basic, venue 20 percent and promoter 10 percent gives platform 225, venue 300, promoter 150, host 825.

## 8. Two ledgers (and why partners cannot withdraw yet)
Files: `04-ledger-and-journal.md`.
```mermaid
flowchart TD
  SALE["Every sale posts to BOTH ledgers"] --> L1["Legacy partner ledger (default read model)"]
  SALE --> L2["Journal v2: double-entry, balanced"]
  L1 --> S1["ticket_revenue and platform_fee entries: settled"]
  L1 --> S2["venue_share, host_payout, promoter_commission entries: PENDING"]
  S2 -.nothing ever advances them.-> NEVER["Available balance is always 0"]
  L2 --> S3["Balances computed from payable accounts,<br/>minus holds"]
  S3 --> S4{"Is the event ended and past the 3 day delay?"}
  S4 -->|no| PEND["Pending"]
  S4 -->|yes| AVAIL["Available"]
  SW["Read-model switch (journal cutover)"] -.has no route or script.-> CODE["Only callable from code"]
  L1 -->|"read model = legacy_reads (default)"| SW
  SW --> L2
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class L2,S3,AVAIL ok
  class NEVER,CODE bad
  class L1,PEND,S2 warn
```
The app reads one model or the other, never the sum. Events only reach `ended` through an admin force-complete, so even the journal model keeps money pending.

## 9. Journal postings by event
Files: `04-ledger-and-journal.md`.
```mermaid
flowchart LR
  subgraph SALE["Ticket sale"]
    A1["collection: Dr provider_clearing / Cr ticket_collection_control, platform_fee_revenue, gst_tax_liability"]
    A2["allocation: Dr ticket_collection_control / Cr platform_commission_revenue, host, venue, promoter payable"]
    A3["provider fee: Dr provider_fee_expense / Cr provider_clearing"]
  end
  subgraph PAYOUT["Payout"]
    B1["request: Dr partner_payable / Cr payout_clearing"]
    B2["completed: Dr payout_clearing / Cr cash_bank_clearing"]
    B3["failed: reversal of the reservation"]
  end
  subgraph REF["Refund (3 journals)"]
    C1["recognition, payable reversal, provider settlement<br/>pro rata across original credits"]
  end
  subgraph CB["Chargeback"]
    D1["hold, provider deduction, hold release<br/>chargeback_reserve and chargeback_loss_receivable"]
  end
  subgraph MAN["Manual adjustment (two supers)"]
    E1["Dr or Cr manual_adjustment_control against partner_payable"]
  end
```

## 10. How the available balance is computed (journal model)
Files: `04-ledger-and-journal.md`.
```mermaid
flowchart TD
  P["Payable lines: host, venue, promoter, partner"] --> NET["net = credits minus debits"]
  NET --> H1["minus pending (inside the settlement delay or event not ended)"]
  H1 --> H2["minus policy reserve (reserve basis points)"]
  H2 --> H3["minus cancellation hold (event cancelled)"]
  H3 --> H4["minus reconciliation hold (unresolved discrepancies)"]
  H4 --> AV["available"]
  NET --> CBH["chargeback holds are tracked separately and reported as held"]
  AV --> LIM["Payout limit = available + reserved + paid"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  class AV,LIM ok
```

## 11. Paying partners
Files: `06-payouts-refunds-disputes.md`.
```mermaid
flowchart TD
  R(["Partner owner or admin requests a payout"]) --> G{"Release gate on?"}
  G -->|"off (production default)"| X["503: payouts not enabled"]
  G -->|on| V{"At least 100 rupees and within available + reserved + paid?"}
  V -->|no| E["Rejected"]
  V -->|yes| RQ["Payout requested + reservation journal"]
  RQ --> BR["Admin runs the batch (gated)"]
  BR --> PR["processing"]
  PR --> MT["Admin records a MANUAL bank transfer:<br/>reference, method, time, proof hash"]
  MT --> PD["paid + completion journal"]
  RQ -.-> FZ["Freeze or release needs two super admins"]
  MT -.-> NG["Recording a transfer is not behind the gate"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class RQ,PD ok
  class X,E bad
  class MT,NG,FZ warn
```
No payout API is ever called: payouts are evidence of a manual transfer.

## 12. Refund paths
Files: `06-payouts-refunds-disputes.md`.
```mermaid
flowchart TD
  subgraph AUTO["Automatic (guest paid but cannot get tickets)"]
    A1["Late payment, seats gone"] --> AR["Full refund, key autorefund:payment id"]
    A2["Promo limit reached at capture"] --> AR2["Full refund, key promo-limit-refund:payment id"]
  end
  subgraph ADMIN["Admin-initiated (no guest or partner API)"]
    B1["Request (order paid, amount within balance)"] --> B2["Approvals by amount:<br/>under 500: none, under 5,000: one, else two<br/>scanned ticket: at least one"]
    B2 --> B3["Execute: Razorpay refund, journals, order refunded, tickets voided"]
  end
  subgraph DISP["Partner-raised dispute"]
    C1["Partner disputes an order"] --> C2{"Admin resolves"}
    C2 -->|upheld| C3["Writes an ignored legacy refund entry, moves no money"]
    C2 -->|denied| C4["Nothing"]
  end
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class AR,AR2,B3 ok
  class C3 bad
```

## 13. Chargebacks
Files: `06-payouts-refunds-disputes.md`.
```mermaid
flowchart TD
  D(["Razorpay dispute event"]) --> C{"Correlated order known?"}
  C -->|no| ERR["Error, event marked support_required, retried"]
  C -->|yes| ST{"Dispute status"}
  ST -->|"open or under review"| HOLD["Post chargeback hold"]
  ST -->|"lost, or closed with deduction"| LOSS["Post loss: partner payable first,<br/>excess to chargeback_loss_receivable"]
  ST -->|"won or closed without deduction"| REL["Release the hold"]
  LOSS -.-> DEC["Who bears the loss and the fee is an open decision"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class HOLD,REL ok
  class ERR bad
  class DEC,LOSS warn
```

## 14. What runs automatically (and what does not)
Files: `07-reconciliation-repair-scheduling.md`.
```mermaid
flowchart LR
  subgraph AUTO["Automatic (per instance, Firestore only)"]
    T1["Every 5 min: release expired holds, expire scanner sessions"]
    T2["Every 5 min: refresh analytics read model"]
    T3["Every 1 s: drain the outbox"]
  end
  subgraph MAN["Manual only"]
    M1["repair:fulfilment script"]
    M2["Reconciliation run (admin console)"]
    M3["Payout batch (admin, gated)"]
    M4["Journal backfill, compare, cutover (code only)"]
  end
  subgraph PLAN["Planned, not built"]
    P1["Hourly Cloudflare cron: reconcile + repair"]
    P2["Alerting on requires_support"]
  end
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class T1,T2,T3 ok
  class M1,M2,M3 warn
  class M4 bad
  class P1,P2 warn
```

## 15. Failure cases at a glance
Files: `08-failure-and-edge-cases.md`.
```mermaid
flowchart TD
  F(["What can go wrong"]) --> X{"Case"}
  X -->|"Webhook and redirect both arrive"| OK1["Safe: same idempotent confirm, one order"]
  X -->|"Payment after the hold expired"| OK2["Revive if seats free, otherwise auto-refund"]
  X -->|"Crash half-way through fulfilment"| OK3["Retry or run repair:fulfilment"]
  X -->|"Refund provider times out"| OK4["Order stays locked, needs reconciliation"]
  X -->|"Payment with no hold or order named"| BAD1["Captured and recorded, no refund, no alert"]
  X -->|"Second payment on the same hold"| BAD2["Captured, no order, no refund"]
  X -->|"Chargeback after the payout"| BAD3["Receivable booked, who pays is undecided"]
  X -->|"payment.failed event"| BAD4["Ignored"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class OK1,OK2,OK3,OK4 ok
  class BAD1,BAD2,BAD3,BAD4 bad
```

## 16. What is merged where (corrected)
Files: `09-status-prs-and-open-decisions.md`.
```mermaid
flowchart TD
  P113["#113 merged: fail-closed payments, Redis rate limiting"] --> M1
  S["#115, #117, #118, #124, #126, #130, #133<br/>closed, but their commits landed through #135"] --> M1["#135 merged 10-18: test-mode checkout (whole stack)"]
  M1 --> M2["#136 merged 10-18: financial engine<br/>journal v2, refund executor, disputes, reconciliation, transition"]
  M2 --> M3["#125 merged: PAYOUTS_ENABLED gate"]
  M3 --> STG["On staging, and identical on the reviewed backend branch"]
  O1["#127 part 2: tax rules store"] -.open.-> NO["Not on staging or main"]
  O2["#129: tax and reconciliation admin routes"] -.open.-> NO
  O3["#119: settlement legs, GST as its own line"] -.open.-> NO
  O4["#131: refund and chargeback decisions doc"] -.open.-> NO
  O5["#116: payments design docs"] -.open.-> NO
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class P113,S,M1,M2,M3,STG ok
  class O1,O2,O3,O4,O5,NO warn
```

## 17. Defect map
Files: `09-status-prs-and-open-decisions.md`.
```mermaid
flowchart TD
  ROOT(["Payments risks"]) --> PAYQ["Getting partners paid"]
  ROOT --> ORPH["Orphan money"]
  ROOT --> HARD["Hardening"]
  ROOT --> TAXQ["Tax and legal"]
  PAYQ --> a1["Legacy ledger never advances: available is always 0"]
  PAYQ --> a2["Journal cutover has no route or script"]
  PAYQ --> a3["Events never end, so money stays pending"]
  ORPH --> b1["Second payment on a converted hold: no order, no refund"]
  ORPH --> b2["Capture with no hold named: no refund, no alert"]
  ORPH --> b3["payment.failed ignored, no scheduled repair"]
  HARD --> c1["Webhook: no rate limit, no replay window"]
  HARD --> c2["Adapter: no fetch timeout"]
  HARD --> c3["Promoter guard weaker than settlement"]
  HARD --> c4["Placeholder buyer contact"]
  TAXQ --> d1["GST on fees only, hard-coded 18 percent"]
  TAXQ --> d2["No TDS or TCS"]
  TAXQ --> d3["Funds flow undecided (RBI question)"]
  TAXQ --> d4["Refund and chargeback loss rules undecided"]
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class a1,a2,b1,d3 bad
  class a3,b2,b3,c1,c2,c3,c4,d1,d2,d4 warn
```

## 18. Suggested fix order
Files: `09-status-prs-and-open-decisions.md`.
```mermaid
flowchart LR
  F1["Step 1: Give the journal cutover an operator path<br/>(or retire the legacy model)"] --> F2["Step 2: Let events end<br/>so money can become available"]
  F2 --> F3["Step 3: Close orphan-capture gaps,<br/>handle payment.failed"]
  F3 --> F4["Step 4: Schedule repair and reconciliation, add alerts"]
  F4 --> F5["Step 5: Harden webhook and adapter"]
  F5 --> F6["Step 6: Fix promoter guard, send attendee details"]
  F6 --> F7["Step 7: Decide tax, funds flow, refund and chargeback rules"]
  F7 --> F8["Step 8: Only then consider enabling payouts"]
  classDef step fill:#e8f0fe,stroke:#4a6fa5,color:#111
  class F1,F2,F3,F4,F5,F6,F7,F8 step
```
