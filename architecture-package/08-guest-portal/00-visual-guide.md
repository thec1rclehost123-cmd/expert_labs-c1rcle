# 00. Guest portal: visual guide

Start here if you prefer pictures. Each diagram points to the file that explains it. Colours:

- 🟩 **green** = live and working
- 🟨 **yellow** = fixture, partial, blocked in production or a decision is pending
- 🟥 **red** = defect or risk (see `07-status-gaps-and-roadmap.md`)
- ⬜ **grey** = infrastructure or neutral step

How to view: GitHub, VS Code (Markdown Preview Mermaid Support), or https://mermaid.live. A browser version is in `visual-guide.html` (needs internet for the Mermaid library).

Contents: 1 System map · 2 Page map (production) · 3 Sign-in and gating · 4 The BFF path · 5 Discovery data flow · 6 RSVP · 7 Paid checkout · 8 Payment recovery · 9 Tickets and the QR · 10 Promoter link flow · 11 What the portal does not use · 12 SEO map · 13 Security findings · 14 Production vs preview · 15 Defect map · 16 Fix order

---

## 1. System map
Files: `02-architecture-auth-and-bff.md`.
```mermaid
flowchart LR
  G["Guest browser"] --> PX["proxy.ts<br/>canonical host, cookie gate for private pages"]
  PX --> PG["Pages (server and client)"]
  PG -->|"same-origin /api/*"| BFF["BFF route handlers<br/>origin check, CSRF, id checks"]
  PG -->|"direct, Bearer token<br/>(tickets page reads)"| GW
  BFF -->|"cookie forwarded"| GW["Gateway /api/v2<br/>(Render)"]
  GW --> CORE["Core services"] --> FS[("Firestore")]
  GW --> RZ["Razorpay (orders, refunds)"]
  G -->|"Razorpay widget script"| RZS["checkout.razorpay.com<br/>(no SRI, no CSP)"]
  RZ -->|webhook| GW
  classDef live fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef infra fill:#eeeeee,stroke:#888,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class PG,BFF,GW,CORE live
  class PX,FS,RZ infra
  class RZS warn
```

## 2. Page map (what a production visitor really gets)
Files: `01-routes-and-screens.md`.
```mermaid
flowchart TD
  HOME["/ home: fixture hero + live upcoming events"] --> EXP["/explore: live, client-side filters, 20 events"]
  EXP --> EVT["/event/id: live"]
  EVT --> CHK["/checkout/id: live, needs sign-in"]
  CHK --> CONF["/confirmation/id: live, decorative pass"]
  CONF --> TIX["/tickets: live wallet + rotating QR"]
  HOME --> APP["/app, /about, /privacy, /terms: marketing and stubs"]
  HOME --> HOSTS["/hosts: empty in production"]
  HOME --> VEN["/venue/slug and /host/slug: live, minimal"]
  HOME --> PRO["/profile: BLOCKED in production"]
  HOME --> HLP["/help: BLOCKED in production"]
  HOME --> SGN["/signup: redirects to login, says use the mobile app"]
  HOME --> VAN["/handle/slug promoter link: resolves then redirects to the event"]
  CHK -.fixture slugs reachable.-> FX["/checkout/neon-nights etc and /confirmation/preview-*"]
  classDef live fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class EXP,EVT,CHK,CONF,TIX,VEN,VAN live
  class HOME,APP,HOSTS,PRO,HLP,SGN warn
  class FX bad
```

## 3. Sign-in and page gating
Files: `02-architecture-auth-and-bff.md`.
```mermaid
flowchart TD
  V(["Visit /profile, /tickets, /help, /checkout or /confirmation"]) --> C{"Session cookie present?"}
  C -->|no| RD["307 to /login?next=path (query string dropped)"]
  C -->|yes| SRV{"Server session valid?"}
  SRV -->|no| RD2["Server redirect to login"]
  SRV -->|yes| PAGE["Page renders, private and no-store"]
  RD --> LG["Login form: email and password"]
  LG --> API["POST /api/auth/login (BFF re-scopes cookie, mints CSRF)"]
  API --> OK{"Login ok?"}
  OK -->|no| ERR["Incorrect email or password, too many attempts, or unavailable"]
  OK -->|yes| NX["safeNextPath: only same-site paths, else /profile"]
  NX --> PAGE
  SU["/signup"] -.redirects to login with a note.-> LG
  PAGE -.-> IDLE["30 minute idle timer logs the user out"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class PAGE,NX,API ok
  class ERR bad
  class SU,IDLE,RD warn
```
The proxy only checks that a cookie exists. The gateway reports every user as role partner, so a partner or admin can sign in here.

