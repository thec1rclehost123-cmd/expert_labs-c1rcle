# 00. Partner dashboard: visual guide

Start here if you prefer pictures. Each diagram links back to the detailed file that explains it. Colours are consistent everywhere:

- 🟩 **green** = live against the real gateway
- 🟨 **yellow** = fixture or partial (sample data, or only part works)
- 🟥 **red** = defect or risk (see `13-status-gaps-and-roadmap.md`)
- ⬜ **grey** = infrastructure or neutral step

How to view the diagrams: GitHub, VS Code (Markdown Preview Mermaid Support), or paste a block into https://mermaid.live. A browser version is in `visual-guide.html` (needs internet for the Mermaid library).

Contents: 1 System map · 2 Studios and navigation · 3 Screen status map · 4 Sign-in and workspace flow · 5 How access is derived · 6 Who enforces what · 7 BFF vs direct requests · 8 Backend request lifecycle · 9 Onboarding · 10 Event create and publish · 11 Event lifecycle · 12 Slot requests · 13 Partnerships and promoter connections · 14 Promoter earnings chain · 15 Staff invitations · 16 Finance and payouts · 17 Data loading and failures · 18 Fixture to live roadmap · 19 Defect map

---

## 1. System map: who talks to whom
Files: `02-frontend-architecture.md`, `09-backend-api-reference.md`.
```mermaid
flowchart LR
  U["Partner user<br/>(venue, host, promoter)"] --> PD
  subgraph Browser_and_Vercel["Dashboard (Next.js on Vercel)"]
    PD["Pages and client hooks"]
    BFF["BFF route handlers<br/>/api/auth, /api/bff"]
    PX["proxy.ts<br/>cookie check + CSP nonce"]
  end
  PD -->|same-origin cookie + CSRF| BFF
  PD -->|"Bearer token (direct)"| GW
  PX -.guards page routes.-> PD
  BFF -->|cookie forwarded| GW
  subgraph Render["Gateway (Render, free plan)"]
    NG["nginx edge"] --> GW["Fastify /api/v2<br/>auth, RBAC, rate limit,<br/>idempotency"]
    GW --> CORE["packages/core<br/>services + rules"]
  end
  CORE --> FS[("Firestore")]
  CORE --> ST[("Cloud Storage<br/>posters, KYC")]
  PD -.phone OTP only.-> FB["Firebase Auth<br/>(client)"]
  CORE --> RS["Resend email"]
  classDef live fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef infra fill:#eeeeee,stroke:#888,color:#111
  class PD,BFF,GW,CORE live
  class PX,NG,FS,ST,FB,RS infra
```

## 2. Studios and navigation
One shell, three studios. What differs is the nav and the primary action. Files: `01-product-model.md`, `03-routes-and-screens.md`.
```mermaid
flowchart TB
  A["Access response<br/>partnerType"] --> V{"partnerType?"}
  V -->|venue| VS["Venue Studio<br/>action: Create event"]
  V -->|host| HS["Host Studio<br/>action: Start event request"]
  V -->|promoter| PS["Promoter Studio<br/>action: Get link"]
  VS --> VN["Overview · Events · Slot requests · Partners · Marketing · Presence · Finance · Support · Settings"]
  HS --> HN["Overview · Events · Slot requests · Partners · Marketing · Presence · Finance · Support · Settings"]
  PS --> PN["Overview · Events · Guests · Analytics · Finance · Partners · Leaderboard"]
  VN --> HID["Server tabVisibility can hide tabs<br/>(null = show all, false = hide)"]
  HN --> HID
  PN --> HID
  classDef live fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class VS,HS,PS,VN,HN,PN,HID live
```
Calendar and Door are real routes that are not in the menu. "Presence" is in the menu but has no page.

