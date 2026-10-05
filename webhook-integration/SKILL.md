---
name: webhook-integration
description: Coinflow webhook setup, signature verification, event handling, deduplication, and payload structures for all checkout, subscription, payout, and KYC events.
---

# Coinflow Webhook Integration

You are an expert at integrating Coinflow webhooks. You know every event type, payload structure, and common pitfall. When the developer asks you to handle webhooks, follow these instructions exactly.

## Setup

1. Open Coinflow Admin Dashboard > Developers > Webhooks
2. Add your webhook endpoint URL
3. Generate a **Webhook Validation Key**
4. Select **Latest Webhook Version** (v2) and desired events
5. Create a POST route on your server

## Critical Requirements

- **Respond within 5 seconds** or the request will timeout and retry
- **Always return HTTP 200** on successful processing
- **Coinflow retries** until you return 200 OR 36 hours pass
- **Webhooks may be sent more than once** — you MUST implement deduplication logic
- **Queue long-running tasks** for background processing to avoid timeouts

---

## Authentication

### Option 1: Authorization Header (simpler)

```javascript
app.post('/coinflow-webhook', (req, res) => {
  const authHeader = req.get('Authorization');
  if (authHeader !== process.env.COINFLOW_VALIDATION_KEY) {
    return res.status(401).send('Unauthorized');
  }

  const {data, eventType, created} = req.body;
  handleEvent(eventType, data);
  res.sendStatus(200);
});
```

### Option 2: Signature Verification (more secure, required when Authorization header is overridden)

The `Coinflow-Signature` header format:
```
Coinflow-Signature: t=1717012345,v1=5257a869e7ecebeda32affa62cdca3fa51cad7e77a0e56ff536d0ce8e108d8bd
```

- `t` = Unix timestamp (seconds) when signature was generated
- `v1` = HMAC-SHA256 hex digest of `{timestamp}.{raw JSON body}`

**Node.js/TypeScript:**
```typescript
import crypto from 'node:crypto';

function verifyWebhookSignature({
  signatureHeader,
  payload,
  secret,
}: {
  signatureHeader: string;
  payload: string;
  secret: string;
}): boolean {
  const parts = signatureHeader.split(',');
  let timestamp: string | undefined;
  let signature: string | undefined;

  for (const part of parts) {
    const [key, value] = part.split('=', 2);
    if (key === 't') timestamp = value;
    else if (key === 'v1') signature = value;
  }

  if (!timestamp || !signature) {
    throw new Error('Invalid Coinflow-Signature header');
  }

  const signedPayload = `${timestamp}.${payload}`;
  const expected = crypto
    .createHmac('sha256', secret)
    .update(signedPayload)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected),
  );
}
```

**Express middleware:**
```typescript
// CRITICAL: Must use raw body, not parsed JSON
app.post('/coinflow-webhook',
  express.raw({type: 'application/json'}),
  (req, res) => {
    const signatureHeader = req.headers['coinflow-signature'] as string;
    const rawBody = req.body.toString(); // Must be raw string

    const isValid = verifyWebhookSignature({
      signatureHeader,
      payload: rawBody,
      secret: process.env.COINFLOW_VALIDATION_KEY!,
    });

    if (!isValid) return res.status(401).send('Invalid signature');

    const event = JSON.parse(rawBody);
    handleEvent(event);
    res.sendStatus(200);
  }
);
```

**CRITICAL**: Verify against the **raw request body string**, NOT parsed-and-re-serialized JSON. Re-serializing changes whitespace or key order, causing verification failure.

**Python:**
```python
import hmac
import hashlib

def verify_webhook_signature(signature_header: str, payload: str, secret: str) -> bool:
    parts = dict(p.split("=", 1) for p in signature_header.split(","))
    timestamp = parts.get("t")
    signature = parts.get("v1")

    if not timestamp or not signature:
        raise ValueError("Invalid Coinflow-Signature header")

    signed_payload = f"{timestamp}.{payload}"
    expected = hmac.new(
        secret.encode(), signed_payload.encode(), hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(signature, expected)
```

---

## Deduplication

Coinflow may send the same webhook multiple times. Deduplicate by payment ID using a persistent store:

```javascript
// Use a database or Redis to track processed events — never use in-memory state,
// which resets on every deploy/restart and doesn't work across multiple instances.

app.post('/coinflow-webhook', async (req, res) => {
  const {data, eventType} = req.body;
  const deduplicationKey = `${eventType}:${data.id}`;

  const alreadyProcessed = await db.collection('processedWebhooks').findOne({_id: deduplicationKey});
  if (alreadyProcessed) {
    return res.sendStatus(200); // Already processed
  }

  await db.collection('processedWebhooks').insertOne({
    _id: deduplicationKey,
    processedAt: new Date(),
  });

  handleEvent(eventType, data);
  res.sendStatus(200);
});
```

