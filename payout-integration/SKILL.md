---
name: payout-integration
description: Coinflow payouts — KYC/KYB flows, payout method setup (bank, debit card, Venmo, PayPal, Interac, IBAN, PIX), quotes, and payout initiation across all regions and speeds.
---

# Coinflow Payout Integration

You are an expert at integrating Coinflow payouts (withdrawals). You know every KYC flow, payout method, regional constraint, and common pitfall. When the developer asks you to implement payouts, follow these instructions exactly.

## Payout Flow Overview

1. **Create Withdrawer** (KYC verification)
2. **Connect Payout Method** (bank, card, Venmo, PayPal, IBAN, PIX, Interac)
3. **Get Quote** (fees and speed options)
4. **Initiate Payout**
5. **Track & Monitor**

---

## SDKs

Coinflow provides official frontend SDKs:

| Framework | Package |
|-----------|---------|
| React | `@coinflowlabs/react` |
| React Native | `@coinflowlabs/react-native` |
| Vue | `@coinflowlabs/vue` |
| Angular | `@coinflowlabs/angular` |

---

## Step 1: Create Withdrawer (KYC)

**Rate limit:** 3 failed KYC attempts per minute.

Every withdrawer must complete verification before their first payout. KYC is one-time — unlimited payouts afterwards.

### Option A: Coinflow-Managed KYC (recommended)

**U.S. Users:**
```bash
POST /api/withdraw/kyc
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: user123

{
  "info": {
    "email": "user@example.com",
    "firstName": "John",
    "surName": "Doe",
    "physicalAddress": "2800 N Damen Ave",
    "city": "Chicago",
    "state": "IL",
    "zip": "60625",
    "country": "US",
    "dob": "19901015",
    "ssn": "1234"
  }
}
```

Response:
```json
{
  "withdrawer": {
    "_id": "672400ead654e5cdd247b33f",
    "currency": "USD",
    "email": "user@example.com",
    "verification": {"status": "approved"}
  }
}
```

**International (Non-U.S.) Users:**
```bash
POST /api/withdraw/kyc
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: user123

{
  "merchantId": "YOUR_MERCHANT_ID",
  "email": "user@example.com",
  "country": "GB"
}
```

**CRITICAL: Handling 451 Response**

If you receive HTTP **451**, additional verification is needed:
```json
{"verificationLink": "https://verify.coinflow.cash/..."}
```

Redirect the user to `verificationLink` — they'll upload a photo ID and take a selfie. Then call `GET /api/withdraw` to check if status is now `approved`.

### Option B: Custom KYC Provider — Pass Documents

```bash
POST /api/withdraw/kyc-doc
Content-Type: multipart/form-data
x-coinflow-auth-session-key: YOUR_SESSION_KEY

email=user@example.com
country=US
idType=ID_CARD
idFront=@id_front.png
idBack=@id_back.png
merchantId=YOUR_MERCHANT_ID
```

### Option C: Sumsub Integration (Enterprise)

Requires tri-party agreement with Sumsub and Coinflow.

```bash
POST /api/withdraw/kyc/share-token
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: user123

{
  "vendor": "sumsub",
  "shareToken": "YOUR_SHARE_TOKEN",
  "country": "US",
  "merchantId": "YOUR_MERCHANT_ID",
  "email": "user@example.com"
}
```

### Option D: KYC Reliance (Enterprise)

Requires Coinflow Compliance approval.

```bash
POST /api/withdraw/kyc/attested
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: user123

{
  "email": "user@example.com",
  "firstName": "John",
  "surName": "Doe",
  "physicalAddress": "201 E Randolph St",
  "city": "Chicago",
  "state": "IL",
  "zip": "60601",
  "country": "US",
  "dob": "10151990",
  "ssn": "1234"
}
```

### KYC Response Times
- KYC (individual): Average under 10 seconds
- KYB (business): Average 30 seconds (automatic), up to 24 hours for manual review
- 90% pass automatic verification

---

## Step 2: Get Withdrawer Details

Check verification status and linked payout methods:

```bash
GET /api/withdraw
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: user123
```

Response:
```json
{
  "withdrawer": {
    "_id": "672400ead654e5cdd247b33f",
    "currency": "USD",
    "email": "user@example.com",
    "verification": {"status": "approved"},
    "bankAccounts": [],
    "cards": [],
    "ibans": [],
    "pixes": []
  }
}
```

---

## Step 3: Connect Payout Method

Each region has different supported methods. Connect the appropriate one.

### U.S. — Bank Account (ACH)

```bash
POST /api/withdraw/account
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: user123

{
  "type": "checking",
  "alias": "Primary Checking",
  "accountNumber": "1111222233330000",
  "routingNumber": "333333334"
}
```

### U.S. — Debit Card

**Requires card tokenization first.** Use `CoinflowCardForm` or API tokenization.

```bash
POST /api/withdraw/debit-card
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: user123

{
  "cardToken": "411111YJM5TX1111",
  "expMonth": "10",
  "expYear": "29"
}
```