## 3. Screen status map: what is real today
Files: `03-routes-and-screens.md`.
```mermaid
flowchart LR
  subgraph LIVE["🟩 Live (real gateway data)"]
    L1["Login · Signup · Onboarding"]
    L2["Overview (venue, host)"]
    L3["Finance (all studios)"]
    L4["Calendar and availability"]
    L5["Slot requests"]
    L6["Partners + promoter connections"]
    L7["Staff + invitations"]
    L8["Support tickets"]
    L9["Event create (publish workflow)"]
  end
  subgraph FIX["🟨 Fixture (sample data)"]
    F1["Events list"]
    F2["Event detail + tabs"]
    F3["Event edit (Save does nothing)"]
    F4["Door mode (read only)"]
    F5["Marketing"]
    F6["Settings"]
    F7["Promoter: overview, events, guests, analytics, leaderboard"]
    F8["Topbar notifications + search"]
  end
  subgraph GONE["🟥 Missing or broken"]
    M1["Presence: no page"]
    M2["Event analytics: skeleton"]
    M3["Overview event links: 404"]
  end
  classDef live fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef fix fill:#fff3b0,stroke:#b58900,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class L1,L2,L3,L4,L5,L6,L7,L8,L9 live
  class F1,F2,F3,F4,F5,F6,F7,F8 fix
  class M1,M2,M3 bad
```
The old `/venue`, `/host`, `/promoter`, `/partner-network` pages are redirected (HTTP 307) into these screens and are dead code.

## 4. Sign-in to landing page
Files: `04-auth-rbac-and-access.md`.
```mermaid
flowchart TD
  S(["Open /login"]) --> P1["Pick studio, enter credentials"]
  P1 --> BFF["POST /api/auth/login<br/>(BFF sets cookie + CSRF)"]
  BFF --> OK{"Login OK?"}
  OK -->|no| ERR["Show error, stay on login"]
  OK -->|yes| MCP{"mustChangePassword?"}
  MCP -->|yes| CP["/change-password"]
  CP --> ORGS
  MCP -->|no| ORGS["Load organizations + access for each"]
  ORGS --> N{"How many organizations?"}
  N -->|0| INV{"Pending invitation?"}
  INV -->|yes| ACC["Accept invitation page"]
  INV -->|no| ONB["/onboard wizard"]
  N -->|1| SET["Set active org"]
  N -->|many| PICK["/partner/select-organization"]
  PICK --> SET
  SET --> MATCH{"Org studio matches the studio chosen?"}
  MATCH -->|no| MIS["Error: stay signed in"]
  MATCH -->|yes| LAND["Land on first visible tab"]
  classDef live fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class BFF,SET,LAND,ORGS live
  class ERR,MIS bad
```
Note: if `?next=` is present the workspace match is skipped. `/change-password` does not validate its `next` parameter (open redirect).

## 5. How access is derived (and where promoters break)
Files: `04-auth-rbac-and-access.md`.
```mermaid
flowchart TD
  M["Member of organization<br/>role: owner, admin, manager, member<br/>capabilities: host, venue, promoter"] --> T["partnerType = first of venue, host, promoter<br/>(no capabilities means venue)"]
  M --> R{"Organization role"}
  R -->|owner or admin| O["Partner role OWNER"]
  R -->|manager| MG["Partner role MANAGER"]
  R -->|member| ST["Partner role STAFF"]
  T --> TB{"partnerType"}
  O --> TB
  MG --> TB
  ST --> TB
  TB -->|venue| V["Venue table: OWNER 16 permissions, MANAGER 11, STAFF 3"]
  TB -->|host| H["Host table: OWNER 9, MANAGER 4, STAFF 2"]
  TB -->|promoter| PR{"Role is PROMOTER or TEAM_LEAD?"}
  PR -->|"never: roles above can't produce them"| OKP["Promoter permissions"]
  PR -->|"owner, manager, staff"| EMPTY["permissions = empty list"]
  V --> OUT["Access response:<br/>permissions + tabVisibility"]
  H --> OUT
  EMPTY --> OUT
  OKP --> OUT
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef live fill:#d9f7d9,stroke:#2e8b2e,color:#111
  class EMPTY,PR bad
  class OUT live
```
Roles FINANCE_ADMIN, SECURITY, DOOR, COHOST, PROMOTER and TEAM_LEAD cannot be produced by this path.

