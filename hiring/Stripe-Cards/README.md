┌────────────────────────────────────────────────────────────────────────┐
│                      PurePulse Client Payment                          │
│               Client Subscription Succeeded ($20 - $100/mo)           │
└──────────────────────────────────┬─────────────────────────────────────┘
│
▼
┌────────────────────────────────────────────────────────────────────────┐
│                       Stripe Webhook Handler                           │
│                      Event: invoice.paid                               │
└──────────────────────────────────┬─────────────────────────────────────┘
│
▼
┌────────────────────────────────────────────────────────────────────────┐
│                      Supabase Backend Engine                           │
│             1. Calculate recurring affiliate share                     │
│             2. Record commission in affiliate_ledger                   │
│             3. Increment spending reserve                              │
└──────────────────────────────────┬─────────────────────────────────────┘
│
▼
┌────────────────────────────────────────────────────────────────────────┐
│                     Stripe Issuing Integration                         │
│             - Dynamic Real-Time Auth Webhook Validation                │
│             - Instant Top-Up / Card Balance Sync                       │
└──────────────────────────────────┬─────────────────────────────────────┘
│
▼
┌────────────────────────────────────────────────────────────────────────┐
│                       Affiliate Cardholder                             │
│       • Apple Pay / Google Wallet Digital Pass                         │
│       • Physical Card POS / ATM / Online Purchases                     │
└────────────────────────────────────────────────────────────────────────┘

---

## 💰 Commission Structure & Economics

Affiliates earn monthly recurring commissions for the lifetime of every active referred website client:

| Plan | Client Subscription | Commission Rate | Monthly Affiliate Payout |
| :--- | :--- | :--- | :--- |
| **Starter** | `$20 / mo` | **10%** | `$2.00 / mo` |
| **Growth** | `$50 / mo` | **40%** | `$20.00 / mo` |
| **Premium** | `$75 / mo` | **45%** | `$33.75 / mo` |
| **Business** | `$100 / mo` | **50%** | **`$50.00 / mo`** |

> 🎁 **Performance Bonus:** Refer at least **1 active client per calendar month** to receive complimentary access to the **VibeCodes Space Business Plan** ($49/mo value) featuring unlimited sites, white-labeling, and team roles.

---

## 🚀 Stripe Issuing API Implementation

### 1. Cardholder Creation (`/v1/issuing/cardholders`)

Create an individual cardholder profile with identity and KYC verification:

```typescript
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-12-18.acacia',
});

export async function createAffiliateCardholder(affiliate: {
  id: string;
  name: string;
  email: string;
  phone: string;
  address: Stripe.AddressParam;
}) {
  return await stripe.issuing.cardholders.create({
    type: 'individual',
    name: affiliate.name,
    email: affiliate.email,
    phone_number: affiliate.phone,
    billing: {
      address: affiliate.address,
    },
    metadata: {
      affiliate_id: affiliate.id,
      platform: 'PurePulse',
    },
  });
}

2. Issue Card (/v1/issuing/cards)
Generate an instant virtual card or dispatch a physical card order:
export async function issueAffiliateCard({
  cardholderId,
  isPhysical = false,
  shippingAddress,
}: {
  cardholderId: string;
  isPhysical?: boolean;
  shippingAddress?: Stripe.Issuing.CardCreateParams.Shipping;
}) {
  const cardParams: Stripe.Issuing.CardCreateParams = {
    cardholder: cardholderId,
    currency: 'usd',
    type: isPhysical ? 'physical' : 'virtual',
    status: 'active',
    spending_controls: {
      spending_limits: [
        {
          amount: 100000, // $1,000 max monthly spend
          interval: 'monthly',
        },
      ],
    },
    metadata: {
      program: 'PurePulse Affiliate Network',
    },
  };

  if (isPhysical && shippingAddress) {
    cardParams.shipping = shippingAddress;
  }

  return await stripe.issuing.cards.create(cardParams);
}

3. Real-Time Authorization Webhook (issuing_authorization.request)
Approve or decline transactions in real time based on the affiliate's current wallet balance:
import { NextRequest, NextResponse } from 'next/server';
import { supabaseAdmin } from '@/lib/supabase';
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(req: NextRequest) {
  const payload = await req.text();
  const sig = req.headers.get('stripe-signature')!;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      payload,
      sig,
      process.env.STRIPE_ISSUING_WEBHOOK_SECRET!
    );
  } catch (err: any) {
    return NextResponse.json({ error: err.message }, { status: 400 });
  }

  if (event.type === 'issuing_authorization.request') {
    const auth = event.data.object as Stripe.Issuing.Authorization;
    const affiliateId = auth.card.cardholder.metadata.affiliate_id;
    const requestedAmount = auth.pending_request.amount; // in cents

    // Verify balance in Supabase
    const { data: wallet } = await supabaseAdmin
      .from('affiliate_wallets')
      .select('available_balance_cents')
      .eq('affiliate_id', affiliateId)
      .single();

    const isApproved = wallet && wallet.available_balance_cents >= requestedAmount;

    return NextResponse.json({
      approved: isApproved,
      metadata: {
        settlement_status: isApproved ? 'cleared' : 'insufficient_funds',
      },
    });
  }

  return NextResponse.json({ received: true });
}

📡 API Endpoints Reference
Card Management
 * POST /v1/issuing/cards — Issue a new card (virtual or physical)
 * POST /v1/issuing/cards/:id — Update card status (freeze, activate, cancel)
 * GET /v1/issuing/cards/:id — Retrieve card details & sensitive metadata
 * GET /v1/issuing/cards — List all cards for a specific cardholder
Testmode & Shipping Lifecycle
 * POST /v1/test_helpers/issuing/cards/:id/shipping/submit — Mark card order submitted
 * POST /v1/test_helpers/issuing/cards/:id/shipping/ship — Mark card order shipped
 * POST /v1/test_helpers/issuing/cards/:id/shipping/deliver — Simulate card delivery
 * POST /v1/test_helpers/issuing/cards/:id/shipping/fail — Simulate shipping failure
 * POST /v1/test_helpers/issuing/cards/:id/shipping/return — Simulate package return
Webhook Events
 * issuing_card.created — Fired when a new card is provisioned
 * issuing_card.updated — Fired on card limit, metadata, or status change
 * issuing_authorization.request — Synchronous webhook for real-time spend decisions
 * issuing_transaction.created — Fired when settled transaction posts to ledger
🎨 Card Design & Brand Specs
┌────────────────────────────────────────────────────────────┐
│  ⚡ PurePulse                                         VISA │
│                                                            │
│     ••••  ••••  ••••  4092                                 │
│                                                            │
│  VALID THRU 08/29                                          │
│  MATTHEW HAGEN                                             │
└────────────────────────────────────────────────────────────┘

 * Base Tone: Obsidian Black (#0B0B0F)
 * Accent Highlight: Electric Pulse Violet (#7C3AED)
 * Typography / Text: Crisp White (#F8FAFC)
 * Card Finish: Matte Velvet PVC with UV Gloss Logo Highlight
 * Packaging Inscription: "Design that moves people forward. Commission unlocked."
⚙️ Environment Variables
Create a .env.local file with the required configuration:
# Stripe Keys
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_ISSUING_WEBHOOK_SECRET=whsec_...

# Supabase
NEXT_PUBLIC_SUPABASE_URL=[https://your-project.supabase.co](https://your-project.supabase.co)
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...

# PurePulse Platform
NEXT_PUBLIC_APP_URL=[https://purepulse.one](https://purepulse.one)
AFFILIATE_PORTAL_URL=[https://vibecodes.space](https://vibecodes.space)

📄 License
Distributed under the MIT License. See LICENSE for details.
© 2026 PurePulse Studio & VibeCodes Network. All rights reserved.
"""
with open("README.md", "w") as f:
f.write(readme_content)
print("README.md generated successfully.")
Your Markdown file is ready:
[file-tag: code-generated-file-0-1787180655945750791]

Here is the exact, copy-pasteable Markdown content for your **GitHub `README.md`**:

```markdown
# ⚡ PurePulse Affiliate Card Program

> **Instant payout card infrastructure for the PurePulse & VibeCodes Affiliate Network.** > Powered by **Stripe Issuing**, **Supabase**, and **Next.js**.

