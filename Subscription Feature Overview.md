

This document describes the subscription and billing system implemented in Factory Portal from a user-facing perspective, including all flows, features, and QA testing guidance.

## How Subscriptions Work

Subscriptions are **per organization**, not per user. One subscription controls access for everyone in the organization. Each plan tier defines limits on the number of users (seats) and teams allowed.

### Subscription Tiers

|              | Trial   | Starter                   | Professional              | Enterprise (B2B)         |
| ------------ | ------- | ------------------------- | ------------------------- | ------------------------ |
| **Price**    | Free    | Per Stripe config ($29)   | Per Stripe config ($99)   | Manual (custom contract) |
| **Duration** | 14 days | Monthly                   | Monthly                   | Unlimited                |
| **Seats**    | 10      | From Stripe metadata (10) | From Stripe metadata (50) | 9999 (unlimited)         |
| **Teams**    | 3       | From Stripe metadata (3)  | From Stripe metadata (10) | 9999 (unlimited)         |

> **Note:** Prices and limits are configured in the **Stripe Dashboard** on each Price object's metadata (`maxSeats`, `maxTeams`). They are NOT hardcoded in the app — changing them in Stripe updates the app automatically.

![Subscription Tiers](Screenshot%202026-03-20%20at%2010.39.04.png)

---

## User Flows

### 1. New User Sign-Up

```
User fills sign-up form → Creates Cognito account + MongoDB user + Organization
                        → 14-day free trial subscription created automatically
                        → User enters the app
                        → Trial banner shows at top: "Your free trial ends in X days"
```

**What happens in the background:**
- User + Organization created in MongoDB (via User Service + Organization Service)
- Trial subscription created in Billing DB: `status: trialing`, `trialEndsAt: +14 days`, `maxSeats: 10`, `maxTeams: 3`
- No Stripe involvement — no card is collected

---

### 2. Trial Expiry

```
14 days pass → trialEndsAt is in the past
             → User opens the app
             → Paywall screen appears instead of the app
             → Message: "Your free trial has expired"
             → Two plan options shown with prices from Stripe
```

**What the user sees:**
- Full-screen paywall with plan cards (Starter, Professional)

![Paywall Screen](Screenshot%202026-03-20%20at%2010.50.03.png)
### 3. Subscribing (Stripe Checkout)

```
User clicks "Subscribe" on a plan → Redirected to Stripe Checkout
                                   → Email pre-filled from user account
                                   → User enters card details
                                   → Payment processed by Stripe
                                   → Redirected back to the app
                                   → Paywall gone — app loads normally
```

![Stripe Checkout](Screenshot%202026-03-20%20at%2010.52.44.png)

### 4. Subscription Management Page (`/subscription`)

Accessible from the sidebar (admin users only).

**Shows:**

| Section | Data Source |
|---------|------------|
| **Plan & Status** (plan name, status chip, billing type) | Our DB |
| **Payment method** (card brand, last 4, expiry + edit button) | **Stripe API** (live) |
| **Usage** (seats X/Y, teams X/Y with progress bars) | **Live count** — users from User Service, teams from Organization Service, limits from our DB |
| **Billing Period** (next charge amount + date, days until, period start/end) | **Stripe API** (live) |
| **Change Plan** (upgrade/downgrade cards) | Plans from Billing Service (enriched with Stripe prices) |
| **Cancel / Resume** | Calls Stripe API |
| **Billing History** (invoice table) | **Stripe API** (live) |

## Enterprise

![Enterprise](Screenshot%202026-03-20%20at%2010.54.15.png)

## Professional

![Professional 1](Screenshot%202026-03-20%20at%2010.55.04.png)
![Professional 2](Screenshot%202026-03-20%20at%2010.56.08.png)

### 5. Upgrade 

```
User on Starter clicks "Upgrade" on Professional card
  → Stripe charges prorated difference immediately
  → Plan changes immediately: new limits apply (50 seats, 10 teams)
```

---

### 6. Downgrade

```
User on Professional clicks "Downgrade" on Starter card
  → IF user has more users/teams than Starter allows:
      → Error: "Cannot downgrade: you have 50 users but Starter allows only 10"
  → IF under limits:
      → Downgrade scheduled for end of billing period
      → Current limits stay until then
      → Banner: "Your plan will change to Starter at the end of the current billing period"
```

---

### 7. Cancel Subscription

```
User clicks "Cancel subscription" → Confirmation dialog:
  "Your subscription will remain active until [date]. After that, you'll lose access."
  → User confirms
  → Subscription marked for cancellation at period end
  → Warning banner: "Your subscription is set to cancel on [date]"
  → "Resume" button appears
```

**After period ends:**
- Stripe cancels the subscription
- Webhook fires → our DB: `status: canceled`
- Next time user opens the app → paywall appears

**Resume (undo cancellation):**
- User clicks "Resume" before period end → cancellation undone, subscription continues normally

---

### 8. Update Payment Method

