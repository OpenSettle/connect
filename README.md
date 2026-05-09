# OpenSettle — Integration Guide

Accept on-chain stablecoin payments on Base, Ethereum, Polygon, Arbitrum, Solana, and Tron.  
Non-custodial: funds land directly in your settlement wallet — OpenSettle never holds them.

[![npm](https://img.shields.io/npm/v/@opensettle/sdk?label=%40opensettle%2Fsdk&color=black)](https://www.npmjs.com/package/@opensettle/sdk)
[![API status](https://img.shields.io/badge/API%20status-live-brightgreen)](https://api.opensettle.io/v1/readyz)

---

## How it works

```
Your server          OpenSettle                  Buyer's wallet
─────────────        ──────────────────────────   ──────────────
create customer  →
create invoice   →
create checkout  →   hosted checkout page    ←   buyer lands here
                     on-chain confirmation   ←   buyer sends payment
                     payment.confirmed       →   webhook → fulfil order
redirect buyer   ←   successUrl redirect
```

1. Your backend creates a **customer + invoice + checkout session** via the SDK.
2. You redirect the buyer to the OpenSettle-hosted checkout page.
3. The buyer sends an on-chain payment (USDC or USDT).
4. OpenSettle fires a signed `payment.confirmed` webhook to your backend.
5. Your webhook handler fulfils the order. The redirect is a convenience — **the webhook is the source of truth**.

---

## Before you start

- [ ] OpenSettle account at [opensettle.io](https://opensettle.io)
- [ ] A verified settlement wallet on the chain you want to bill in (Base recommended for first-time integrators)
- [ ] Node.js ≥ 18 on your server
- [ ] A public HTTPS endpoint for receiving webhooks

---

## Dashboard setup

### 1. Create a workspace

Sign in at [opensettle.io/login](https://opensettle.io/login) → **Org switcher → Create workspace**.

The workspace **slug** you choose becomes your customer portal URL:  
`https://opensettle.io/portal/<slug>`

> **Enroll a passkey first.** Go to **Settings → Security** and add a passkey before touching wallets or API keys. Several sensitive actions require either a passkey session or a login within the last 5 minutes — you'll hit this gate repeatedly without one.

---

### 2. Connect a settlement wallet

**Wallets → Connect wallet** → pick your chain.

For EVM chains, the wallet picker offers four routes: **WalletConnect · Binance · OKX · Coinbase** — all via WalletConnect v2 (AppKit). Sign the EIP-191 ownership challenge.  
Solana wallets sign an Ed25519 challenge; Tron wallets sign a TIP-191 challenge.  
Mark one wallet as **Default** per chain.

Recommended chain order (lowest to highest gas): **Solana → Tron → Base → Arbitrum → Polygon → Ethereum**

| Chain | Token | Notes |
|-------|-------|-------|
| Solana | USDC | Sub-second finality, near-zero fee |
| Tron | USDT | TRC-20 — popular for cross-border USDT |
| Base | USDC, USDT | Lowest-gas EVM, fast finality |
| Arbitrum | USDC, USDT | L2, fast — good alongside Base |
| Polygon | USDC, USDT | High throughput, low fee |
| Ethereum | USDC, USDT | Add once volume justifies gas cost |

---

### 3. Get your workspace ID

**Settings → General** — your workspace ID is displayed at the top of the page.  
It looks like `ws_01jt…` and never changes.

Set it as `OPENSETTLE_WORKSPACE_ID` in your server environment. Every API call is scoped to a workspace — the SDK won't work without it.

---

### 4. Create an API key

**Developers → API keys → Create key** (requires re-authentication or passkey session).

- Give the key a name (e.g. `prod-server`).
- Choose **Restricted** permissions for standard integrations.
- The secret is shown **once** — copy it to a password manager immediately.
- Set it as `OPENSETTLE_API_KEY` in your server environment.

---

### 5. Create subscription plans

**Subscriptions → Plans → Create plan** — give it a name, price (USD), and billing interval (monthly or yearly).

After saving, the plan detail page shows the **Price ID** (`price_01…`). Copy it — you'll reference it when creating subscription checkouts.

> Chain and token are **not** set on the plan. They are specified per-checkout by your backend (see below). A single plan can be paid in USDC on Base or USDT on Polygon — it's your choice at checkout time.

---

### 6. Register a webhook endpoint

**Webhooks → Add endpoint**.

- **URL:** `https://your-domain.com/webhooks/opensettle` (must be HTTPS, publicly reachable; localhost / RFC1918 / cloud-metadata IPs are blocked by the SSRF guard).
- **Events:**
  - One-off payments: subscribe to `payment.confirmed`, `payment.failed`, `payment.refunded`.
  - Subscriptions: subscribe to `subscription.created` (activation), plus `subscription.renewed`, `subscription.past_due`, `subscription.canceled`.
- Hit **Send test event** after saving — you should see a 200 from your endpoint in the deliveries list before you take any real money.
- The **signing secret** is shown once after creation — copy it immediately.
- Set it as `OPENSETTLE_WEBHOOK_SECRET` in your server environment.

If you lose the signing secret, rotate it from the endpoint detail page (a grace window keeps the old secret valid while you redeploy).

---

## Backend integration

### Install

```bash
npm install @opensettle/sdk
# pnpm add @opensettle/sdk
# yarn add @opensettle/sdk
```

### Initialize the client

```ts
import { OpenSettle } from "@opensettle/sdk";

const os = new OpenSettle({
  apiKey: process.env.OPENSETTLE_API_KEY!,          // sk_live_… or sk_test_…
  workspaceId: process.env.OPENSETTLE_WORKSPACE_ID!, // ws_01… from Settings → General
  baseUrl: "https://api.opensettle.io",
});
```

> **Both `apiKey` and `workspaceId` are required.** The SDK scopes every request to your workspace — omitting `workspaceId` causes 404s on all API calls.

---

### One-off payment

The flow is always: **customer → invoice → checkout → redirect**.

```ts
// 1. Create (or retrieve) the customer
const customer = await os.customers.create({
  email: "buyer@example.com",
  name: "Jane Doe",        // optional
});

// 2. Create an invoice — this is where you specify the amount and chain
const invoice = await os.invoices.create({
  customerId: customer.id,
  chain: "base",           // "base" | "ethereum" | "polygon" | "arbitrum" | "solana" | "tron"
  token: "USDC",           // "USDC" | "USDT"
  lineItems: [
    {
      description: "Pro plan – 1 month",
      quantity: 1,
      unitAmountMinor: 10_000000, // 10 USDC — always integers, 6 decimal places
    },
  ],
  dueInDays: 0,            // due immediately
  metadata: { orderId: "ord_123" },
});

// 3. Create the hosted checkout session
const checkout = await os.checkouts.create({
  mode: "payment",
  customerId: customer.id,
  invoiceId: invoice.id,
  successUrl: `https://your-domain.com/payments/success?invoice=${invoice.id}`,
  cancelUrl:  "https://your-domain.com/payments/cancel",
});

// 4. Redirect the buyer to the OpenSettle-hosted checkout page
return Response.redirect(`https://opensettle.io${checkout.hostedUrl}`, 303);
```

> **Amounts are always integers.** USDC and USDT both use 6 decimal places.  
> 1 USDC = `1_000000` · 10 USDC = `10_000000` · 0.50 USDC = `500000`

---

### Subscription (SDK)

Reference a `priceId` from your plans instead of creating an invoice.

```ts
// Prices are created in the dashboard: Subscriptions → Plans → Create plan
const checkout = await os.checkouts.create({
  mode: "subscription",
  customerId: customer.id,
  priceId: "price_xxxxxxxxxx",    // from Dashboard → Plans
  chain: "base",                  // required for subscription mode
  token: "USDC",                  // required for subscription mode
  successUrl: "https://your-domain.com/subscriptions/activated",
  cancelUrl:  "https://your-domain.com/subscriptions/cancel",
  metadata: { userId: "usr_123", planId: "pro" }, // propagated to subscription + webhook events
});

return Response.redirect(`https://opensettle.io${checkout.hostedUrl}`, 303);
```

> **`chain` and `token` are required for `mode: "subscription"`** — unlike one-off payments where they are set on the invoice, subscription checkouts need them explicitly.

> **Pass `metadata`** with any identifiers you need to recover in webhook handlers (e.g. your internal `userId`, `planId`). This metadata is copied to the subscription record and included in all subscription lifecycle webhook events.

Link customers to their self-service portal at `https://opensettle.io/portal/<your-slug>` to manage subscriptions and view invoices.

---

### Subscription (no SDK — Vercel Edge / Cloudflare Workers)

If you are running in an edge runtime that can't use Node.js packages, call the REST API directly.

```ts
// POST /api/create-checkout — Vercel Edge Function example
export const config = { runtime: "edge" };

const OS_API_KEY   = process.env.OPENSETTLE_API_KEY!;
const OS_BASE_URL  = process.env.OPENSETTLE_BASE_URL ?? "https://api.opensettle.io";
const OS_WORKSPACE = process.env.OPENSETTLE_WORKSPACE_ID!;

export default async function handler(req: Request) {
  const { priceId, customerEmail, userId, planId, attemptId } = await req.json();

  // Idempotency-Key must be STABLE across retries of the same checkout intent.
  // Generate `attemptId` on the client (e.g. crypto.randomUUID()) once per click
  // and pass it through. Do NOT bake `Date.now()` into the key — it changes on
  // every retry and defeats the point.
  const idempotencyKey = `checkout-${userId}-${planId}-${attemptId}`;

  const res = await fetch(
    `${OS_BASE_URL}/v1/workspaces/${OS_WORKSPACE}/checkouts`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${OS_API_KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,  // required
      },
      body: JSON.stringify({
        mode: "subscription",
        priceId,
        customerEmail,            // OpenSettle finds or creates the customer by email
        chain: "base",            // required for subscription mode
        token: "USDC",            // required for subscription mode
        successUrl: "https://your-domain.com/billing?checkout=success",
        cancelUrl:  "https://your-domain.com/billing?checkout=cancel",
        metadata: { userId, planId }, // recovered in webhook events
      }),
    },
  );

  if (!res.ok) {
    const err = await res.json();
    return Response.json({ error: "Failed to create checkout", detail: err }, { status: 502 });
  }

  const data = await res.json();
  // Response shape: { checkout: { hostedUrl: "/checkout/<token>", ... } }
  const hostedPath: string = data.checkout?.hostedUrl ?? "";
  if (!hostedPath) {
    return Response.json({ error: "Missing hostedUrl in response" }, { status: 502 });
  }

  // Return the full URL — prepend the OpenSettle host to the path
  return Response.json({ url: `https://opensettle.io${hostedPath}` });
}
```

**Key differences from the SDK flow:**

| Detail | Value |
|--------|-------|
| `Idempotency-Key` header | Required on every mutating request |
| `customerEmail` | Pass instead of `customerId` — OpenSettle finds or creates the customer |
| `chain` + `token` | Always required for `mode: "subscription"` |
| Response field | `data.checkout.hostedUrl` (a path — prepend `https://opensettle.io`) |

---

## Webhooks

### Verify every request

OpenSettle signs every webhook with HMAC-SHA256. The signature arrives in the `x-opensettle-signature` header with the format `t=<unix-seconds>,v1=<hex-digest>`.

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

function verifyOpenSettleWebhook(
  rawBody: string,        // the unparsed request body string
  sigHeader: string,      // value of the x-opensettle-signature header
  secret: string,         // your OPENSETTLE_WEBHOOK_SECRET
): void {
  const [tsPart = "", v1Part = ""] = sigHeader.split(",");
  const ts  = tsPart.split("=")[1] ?? "";
  const sig = v1Part.split("=")[1] ?? "";

  const expected = createHmac("sha256", secret)
    .update(`${ts}.${rawBody}`)
    .digest("hex");

  // timingSafeEqual throws if the two buffers differ in length — pre-check.
  // SHA-256 hex is always 64 chars; anything else is malformed up front.
  if (sig.length !== expected.length) {
    throw new Error("signature mismatch");
  }
  if (!timingSafeEqual(Buffer.from(sig, "hex"), Buffer.from(expected, "hex"))) {
    throw new Error("signature mismatch");
  }

  // Reject webhooks older than 5 minutes
  if (!ts || Math.abs(Date.now() / 1000 - Number(ts)) > 300) {
    throw new Error("webhook is stale");
  }
}
```

### Handle events

```ts
// Express example — use express.raw() so rawBody is the unmodified buffer
app.post(
  "/webhooks/opensettle",
  express.raw({ type: "application/json" }),
  async (req, res) => {
    const sig = req.headers["x-opensettle-signature"] as string ?? "";

    try {
      verifyOpenSettleWebhook(
        req.body.toString(),
        sig,
        process.env.OPENSETTLE_WEBHOOK_SECRET!,
      );
    } catch {
      return res.status(400).send("bad signature");
    }

    const event = JSON.parse(req.body.toString()) as { type: string; data: unknown };

    switch (event.type) {
      case "payment.confirmed":
        await fulfillOrder(event.data);   // grant access, send receipt
        break;
      case "payment.refunded":
        await revokeAccess(event.data);
        break;
      case "payment.failed":
        // notify user, offer retry
        break;
    }

    res.status(200).send("ok");
  },
);
```

**Three rules for webhook handlers:**

1. **Respond within 15 seconds.** OpenSettle retries up to 10 times with exponential backoff (1 min → 5 min → 15 min → 1 hr → 6 hrs). After 20 consecutive failures the endpoint is disabled automatically.
2. **Make handlers idempotent.** The same event may be delivered more than once — a duplicate `payment.confirmed` must not double-fulfil an order.
3. **Never trust the redirect alone.** The buyer could close their browser before the redirect fires. Always fulfil from the webhook.

---

### Webhook event reference

| Event | When to act |
|-------|-------------|
| `payment.confirmed` | On-chain confirmation received — fulfil the order |
| `payment.failed` | Payment attempt failed — notify buyer |
| `payment.refunded` | Refund confirmed on-chain — revoke / adjust access |
| `subscription.created` | New subscription started — activate access |
| `subscription.trial_ended` | Trial finished and the first paid period has begun — billing has started |
| `subscription.renewed` | Recurring payment confirmed — extend access period |
| `subscription.past_due` | Renewal payment failed — subscriber will retry |
| `subscription.canceled` | Subscription cancelled — revoke access |
| `invoice.paid` | Invoice paid |
| `invoice.past_due` | Invoice overdue |

### Subscription event payload shapes

Subscription events come in two shapes depending on the trigger:

- **Full object** — `event.data.subscription` is the complete record.
  - Fired by `subscription.created` and by `subscription.canceled` when an admin/customer cancels immediately.
- **Reference** — `event.data` carries `subscriptionId` plus a few flat fields.
  - Fired by `subscription.trial_ended`, `subscription.renewed`, `subscription.past_due`, and `subscription.canceled` when the cancellation is system-driven (period-end or dunning).

Read defensively from both locations — the field may be on `data.subscription.metadata` or directly on `data.metadata`:

```ts
type SubEvent = {
  type: string;
  data: {
    // full-object form
    subscription?: {
      id: string;
      priceId: string;
      currentPeriodEnd: string;
      status: string;
      metadata?: Record<string, string>;
    };
    // reference form
    subscriptionId?: string;
    nextBillingDate?: string;     // subscription.renewed
    reason?: string;               // subscription.canceled — "period_end" | "dunning_exhausted" | …
    metadata?: Record<string, string>;
  };
};

function readSubscriptionRef(event: SubEvent) {
  const sub = event.data.subscription;
  const subscriptionId = sub?.id ?? event.data.subscriptionId ?? null;
  const metadata = sub?.metadata ?? event.data.metadata ?? {};
  return { subscriptionId, metadata };
}
```

> **Why metadata propagates:** When you pass `metadata` to the checkout, OpenSettle copies it to the subscription record. Most lifecycle events include that metadata so you can recover `userId`, `planId`, etc. without a database lookup. **Caveat:** the dunning-exhausted `subscription.canceled` is dispatched without metadata — if you must react to it, key off `subscriptionId` against your own table rather than relying on metadata being present.

---

## Go-live checklist

- [ ] `https://api.opensettle.io/v1/readyz` returns `200` with a body where `ok === true` and `db === "ok"` (the response includes additional diagnostic fields — match those two keys, not the whole body)
- [ ] At least one **verified default wallet** per chain you advertise
- [ ] Webhook endpoint registered with `payment.confirmed` subscribed
- [ ] `OPENSETTLE_API_KEY`, `OPENSETTLE_WORKSPACE_ID`, and `OPENSETTLE_WEBHOOK_SECRET` set in prod env — never committed to git
- [ ] Webhook handler correctly **rejects** a deliberately bad signature (returns 400)
- [ ] Webhook handler correctly **accepts** `payment.confirmed` and is **idempotent** (duplicate delivery does not double-fulfil)
- [ ] Test transaction on **mainnet** ≤ $1 on each chain you advertise — Base first
- [ ] `payment.refunded` path tested end to end
- [ ] Your status page links to `https://api.opensettle.io/v1/readyz` so your oncall can rule OpenSettle in or out

---

## Content-Security-Policy

If your site has a CSP, add these directives:

```
connect-src https://api.opensettle.io;
frame-src https://opensettle.io;   /* only if embedding checkout in an iframe */
```

Top-level redirect (`303`) is preferred over iframe embedding.

---

## Support

- Dashboard: [opensettle.io](https://opensettle.io)
- API status: [api.opensettle.io/v1/readyz](https://api.opensettle.io/v1/readyz)
- Issues: [github.com/OpenSettle/connect/issues](https://github.com/OpenSettle/connect/issues)