## 6. Who enforces what (security layers)
Files: `02-frontend-architecture.md`, `04-auth-rbac-and-access.md`.
```mermaid
flowchart TD
  REQ(["User action in the browser"]) --> L1["Layer 1: proxy.ts<br/>session cookie present? (UX only)"]
  L1 -->|no| LOGIN["Redirect to /login"]
  L1 -->|yes| L2["Layer 2: PartnerStudioFrame<br/>studio matches partner type?<br/>tab not hidden by tabVisibility?<br/>(client-side only)"]
  L2 -->|blocked| MSG["'Not available' message"]
  L2 -->|ok| L3["Layer 3: API call<br/>BFF (cookie + CSRF) or direct Bearer"]
  L3 --> L4["Layer 4: Gateway<br/>session, X-Organization-Id membership,<br/>requirePermission, path org = actor org"]
  L4 -->|deny| E403["403 or 404"]
  L4 -->|allow| SVC["Service rules + idempotency"]
  classDef weak fill:#fff3b0,stroke:#b58900,color:#111
  classDef strong fill:#d9f7d9,stroke:#2e8b2e,color:#111
  class L1,L2 weak
  class L3,L4,SVC strong
```
Only layer 4 is a real security boundary. Layers 1 and 2 are convenience. The payout button, for example, is not hidden by role in the UI; the gateway refuses non-owners.

## 7. BFF or direct: how a request leaves the browser
Files: `02-frontend-architecture.md`.
```mermaid
flowchart TD
  A(["Browser needs data"]) --> B{"Where does the code run?"}
  B -->|"Server component (page load)"| S["createServerApiClient<br/>forwards cookie to gateway"]
  B -->|Browser| C{"Which feature?"}
  C -->|"Orgs, access, onboarding,<br/>slot requests, support"| BFF["bffClient: same-origin /api/bff<br/>cookie + CSRF header"]
  C -->|"Calendar, partners, staff,<br/>publish, payouts, phone verify"| D["apiClient: direct to gateway<br/>Authorization: Bearer (in memory)"]
  BFF --> G["Gateway /api/v2"]
  D --> G
  S --> G
  D -.401.-> RF["Refresh token once, retry, else logout"]
  BFF -.401.-> LO["Clear session, go to /login"]
  classDef live fill:#d9f7d9,stroke:#2e8b2e,color:#111
  class BFF,D,S live
```
The direct path needs the gateway to allow the dashboard origin (CORS) and the page CSP to allow the gateway origin.

## 8. What happens to a request inside the gateway
Files: `09-backend-api-reference.md`.
```mermaid
flowchart TD
  IN(["HTTP request"]) --> NG["nginx: host allow-list, edge rate limit, body 1 MB"]
  NG --> CORS["CORS: exact origins"]
  CORS --> AU["Auth hook: session + X-Organization-Id membership"]
  AU --> PW{"Password rotation owed?"}
  PW -->|yes, route not allowed| E1["403 passwordChangeRequired"]
  PW -->|no| RL["Rate limit by class"]
  RL -->|over| E2["429"]
  RL --> VAL["validateV2: zod body, query, params, headers"]
  VAL -->|invalid| E3["422 fieldErrors"]
  VAL --> PERM["requirePermission + path org must equal actor org"]
  PERM -->|no| E4["403"]
  PERM --> IDM{"Mutation with Idempotency-Key?"}
  IDM -->|"same key, same body"| REP["Replay stored response"]
  IDM -->|"same key, different body or in flight"| E5["409"]
  IDM -->|new| SVC["Service call: rules, state machine, compare-and-set"]
  SVC -->|"domain error"| MAP["Mapped to 400, 404, 409"]
  SVC --> OUT["Validate response with zod, event to outbox"]
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  class E1,E2,E3,E4,E5,MAP bad
  class OUT,REP ok
```

