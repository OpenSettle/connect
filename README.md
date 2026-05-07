# OpenSettle — Integration Guide

Accept on-chain stablecoin payments on Base, Ethereum, Polygon, and Arbitrum.  
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
- [ ] A verified EVM settlement wallet (Base recommended for v1)
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

**Settings → Wallets → Connect wallet** → pick your chain.

The wallet picker offers four options: **WalletConnect · Binance · OKX · Coinbase** — all routed through WalletConnect v2 (AppKit). Sign the EIP-191 ownership challenge. Mark one wallet as **Default** per chain.

Recommended chain order (lowest to highest cost): **Base → Arbitrum → Polygon → Ethereum**

| Chain | Token | Notes |
|-------|-------|-------|
| Base | USDC, USDT | Start here — lowest gas, fastest finality |
| Arbitrum | USDC, USDT | L2, fast — good alongside Base |
| Polygon | USDC, USDT | High throughput, low fee |
| Ethereum | USDC, USDT | Add once volume justifies gas cost |

> Solana and Tron are not yet available.

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

### 5. Register a webhook endpoint

**Developers → Webhooks → Add endpoint**.

- **URL:** `https://your-domain.com/webhooks/opensettle` (must be HTTPS, publicly reachable)
- **Events:** subscribe to at minimum `payment.confirmed`, `payment.failed`, `payment.refunded`
- The **signing secret** is shown once after creation — copy it immediately.
- Set it as `OPENSETTLE_WEBHOOK_SECRET` in your server environment.

If you lose the signing secret, delete the endpoint and create a new one.

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
  apiKey: process.env.OPENSETTLE_API_KEY!,         // sk_live_… or sk_test_…
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
  chain: "base",           // "base" | "ethereum" | "polygon" | "arbitrum"
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

### Subscription

Reference a `priceId` from your product catalogue instead of creating an invoice.

```ts
// Prices are created in the dashboard: Products → New product → Add price
const checkout = await os.checkouts.create({
  mode: "subscription",
  customerId: customer.id,
  priceId: "price_xxxxxxxxxx",    // from Dashboard → Products
  successUrl: "https://your-domain.com/subscriptions/activated",
  cancelUrl:  "https://your-domain.com/subscriptions/cancel",
});

return Response.redirect(`https://opensettle.io${checkout.hostedUrl}`, 303);
```

Link customers to their self-service portal at `https://opensettle.io/portal/<your-slug>` to manage subscriptions and view invoices.

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

  // Use timingSafeEqual to prevent timing attacks
  if (!timingSafeEqual(Buffer.from(sig), Buffer.from(expected))) {
    throw new Error("signature mismatch");
  }

  // Reject webhooks older than 5 minutes
  if (Math.abs(Date.now() / 1000 - Number(ts)) > 300) {
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
| `subscription.created` | New subscription started |
| `subscription.renewed` | Recurring payment confirmed |
| `subscription.past_due` | Renewal payment failed — subscriber will retry |
| `subscription.canceled` | Subscription cancelled |
| `invoice.paid` | Invoice paid |
| `invoice.past_due` | Invoice overdue |

---

## Go-live checklist

- [ ] `https://api.opensettle.io/v1/readyz` returns `{ "ok": true, "db": "ok" }`
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
