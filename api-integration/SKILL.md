---
name: api-integration
description: Coinflow API authentication, session keys, environment setup, rate limits, SDK configuration, subscriptions, marketplaces, and cross-cutting concerns.
---

# Coinflow API Integration

You are an expert at integrating Coinflow's API. You know every authentication pattern, environment configuration, rate limit, and cross-cutting concern. When the developer asks about Coinflow API setup, authentication, subscriptions, marketplaces, or general integration topics, follow these instructions exactly.

## Environments

| Environment | API URL | UI URL | Dashboard |
|-------------|---------|--------|-----------|
| **sandbox** | `https://api-sandbox.coinflow.cash` | `https://sandbox.coinflow.cash` | `https://sandbox.merchant.coinflow.cash` |
| **prod** | `https://api.coinflow.cash` | `https://coinflow.cash` | `https://merchant.coinflow.cash` |

Always default to sandbox during development. The sandbox is fully functional with test data.

---

## Authentication

### API Key

Every server-to-server call requires the `Authorization` header with your API key from the merchant dashboard.

```bash
Authorization: YOUR_API_KEY
```

### Session Keys

For frontend or user-scoped operations, create a session key:

```bash
GET /api/auth/session-key
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: customer123
```

Response:
```json
{"key": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
```

- **Valid for 30 minutes** — must refresh before expiry
- Pass as `x-coinflow-auth-session-key` header

### Wallet Authentication

For blockchain wallet-based flows:

```bash
x-coinflow-auth-wallet: WALLET_PUBLIC_KEY
x-coinflow-auth-blockchain: solana
x-coinflow-auth-signed-message: WALLET_SIGNATURE
```

### Full Header Reference

| Header | Purpose | When Required |
|--------|---------|--------------|
| `Authorization` | API Key | All server-to-server calls |
| `x-coinflow-auth-session-key` | Session JWT | Frontend/user-scoped calls |
| `x-coinflow-auth-user-id` | Customer identifier | Payout, customer management |
| `x-coinflow-auth-wallet` | Wallet public key | Wallet-based flows |
| `x-coinflow-auth-blockchain` | Chain identifier | Wallet-based flows |
| `x-coinflow-auth-signed-message` | Wallet signature | Signature verification |
| `x-coinflow-auth-merchant-id` | Merchant ID | Merchant-initiated flows |
| `x-coinflow-submerchant-id` | Seller ID | Marketplace flows |
| `x-device-id` | Device fingerprint | Checkout with chargeback protection |
| `x-session-id` | Session ID | Checkout with fraud detection |
| `x-coinflow-client-ip` | Customer IP address | Card checkout |

---

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| **Global** | 100 requests / 1 minute |
| **Login by Email** | 3 / 90 seconds |
| **Login by MerchantID** | 3 / 90 seconds |
| **KYC** | 3 / 1 minute |
| **Checkout JWT** | 5 / 10 minutes |
| **Not Found (404)** | 10 / 10 seconds |
| **Support Tickets** | 60 / 1 hour |
| **Failed Checkouts** | 5 / 10 minutes (sandbox) |

---

## API Endpoint Groups

| Group | Purpose |
|-------|---------|
| **Checkout** | Payment initiation (card, ACH, crypto, Apple Pay, etc.) |
| **Merchant** | Dashboard, payments, settlements, customers |
| **Withdraw** | User withdrawals (bank, card, IBAN, PIX, etc.) |
| **Customer** | User account/payment method management |
| **Subscription** | Recurring payments |
| **Marketplace** | Seller/buyer marketplace features |
| **Tokenize** | PCI-compliant card tokenization |
| **Auth** | Session keys, verification |
| **Refunds** | Refund processing |
| **Events** | Fraud detection event tracking |

---

## Destination Tokenization

When settling to a specific wallet, tokenize it:

```bash
POST /api/checkout/destination-auth-key
{
  "blockchain": "base",
  "destination": "0x70bd2A7a9eedE3aCb62cF839cEa1008b72FF7844"
}
```