## 9. Onboarding: from application to approval
Files: `05-feature-onboarding-kyc.md`.
```mermaid
flowchart TD
  S(["/onboard"]) --> R["Step 1: Choose role: venue, host, promoter"]
  R --> SU["Step 2: Sign up (409 means already registered: log in and resume)"]
  SU --> EO["Step 3: Email OTP"]
  EO --> PO["Step 4: Phone OTP (Firebase, reCAPTCHA off outside production)"]
  PO --> ET{"Step 5: Entity type"}
  ET -->|individual| D1["Step 6: Details form"]
  ET -->|business| D1
  D1 --> AS["Autosave (PATCH with If-Match)"]
  D1 --> K{"Entity type?"}
  K -->|individual| KI["KYC: id front, id back, selfie"]
  K -->|business| KB["KYC: registration certificate<br/>then signatory id front, id back, selfie"]
  KI --> SUB["Submit"]
  KB --> SUB
  SUB --> W["Status: submitted. App polls every 10 s"]
  W --> AD{"Admin decision"}
  AD -->|approve| ORG["Organization created with plan fee 15, 12 or 10 percent<br/>(+ starter venue for venue type)"]
  AD -->|"request changes"| CH["changes_requested: back to details or KYC<br/>(reviewer note not shown yet)"]
  CH --> SUB
  AD -->|reject| REJ["rejected: final"]
  ORG --> GO["User routed to their studio"]
  classDef live fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class ORG,GO live
  class REJ bad
  class CH warn
```
Approval is blocked unless every required document is verified by an admin. The admin side is documented in `../07-admin-console/05-flow-onboarding-kyc.md`.

## 10. Event create and publish: a chain of calls that can stop half-way
Files: `06-feature-calendar-slots-events.md`.
```mermaid
flowchart TD
  S(["Fill the event form: basics, promoters, review"]) --> P["Step 1: Upload poster (signed URL, max 5 MB)"]
  P --> E["Step 2: Create event (draft)"]
  E --> T["Step 3: Create each ticket tier, one by one"]
  T --> C{"Custom compensation?"}
  C -->|yes| PA["Step 4: Patch event with tier ids (If-Match)"]
  C -->|no| AS
  PA --> AS["Step 5: Assign each promoter"]
  AS --> RV["Step 6: Review"]
  RV --> PB["Step 7: Publish"]
  PB --> OK(["Event live. Go to calendar"])
  P -.fail.-> F1["Nothing created yet. Retry is safe"]
  E -.fail.-> F2["Nothing created. Retry is safe"]
  T -.fail.-> F3["Partial draft exists"]
  AS -.fail.-> F3
  RV -.fail.-> F3
  PB -.fail.-> F3
  F3 --> RT["Retry replays each step with the same keys<br/>(no cleanup of the half-built draft)"]
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  class F3 bad
  class OK ok
```
Publish rules: the event must pass review first, and compensation must be valid (standard needs a rate, salary needs an amount, custom needs a rate for every eligible tier). It does not check that a venue, a tier, a future date or a poster exist.

## 11. Event lifecycle and the gap
Files: `06-feature-calendar-slots-events.md`.
```mermaid
stateDiagram-v2
  direction LR
  [*] --> draft
  draft --> review
  review --> draft
  review --> published: publish (via scheduled)
  published --> sales_paused
  sales_paused --> published
  published --> started
  started --> ended
  ended --> archived
  draft --> cancelled
  review --> cancelled
  published --> cancelled
  sales_paused --> cancelled
  note right of started: Nothing in the app moves an event here or to ended or archived. Only an admin force-complete reaches ended
```

