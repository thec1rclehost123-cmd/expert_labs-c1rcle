# 13. Sequence flows

## Admin sign-in and session
```mermaid
sequenceDiagram
  participant B as Browser (admin console)
  participant N as Console BFF (/api/auth)
  participant G as Gateway
  B->>N: POST /api/auth/login (same-origin)
  N->>G: POST /api/v2/auth/login
  G-->>N: session cookie + accessToken, expiresAt
  N-->>B: re-scoped cookie + CSRF cookie (admin.c1rcle.csrf)
  B->>B: token kept in memory, hard redirect to /
  B->>G: Bearer calls to /api/v2/admin/* (cross-origin)
  G->>G: requireAdmin (v2_admins) then authorize(action)
  alt 401
    B->>N: POST /api/auth/refresh (CSRF), retry once
  end
  Note over B,G: No role in the browser session, no mustChangePassword handling
```

## Dual-control action (commission change)
```mermaid
sequenceDiagram
  participant A as Super admin A
  participant G as Gateway
  participant S as Super admin B
  A->>G: POST /admin/proposals {COMMISSION_ADJUST, reason, payload} (IK)
  G->>G: status pending, audit proposal.raise
  S->>G: POST /admin/proposals/:id/approve
  G->>G: require S != A, S is super, status approved, audit
  Note over G: No expiry and no executed state
  S->>G: POST /admin/proposals/:id/adjust-commission (IK)
  G->>G: read payload from stored proposal, set platformFeePercent, audit
  Note over G: Can be executed again with a new key
```

## Onboarding approval
```mermaid
sequenceDiagram
  participant A as Admin (TIER2)
  participant G as Gateway
  participant ST as Cloud Storage
  A->>G: GET /admin/onboarding/applications/:id
  loop each document
    A->>G: GET .../documents/:label/read-url
    G->>ST: sign GET (10 min)
    A->>A: open signed URL in new window
    A->>G: POST .../verify or .../reject (reason)
  end
  A->>G: POST .../approve (IK)
  G->>G: all required docs verified? else 400
  G->>G: create organization (plan fee), starter venue if venue
  G->>G: save request approved, audit
```

## Refund lifecycle
```mermaid
sequenceDiagram
  participant A as Admin
  participant G as Gateway
  participant RZ as Razorpay
  A->>G: POST /admin/refunds {orderId, amount, reason}
  G->>G: order paid -> refund_requested, freeze tickets, approversRequired by amount
  A->>G: POST /refunds/:id/approve (other admin, 0, 1 or 2 approvals)
  A->>G: POST /refunds/:id/execute
  G->>G: claim executing (compare-and-set)
  G->>RZ: refund (key refund:id)
  alt definite failure
    G->>G: failed, release tickets, restore order
  else ambiguous
    G->>G: stay executing, 409, wait for webhook or reconciliation
  else processed
    G->>G: post journals, order refunded, void entitlements, settled
  end
```

## Payout (with release gate)
```mermaid
sequenceDiagram
  participant P as Partner
  participant A as Admin
  participant G as Gateway
  P->>G: POST /organizations/:id/payouts
  G-->>P: 503 when PAYOUTS_ENABLED is false
  A->>G: POST /admin/payouts/batch-run
  G-->>A: 503 when gate off
  A->>G: POST /admin/payouts/:id/complete {reference, method, proof}
  Note over G: Not behind the gate. Posts completion journal, then marks paid
```

## Reconciliation and adjustment
```mermaid
sequenceDiagram
  participant A as Admin
  participant G as Gateway
  A->>G: POST /admin/reconciliation/runs
  G->>G: scan orders, provider payments, entitlements, payouts, create discrepancies
  A->>G: GET /admin/reconciliation/discrepancies?state=open
  alt fix data upstream
    A->>G: POST .../discrepancies/:id/retry (re-runs whole scan)
  else manual correction
    A->>G: raise FINANCIAL_ADJUSTMENT proposal, super B approves
    A->>G: POST /admin/proposals/:id/post-financial-adjustment
  end
  A->>G: POST .../discrepancies/:id/resolve (reason)
  Note over G: Resolve does not repair data, next run can reopen it
```

## Support ticket handling
```mermaid
sequenceDiagram
  participant U as Guest or partner
  participant G as Gateway
  participant A as Admin (any role)
  U->>G: POST /support/tickets (requester from session)
  A->>G: GET /admin/support/tickets?status=open
  A->>G: POST .../assign (name supplied by caller)
  A->>G: POST .../reply (status waiting_on_customer)
  U->>G: POST /support/tickets/:id/messages (back to in_progress)
  A->>G: POST .../resolve (reason), then .../close
```