Response:
```json
{"destinationAuthKey": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
```

Pass `destinationAuthKey` in checkout requests when settling to a specific wallet.

---

## Subscriptions

### Create a Plan

Via API:
```bash
POST /api/merchant/subscription/plans
Authorization: YOUR_API_KEY

{
  "name": "Premium Monthly",
  "code": "premium_monthly",
  "description": "Monthly premium access",
  "interval": "Monthly",
  "duration": 12,
  "amount": {"cents": 999, "currency": "USD"}
}
```

Intervals: `Monthly`, `Yearly`, `Weekly`, `Daily`

Duration is optional — omit for no automatic expiry.

### Get Available Plans

```bash
GET /api/subscription/YOUR_MERCHANT_ID/plans
```

### Subscribe via UI

```jsx
<CoinflowPurchase
  {...baseProps}
  planCode="premium_monthly"
  chargebackProtectionData={[...]}
/>
```

First payment is drawn immediately.

### Subscribe via API

```bash
POST /api/subscription/YOUR_MERCHANT_ID/subscribers/card
{
  "planCode": "premium_monthly",
  "card": {...cardData},
  "chargebackProtectionData": [...]
}
```

### Get Customer Subscriptions

```bash
GET /api/subscription/YOUR_MERCHANT_ID/subscribers
x-coinflow-auth-session-key: JWT_TOKEN
```

### Cancel Subscription

```bash
PATCH /api/subscription/YOUR_MERCHANT_ID/subscribers/SUBSCRIPTION_ID
{
  "status": "Canceled"
}
```

### Subscription States

| State | Meaning |
|-------|---------|
| Active | In good standing, will renew |
| Canceled | Canceled by merchant or customer |
| Expired | Plan no longer available |
| Concluded | Duration reached |
| Failed | Recurring payment failed |
| Blocked | Customer blocked in ecosystem |

### Subscription Webhooks

- `Subscription Created` — New subscription activated
- `Subscription Canceled` — Includes `reason` field
- `Subscription Expired` — Plan expired
- `Subscription Failure` — Payment failed
- `Subscription Concluded` — Duration completed
- `Settled` (with `subscription` object) — Renewal payment completed

**Chargeback protection covers 100%** of subscription chargebacks when enabled.

---

## Marketplaces

### Architecture

Four players: **Coinflow** ↔ **Marketplace Platform** ↔ **Sellers** ↔ **End Users**

### Setup Flow

1. Configure marketplace account + generate API keys
2. Create seller registration links → sellers undergo KYC/KYB
3. Sellers access seller dashboard
4. Generate purchase/subscription links for end-users
5. End-users checkout; funds auto-distributed between seller and marketplace

### Seller Registration

```bash
POST /api/marketplace/link/seller/registration
{
  "email": "seller@example.com",
  "sellerId": "seller-123",
  "country": "US",
  "redirectUrl": "https://yourapp.com/seller-dashboard"
}
```

### Seller Login

```bash
POST /api/marketplace/link/seller/login
{"email": "seller@example.com"}
```

### Purchase Link (with fee split)

```bash
POST /api/checkout/link
{
  "subtotal": {"currency": "USD", "cents": 1000},
  "feePercentage": 10,
  "sellerId": "seller-123",
  "email": "buyer@example.com",
  "chargebackProtectionData": [...],
  "webhookInfo": {...}
}
```

### Subscription Link

```bash
POST /api/marketplace/link/subscription
{
  "planCode": "premium_monthly",
  "feePercentage": 10,
  "sellerId": "seller-123",
  "email": "buyer@example.com",
  "chargebackProtectionData": [...]
}
```

### Marketplace Credits

1. Generate wallet for payer: `POST /api/marketplace/wallet`
2. Purchase credits: `POST /api/checkout/link` with `settlementType: "Credits"`
3. Check balance: `GET /api/customer/balances/{merchantId}`
4. Spend credits: `POST /api/marketplace/link/purchase/credits`