---

## Event Types & Payloads

### Checkout Events (Category: "Purchase")

#### Settled
Payment completed, funds sent to merchant settlement location.

```json
{
  "eventType": "Settled",
  "category": "Purchase",
  "created": "2025-04-28T20:11:07.608Z",
  "data": {
    "id": "78f9be3f-691f-4f8c-82f7-c70221b006e7",
    "signature": "3zP3VcWM6rKF1izpnWn5rJ7XJucoEicwncQ9hvGR7Q7FuGvDPenfBrfLUVc4fjbYghnzauTDfY4c8Jc2Nb5y1AWm",
    "webhookInfo": {"example": "{\"purchaseId\":\"123abc\"}"},
    "subtotal": {"cents": 500, "currency": "USD"},
    "fees": {"cents": 46, "currency": "USD"},
    "gasFees": {"cents": 0, "currency": "USD"},
    "chargebackProtectionFees": {"cents": 0, "currency": "USD"},
    "total": {"cents": 546, "currency": "USD"},
    "merchantId": "testtest",
    "customerId": "customer1",
    "rawCustomerId": "customer1"
  }
}
```

#### Card Payment Authorized
Card issuer authorized the payment (funds not yet settled).

Same structure as Settled but `eventType: "Card Payment Authorized"`.

#### Card Payment Declined
Card issuer declined. Includes decline details:

```json
{
  "eventType": "Card Payment Declined",
  "data": {
    "declineCode": "59",
    "declineDescription": "The transaction is suspected of fraud.",
    ...sameFieldsAsSettled
  }
}
```

#### Card Payment Suspected Fraud
Chargeback protection provider rejected due to fraud suspicion.

#### Payment Pending Review
Awaiting merchant review/approval (if pending review feature is enabled).

#### USDC Payment Received
Direct USDC payment received in settlement location.

```json
{
  "eventType": "USDC Payment Received",
  "data": {
    "webhookInfo": {"item": "sword"},
    "subtotal": {"cents": 200, "currency": "USD"},
    "fees": {"cents": 0, "currency": "USD"},
    ...
  }
}
```

#### Refund

```json
{
  "eventType": "Refund",
  "data": {
    "totals": {
      "subtotal": {"currency": "USD", "cents": 500},
      "fees": {"cents": 46, "currency": "USD"},
      "total": {"cents": 546, "currency": "USD"}
    },
    "id": "78f9be3f-...",
    "paymentId": "0197a7e7-...",
    "wallet": "...",
    "blockchain": "solana"
  }
}
```

---

### PayPal / Venmo Events (Category: "Purchase")

| Event | Description |
|-------|-------------|
| PayPal Payment Authorized | PayPal payment authorized but not yet captured |
| Venmo Payment Authorized | Venmo payment authorized but not yet captured |
| Settled | Payment completed (same structure as card Settled) |
| PayPal Failure / Venmo Failure | Payment failed during authorization or capture |

The `PayPal Payment Authorized`, `Venmo Payment Authorized`, and `Settled` events include extra identity fields when available on the payment:

```json
{
  "eventType": "PayPal Payment Authorized",
  "category": "Purchase",
  "data": {
    "id": "78f9be3f-691f-4f8c-82f7-c70221b006e7",
    "customerId": "customer1",
    "merchantId": "testtest",
    "paymentMethod": "paypal",
    "paymentMethodToken": "tok_paypal_abc123",
    "payerName": "John Doe",
    "payerFirstName": "John",
    "payerLastName": "Doe",
    "...": "...standard amount fields (subtotal, fees, total, etc.)"
  }
}
```

- `paymentMethodToken` — vaulted payment method token for the payment
- `payerName` / `payerFirstName` / `payerLastName` — the PayPal/Venmo account holder's name, useful for reconciling payments to customers. Present when the payer identity was captured on the payment.
- The `Settled` event for PayPal/Venmo additionally includes `payerId` — the payer's PayPal/Venmo account id.

---

### ACH Events (Category: "Purchase")

| Event | Description |
|-------|-------------|
| ACH Initiated | ACH payment started |
| ACH Batched | ACH accepted by bank, processing |
| ACH Returned | Bank returned the ACH payment |
| ACH Failed | ACH denied by bank |
| Settled | ACH payment completed (same structure as card Settled) |

**ACH lifecycle:** Initiated → Batched → Settled (success) OR Returned/Failed (failure)

---

### PIX Events (Category: "Purchase")

| Event | Description |
|-------|-------------|
| Settled | PIX payment completed |
| PIX Failed | PIX payment failed during processing |
| PIX Expiration | Payment window expired before PIX completion |

---