```
User clicks edit icon (pencil) next to payment method
  → Redirected to Stripe Checkout (setup mode — no charge)
  → User enters new card
  → Redirected back to /subscription
  → Payment method display updated
```

---

### 9. Billing History

Shows a table of past invoices on the subscription page:
- **Date** — when the invoice was created
- **Amount** — how much was charged
- **Status** — paid (green), open (blue), uncollectible (red)
- **Link** — opens the Stripe-hosted invoice in a new tab (viewable/downloadable)

All data is fetched **live from Stripe** — not stored in our DB.

---

## Seat & Team Limits

### How Limits Work

| Event | What Happens |
|-------|-------------|
| Admin creates a user | System counts current users live → compares with `maxSeats` from subscription → blocks if at limit |
| Admin deletes a user | Next user creation succeeds (live count decreased) |
| Admin adds a team (edit organization) | System counts current teams live → compares with `maxTeams` → blocks if at limit |
| Admin deletes a team | Next team creation succeeds |
| Limit reached in UI | Warning banner appears inside the form/modal: "[Seats/Teams] limit reached (X/Y). Upgrade your plan to add more." + "Upgrade Plan" button |

**Key design:** Counts are **never stored** — they're computed live from the User and Organization collections every time. This means they're always accurate with zero drift risk.

### Organization Creation Restriction

- **B2B (manual billing) users:** Can create multiple organizations
- **Stripe subscribers:** Cannot create additional organizations — the "New Organization" button is hidden

---

## Data Sources Reference

| Data                                               | Source                                           | Notes                                     |
| -------------------------------------------------- | ------------------------------------------------ | ----------------------------------------- |
| Plan name, status, billingType, maxSeats, maxTeams | **Our DB** (SUBSCRIPTIONS collection)            | Updated via Stripe webhooks               |
| Billing period dates (start, end)                  | **Stripe API** (live from subscription item)     | Fetched on subscription page load         |
| Next charge amount                                 | **Stripe API** (from price on subscription item) |                                           |
| Payment method (card brand, last 4, expiry)        | **Stripe API** (from payment method)             |                                           |
| User count (current seats)                         | **Our Service** (live query)                     | Counts users with matching organizationID |
| Team count (current teams)                         | **Our Service** (live query)                     | Counts teams array length on org document |
| Plan prices and limits for plan cards              | **Stripe API** (from price amount + metadata)    | Cached 5 min in Billing Service           |
| Invoices                                           | **Stripe API** (invoices.list)                   | Not stored in our DB                      |
| Cancel at period end flag                          | **Stripe API** (from subscription object)        |                                           |
| Pending downgrade                                  | **Our DB** (pendingPlan field)                   | Cleared by webhook when applied           |
| Trial end date                                     | **Our DB** (trialEndsAt field)                   | Set on signup, no Stripe involvement      |

---

## Existing B2B Customers

Existing customers were migrated with:
- `billingType: 'manual'` — bypasses all Stripe checks
- `status: 'active'` — always passes paywall
- `maxSeats: 9999`, `maxTeams: 9999` — effectively no limits
- **No Stripe subscription** — no billing period, no invoices, no payment method
- Subscription page shows plan info but no billing details or change plan options

**They see zero changes** — the app works exactly as before.

---

## QA Testing Guide

### Prerequisites
- Access to Stripe Dashboard (test mode) — for verifying payments and webhooks

### Test Cards

| Card Number           | Scenario                                  |
| --------------------- | ----------------------------------------- |
| `4242 4242 4242 4242` | Successful payment                        |
| `4000 0000 0000 0341` | Attaches OK but **fails on first charge** |
| `4000 0000 0000 9995` | Decline — insufficient funds              |
| `4000 0025 0000 3155` | Requires 3D Secure authentication         |

All test cards use any future expiry date and any 3-digit CVC.

### Test Scenarios

#### A. New User Sign-Up + Trial

| Step | Action | Expected Result | Verify In |
|------|--------|----------------|-----------|
| 1 | Register a new user | User created, logged in | App |
| 2 | Check top of app | Trial banner visible: "Your free trial ends in 14 days" | App |
| 3 | Go to `/subscription` | Shows: Plan=Starter, Status=Trial, Trial ends=[date +14 days] | App |
| 4 | Check Stripe Dashboard | No customer/subscription (trial is DB-only) | Stripe |

#### B. Trial Expiry → Paywall

| Step | Action                                                                                    | Expected Result                                | Verify In |
| ---- | ----------------------------------------------------------------------------------------- | ---------------------------------------------- | --------- |
| 1    | Ask developer to set `trialEndsAt` to a past date (since trial is handled only in our db) | —                                              | Developer |
| 2    | Refresh the app                                                                           | Paywall appears: "Your free trial has expired" | App       |
| 3    | Verify plan cards show                                                                    | Two plans with prices from Stripe              | App       |

#### C. Subscribe from Paywall

