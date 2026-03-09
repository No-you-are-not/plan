
The system would be designed to use a **hybrid payment approach** to support both web and mobile users:

- **Stripe** — for web checkout and billing management
- **RevenueCat** — for in-app purchases (iOS/Apple), connected to Apple's payment system

Subscriptions are **per organization**, with a fixed price per tier. Each tier allows a maximum number of users.

---

## Key Principles


| **Per-organization billing**        | One subscription per organization, not per user                      |
| ----------------------------------- | -------------------------------------------------------------------- |
| **Tier-based pricing**              | Fixed price per tier, each tier allows a maximum number of users     |
| **No disruption to existing users** | Current B2B customers continue as-is — no forced migration           |
| **Hybrid payments**                 | Stripe for web, RevenueCat for mobile, kept in sync                  |
| **Whole-app gating**                | Subscription gates access to the entire app, not individual features |

---

## Subscription Tiers (just an example, can be changed later)

| | Trial | Starter | Professional | Enterprise |
|---|---|---|---|---|
| **Price** | Free | Fixed/month | Fixed/month | Custom (B2B contract) |
| **Duration** | 14 days | Monthly or Annual | Monthly or Annual | Custom |
| **Seats** | Up to 5 | Up to 10 | Up to 50 | Unlimited |
| **Teams** | 1 | 3 | 10 | Unlimited |
| **Support** | — | Standard | Priority | Dedicated |

> **Note:** Existing B2B customers will be mapped to the **Enterprise** tier with manual billing — nothing changes for them.

---

## How It Works

### New Customer Flow

```
Sign Up → Create Organization → 14-Day Free Trial
                                      │
                          Trial expires │
                                      ▼
                    ┌─── Web ──► Stripe Checkout
                    │
    Choose payment ─┤
                    │
                    └─── iOS ──► RevenueCat / App Store
                                      │
                                      ▼
                          Subscription Active ✓
```

### Existing B2B Customer Flow

```
Current State                    After Migration
─────────────                    ───────────────
Org exists in system      →      Subscription created automatically
Users have roles          →      No changes to users
Pay via B2B contract      →      billingType = "manual", status = "active"
Full access               →      Full access continues unchanged
```

**Zero disruption.** Existing customers won't see any difference. Their subscription is marked as `manual` billing, which bypasses all payment checks.

---

## Access Control

When a user makes a request, the system checks three layers:

```
1. Authentication (AWS Cognito)     — Is the user logged in?
2. Role Check (@auth directive)     — Does the user have the right role?
3. Subscription Check (NEW)         — Does their org have an active subscription?
```

For manually-billed organizations (existing B2B), the subscription check always passes.

### Admin vs TeamAdmin Permissions (suggestion, can be disscussed)

| Action                                      | Admin | TeamAdmin         |
| ------------------------------------------- | ----- | ----------------- |
| View subscription status                    | ✓     | ✓                 |
| View billing history & invoices (read-only) | ✓     | ✓                 |
| Change plan (upgrade/downgrade)             | ✓     | ✗                 |
| Update payment method                       | ✓     | ✗                 |
| Cancel subscription                         | ✓     | ✗                 |
| Add/remove users (seats)                    | ✓     | ✓ (own team only) |

---

## Stripe ↔ RevenueCat Sync

The **Billing Service** (backend) is the single source of truth. Both Stripe and RevenueCat report to it via webhooks:

```
┌──────────────────┐          ┌─────────────────┐
│   Stripe         │──webhook──►                 │
│   (Web Payments) │          │  Billing Service │──► Subscription DB
│                  │          │  (Source of Truth)│
└──────────────────┘          │                  │
                              │                  │
┌──────────────────┐          │                  │
│   RevenueCat     │──webhook──►                 │
│   (iOS Payments) │          └─────────────────┘
└──────────────────┘
```

**One subscription at a time:** A user can only have one active payment source — either Stripe or RevenueCat, never both. If a subscription is active through one channel, the other channel's paywall is blocked and shows a message: *"Your subscription is managed via [Web / App Store]. Please manage it there."*

### Switching Between Payment Providers

Users should be able to switch from Stripe to RevenueCat (or vice versa). The process is supposed to be designed to be seamless