**For mobile apps (no CVV needed for debit payouts):**
```bash
# 1. Get config
GET /api/checkout/mobile-tokenize-config

# 2. Tokenize card only (no CVV)
POST /api/tokenize/mobile/card-only
{"data": "4111111111111111", ...config}

# 3. Add debit card
POST /api/withdraw/debit-card
{"cardToken": "411111YJM5TX1111", "expYear": "30", "expMonth": "10"}
```

**For web apps, use `CoinflowCardNumberForm` (card number and expiration, no CVV collection):**
```jsx
import {CoinflowCardNumberForm, CardFormRef} from '@coinflowlabs/react';

const ref = useRef<CardFormRef>(null);

<CoinflowCardNumberForm
  merchantId="your-merchant-id"
  ref={ref}
  env="sandbox"
/>

// On submit:
const {token, expMonth, expYear} = await ref.current.tokenize();
// Then POST /api/withdraw/debit-card with token, expMonth and expYear
```

### U.S. — Venmo

```bash
POST /api/withdraw/venmo
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: user123

{
  "merchantId": "YOUR_MERCHANT_ID",
  "phoneNumber": "1234567890"
}
```

### U.S. — PayPal

```bash
POST /api/withdraw/paypal
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: user123

{
  "phoneNumber": "1234567890",
  "merchantId": "YOUR_MERCHANT_ID"
}
```

### Canada — Interac

```bash
POST /api/withdraw/interac
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: user123

{
  "phoneNumber": "1234567890",
  "merchantId": "YOUR_MERCHANT_ID"
}
```

### EU/UK — IBAN

```bash
POST /api/withdraw/iban
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: user123

{
  "alias": "Primary Account",
  "number": "GB33BUKB20201555555555",
  "sortCode": "201234"
}
```

### Brazil — PIX

```bash
POST /api/withdraw/pix
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: user123

{
  "pixKey": "user@example.com"
}
```

### Response

All connect methods return the withdrawer with the new method's **token**:
```json
{
  "withdrawer": {
    "bankAccounts": [{
      "token": "0dc36240-29fc-4708-81f6-bcab19e6e597",
      "alias": "Primary Checking",
      "last4": "0000",
      "routingNumber": "333333334"
    }]
  }
}
```

Save the `token` — you'll need it for initiating payouts.

---

## Step 4: Get Quote

```bash
GET /api/withdraw/quote?token=USDC_TOKEN&amount=100&merchantId=YOUR_MERCHANT_ID&usePermit=true
Authorization: YOUR_API_KEY
x-coinflow-auth-blockchain: solana
x-coinflow-auth-user-id: user123
```

For card payouts, add `accountToken=<card token>`. If your account has a separate fee for non-USD (global) cards, a `card` quote without a known card (no `accountToken`, or `APPLE_PAY`) shows the higher of the two fees; the payout charges the fee for the actual card, so it is never more than quoted.

**USDC Token addresses:**
- Sandbox (Solana): `4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU`
- Production (Solana): `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`

Response:
```json
{
  "quote": {"cents": 10000, "currency": "USD"},
  "asap": {
    "fee": {"cents": 0, "currency": "USD"},
    "finalSettlement": {"cents": 10000, "currency": "USD"},
    "limit": {"cents": 1000000, "currency": "USD"},
    "expectedDeliveryDate": "in a few minutes",
    "expectedDeliveryDateISO": "2026-04-09T16:45:00.000Z"
  },
  "same_day": {
    "fee": {"cents": 50},
    "finalSettlement": {"cents": 9950},
    "limit": {"cents": 10000000}
  },
  "standard": {
    "fee": {"cents": 100},
    "finalSettlement": {"cents": 9900},
    "limit": {"cents": 10000000}
  },
  "card": {
    "fee": {"cents": 200},
    "finalSettlement": {"cents": 9800},
    "limit": {"cents": 5000000}
  }
}
```

---

## Step 5: Initiate Payout

### From Coinflow Wallet (Delegated)

```bash
POST /api/merchant/withdraws/payout/delegated
Authorization: YOUR_API_KEY

{
  "amount": {"cents": 10000},
  "speed": "same_day",
  "account": "0dc36240-29fc-4708-81f6-bcab19e6e597",
  "userId": "user123"
}
```

Response:
```json
{
  "signature": "4Cv4nbe6fkGpdSYcqhPHXkdndeiyTa8mhFoWW5x3vroxRibAUssrbXZ5VW4vxkPedcX3xTRKu7ZpkJXWKdJBCGuq"
}
```

### From Merchant's Own Wallet (BYO)

```bash
POST /api/merchant/withdraws/payout
Authorization: YOUR_API_KEY

{
  "blockchain": "solana",
  "amount": {"cents": 400},
  "speed": "standard",
  "wallet": "Dk462fqWGi3W6ak7ufDP4TCTfghbj4TQYukSnF4fx62J",
  "userId": "user123",
  "account": "4db3391e-3199-4c49-9897-9ad2e73390b6"
}
```

### Self-Custodial Wallet Payout (User Signs)