| Step | Action | Expected Result | Verify In |
|------|--------|----------------|-----------|
| 1 | Click "Subscribe" on Starter | Redirected to Stripe Checkout | Stripe |
| 2 | Enter test card `4242 4242 4242 4242` | Payment succeeds | Stripe |
| 3 | Redirected back to app | App loads (no paywall) | App |
| 4 | Go to `/subscription` | Plan=Starter, Status=Active, payment method shown, billing period shown | App |
| 5 | Check Stripe Dashboard → Subscriptions | New subscription visible | Stripe |

#### D. Upgrade

| Step | Action | Expected Result | Verify In |
|------|--------|----------------|-----------|
| 1 | On `/subscription`, find "Change Plan" section | Starter highlighted as current, Professional shows "Upgrade" button | App |
| 2 | Click "Upgrade" | Success message, plan changes to Professional immediately | App |
| 3 | Check usage limits | maxSeats and maxTeams increased | App |
| 4 | Check Stripe Dashboard → Subscriptions | Price changed, prorated invoice created | Stripe |

#### E. Downgrade

| Step | Action | Expected Result | Verify In |
|------|--------|----------------|-----------|
| 1 | On `/subscription`, click "Downgrade" to Starter | If under limits: success, "Downgrade scheduled" message | App |
| 2 | If over limits | Error message: "Cannot downgrade: you have X users but Starter allows only Y" | App |
| 3 | Check pending downgrade banner | "Your plan will change to Starter at the end of the current billing period" | App |
| 4 | Check Stripe Dashboard | Subscription shows scheduled downgrade | Stripe |

#### F. Cancel Subscription

| Step | Action | Expected Result | Verify In |
|------|--------|----------------|-----------|
| 1 | Click "Cancel subscription" | Confirmation dialog appears | App |
| 2 | Confirm cancellation | Warning banner: "Your subscription is set to cancel on [date]" | App |
| 3 | Check Stripe Dashboard | `cancel_at_period_end: true` | Stripe |
| 4 | Click "Resume" | Cancellation undone, back to normal | App |
| 5 | Check Stripe Dashboard | `cancel_at_period_end: false` | Stripe |

#### G. Update Payment Method

| Step | Action | Expected Result | Verify In |
|------|--------|----------------|-----------|
| 1 | Click edit icon next to payment method | Redirected to Stripe setup checkout | Stripe |
| 2 | Enter new test card | Card accepted | Stripe |
| 3 | Redirected back to `/subscription` | New card's last 4 digits shown | App |
| 4 | Check Stripe Dashboard → Customer | New payment method attached, set as default | Stripe |

#### H. Billing History

| Step | Action | Expected Result | Verify In |
|------|--------|----------------|-----------|
| 1 | Subscribe and/or upgrade (create at least 1 invoice) | — | — |
| 2 | Go to `/subscription` | Billing History table shows with at least 1 row | App |
| 3 | Click invoice link icon | Opens Stripe-hosted invoice in new tab | Browser |
| 4 | Verify amount and status match | Paid invoices green, amounts correct | App + Stripe |

#### I. Seat Limit Enforcement

| Step | Action | Expected Result | Verify In |
|------|--------|----------------|-----------|
| 1 | Subscribe to Starter (10 seat limit) | — | — |
| 2 | Create users until at 10 | Users created successfully | App |
| 3 | Try to create 11th user | Warning banner: "Seat limit reached (10/10)" + submit disabled | App |
| 4 | Delete a user | — | App |
| 5 | Try to create user again | Succeeds (now 10/10 again) | App |

#### J. Team Limit Enforcement

| Step | Action | Expected Result | Verify In |
|------|--------|----------------|-----------|
| 1 | Subscribe to Starter (3 team limit) | — | — |
| 2 | Edit organization, add teams until 3 | Teams added | App |
| 3 | Try to add 4th team | Warning: "Team limit reached (3/3)" + add row blocked | App |
| 4 | Delete a team | — | App |
| 5 | Add team again | Succeeds | App |

#### K. B2B User (Manual Billing)

| Step | Action | Expected Result | Verify In |
|------|--------|----------------|-----------|
| 1 | Log in as existing B2B user | App loads normally (no paywall, no trial banner) | App |
| 2 | Go to `/subscription` | Shows Plan=Enterprise, Status=Active, Billing=Manual | App |
| 3 | No billing period, no payment method, no invoices, no change plan | Only plan info and usage visible | App |
| 4 | Create users freely | No seat limit (9999) | App |

---

## Webhook Events We Handle

| Stripe Event | What We Do |
|-------------|-----------|
| `checkout.session.completed` (subscription mode) | Create/update subscription: status=active, set Stripe IDs, set limits from price metadata |
| `checkout.session.completed` (setup mode) | Update subscription's default payment method to the new card |
| `customer.subscription.updated` | Sync status (active/past_due/canceled), sync limits from price metadata, clear pendingPlan |
| `customer.subscription.deleted` | Set status=canceled, clear pendingPlan |

**To verify webhooks in Stripe Dashboard:** Go to Developers → Webhooks → look at recent events and their delivery status.