---

## Supported Blockchains

| Blockchain | ID | Notes |
|------------|----|-------|
| Solana | `solana` | Recommended (Coinflow covers gas) |
| Ethereum | `eth` | EVM |
| Base | `base` | EVM |
| Polygon | `polygon` | EVM |
| Arbitrum | `arbitrum` | EVM |
| Stellar | `stellar` | XDR transactions |
| Monad | `monad` | EVM |
| Tempo | `tempo` | EVM |

For users without wallets, use `sessionKey` authentication or generate deterministic wallets.

---

## Dashboard Roles

| Role | Permissions |
|------|------------|
| **Viewer** | Read-only |
| **Chargeback** | Read + respond to chargebacks + override protection |
| **Editor** | Write actions (refunds, webhooks, API keys, settings) |
| **Admin** | Everything + user management + 2FA setup |

2FA is **required** for all logins.

---

## File Upload

Upload evidence files (for chargebacks, etc.):

```bash
# 1. Get presigned URL
POST /api/merchant/files/upload-url
Authorization: YOUR_API_KEY
{"fileName": "evidence.pdf", "contentType": "application/pdf"}

# Returns: {uploadUrl: "...", fileKey: "merchants/id/uuid.pdf"}

# 2. Upload file
PUT {uploadUrl}
Content-Type: application/pdf
Body: <file binary>

# 3. Reference in API calls
POST /api/merchant/chargebacks/PAYMENT_ID/respond
{"fileKey": "merchants/id/uuid.pdf"}
```

Accepted types: PDF, PNG, JPEG, GIF, ZIP, DOCX, XLSX, CSV.
Presigned URL valid for **1 hour**.

---

## Error Handling Patterns

### Common HTTP Status Codes

| Status | Meaning | Action |
|--------|---------|--------|
| 200 | Success | Process response |
| 400 | Bad Request | Check request body/params |
| 401 | Unauthorized | Check API key / session key |
| 403 | Forbidden | Feature not enabled on account |
| 404 | Not Found | Check endpoint URL / resource ID |
| 410 | Gone | Resource expired (CoF limits, token expiry) |
| 412 | Precondition Failed | 3DS challenge required |
| 422 | Unprocessable | Missing required field (e.g., address for AVS) |
| 429 | Rate Limited | Back off and retry |
| 451 | Unavailable for Legal Reasons | Additional KYC verification needed |
| 500+ | Server Error | Retry with exponential backoff |

### Retry Strategy

```javascript
const MAX_RETRIES = 3;
const BASE_DELAY = 100; // ms

async function apiCallWithRetry(fn) {
  for (let attempt = 0; attempt < MAX_RETRIES; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (error.response?.status < 500 && error.response?.status !== 429) {
        throw error; // Don't retry client errors
      }
      const delay = BASE_DELAY * Math.pow(2, attempt) + Math.random() * 100;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  throw new Error('Max retries exceeded');
}
```

---

## Testnet Resources

### Solana
- USDC Token (Sandbox): `4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU`
- USDC Token (Prod): `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`

### EVM
- ETH Sepolia, Base Sepolia, Polygon Amoy
- Requires wETH/wMATIC deposits via contract function for testnet liquidity

### ACH Testing
Use sandbox credentials for bank account linking in sandbox environment. See Coinflow docs for test bank account details.

---

## Coinflow Credits System

Credits are a store-of-value:
- 1 credit = $1 USD (1:1 equivalence)
- Held in digital wallet
- **Not withdrawable** (app-locked to merchant platform)
- No expiration
- When spent, funds settle as USDC to merchant

Use `settlementType: "Credits"` in checkout to issue credits.

---

## Common Integration Patterns