**Key design rules:**
- Switching is only allowed when the current subscription is **canceled** or **expired** — never while active, to prevent double billing
- The Billing Service tracks `billingType` as a mutable field, not a permanent choice
- During the gap between cancellation and new subscription, the user retains access until the current period ends (they already paid for it)
- The subscription document is **reused**, not deleted — history and seat configuration are preserved, only the payment source fields change
- Works in both directions: Stripe → RevenueCat and RevenueCat → Stripe

> **For the user it looks like:** Cancel on one platform → finish the period you paid for → subscribe on the other platform. No data loss, no downtime.

---

## Seat Limits

Each subscription tier has a **fixed price** and a **maximum number of allowed users**. The price does not change based on how many users are active — it only changes when the organization upgrades or downgrades to a different tier.

Seats = number of active users in an organization with app-relevant roles.

| Event | What Happens |
|---|---|
| Admin adds a user | System checks if `currentSeats < maxSeats` for the tier |
| Limit not reached | User is created normally |
| Limit reached | User creation is blocked — admin sees a prompt to upgrade to a higher tier |
| Admin removes/disables a user | Seat count decreases, freeing up space for new users |
| Admin upgrades tier | `maxSeats` increases, more users can be added |


---

## Technical Changes

### What's New

| Component                  | Description                                                           |
| -------------------------- | --------------------------------------------------------------------- |
| **Billing Service**        | New microservice handling subscriptions, webhooks, seat management    |
| **Subscription Model**     | MongoDB collection storing plan, status, seats, Stripe/RevenueCat IDs |
| **Billing Events Log**     | Audit trail of all billing-related events                             |
| **Subscription Directive** | New GraphQL directive `@requiresSubscription` for access gating       |
| **Subscription UI**        | Admin panel page for managing billing, viewing invoices               |
| **Paywall Screen**         | Shown when trial expires or subscription lapses                       |

### What Changes

| Component | Change |
|---|---|
| **Organization Model** | Adds optional `subscription` reference and `billingEmail` field |
| **GraphQL Gateway** | Adds subscription check directive alongside existing auth |
| **User Creation Flow** | Adds seat limit check before allowing new users |

### What Stays the Same

| Component | Why |
|---|---|
| **User Model** | Subscription lives on the org, not the user — no user schema changes |
| **Roles & Permissions** | Existing RBAC is untouched; subscription is an additional layer |
| **Authentication** | AWS Cognito stays as-is |
| **All existing functionality** | Nothing is removed or modified for current users |

---

## Implementation Phases

### Phase 1 — Backend Foundation

- Create Billing Service microservice
- Define Subscription and BillingEvent models
- Add subscription reference to Organization schema
- Build subscription CRUD API (exposed via GraphQL Gateway)
- Run migration script to add subscriptions for all existing organizations

### Phase 2 — Stripe Integration (Web)

- Configure Stripe Products with a fixed Price per tier
- Implement Stripe Checkout session creation
- Build webhook handler for payment events

### Phase 3 — Frontend Subscription UI

- Subscription status page in admin panel
- Stripe Checkout redirect for new subscriptions
- Page for billing management
- Paywall/gate screen when subscription is missing or expired
- Seat limit enforcement in user creation flow
- Trial countdown banner (shows days remaining)

### Phase 4 — RevenueCat Integration (Mobile)

- Configure RevenueCat dashboard with matching products
- Add RevenueCat SDK to the Capacitor iOS app
- Build webhook handler for RevenueCat events
- Add mobile paywall screen
- Implement cross-platform sync (prevent duplicate subscriptions)

---

## Risk Mitigation

| Risk                               | Mitigation                                                |
| ---------------------------------- | --------------------------------------------------------- |
| Breaking existing users            | `billingType: 'manual'` bypasses all payment checks       |
| Payment failures locking users out | 7-day grace period for `past_due` status                  |
| Stripe/RevenueCat out of sync      | Single source of truth in our DB; both write via webhooks |

---

## Summary

This plan adds subscription billing to 5i Factory Portal with **zero impact on existing customers**. It supports both web (Stripe) and mobile (RevenueCat/Apple) payments, scales pricing by seats, and is implemented incrementally across 5 phases — each delivering standalone value.
