# Payments via Paygate

Plan for taking payments in Raaga through [Paygate](https://github.com/labsprogiciel-coder/paygate), the portfolio's shared payment gateway. **PI-SPI bank payments first, mobile money later.** Nothing here is built yet.

Raaga is the seller of record, so it only **collects**: no payouts, no escrow. Paygate talks to the providers, keeps the ledger and tells Raaga the outcome with a signed webhook. Raaga decides what "paid" means for a cart, a quote or an order.

```
 storefront ──► Medusa backend ──POST /v1/collections──► Paygate ──► PI-SPI (later: pawaPay)
                     ▲                                      │
                     └──── signed payment.* webhook ────────┘
```

## Decisions

- **Limits:** Raaga works within Paygate's limits as they are today (mapped [below](#paygate-limits-and-what-raaga-does)). Orders above them are paid by hand.
- **Countries:** each country needs its own PI-SPI account, integrated into Paygate. Paygate has one today (Côte d'Ivoire), so PI-SPI is offered to buyers in that country only.
- **Confirmation:** a person confirms each PI-SPI payment in the Paygate console (`PISPI_POLL` off). Rules for confirming automatically may come later. So checkout does **not** wait for the payment: the order is placed, then paid.
- **Payment requests (RTP):** only the buyer's PI-SPI alias is needed.

## Today

- The only payment provider is Medusa's manual `pp_system_default`; payment is arranged by hand.
- Accepting a quote (`customer-accept-quote`) turns its draft order into an order with no payment step.
- Markets: XOF (WAEMU countries) and USD (rest of world). **Paygate is XOF only**, so the USD region keeps manual payment.

## Paygate limits and what Raaga does

From Paygate's [`docs/api.md`](https://github.com/labsprogiciel-coder/paygate/blob/main/docs/api.md) and `src/payments.ts`:

| Paygate limit | Value | Raaga does |
|---|---|---|
| Currency | XOF only | Offer Paygate methods on the XOF region only; USD stays manual |
| Amount unit | Whole XOF, no decimals | Round the order total to a whole XOF before sending; keep XOF prices whole |
| Minimum | 100 XOF | Hide Paygate methods below it (never happens in practice for parts) |
| Maximum per payment | Tenant `maxAmount`, default **2,000,000 XOF** | Hide PI-SPI when the total is above it; the buyer pays by hand (bank transfer). Show why: "Bank payment up to 2,000,000 FCFA; above that we send payment details." Read the live value from `GET /v1/me` rather than hard-coding it |
| Countries with PI-SPI | One PI-SPI account per country; **Côte d'Ivoire** now | Offer PI-SPI only when the shipping/billing country has an account; others pay by hand |
| `reference` | 3–64 of `A-Za-z0-9_-`, unique per tenant | The Medusa payment session id (`payses_…`) or payment collection id (`pay_col_…`): unique, within the pattern |
| Same reference, different details | `409 idempotency_conflict` | A changed amount always gets a new session, so a new reference |
| `metadata` | Object, at most 2 KB | Only ids: `{ cart_id, order_id?, quote_id? }` |
| Expiry, QR | 60 s – 24 h, default 3 h | Ask for **24 h** (`expiresInSeconds: 86400`): a person confirms, and B2B buyers may pay from an office later in the day |
| Expiry, RTP | 60 s – 24 h, default 1 h | Ask for **24 h**, same reason |
| Late payments | Honoured after expiry (`late: true`) | Never cancel an order only because its payment expired; a late `payment.completed` still pays it |
| Payment requests | **3 per buyer alias per day** (`429 rtp_limit`) | Show "Resend" at most 3 times a day; after that, show the QR |
| Settlement | PI-SPI record, **exact** amount, right category | Show the exact amount and tell the buyer not to split or round it; a different amount is a `payment.mismatch` |
| Reads | Provider asked at most every 4 s per payment | Don't poll; rely on webhooks. A "check again" button on the order page is enough |
| Webhooks | At least once, any order; retried 10 s → 24 h, then dead | Dedupe on event `id`; answer 2xx fast; `GET` the payment when unsure |
| Refunds | Not supported | Refund by bank transfer, recorded by hand in Medusa |
| Sandbox | Mobile money only | Test PI-SPI with the participant's test environment, or confirm by hand in the console |

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

Each id becomes a provider the storefront can offer (`pp_paygate_pispi-qr`), enabled on the XOF region only and filtered by the [limits](#paygate-limits-and-what-raaga-does) above.

| Provider method | Does |
|---|---|
| `initiatePayment` | `POST /v1/collections` with `reference` = the Medusa payment session id, `amount` rounded to a whole XOF, `rail`, `expiresInSeconds: 86400`, `metadata: { cart_id, quote_id? }`. Stores the Paygate payment (id, `qr`, `expiresAt`) in the session `data`. |
| `authorizePayment` | Returns `authorized` once the collection exists, so the cart can become an order **before** the money arrives (the bank-transfer pattern) |
| `capturePayment` / `getPaymentStatus` | `GET /v1/collections/{id}`: `completed` → captured; anything else stays authorized |
| `updatePayment` | The amount changed: start a new collection with a new reference (a reference cannot change its amount) |
| `cancelPayment` / `deletePayment` | Nothing at Paygate (collections expire on their own); clear the session |
| `refundPayment` | Not supported by Paygate; refunds stay manual (bank transfer) and are recorded in Medusa |
| `getWebhookActionAndData` | Verify `Paygate-Signature` on the raw body (port Paygate's `src/signing.ts`), reject if `t` is older than 5 min; map `payment.completed` → `captured`, others → `not_supported` (handled by a subscriber that alerts Raaga); return the session id from `data.reference` |

**Webhook URL.** A Paygate tenant has one webhook URL. Point it at Medusa's built-in payment hook for the first provider (`/hooks/payment/paygate_pispi-qr`). Medusa finds the session by id, and every Paygate provider shares the same verification code, so mobile money later uses the same endpoint.

**Config** (`apps/backend/.env`): `PAYGATE_URL`, `PAYGATE_API_KEY`, `PAYGATE_WEBHOOK_SECRET`.

### 2. Checkout: order first, payment confirmed by a person

Because each PI-SPI payment is confirmed by hand, the buyer cannot wait on the checkout page:

1. Buyer picks "Bank payment (PI-SPI)" in the payment step (XOF region, a country with a PI-SPI account, total within the limit).
2. Placing the order creates the Paygate collection and completes the cart. The order is **placed, awaiting payment**.
3. The order confirmation page (and email) shows the QR, the exact amount, the expiry (24 h) and steps in French and English. The order page keeps showing it until it is paid.
4. The buyer scans and pays in their bank app.
5. Raaga confirms the payment in the Paygate console (`POST /admin/payments/{id}/resolve`, `completed`). Paygate sends `payment.completed`; Medusa captures the payment. **Fulfilment starts only on capture.**
6. Unpaid after 24 h: the order stays open, marked unpaid; Raaga is alerted and follows up by phone or email. A late payment still settles it.

Storefront work: a `pp_paygate_*` entry in `src/lib/constants.tsx` (`paymentInfoMap`), a PI-SPI QR component, a "pay for this order" block on the order confirmation and order details pages, and copy in both languages.

### 3. Accepted quotes

Quotes become orders without going through the cart, so they need their own path:

- After `customer-accept-quote` converts the draft order, create a payment collection for the order and a Paygate collection for its total (`reference` = the payment collection's id).
- The quote page shows the QR (or the RTP prompt) to pay the accepted price, same as an order page.
- `payment.completed` marks the order's payment captured; fulfilment starts only then.

This is also where negotiated prices (see the RFQ ideas) get paid, so it is worth building right after checkout.

### 4. Edge cases to handle

| Case | Handling |
|---|---|
| Duplicate webhook | Dedupe on the Paygate event `id` (Medusa's webhook processing is idempotent on the session, but log the event id) |
| `payment.expired` | Order stays open and unpaid; alert Raaga to follow up |
| `late: true` | Capture the order's payment as usual. If the order was cancelled meanwhile, alert Raaga to reinstate it or refund by bank transfer |
| `payment.mismatch` | Nothing settled. Alert Raaga; the operator resolves it in the Paygate console |
| `payment.failed` (RTP declined) | Order stays open; the buyer can pay by QR instead |
| Total above `maxAmount`, or no PI-SPI account for the country | PI-SPI is not offered; manual payment |

## Phases

| Phase | Scope |
|---|---|
| **1. PI-SPI QR, order first** | Provider module, webhook, QR on the order pages, XOF region and Côte d'Ivoire only, operator confirmation. Paygate tenant created. |
| **2. PI-SPI for accepted quotes** | Payment collection after quote acceptance, QR on the quote page, fulfilment waits for payment. |
| **3. PI-SPI payment requests (RTP)** | Store the buyer's PI-SPI alias on the company (next to the KYB fields); "Send to my bank app" next to the QR, at most 3 times a day. |
| **4. More countries** | As each country's PI-SPI account is added to Paygate, enable PI-SPI for that country. Needs the Paygate change below. |
| **5. Mobile money (pawaPay)** | `mobile-money` provider: operator + phone, `GET /v1/methods` to show only what works for that number and amount, Orange Burkina Faso one-time code (`pre_auth_code_required`), Wave redirect (`successUrl` / `failedUrl`), approval instructions while `processing`. Likely for smaller orders only, given operator limits. |
| **Later** | Automatic confirmation rules in Paygate (e.g. turn on polling per country or under an amount), which would make PI-SPI checkout instant. |

## Needed in Paygate

- **Several PI-SPI accounts.** Today Paygate has one receiving alias and one country (`PI_SPI_ACCOUNT_ALIAS`, `PISPI_COUNTRY`). For phase 4 it needs one account per country, and a way for a collection to say which country it is for (for example a `country` field on `pispi_qr` / `pispi_rtp` collections), so the QR carries the right alias and country code.
- **Countries in `GET /v1/methods`.** So Raaga can ask which countries PI-SPI is open for instead of keeping its own list.

## Setting up the Raaga tenant in Paygate

```bash
A='Authorization: Bearer <ADMIN_API_KEY>'
curl -X POST $GW/admin/tenants -H "$A" -H 'content-type: application/json' \
  -d '{"id":"raaga","name":"Raaga","idNamespace":"raaga","webhookUrl":"https://<backend>/hooks/payment/paygate_pispi-qr"}'
curl -X POST $GW/admin/tenants/raaga/keys -H "$A" -d '{"label":"prod"}'   # → PAYGATE_API_KEY (shown once)
curl $GW/admin/tenants/raaga/webhook-secret -H "$A"                      # → PAYGATE_WEBHOOK_SECRET
```

No `maxAmount`: the tenant uses the gateway default (2,000,000 XOF).

## Open question

- **Refunds.** Paygate does not refund. Confirm that refunds by bank transfer, recorded by hand in Medusa, are acceptable.