### Pattern 1: Session-Based (No Wallet)
```jsx
<CoinflowPurchase
  sessionKey={jwtToken}
  merchantId="id"
  env="sandbox"
  subtotal={{cents: 500, currency: "USD"}}
  onSuccess={handleSuccess}
/>
```

### Pattern 2: Wallet-Based
```jsx
<CoinflowPurchase
  wallet={solanaWallet}
  connection={connection}
  blockchain="solana"
  merchantId="id"
  onSuccess={handleSuccess}
/>
```

### Pattern 3: Checkout Link (No Frontend Code)
```bash
POST /api/checkout/link
# Embed returned URL in iframe or redirect
```

### Pattern 4: Full API (Custom UI)
```
GET /api/auth/session-key → POST /api/checkout/totals → Tokenize Card → POST /api/checkout/card
```

---

## Iframe Communication

When embedding Coinflow UI in iframes, communication happens via `postMessage`:

### Message Types from Coinflow

| Message | Purpose |
|---------|---------|
| `{method: 'loaded'}` | UI finished loading |
| `{method: 'heightChange', data: '500'}` | Content height changed |
| `{data: 'success', info: {paymentId: '...'}}` | Payment completed |
| `{data: 'authDeclined', info: {...}}` | Payment declined |
| `{data: 'accountLinked', info: {type: 'bank'}}` | Account linked (bank/card/iban/pix) |

### Listening for Messages

```javascript
window.addEventListener('message', (event) => {
  // Always validate origin with an exact match (see the table above); a
  // substring check like event.origin.includes('coinflow.cash') is bypassable.
  if (event.origin !== 'https://coinflow.cash') return;

  const data = typeof event.data === 'string' ? JSON.parse(event.data) : event.data;

  switch (data.method || data.data) {
    case 'loaded':
      setLoading(false);
      break;
    case 'heightChange':
      setHeight(data.data);
      break;
    case 'success':
      handlePayment(data.info.paymentId);
      break;
    case 'authDeclined':
      handleDecline(data.info);
      break;
    case 'accountLinked':
      handleAccountLinked(data.info.type); // 'bank' | 'card' | 'iban' | 'pix'
      break;
  }
});
```

### Loading Skeleton

Show a skeleton while the iframe loads:
```jsx
const [iframeLoaded, setIframeLoaded] = useState(false);

// Listen for 'loaded' message
// Show skeleton overlay with opacity transition:
<div style={{
  opacity: iframeLoaded ? 0 : 1,
  transition: 'opacity 300ms ease',
  pointerEvents: iframeLoaded ? 'none' : 'auto'
}}>
  <Skeleton />
</div>
```

---

## Security Best Practices

1. **Never expose API keys in frontend code** — Use session keys for frontend, API keys for server-only
2. **Always validate webhook signatures** — See webhook skill
3. **Use tokenization for all card data** — Never handle raw card numbers
4. **Validate postMessage origins** — Compare with an exact match, e.g. `event.origin === 'https://coinflow.cash'` (a substring check like `event.origin.includes('coinflow.cash')` is satisfied by attacker origins such as `https://coinflow.cash.evil.test`)
5. **Add chargeback protection to every page** — Not just checkout pages
6. **Pass real IP addresses** — `x-coinflow-client-ip` must be the actual customer IP, not your server IP
7. **Use HTTPS everywhere** — Coinflow will not load in insecure contexts

---

## FAQ Quick Reference

| Question | Answer |
|----------|--------|
| Minimum purchase? | $0.50 |
| Card limit per transaction? | $5,000 (request increase from sales) |
| Card limit per card? | $10,000 |
| ACH limit? | $50,000 |
| Session key validity? | 30 minutes |
| KYC response time? | Average <10 seconds |
| KYB response time? | 30 seconds auto, up to 24 hours manual |
| Refund timeline? | 2-3 days |
| Webhook retry window? | 36 hours |
| Webhook timeout? | 5 seconds |
| Token validity (unused)? | 7 days |
| Token validity (used once)? | 5 minutes (prod), indefinite (sandbox) |