## 4. The BFF path and its checks
Files: `02-architecture-auth-and-bff.md`.
```mermaid
flowchart TD
  B(["Browser calls /api/..."]) --> O{"Same origin?"}
  O -->|"Origin mismatch or cross-site"| X1["Rejected"]
  O -->|"ok (a missing Origin is allowed)"| W{"State-changing call?"}
  W -->|yes| CS{"CSRF header matches the cookie?"}
  CS -->|no| X2["Rejected"]
  CS -->|yes| ID
  W -->|no| ID{"Ids validated by regex?"}
  ID -->|"orders, qr, verify, wallet, recovery"| FW["Forward with the browser cookie"]
  ID -->|"help ticket routes"| NV["No validation: ticketId goes into the gateway path"]
  FW --> GWY["Gateway"]
  NV --> GWY
  NV -.-> INJ["Path injection risk"]
  GWY -.-> IP["Client IP is not forwarded: anonymous auth shares one rate-limit bucket"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class FW,GWY ok
  class X1,X2,INJ,NV,IP bad
```
The help POST routes check origin but not CSRF, and the help GET detail route does not check origin.

## 5. Discovery data flow
Files: `03-discovery-seo-and-promoter-links.md`.
```mermaid
flowchart LR
  EXP["Explore page"] --> D["GET /public/discovery<br/>20 soonest events"]
  D --> V["N venue reads (one per event)"]
  EXP --> FILT["Search, city, date and sort run in the browser"]
  EVT["Event page"] --> E1["GET /public/events/id"]
  E1 --> E2["GET venue by id"]
  E1 --> E3["GET host by id"]
  E1 -.already embeds.-> EMB["venue and organizer in the same response"]
  SM["Sitemap"] --> D
  SM --> NV["Venues and hosts: always empty"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class D,E1,E2,E3,V ok
  class FILT,EMB,NV warn
```
No public list endpoint exists for venues or hosts, and the gateway sends no cache headers on public reads.

## 6. Free ticket (RSVP)
Files: `04-booking-and-checkout.md`.
```mermaid
flowchart TD
  S(["Pick a free tier (max 1 ticket)"]) --> Q["Confirm"]
  Q --> R["POST /api/rsvp {eventId, tierId}"]
  R --> G{"Gateway result"}
  G -->|created| OK["Paid 0 order + ticket, go to /confirmation/orderId"]
  G -->|"conflict (already RSVPed)"| DUP["Message: already on the list, link to My Tickets"]
  G -->|unauthorized| LG["Redirect to login"]
  G -->|other| ER["RSVP failed"]
  OK -.contact saved as.-> PH["RSVP Guest, no email, no phone<br/>(attendee details are not sent)"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class OK ok
  class ER bad
  class DUP,LG,PH warn
```

## 7. Paid checkout
Files: `04-booking-and-checkout.md`, `../10-payments-and-finance/02-checkout-and-payment-flow.md`.
```mermaid
flowchart TD
  S1["Step 1: choose tiers"] --> S2["Step 2: name, email (phone optional)<br/>goes only to Razorpay prefill"]
  S2 --> Q["Quote from the server (total shown from the quote only)"]
  Q --> H["Hold for 10 minutes (no countdown shown)"]
  H --> PC{"Hold total equals quoted total?"}
  PC -->|no| RQ["Re-quote with a new key"]
  RQ --> H
  PC -->|yes| PA["Payment attempt: one Razorpay order per hold"]
  PA --> WG["Razorpay widget"]
  WG --> RES{"Result"}
  RES -->|dismissed| BACK["Back to the page, hold still active for a few minutes"]
  RES -->|paid| VF["Verify with signature"]
  VF --> VR{"Gateway answer"}
  VR -->|confirmed| CF["Confirmation page"]
  VR -->|"409 refund message"| RF["Show refund message, clear recovery cookie"]
  VR -->|"other error"| PR["Processing: poll My Tickets about 30 s,<br/>then tell the guest to contact support"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class CF,Q,H,PA ok
  class RQ,BACK,PR warn
  class RF bad
```
Not sent by the portal: attendee, promo code, referral code. Not shown: hold countdown, QR on the confirmation page.