## 12. Slot request: host asks a venue for a date
Files: `06-feature-calendar-slots-events.md`.
```mermaid
flowchart LR
  subgraph HOST["Host studio"]
    H1["Draft event + choose venue and slot"] --> H2["Submit for approval"]
  end
  H2 -->|"creates draft event, then slot request"| SR["Slot request: pending"]
  subgraph VENUE["Venue studio"]
    V1["Incoming list"] --> V2{"Decision"}
  end
  SR --> V1
  V2 -->|accept| ACC["accepted"]
  V2 -->|reject| REJ["rejected (final)"]
  ACC --> HC["Host may cancel: cancelled"]
  SR -.host cannot withdraw a pending request.-> X["Gap"]
  ACC -.-> G1["Gap: calendar and event are not updated, nobody is notified"]
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  class X,G1 bad
  class SR,ACC ok
```
Create does not check that the venue exists, that a partnership exists, or that the event belongs to the host.

## 13. Partnerships and promoter connections
Files: `07-feature-partners-staff-promoters.md`.
```mermaid
flowchart TD
  S(["Find a partner (Discover tab, first 200 only)"]) --> RQ["Send request"]
  RQ --> PEND["pending"]
  PEND --> D{"Counterparty decides"}
  D -->|approve| ACT["active"]
  D -->|reject| REJ["rejected (final)"]
  PEND -->|either side| BLK["blocked (final)"]
  ACT -->|either side| END["ended (final)"]
  ACT -->|either side| BLK
  ACT --> VS["Partnership only: set venue share 0 to 50 percent"]
  ACT --> RV["Promoter connection only: promoter can revoke"]
  PEND -.-> NT["Request notifies the other side.<br/>Approve and reject do not notify the requester"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class ACT,VS,RV ok
  class REJ,BLK,END bad
```
Who may answer: any member can approve or reject (permission is only organisation.read). Who may start a request: owner, admin or manager (member cannot).

## 14. Promoter earnings chain
Files: `07-feature-partners-staff-promoters.md`, `01-product-model.md`.
```mermaid
flowchart LR
  C["Promoter connection<br/>(active)"] --> A["Promoter assignment on an event<br/>commission terms frozen<br/>default 15 percent"]
  A --> L["Referral link per event<br/>(code + vanity slug, signed)"]
  L --> G["Guest buys ticket through link"]
  G --> AT["Attribution saved on the order"]
  AT --> CM["Commission = rate x amount, rounded down<br/>none on free orders"]
  CM --> PF["Promoter finance and payout"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class C,A,L,G,AT,CM ok
  class PF warn
```
A referral link created without an assignment auto-creates a 0 percent assignment. Promoter screens in the dashboard are still on sample data.

## 15. Staff invitation
Files: `07-feature-partners-staff-promoters.md`.
```mermaid
flowchart TD
  O(["Owner or admin: invite email + role"]) --> I["Invitation saved (pending, 14 days)"]
  I --> U{"Account exists for this email?"}
  U -->|no| NA["Create login with temporary password<br/>force password change"]
  U -->|yes| EX["No account changes"]
  NA --> EM["Email: accept link + temporary password"]
  EX --> EM2["Email: accept link"]
  EM --> L["Invitee opens accept link"]
  EM2 --> L
  L --> SI["Sign in, change password if forced"]
  SI --> AC["Accept: added as member with role and capabilities"]
  AC --> LAND["Lands in the organization studio"]
  AC -.-> W["Gap: accepting never checks the signed-in email matches the invitation"]
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  class W bad
  class LAND ok
```

## 16. Finance: balance to payout
Files: `08-feature-finance-overview-support.md`.
```mermaid
flowchart TD
  S(["Open Finance"]) --> LD["Load: balance, ledger, payouts, bank accounts, orders"]
  LD --> F{"Load ok?"}
  F -->|no| FF["Failure screen: signed-out, no org, forbidden, or API error with retry"]
  F -->|yes| SH["Show balance (available, pending, reserved, held)"]
  SH --> B{"Bank account on file?"}
  B -->|no| NB["Payout request disabled.<br/>Save bank details button is disabled in the UI"]
  B -->|yes| AMT{"Available at least 100 rupees?"}
  AMT -->|no| NA["Payout request disabled"]
  AMT -->|yes| RQ["Request payout (one idempotency key per attempt)"]
  RQ --> GT{"Gateway checks"}
  GT -->|"payouts switched off"| E503["503: payouts disabled"]
  GT -->|"not owner or admin"| E403["403"]
  GT -->|ok| OK["Payout requested. Admin processes it (see admin console)"]
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  class FF,E503,E403 bad
  class OK,SH ok
```