Using React SDK:
```jsx
import {CoinflowWithdraw} from '@coinflowlabs/react';

<CoinflowWithdraw
  wallet={{
    publicKey: new PublicKey('WALLET_PUBKEY'),
    sendTransaction: (tx) => Promise<string>,
    signMessage: (msg) => Promise<string>
  }}
  merchantId="YOUR_MERCHANT_ID"
  env="sandbox"
  blockchain="solana"
  onSuccess={(...args) => console.log('Withdraw Success', args)}
  email="user@email.com"
  amount={1}
/>
```

---

## Speed Options by Region

| Speed | Method | Region | Timing |
|-------|--------|--------|--------|
| `asap` | RTP (Real-Time Payments) | US | Instant |
| `same_day` | ACH Same Day | US | Same business day |
| `standard` | ACH Standard | US | 1-3 business days |
| `card` | Debit Card Push | US | Instant |
| `venmo` | Venmo | US | Instant |
| `paypal` | PayPal | US | Instant |
| `interac` | Interac | Canada | Instant |
| `iban` | SEPA Transfer | EU/UK | Instant |
| `pix` | PIX | Brazil | Instant |
| `eft` | Electronic Funds Transfer | Canada | 1-3 business days |

**CRITICAL: `effectiveSpeed` may differ from requested speed.** Same-Day ACH requests above NACHA's $1,000,000 per-transaction limit are automatically downgraded to `standard`. Always check the `effectiveSpeed` field in the response.

---

## Payout Methods by Region

| Region | Methods |
|--------|---------|
| **United States** | Bank Account (ACH), Debit Card, Venmo, PayPal, RTP |
| **Canada** | Interac, EFT |
| **EU/UK** | IBAN (SEPA) |
| **Brazil** | PIX |
| **Other (100+ countries)** | USDC to Solana wallet |

---

## Apple Pay Payouts

For pushing funds to Apple Pay-enabled cards:

```javascript
const config = {
  buttonSelector: "#apple-pay-disburse",
  merchantIdentifier: "merchant.main.coinflow",
  merchantId: "YOUR_MERCHANT_ID",
  coinflowApiUrl: "https://api-sandbox.coinflow.cash",
  validateMerchantPath: "/api/checkout/apple-pay/validatemerchant",
  label: "Withdrawal",
  totalAmountCents: 1100,
  grossAmountCents: 1200,
  instantFundsOutFeeCents: 100,
  requiredRecipientContactFields: ["name", "postalAddress"],
  onSuccess: (details) => { /* handle Apple Pay token */ },
  onError: (err) => { /* handle error */ }
};
```

**Key requirements:**
- Must validate merchant from your backend
- Line items order: Total Amount → Extra Items → IFO Fee → Disbursement Amount
- Instant Funds Out fee required if `supportsInstantFundsOut` capability enabled
- Browser must support PaymentRequest API

---

## Monitoring

### Check Wallet Balance

```bash
GET /api/merchant/balance
Authorization: YOUR_API_KEY
```

### View Payouts

Dashboard: [sandbox.merchant.coinflow.cash/withdraws](https://sandbox.merchant.coinflow.cash/withdraws)

### Withdrawal History

```bash
GET /api/withdraw/history
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: user123
```

---

## Marketplace Seller Payouts

For marketplace platforms where sellers need to receive payouts:

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
// OR
{"sellerId": "seller-123"}
```

Sellers can withdraw from their own dashboard, or you can initiate payouts via API.

### Seller Payout via API
1. Get wallet balance: `GET /api/withdraw/balances`
2. Add payout destination (bank, card, IBAN, PIX)
3. Get withdraw transaction: `POST /api/withdraw/transaction`
4. Sign and send transaction (wallet signs)

---

## Funding Your Payout Wallet

### Production
1. **USDC on Solana**: Contact Coinflow for wallet address
2. **Wire transfer**: Contact Coinflow for instructions

### Sandbox
Funds are pre-loaded in sandbox for testing.

---

## Common Pitfalls

1. **Forgetting KYC before first payout** — Every user needs KYC once. Call `GET /api/withdraw` to check status before attempting payout.

2. **Not handling 451 response** — Additional verification needed. Redirect user to `verificationLink`.

3. **Using wrong USDC token in quote** — Sandbox and production have different USDC token addresses.

4. **Ignoring `effectiveSpeed`** — Requested speed may be downgraded. Always use the effective speed for user-facing messaging.

5. **Missing required headers** — All payout endpoints need `Authorization`, `x-coinflow-auth-user-id`, and often `x-coinflow-auth-blockchain`.

6. **Not tokenizing debit cards** — PCI compliance requires tokenization. Raw card numbers must never touch your servers.

7. **CVV for debit card payouts** — Unlike checkout, debit card payout tokenization does NOT require CVV. Use `CoinflowCardNumberForm` or `POST /api/tokenize/mobile/card-only`.

8. **24-hour wait for USDC withdrawals** — International users withdrawing USDC to Solana wallet must wait 24 hours after verification before first withdrawal.