## 8. Payment recovery
Files: `04-booking-and-checkout.md`.
```mermaid
sequenceDiagram
  participant U as Guest browser
  participant B as BFF
  participant G as Gateway
  U->>B: create hold
  B->>G: hold
  G-->>B: hold id
  B-->>U: set httpOnly cookie with the hold id (15 min)
  Note over U: Guest closes the tab or loses connection mid-payment
  U->>B: returns to the checkout page, GET recovery
  B->>G: status for the hold id from the cookie
  G-->>B: pending, processing, fulfilled, failed, cancelled, expired, refunded or requires_support
  B-->>U: status
  alt fulfilled
    U->>U: go to the confirmation page
  else refunded or terminal
    U->>B: DELETE recovery cookie
  end
```

## 9. Tickets and the rotating QR
Files: `05-tickets-wallet-profile-support.md`.
```mermaid
flowchart TD
  T["/tickets"] --> W["GET /api/wallet/tickets (first 50 entitlements)"]
  W --> EV["Browser reads event and venue for each distinct event"]
  EV --> CARD["Cards grouped by order and tier"]
  CARD --> M["Open a ticket"]
  M --> ST{"Ticket status"}
  ST -->|active| QR["GET /api/tickets/first-ticket-id/qr every 30 s, shown as a QR"]
  ST -->|"used, cancelled, pending"| NQ["No entry code message"]
  QR -.-> ONE["Only the FIRST ticket's QR is reachable for a multi-ticket card"]
  QR -.-> OFF["No offline copy: a failed fetch shows Entry code unavailable"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class W,CARD,QR ok
  class NQ,OFF warn
  class ONE bad
```

## 10. Promoter link flow (what is broken)
Files: `03-discovery-seo-and-promoter-links.md`.
```mermaid
flowchart LR
  A["Guest opens /handle/slug"] --> B["Server resolves and redirects to /event/slug?ref=CODE"]
  B --> C["Event page ignores ?ref"]
  C --> D["No click is posted (tracker is never rendered)"]
  C --> E["Checkout sends no referral code"]
  E --> F["Order has no promoter attribution"]
  F --> G["Promoter earns no commission, conversions stay at 0"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class A,B ok
  class C,D,E,F,G bad
```

## 11. Backend features the portal does not use
Files: `05-tickets-wallet-profile-support.md`, `06-backend-public-and-guest-api.md`.
```mermaid
flowchart TD
  BE["Gateway guest-side features"] --> U1["Used: public reads, quote, hold, attempt, verify, RSVP,<br/>order by id, wallet tickets, ticket QR, profile read, support intake"]
  BE --> N1["Promo codes and referral codes"]
  BE --> N2["Attendee details on the order"]
  BE --> N3["Profile save (PUT /profile/me)"]
  BE --> N4["Follows and guest notifications"]
  BE --> N5["Cover wallet QR (guest can mint, cannot find the wallet id)"]
  BE --> N6["Order list, order status, wallet summary"]
  BE --> N7["Search endpoint, venue and host lists (list does not exist)"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class U1 ok
  class N1,N2,N3,N4,N5,N6,N7 warn
```

## 12. SEO map
Files: `03-discovery-seo-and-promoter-links.md`.
```mermaid
flowchart TD
  S["SEO setup"] --> R["robots.ts: production allows crawling, other environments disallow"]
  S --> SM["sitemap.ts: home, app, explore (if events), live events"]
  S --> MD["Metadata helpers: canonical, Open Graph, Twitter"]
  S --> JL["JSON-LD: Organization and WebSite on home, Breadcrumbs on events"]
  SM --> BUG1["Event pages are listed in the sitemap..."]
  BUG1 --> BUG2["...but every event page sends robots noindex"]
  JL --> BUG3["The Event schema block is in a component no page renders"]
  MD --> BUG4["Event and explore pages hard-code the canonical origin"]
  SM --> BUG5["Venues and hosts are never listed"]
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class R,MD ok
  class BUG1,BUG2,BUG3,BUG4,BUG5 bad
```