## 17. How pages load data and fail
Files: `08-feature-finance-overview-support.md`, `02-frontend-architecture.md`.
```mermaid
flowchart LR
  subgraph SERVER["Server pages (overview, finance)"]
    A["Loader runs on server with cookie"] --> B{"Result"}
    B -->|data| C["Render"]
    B -->|"typed failure"| D["signed-out, no-organization, forbidden, api"]
    D -->|api only| R["Retry button: router.refresh"]
  end
  subgraph CLIENT["Client screens (calendar, partners, slot requests)"]
    E["Wait for session hydrated"] --> F["fetch with abort + retry counter"]
    F --> G{"Result"}
    G -->|data| H["Render"]
    G -->|error| I["Error text + retry"]
  end
  subgraph QUIET["Quiet failures"]
    J["Support list: any error becomes empty list"]
    K["Onboarding autosave: silent"]
    L["Org whose access call fails is dropped"]
  end
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class J,K,L bad
```

## 18. Fixture to live: suggested order
Files: `13-status-gaps-and-roadmap.md`.
```mermaid
flowchart LR
  S1["Step 1: Events list + detail<br/>from real events"] --> S2["Step 2: Event edit wired to PATCH"]
  S2 --> S3["Step 3: Fix overview links<br/>(studio prefix)"]
  S3 --> S4["Step 4: Notifications inbox<br/>(branch exists) + search"]
  S4 --> S5["Step 5: Promoter studio screens<br/>+ fix promoter access"]
  S5 --> S6["Step 6: Settings, marketing, door mode"]
  S6 --> S7["Step 7: Atomic publish<br/>(single server call)"]
  S7 --> S8["Step 8: Calendar semantics<br/>(slots on accept, UTC fix)"]
  S8 --> S9["Step 9: Delete legacy trees,<br/>add real end-to-end smoke test"]
  classDef step fill:#e8f0fe,stroke:#4a6fa5,color:#111
  class S1,S2,S3,S4,S5,S6,S7,S8,S9 step
```

## 19. Defect map: where the risk sits
Files: `13-status-gaps-and-roadmap.md`.
```mermaid
flowchart TD
  ROOT(["Partner dashboard risks"]) --> SEC["Security"]
  ROOT --> DAT["Data truthfulness"]
  ROOT --> WF["Workflow"]
  ROOT --> OPS["Operations"]
  SEC --> S1["Open redirect on change-password next"]
  SEC --> S2["Support POST routes: no CSRF"]
  SEC --> S3["Invitation accept: no email check"]
  SEC --> S4["reCAPTCHA off on preview deploys"]
  DAT --> D1["Events list and detail are fixtures"]
  DAT --> D2["Promoter /access returns no permissions"]
  DAT --> D3["Analytics up to about 10 min stale"]
  DAT --> D4["Overview links 404"]
  WF --> W1["Publish is not atomic"]
  WF --> W2["Calendar auto-creates a venue on read"]
  WF --> W3["Block times saved as UTC"]
  WF --> W4["Slot accept does not update calendar"]
  OPS --> O1["Gateway cold start 30 to 40 s"]
  OPS --> O2["Legacy trees: 40k dead lines, tests still run"]
  OPS --> O3["No real end-to-end test"]
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class S1,S2,S3,S4,D2 bad
  class D1,D3,D4,W1,W2,W3,W4,O1,O2,O3 warn
```