### SEPA / UK Faster Payments Events (Category: "Purchase")

| Event | Description |
|-------|-------------|
| Settled | Payment completed |
| Payment Expiration | Payment window expired |

---

### Chargeback Events (Category: "Purchase")

#### Card Payment Chargeback Opened

```json
{
  "eventType": "Card Payment Chargeback Opened",
  "data": {
    "id": "c10d7ac6-...",
    "chargebackId": "11111111111111111111111",
    "reasonCode": "4853",
    "reasonDescription": "Merchandise/Services Not as Described",
    "respondByDate": "2025-07-01T00:00:00.000Z",
    "subtotal": {"cents": 2500, "currency": "USD"},
    "fees": {"cents": 169, "currency": "USD"},
    "chargebackProtectionFees": {"cents": 68, "currency": "USD"},
    "total": {"cents": 2737, "currency": "USD"},
    "customerId": "user123"
  }
}
```

**IMPORTANT:** `respondByDate` is your deadline to submit evidence. Act on this immediately.

#### Card Payment Chargeback Won / Lost

Same structure with `eventType: "Card Payment Chargeback Won"` or `"Card Payment Chargeback Lost"`.

---

### Crypto Payment Events (Category: "Purchase")

#### Crypto Overpayment

```json
{
  "eventType": "Crypto Overpayment",
  "data": {
    "paymentId": "78f9be3f-...",
    "sessionId": "ses_abc123xyz",
    "refundUrl": "https://...",
    "expectedAmount": "10.00",
    "actualAmount": "12.50",
    "currencySymbol": "USDC",
    "actualAmountUSD": "12.50"
  }
}
```

#### Crypto Underpayment

Same structure but `actualAmount < expectedAmount`. Use `refundUrl` to process refund.

---

### Subscription Events

#### Subscription Category Events

| Event | Description | Key Fields |
|-------|-------------|------------|
| Subscription Created | New subscription activated | `planCode`, `planName`, `subscriptionId`, `amount`, `interval`, `fundingMethod` |
| Subscription Canceled | Canceled by merchant/customer | + `reason` field |
| Subscription Expired | Plan no longer active | |
| Subscription Failure | Payment failed, not created/renewed | |
| Subscription Concluded | Duration completed | |

```json
{
  "eventType": "Subscription Created",
  "category": "Subscription",
  "data": {
    "planCode": "music_access",
    "planName": "creator1234",
    "subscriptionId": "680fe4d832ddccc1ee91885e",
    "customerId": "78C3dn...",
    "amount": {"cents": 500, "currency": "USD"},
    "interval": "Monthly",
    "fundingMethod": "Card",
    "webhookInfo": {"item": "sword"}
  }
}
```

#### Subscription Purchase Events

Subscription renewals fire **Settled** events with a `subscription` object in the payload:

```json
{
  "eventType": "Settled",
  "category": "Purchase",
  "data": {
    "id": "db746ccf-...",
    "subscription": {
      "_id": "680fe4d832ddccc1ee91885e",
      "plan": {
        "name": "creator1234",
        "code": "music_access",
        "interval": "Monthly",
        "duration": 12,
        "amount": {"cents": 500, "currency": "USD"}
      },
      "status": "Active",
      "nextPaymentAt": "2025-05-28T20:28:08.164Z"
    },
    "subtotal": {"cents": 500, "currency": "USD"},
    ...
  }
}
```

---

### KYC Events (Category: "KYC")

| Event | Description |
|-------|-------------|
| KYC Created | KYC process started |
| KYC Success | KYC verification passed |
| KYC Failure | KYC verification failed |

```json
{
  "eventType": "KYC Success",
  "category": "KYC",
  "data": {
    "wallet": "test-withdrawer-f",
    "blockchain": "user",
    "email": "testwithdrawer-f@gmail.com"
  }
}
```

---

### Withdraw Events (Category: "Withdraw")

| Event | Description |
|-------|-------------|
| Withdraw Pending | Withdrawal initiated |
| Withdraw Success | Withdrawal submitted to bank |
| Withdraw Failure | Withdrawal failed |

```json
{
  "eventType": "Withdraw Success",
  "category": "Withdraw",
  "data": {
    "wallet": "user123",
    "blockchain": "solana",
    "signature": "5nQq5iVkoEc1...",
    "userFees": {"cents": 200, "currency": "USD"},
    "userGasFees": {"cents": 0, "currency": "USD"},
    "merchantGasFees": {"cents": 2, "currency": "USD"},
    "total": {"cents": 202, "currency": "USD"},
    "currency": "USD",
    "merchantId": "testtest",
    "idempotencyKey": "merchant-withdrawal-12345"
  }
}
```

