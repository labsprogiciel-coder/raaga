# Payments via Paygate

Plan for taking payments in Raaga through [Paygate](https://github.com/labsprogiciel-coder/paygate), the portfolio's shared payment gateway. **PI-SPI bank payments first, mobile money later.** Nothing here is built yet.

Raaga is the seller of record, so it only **collects**: no payouts, no escrow. Paygate talks to the providers, keeps the ledger and tells Raaga the outcome with a signed webhook. Raaga decides what "paid" means for a cart, a quote or an order.

```
 storefront ──► Medusa backend ──POST /v1/collections──► Paygate ──► PI-SPI (later: pawaPay)
                     ▲                                      │
                     └──── signed payment.* webhook ────────┘
```

## Today

- The only payment provider is Medusa's manual `pp_system_default`; payment is arranged by hand.
- Accepting a quote (`customer-accept-quote`) turns its draft order into an order with no payment step.
- Markets: XOF (WAEMU countries) and USD (rest of world). **Paygate is XOF only**, so the USD region keeps manual payment.

## What Paygate gives us (and its rules)

From Paygate's [`docs/api.md`](https://github.com/labsprogiciel-coder/paygate/blob/main/docs/api.md):

| | |
|---|---|
| Auth | `Authorization: Bearer pgk_<id>_<secret>`, one key for the Raaga tenant |
| Create | `POST /v1/collections` with `reference`, `amount`, `rail`, `metadata`; creates **and** starts the payment |
| Read | `GET /v1/collections/{id or reference}` (also nudges the provider, at most every 4 s) |
| Amounts | Whole XOF, minimum 100, maximum the tenant's `maxAmount` (gateway default **2,000,000 XOF**) |
| Idempotency | Same `reference` + same details → same payment; different details → `409` |
| Webhooks | `payment.completed` / `failed` / `expired` / `mismatch`; at least once, dedupe on event `id`; HMAC signature in `Paygate-Signature` |
| Late payments | An expired or failed collection can still complete; it arrives as `payment.completed` with `late: true` |

PI-SPI specifics:

- **`pispi_qr`**: Paygate returns `qr` (`alias`, `countryCode`, `qrType: "DYNAMIC"`, `referenceLabel`, `amount`). We render it as a PI-SPI dynamic QR; the buyer scans it in their bank app. Default window **3 h**.
- **`pispi_rtp`**: Paygate sends a payment request to the buyer's PI-SPI alias (a UUID); they approve it in their bank app. At most **3 requests per buyer per day**. Default window 1 h.
- Settlement is PI-SPI's own record of received payments: **exactly** the amount, the right category, on our alias. Anything else is `payment.mismatch` and nothing settles.
- With `PISPI_POLL` off, nothing settles automatically; an operator confirms in the Paygate console.
- Paygate's receiving alias is for one country (`PISPI_COUNTRY`, currently `CI`).
- Paygate's sandbox mode simulates mobile money only. PI-SPI is tested against the participant's test environment, or with polling off and operator confirmation.

## Design

### 1. A `paygate` payment provider module

`apps/backend/src/modules/paygate/`: a Medusa payment provider (`AbstractPaymentProvider`), registered under the Payment module in `medusa-config.ts`:

```ts
[Modules.PAYMENT]: {
  resolve: "@medusajs/medusa/payment",
  options: {
    providers: [
      { resolve: "./src/modules/paygate", id: "pispi-qr", options: { rail: "pispi_qr", ...paygateEnv } },
      // later: { id: "pispi-rtp", options: { rail: "pispi_rtp" } }, { id: "mobile-money", options: { rail: "mobile_money" } }
    ],
  },
},
```

Each id becomes a provider the storefront can offer (`pp_paygate_pispi-qr`), enabled on the XOF region only.

| Provider method | Does |
|---|---|
| `initiatePayment` | `POST /v1/collections` with `reference` = the Medusa payment session id, `amount` rounded to a whole XOF, `rail`, `metadata: { cart_id, quote_id? }`. Stores the Paygate payment (id, `qr`, `expiresAt`) in the session `data`. |
| `getPaymentStatus` / `authorizePayment` | `GET /v1/collections/{id}`: `completed` → captured, `processing` → pending, `failed`/`expired` → error |
| `updatePayment` | The amount changed: start a new collection with a new reference (a reference cannot change its amount) |
| `cancelPayment` / `deletePayment` | Nothing at Paygate (collections expire on their own); clear the session |
| `refundPayment` | Not supported by Paygate; refunds stay manual (bank transfer) and are recorded in Medusa |
| `getWebhookActionAndData` | Verify `Paygate-Signature` on the raw body (port Paygate's `src/signing.ts`), reject if `t` is older than 5 min; map `payment.completed` → `captured`, `failed`/`expired` → `failed`, others → `not_supported`; return the session id from `data.reference` |

**Webhook URL.** A Paygate tenant has one webhook URL. Point it at Medusa's built-in payment hook for the first provider (`/hooks/payment/paygate_pispi-qr`). Medusa finds the session by id, and every Paygate provider shares the same verification code, so mobile money later uses the same endpoint.

**Config** (`apps/backend/.env`): `PAYGATE_URL`, `PAYGATE_API_KEY`, `PAYGATE_WEBHOOK_SECRET`.

### 2. Checkout (cart → order)

1. Buyer picks "Bank payment (PI-SPI)" in the payment step (XOF region only).
2. Backend creates the session → Paygate collection; the storefront renders the QR from the session `data`, with the amount, the expiry time, and steps in French and English.
3. The storefront polls the cart's payment session status (or the order lookup) while the QR is shown.
4. Paygate sees the bank payment and sends `payment.completed`; Medusa marks the payment captured and completes the cart into an order.
5. If the window passes, the buyer starts again (a new session, a new QR).

Storefront work: a `pp_paygate_*` entry in `src/lib/constants.tsx` (`paymentInfoMap`), a QR component in `modules/checkout/components/payment`, a "waiting for your bank payment" state on the review step, and copy in both languages.

### 3. Accepted quotes

Quotes become orders without going through the cart, so they need their own path:

- After `customer-accept-quote` converts the draft order, create a payment collection for the order and a Paygate collection for its total (`reference` = the payment collection's id).
- The quote page shows the QR (or the RTP prompt) to pay the accepted price.
- `payment.completed` marks the order's payment captured; fulfilment starts only then.

This is also where negotiated prices (see the RFQ ideas) get paid, so it is worth building right after checkout.

### 4. Edge cases to handle

| Case | Handling |
|---|---|
| Duplicate webhook | Dedupe on the Paygate event `id` (Medusa's webhook processing is idempotent on the session, but log the event id) |
| `late: true` after the cart expired | If the cart can still complete, complete it. Otherwise alert Raaga (Slack/email) to create the order by hand or refund by bank transfer |
| `payment.mismatch` | Nothing settled. Alert Raaga; the operator resolves it in the Paygate console |
| Order above `maxAmount` | Hide PI-SPI for that cart and fall back to manual payment, until the Raaga tenant's `maxAmount` is raised |
| Amount with decimals | XOF has no minor unit; round to a whole number before sending, and keep Medusa's totals whole in XOF |
| USD region | Manual payment only |

## Phases

| Phase | Scope |
|---|---|
| **1. PI-SPI QR at checkout** | Provider module, webhook, QR in the storefront, XOF region only. Paygate tenant created. |
| **2. PI-SPI for accepted quotes** | Payment collection after quote acceptance, QR on the quote page, fulfilment waits for payment. |
| **3. PI-SPI payment requests (RTP)** | Store the buyer's PI-SPI alias on the company (next to the KYB fields); "Send to my bank app" instead of scanning. Respect the 3-per-day limit. |
| **4. Mobile money (pawaPay)** | `mobile-money` provider: operator + phone at checkout, `GET /v1/methods` to show only what works for that number and amount, Orange Burkina Faso one-time code (`pre_auth_code_required`), Wave redirect (`successUrl` / `failedUrl`), approval instructions while `processing`. Likely for smaller orders only, given operator limits. |

## Setting up the Raaga tenant in Paygate

```bash
A='Authorization: Bearer <ADMIN_API_KEY>'
curl -X POST $GW/admin/tenants -H "$A" -H 'content-type: application/json' \
  -d '{"id":"raaga","name":"Raaga","idNamespace":"raaga","webhookUrl":"https://<backend>/hooks/payment/paygate_pispi-qr","maxAmount":<agreed limit>}'
curl -X POST $GW/admin/tenants/raaga/keys -H "$A" -d '{"label":"prod"}'   # → PAYGATE_API_KEY (shown once)
curl $GW/admin/tenants/raaga/webhook-secret -H "$A"                      # → PAYGATE_WEBHOOK_SECRET
```

## Open questions

1. **Amount limit.** Parts orders often exceed 2,000,000 XOF. What `maxAmount` should Raaga's tenant have, and what are the PI-SPI participant's own per-payment limits?
2. **Countries.** Paygate's PI-SPI alias is for Côte d'Ivoire. Can buyers in other WAEMU countries pay it (PI-SPI is meant to be regional), or does each country need its own alias?
3. **Polling.** Will `PISPI_POLL` be on in production, or does an operator confirm each bank payment? That decides whether checkout is instant or waits for a person.
4. **Payment request label.** Paygate sends RTPs with the label `Payment`. A label like "Raaga order #…" would help buyers recognise it; that is a small change in Paygate.
5. **Refunds.** Paygate does not refund. Confirm that refunds by bank transfer, recorded by hand in Medusa, are acceptable.
