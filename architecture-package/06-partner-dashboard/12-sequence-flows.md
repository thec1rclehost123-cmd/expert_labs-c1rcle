# 12. Sequence flows

## Login and workspace selection
```mermaid
sequenceDiagram
  participant B as Browser
  participant N as Dashboard BFF (/api/auth, /api/bff)
  participant G as Gateway
  B->>N: POST /api/auth/login
  N->>G: POST /api/v2/auth/login
  G-->>N: session cookie + token
  N-->>B: re-scoped cookie + CSRF cookie
  B->>N: GET /api/bff/organizations
  N->>G: GET /organizations
  loop each org
    B->>N: GET /api/bff/organizations/:id/access
    N->>G: GET /organizations/:id/access (X-Organization-Id)
  end
  alt zero orgs
    B->>B: pending invite or /onboard
  else one org
    B->>B: setActiveOrg, land on first visible tab
  else many
    B->>B: /partner/select-organization
  end
```

## Onboarding to approval
```mermaid
sequenceDiagram
  participant U as Applicant
  participant W as Wizard (/onboard)
  participant N as BFF
  participant G as Gateway
  participant FB as Firebase (phone)
  participant A as Platform admin
  U->>W: choose role, sign up
  W->>N: POST /api/auth/signup, otp/send, otp/verify
  W->>FB: signInWithPhoneNumber (reCAPTCHA)
  W->>G: POST /onboarding/verify-document (phone, idToken) [direct Bearer]
  W->>N: POST /api/bff/onboarding/applications (IK)
  W->>N: PATCH .../applications/:id (If-Match) autosave
  W->>N: POST .../documents/upload (BFF: upload-url, PUT storage, record)
  W->>N: POST .../submit (IK, If-Match)
  loop every 10 s
    W->>N: GET /api/bff/onboarding/me
  end
  A->>G: verify each document, POST approve (TIER2)
  G->>G: create org (+ starter venue), plan fee, audit
  W->>W: approved, routeAfterAuth
```

## Slot request
```mermaid
sequenceDiagram
  participant H as Host dashboard
  participant N as BFF
  participant G as Gateway
  participant V as Venue dashboard
  H->>N: POST /api/bff/slot-requests (draft, key)
  N->>G: POST /organizations/:host/events (key-event)
  N->>G: POST /venues/:v/slot-requests (key-slot-request)
  V->>N: GET /api/bff/slot-requests?direction=incoming
  V->>N: POST .../:id/accept
  N->>G: POST /venues/slot-requests/:id/accept (venue.schedule)
  Note over G: status accepted. Calendar and event are NOT changed. No notification
```

## Event create and publish (browser-orchestrated)
```mermaid
sequenceDiagram
  participant B as Browser (publishVenueEvent)
  participant G as Gateway
  participant S as Cloud Storage
  B->>G: POST /organizations/:id/poster/upload-url
  B->>S: PUT poster (signed URL)
  B->>G: POST /organizations/:id/events (IK wf-event)
  loop each tier
    B->>G: POST /events/:id/ticket-tiers (IK wf-tier-n)
  end
  opt custom compensation
    B->>G: PATCH /events/:id (If-Match)
  end
  loop each promoter
    B->>G: POST /events/:id/promoter-assignments
  end
  B->>G: POST /events/:id/review
  B->>G: POST /events/:id/publish
  Note over B,G: Not atomic. A failure leaves a partial draft, retries replay idempotently
```

## Partnership request and answer
```mermaid
sequenceDiagram
  participant H as Host
  participant G as Gateway
  participant V as Venue
  H->>G: POST /partnerships {venueId, initiatedBy host}
  G->>G: status pending, emit partnership.requested
  G-->>V: notification (inbox exists, dashboard does not read it)
  V->>G: POST /partnerships/:id/approve (counterparty only)
  G->>G: status active (no event, requester not notified)
  V->>G: POST /partnerships/:id/venue-share {rate 0..50}
```

## Staff invite and accept
```mermaid
sequenceDiagram
  participant O as Owner/admin
  participant G as Gateway
  participant E as Email (Resend)
  participant S as Invitee
  O->>G: POST /organizations/:id/invitations (IK)
  G->>G: save invitation, provision login + temp password, set rotation required
  G->>E: accept URL + temporary password
  S->>S: open /invitations/:id/accept
  S->>G: sign in, forced /change-password
  S->>G: POST /invitations/:id/accept
  Note over G: no check that session email equals invitation email
  G-->>S: organisation, member added with capabilities
```

## Payout request
```mermaid
sequenceDiagram
  participant B as Browser (PayoutRequestAction)
  participant G as Gateway
  B->>G: POST /organizations/:id/payouts {amountPaise, bankAccountId, idempotencyKey}
  alt PAYOUTS_ENABLED false
    G-->>B: 503
  else finance.payouts.manage missing
    G-->>B: 403
  else ok
    G-->>B: 201 payout, router.refresh()
  end
```