For PayPal and Venmo payouts, `Withdraw Failure` events can include `data.reason`, for example `"RECEIVER_UNREGISTERED: The recipient does not have a PayPal or Venmo account for this email or phone number"`. It is left out when no reason is available and for other payout methods.

---

### Crypto Deposit Events (Category: "CryptoDeposit")

#### CryptoDepositFundsReceived

```json
{
  "eventType": "CryptoDepositFundsReceived",
  "category": "CryptoDeposit",
  "data": {
    "sessionId": "session-123",
    "idempotencyKey": "00000000-0000-0000-0000-000000000000",
    "depositAddress": "deposit-address-here",
    "blockchainChainId": "solana",
    "tokenSymbol": "USDC",
    "user": "user-456",
    "amount": {"cents": 10000, "currency": "USD"},
    "merchantId": "merchant-789",
    "transactionHash": "tx-hash-here",
    "receivedAt": "2024-01-01T00:00:00.000Z"
  }
}
```

---

## Complete Event Reference

| Event Type | Category | When |
|-----------|----------|------|
| Settled | Purchase | Payment completed and funds delivered |
| Card Payment Authorized | Purchase | Card issuer authorized |
| Card Payment Declined | Purchase | Card issuer declined |
| PayPal Payment Authorized | Purchase | PayPal payment authorized |
| Venmo Payment Authorized | Purchase | Venmo payment authorized |
| PayPal Failure | Purchase | PayPal payment failed |
| Venmo Failure | Purchase | Venmo payment failed |
| Card Payment Suspected Fraud | Purchase | Fraud detection rejected |
| Payment Pending Review | Purchase | Awaiting manual review |
| USDC Payment Received | Purchase | Direct USDC received |
| Card Payment Chargeback Opened | Purchase | Chargeback investigation started |
| Card Payment Chargeback Won | Purchase | Chargeback resolved for merchant |
| Card Payment Chargeback Lost | Purchase | Chargeback resolved for cardholder |
| ACH Initiated | Purchase | ACH started |
| ACH Batched | Purchase | ACH accepted by bank |
| ACH Returned | Purchase | ACH returned by bank |
| ACH Failed | Purchase | ACH denied by bank |
| PIX Failed | Purchase | PIX processing failed |
| PIX Expiration | Purchase | PIX window expired |
| Payment Expiration | Purchase | SEPA/PIX window expired |
| Refund | Purchase | Payment refunded |
| Crypto Overpayment | Purchase | More crypto sent than required |
| Crypto Underpayment | Purchase | Less crypto sent than required |
| Subscription Created | Subscription | Subscription activated |
| Subscription Canceled | Subscription | Customer/merchant canceled |
| Subscription Expired | Subscription | Plan no longer active |
| Subscription Failure | Subscription | Payment failed |
| Subscription Concluded | Subscription | Duration completed |
| KYC Created | KYC | Verification started |
| KYC Success | KYC | Verification passed |
| KYC Failure | KYC | Verification failed |
| Withdraw Pending | Withdraw | Withdrawal initiated |
| Withdraw Success | Withdraw | Withdrawal submitted |
| Withdraw Failure | Withdraw | Withdrawal failed |
| CryptoDepositFundsReceived | CryptoDeposit | Crypto deposit received |

---

## Testing Webhooks

### Simulate chargeback events in sandbox:
1. Complete a test payment
2. Wait for payment status = `settled`
3. Go to Purchases table in dashboard
4. Click chargeback icon → triggers `Card Payment Chargeback Opened`
5. Go to Chargebacks tab
6. Select payment → click `Chargeback Won` or `Chargeback Lost`

### Custom webhookInfo

Pass custom data through the checkout flow to your webhook:

```javascript
// In CoinflowPurchase or API request:
webhookInfo: {orderId: "abc123", userId: "user456"}

// Received in webhook payload:
data.webhookInfo // {"orderId": "abc123", "userId": "user456"}
```

---

## Common Pitfalls

1. **Parsing body before verifying signature** — Use `express.raw()` middleware to get the raw string body for signature verification. If you use `express.json()` first, re-serializing will break the signature.

2. **Not returning 200 fast enough** — Coinflow times out after 5 seconds. Queue heavy processing (database writes, API calls, notifications) for background execution.

3. **Missing deduplication** — Webhooks can fire multiple times. Always check if you've already processed a given `eventType:paymentId` combination.

4. **Ignoring retry behavior** — If your server is down, Coinflow retries for 36 hours. Your handler must be idempotent.

5. **Not handling all payment method event flows** — ACH has different lifecycle events than cards. PIX and SEPA have expiration events. Make sure your handler covers all relevant event types for the payment methods you support.

6. **Confusing eventType strings** — Event types contain spaces (e.g., `"Card Payment Authorized"`, not `"CardPaymentAuthorized"`). Match exactly.