## 13. Security findings map
Files: `07-status-gaps-and-roadmap.md`.
```mermaid
flowchart TD
  ROOT(["Guest portal security"]) --> FE["Frontend"]
  ROOT --> BE["Backend"]
  FE --> f1["Help ticket routes: unvalidated id goes into the gateway path"]
  FE --> f2["No client IP forwarded: shared rate-limit bucket"]
  FE --> f3["No CSP, no HSTS, Razorpay script without SRI"]
  FE --> f4["Help POST: origin check only, no CSRF"]
  FE --> f5["Bad percent encoding gives a 500"]
  BE --> b1["Support tickets return internal notes, assignee, timeline, SLA to the guest"]
  BE --> b2["Every user is reported as role partner"]
  BE --> b3["Referral secret falls back to dev-secret if unset"]
  BE --> b4["Promoter clicks can be inflated anonymously"]
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class f1,b1 bad
  class f2,f3,f4,f5,b2,b3,b4 warn
```

## 14. Production vs preview
Files: `01-routes-and-screens.md`.
```mermaid
flowchart LR
  subgraph PROD["Production"]
    p1["Live: explore, event, checkout, confirmation, tickets"]
    p2["Blocked: profile, help"]
    p3["Empty: hosts. 404: public profile"]
    p4["Fixture still reachable: checkout slugs, preview confirmations"]
  end
  subgraph PREV["Preview and development"]
    v1["Everything in production"]
    v2["Plus live: profile, help"]
    v3["Plus fixtures: hosts, public profile, home drops, venue and host pages"]
    v4["robots: noindex nofollow on every response"]
  end
  classDef ok fill:#d9f7d9,stroke:#2e8b2e,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  class p1,v1,v2 ok
  class p2,p3,v3,v4 warn
  class p4 bad
```

## 15. Defect map
Files: `07-status-gaps-and-roadmap.md`.
```mermaid
flowchart TD
  ROOT(["Guest portal risks"]) --> REV["Revenue and data"]
  ROOT --> TRUST["Trust and security"]
  ROOT --> REACH["Reach and SEO"]
  ROOT --> UX["Experience"]
  REV --> r1["Promoter attribution broken"]
  REV --> r2["Placeholder buyer contact on every order"]
  REV --> r3["No promo code entry"]
  TRUST --> t1["Internal support notes visible to guests"]
  TRUST --> t2["Help route path injection"]
  TRUST --> t3["No CSP or HSTS"]
  REACH --> s1["Event pages noindex"]
  REACH --> s2["No web sign-up, profile and help blocked in production"]
  REACH --> s3["Venues and hosts missing from sitemap"]
  UX --> u1["Confirmation page shows Guest and a wrong reference"]
  UX --> u2["Only first ticket's QR; profile edit does nothing"]
  UX --> u3["No hold countdown, fonts never load"]
  classDef bad fill:#ffd6d6,stroke:#c0392b,color:#111
  classDef warn fill:#fff3b0,stroke:#b58900,color:#111
  class r1,t1,t2 bad
  class r2,r3,t3,s1,s2,s3,u1,u2,u3 warn
```

## 16. Suggested fix order
Files: `07-status-gaps-and-roadmap.md`.
```mermaid
flowchart LR
  F1["Step 1: Redact internal support fields,<br/>validate help route ids, add CSRF"] --> F2["Step 2: Forward client IP from the BFF"]
  F2 --> F3["Step 3: Send attendee, promo and referral codes,<br/>wire the click tracker"]
  F3 --> F4["Step 4: Decide guest role and web sign-up"]
  F4 --> F5["Step 5: Fix event SEO and sitemap"]
  F5 --> F6["Step 6: Enable profile and help in production, add profile save"]
  F6 --> F7["Step 7: Per-ticket QR, wallet pagination, order history"]
  F7 --> F8["Step 8: CSP, HSTS, SRI, remove fixture routes and dead code"]
  classDef step fill:#e8f0fe,stroke:#4a6fa5,color:#111
  class F1,F2,F3,F4,F5,F6,F7,F8 step
```