[![Stripe Issuing](https://img.shields.io/badge/Stripe-Issuing%20API-635BFF?style=flat-square&logo=stripe&logoColor=white)](https://docs.stripe.com/issuing)
[![Next.js 15](https://img.shields.io/badge/Next.js-15%20App%20Router-black?style=flat-square&logo=next.js)](https://nextjs.org/)
[![Supabase](https://img.shields.io/badge/Supabase-Auth%20%26%20Postgres-3ECF8E?style=flat-square&logo=supabase&logoColor=white)](https://supabase.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](LICENSE)

---

## 📖 Overview

The **PurePulse Affiliate Card Program** turns recurring monthly affiliate commissions into instant liquidity. Modeled after on-demand earnings programs like *Uber Pro Card* and *DoorDash DasherDirect*, affiliates receive their recurring payouts directly to a dedicated virtual or custom physical **PurePulse Visa® Debit Card**.

### 🌟 Key Highlights
- **Zero Payout Latency:** Earn recurring monthly payouts credited instantly to card balances upon successful client invoice settlement.
- **Virtual & Physical Cards:** Issue instant Apple Pay / Google Wallet virtual passes (`$0.10/card`) or deliver custom matte-obsidian physical cards (`$3.50/card`).
- **Real-Time Authorization Engine:** Dynamic transaction approvals verified against live Supabase affiliate balances via webhooks under 2 seconds.
- **Tiered Recurring Earnings:** From 10% to 50% lifetime recurring payouts on website subscriptions.

---

## 🏛 Architecture & Transaction Flow


┌────────────────────────────────────────────────────────────────────────┐
│                      PurePulse Client Payment                          │
│               Client Subscription Succeeded ($20 - $100/mo)           │
└──────────────────────────────────┬─────────────────────────────────────┘
│
▼
┌────────────────────────────────────────────────────────────────────────┐
│                       Stripe Webhook Handler                           │
│                      Event: invoice.paid                               │
└──────────────────────────────────┬─────────────────────────────────────┘
│
▼
┌────────────────────────────────────────────────────────────────────────┐
│                      Supabase Backend Engine                           │
│             1. Calculate recurring affiliate share                     │
│             2. Record commission in affiliate_ledger                   │
│             3. Increment spending reserve                              │
└──────────────────────────────────┬─────────────────────────────────────┘
│
▼
┌────────────────────────────────────────────────────────────────────────┐
│                     Stripe Issuing Integration                         │
│             - Dynamic Real-Time Auth Webhook Validation                │
│             - Instant Top-Up / Card Balance Sync                       │
└──────────────────────────────────┬─────────────────────────────────────┘
│
▼
┌────────────────────────────────────────────────────────────────────────┐
│                       Affiliate Cardholder                             │
│       • Apple Pay / Google Wallet Digital Pass                         │
│       • Physical Card POS / ATM / Online Purchases                     │
└────────────────────────────────────────────────────────────────────────┘

---

## 💰 Commission Structure & Economics

Affiliates earn monthly recurring commissions for the lifetime of every active referred website client:

| Plan | Client Subscription | Commission Rate | Monthly Affiliate Payout |
| :--- | :--- | :--- | :--- |
| **Starter** | `$20 / mo` | **10%** | `$2.00 / mo` |
| **Growth** | `$50 / mo` | **40%** | `$20.00 / mo` |
| **Premium** | `$75 / mo` | **45%** | `$33.75 / mo` |
| **Business** | `$100 / mo` | **50%** | **`$50.00 / mo`** |

> 🎁 **Performance Bonus:** Refer at least **1 active client per calendar month** to receive complimentary access to the **VibeCodes Space Business Plan** ($49/mo value) featuring unlimited sites, white-labeling, and team roles.

---

## 🚀 Stripe Issuing API Implementation

### 1. Cardholder Creation (`/v1/issuing/cardholders`)

Create an individual cardholder profile with identity and KYC verification:

```typescript
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-12-18.acacia',
});

export async function createAffiliateCardholder(affiliate: {
  id: string;
  name: string;
  email: string;
  phone: string;
  address: Stripe.AddressParam;
}) {
  return await stripe.issuing.cardholders.create({
    type: 'individual',
    name: affiliate.name,
    email: affiliate.email,
    phone_number: affiliate.phone,
    billing: {
      address: affiliate.address,
    },
    metadata: {
      affiliate_id: affiliate.id,
      platform: 'PurePulse',
    },
  });
}

2. Issue Card (/v1/issuing/cards)
Generate an instant virtual card or dispatch a physical card order:
export async function issueAffiliateCard({
  cardholderId,
  isPhysical = false,
  shippingAddress,
}: {
  cardholderId: string;
  isPhysical?: boolean;
  shippingAddress?: Stripe.Issuing.CardCreateParams.Shipping;
}) {
  const cardParams: Stripe.Issuing.CardCreateParams = {
    cardholder: cardholderId,
    currency: 'usd',
    type: isPhysical ? 'physical' : 'virtual',
    status: 'active',
    spending_controls: {
      spending_limits: [
        {
          amount: 100000, // $1,000 max monthly spend
          interval: 'monthly',
        },
      ],
    },
    metadata: {
      program: 'PurePulse Affiliate Network',
    },
  };

  if (isPhysical && shippingAddress) {
    cardParams.shipping = shippingAddress;
  }

  return await stripe.issuing.cards.create(cardParams);
}

3. Real-Time Authorization Webhook (issuing_authorization.request)
Approve or decline transactions in real time based on the affiliate's current wallet balance:
import { NextRequest, NextResponse } from 'next/server';
import { supabaseAdmin } from '@/lib/supabase';
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(req: NextRequest) {
  const payload = await req.text();
  const sig = req.headers.get('stripe-signature')!;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      payload,
      sig,
      process.env.STRIPE_ISSUING_WEBHOOK_SECRET!
    );
  } catch (err: any) {
    return NextResponse.json({ error: err.message }, { status: 400 });
  }

  if (event.type === 'issuing_authorization.request') {
    const auth = event.data.object as Stripe.Issuing.Authorization;
    const affiliateId = auth.card.cardholder.metadata.affiliate_id;
    const requestedAmount = auth.pending_request.amount; // in cents

    // Verify balance in Supabase
    const { data: wallet } = await supabaseAdmin
      .from('affiliate_wallets')
      .select('available_balance_cents')
      .eq('affiliate_id', affiliateId)
      .single();

    const isApproved = wallet && wallet.available_balance_cents >= requestedAmount;

    return NextResponse.json({
      approved: isApproved,
      metadata: {
        settlement_status: isApproved ? 'cleared' : 'insufficient_funds',
      },
    });
  }

  return NextResponse.json({ received: true });
}

📡 API Endpoints Reference
Card Management
 * POST /v1/issuing/cards — Issue a new card (virtual or physical)
 * POST /v1/issuing/cards/:id — Update card status (freeze, activate, cancel)
 * GET /v1/issuing/cards/:id — Retrieve card details & sensitive metadata
 * GET /v1/issuing/cards — List all cards for a specific cardholder
Testmode & Shipping Lifecycle
 * POST /v1/test_helpers/issuing/cards/:id/shipping/submit — Mark card order submitted
 * POST /v1/test_helpers/issuing/cards/:id/shipping/ship — Mark card order shipped
 * POST /v1/test_helpers/issuing/cards/:id/shipping/deliver — Simulate card delivery
 * POST /v1/test_helpers/issuing/cards/:id/shipping/fail — Simulate shipping failure
 * POST /v1/test_helpers/issuing/cards/:id/shipping/return — Simulate package return
Webhook Events
 * issuing_card.created — Fired when a new card is provisioned
 * issuing_card.updated — Fired on card limit, metadata, or status change
 * issuing_authorization.request — Synchronous webhook for real-time spend decisions
 * issuing_transaction.created — Fired when settled transaction posts to ledger
🎨 Card Design & Brand Specs
┌────────────────────────────────────────────────────────────┐
│  ⚡ PurePulse                                         VISA │
│                                                            │
│     ••••  ••••  ••••  4092                                 │
│                                                            │
│  VALID THRU 08/29                                          │
│  MATTHEW HAGEN                                             │
└────────────────────────────────────────────────────────────┘

 * Base Tone: Obsidian Black (#0B0B0F)
 * Accent Highlight: Electric Pulse Violet (#7C3AED)
 * Typography / Text: Crisp White (#F8FAFC)
 * Card Finish: Matte Velvet PVC with UV Gloss Logo Highlight
 * Packaging Inscription: "Design that moves people forward. Commission unlocked."
⚙️ Environment Variables
Create a .env.local file with the required configuration:
# Stripe Keys
STRIPE_SECRET_KEY=sk_live_...
STRIPE_PUBLISHABLE_KEY=pk_live_...
STRIPE_ISSUING_WEBHOOK_SECRET=whsec_...

# Supabase
NEXT_PUBLIC_SUPABASE_URL=[https://your-project.supabase.co](https://your-project.supabase.co)
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...

# PurePulse Platform
NEXT_PUBLIC_APP_URL=[https://purepulse.one](https://purepulse.one)
AFFILIATE_PORTAL_URL=[https://vibecodes.space](https://vibecodes.space)

📄 License
Distributed under the MIT License. See LICENSE for details.
© 2026 PurePulse Studio & VibeCodes Network. All rights reserved.

