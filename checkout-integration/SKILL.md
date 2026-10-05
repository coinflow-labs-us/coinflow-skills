---
name: checkout-integration
description: Coinflow payment checkout — cards, ACH, Apple Pay, Google Pay, SEPA, PIX, USDC, crypto. Covers React SDK, API, checkout links, 3DS, tokenization, saved cards, Card on File, zero auth, and chargeback protection.
---

# Coinflow Checkout Integration

You are an expert at integrating Coinflow payment checkout into web and mobile applications. You know every payment method, edge case, and implementation pattern. When the developer asks you to implement checkout, follow these instructions exactly.

## Environments

| Environment | API Base URL | Checkout UI URL |
|-------------|-------------|-----------------|
| **sandbox** | `https://api-sandbox.coinflow.cash` | `https://sandbox.coinflow.cash` |
| **prod** | `https://api.coinflow.cash` | `https://coinflow.cash` |

Always default to **sandbox** during development. Never use production URLs in example code unless the developer explicitly asks.

## Three Implementation Approaches

Coinflow supports three checkout approaches. Ask the developer which one they want if unclear.

### 1. Checkout Link (Fastest — no frontend code)

Generate a hosted checkout URL and redirect users or embed in an iframe.

```bash
POST /api/checkout/link
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: customer123

{
  "subtotal": {"currency": "USD", "cents": 500},
  "email": "user@gmail.com",
  "blockchain": "solana",
  "settlementType": "USDC",
  "chargebackProtectionData": [
    {"productName": "Widget", "productType": "inGameProduct", "quantity": 1}
  ],
  "allowedPaymentMethods": ["card", "ach", "applePay", "googlePay"]
}
```

Listen for success via postMessage:
```javascript
window.addEventListener('message', event => {
  // Match the exact origin of the checkout link (see the table above).
  if (event.origin !== 'https://coinflow.cash') return;
  if (typeof event.data === 'string') {
    const data = JSON.parse(event.data);
    if (data.data === 'success') {
      console.log('paymentId:', data.info.paymentId);
    }
  }
});
```

**IMPORTANT**: When embedding in an iframe, add `&useHeightChange=true` to the URL and listen for height change messages to avoid scroll issues:
```javascript
window.addEventListener('message', ({data, origin}) => {
  // Match the exact origin of the iframe you embed (see the table above);
  // a substring check like origin.includes('coinflow.cash') is bypassable.
  if (origin !== 'https://coinflow.cash') return;
  try {
    const message = JSON.parse(data);
    if (message.method === 'heightChange') setHeight(message.data);
  } catch (e) {}
});
```

### 2. React SDK (Pre-built UI)

Coinflow provides official frontend SDKs:

| Framework | Package |
|-----------|---------|
| React | `@coinflowlabs/react` |
| React Native | `@coinflowlabs/react-native` |
| Vue | `@coinflowlabs/vue` |
| Angular | `@coinflowlabs/angular` |

```bash
npm install @coinflowlabs/react
```

```jsx
import {CoinflowPurchase} from '@coinflowlabs/react';

<CoinflowPurchase
  wallet={wallet}
  merchantId="your-merchant-id"
  blockchain="solana"
  connection={connection}
  env="sandbox"
  subtotal={{cents: 500, currency: "USD"}}
  settlementType="USDC"
  onSuccess={({paymentId}) => console.log('Paid:', paymentId)}
  chargebackProtectionData={[
    {productName: "Widget", productType: "inGameProduct", quantity: 1}
  ]}
/>
```

**Dynamic height**: Pass `handleHeightChange` to avoid hardcoded heights:
```jsx
const [height, setHeight] = useState(0);
const handleHeight = useCallback((newHeight) => setHeight(Number(newHeight)), []);

<div style={{height: `${height}px`}}>
  <CoinflowPurchase handleHeightChange={handleHeight} {...props} />
</div>
```

### 3. API Integration (Full control, custom UI)

This is a multi-step process. See the detailed flow in the "Card Checkout" section below.

---

## Payment Methods

Coinflow supports many payment methods. Each has specific requirements and edge cases.

### Credit/Debit Cards

**Supported globally** (except restricted countries). Brands: Visa, Mastercard, Discover, Amex.

**Required fields for card checkout:**
- `cardToken` (from tokenization — never raw card numbers)
- `expYear` (2-digit: "30")
- `expMonth` (2-digit: "10")
- `email`, `firstName`, `lastName`
- `address1`, `city`, `zip`, `state`, `country`

**Card Tokenization** — You MUST tokenize cards. Never send raw card numbers to your server.

Option A — CoinflowCardForm component (recommended):
```jsx
import {CoinflowCardForm} from '@coinflowlabs/react';

const cardFormRef = useRef(null);

<CoinflowCardForm
  ref={cardFormRef}
  merchantId="your-merchant-id"
  env="sandbox"
  theme={{font: 'Inter', style: 'rounded'}}
  onLoad={() => console.log('Card form ready')}
/>

// On submit:
const {token, expMonth, expYear} = await cardFormRef.current.tokenize();
```

Option B — API tokenization (requires PCI DSS certification):
```bash
# Step 1: Get tokenization config
GET /api/checkout/mobile-tokenize-config
Authorization: YOUR_API_KEY
# Returns: tokenization config (Timestamp, IDs, keys)

# Step 2: Tokenize using the config values from Step 1
POST /api/tokenize/mobile/card-and-cvv
{
  "data": "4111111111111111",
  "cvv": "123",
  ...configFromStep1
}
```

**CRITICAL — Token lifecycle:**
- Tokens expire after 7 days if unused
- Once used, tokens are valid for 5 minutes (sandbox: indefinite)
- CVV associations are **single-use only** — must re-associate for each transaction

**Card Checkout API flow:**

```bash
# 1. Get session key
GET /api/auth/session-key
Authorization: YOUR_API_KEY
x-coinflow-auth-user-id: customer123
# Returns: {"key": "JWT_TOKEN"} — valid 30 minutes

# 2. Get fee totals
POST /api/checkout/totals/YOUR_MERCHANT_ID
x-coinflow-auth-session-key: JWT_TOKEN
{"subtotal": {"cents": 500}, "settlementType": "USDC"}

# 3. Tokenize card (see above)

# 4. Checkout
POST /api/checkout/card/YOUR_MERCHANT_ID
x-coinflow-auth-session-key: JWT_TOKEN
x-device-id: DEVICE_FINGERPRINT_ID
x-coinflow-client-ip: CUSTOMER_IP
{
  "subtotal": {"currency": "USD", "cents": 500},
  "card": {
    "cardToken": "411111YJM5TX1111",
    "expYear": "30", "expMonth": "10",
    "email": "user@example.com",
    "firstName": "John", "lastName": "Doe",
    "address1": "123 Main St", "city": "Chicago",
    "zip": "60601", "state": "IL", "country": "US"
  },
  "settlementType": "USDC",
  "chargebackProtectionData": [...]
}
# Returns: {"paymentId": "uuid"}
```

### ACH (US Only)

- Bank transfers via secure bank linking
- Settles in ~3 business days
- Lower fees than cards
- **ACH error codes to handle**: R01 (Insufficient Funds), R02 (Account Closed), R03 (No Account), R04 (Invalid Account Number), R05 (Unauthorized), R07 (Auth Revoked), R08 (Payment Stopped), R09 (Uncollected Funds)

### Apple Pay

**Requirements:**
- iOS 17+ and macOS 14+ minimum
- Domain verification required (`.well-known/apple-developer-merchantid-domain-association`)
- Real credit card required for sandbox testing (NOT charged)
- **Restricted categories**: tobacco, marijuana, firearms, illegal drugs, pornography, non-approved fundraising

```jsx
import {CoinflowApplePayButton} from '@coinflowlabs/react';

<CoinflowApplePayButton
  {...purchaseProps}
  color="black"
  onError={(message) => console.error(message)}
/>
```

### Google Pay

**Requirements:**
- Domain verification with Google
- Mobile apps require Google Pay developer registration
- Real test cards (NOT charged on sandbox)
- BillingAddressParameters required for AVS

```jsx
import {CoinflowGooglePayButton} from '@coinflowlabs/react';

<CoinflowGooglePayButton
  {...purchaseProps}
  color="black"
  onError={(message) => console.error(message)}
/>
```

### SEPA & UK Faster Payments

- **SEPA**: Euro-denominated IBANs only, SEPA member countries
- **UK Faster Payments**: GBP, UK banks only
- Both settle **instantly**
- Higher transaction limits, lower fees than cards
- Presentment currency shown to customers in native currency

### PIX (Brazil)

- QR code based, instant settlement
- Lowest fees of all methods
- Nearly all Brazilian banks supported

### USDC / Crypto

- User must have USDC in wallet
- Settles instantly
- No payer wallet needed for fiat methods

### Enabling/restricting payment methods

Use `allowedPaymentMethods` to control which methods are shown:
```javascript
allowedPaymentMethods: ['card', 'ach', 'applePay', 'googlePay', 'usdc', 'sepa', 'fasterPayments', 'pix', 'crypto', 'wire', 'cashApp', 'credits']
```

---

## 3D Secure (3DS)

3DS adds cardholder authentication. The bank decides whether to require it.

**Three outcomes:**
1. **Frictionless** — Bank authenticates silently. No user action needed.
2. **Friction Challenge** — User must complete OTP, biometric, or knowledge-based auth.
3. **Rejection** — Bank rejects the transaction. Do NOT proceed.

**CRITICAL**: If using the API (not React SDK), you MUST handle 3DS challenges yourself.

### Handling 3DS via API

The checkout endpoint returns HTTP **412** when a challenge is required:

```javascript
try {
  const response = await axios.post(
    `${API_URL}/api/checkout/card/${merchantId}`,
    {
      subtotal: {cents: 198},
      authentication3DS: {
        colorDepth: window.screen.colorDepth,
        screenHeight: window.screen.height,
        screenWidth: window.screen.width,
        timeZone: -new Date().getTimezoneOffset(),
      },
      card: {...cardData},
    },
    {headers}
  );
  // Success — no challenge needed
  return response.data.paymentId;
} catch (error) {
  if (error.response?.status === 412) {
    const {transactionId, creq, url} = error.response.data;
    // Show challenge modal (see below)
  }
}
```

**Challenge modal implementation:**
```jsx
// challenge_success is posted from either the UI or the API origin (see the
// table above); compare exactly, never with a substring check.
const CHALLENGE_ORIGINS = ['https://coinflow.cash', 'https://api.coinflow.cash'];

function ChallengeModal({url, creq, transactionId, onComplete}) {
  useEffect(() => {
    const handler = (event) => {
      if (!CHALLENGE_ORIGINS.includes(event.origin)) return;
      if (event.data === 'challenge_success') onComplete(transactionId);
    };
    window.addEventListener('message', handler);
    return () => window.removeEventListener('message', handler);
  }, [transactionId, onComplete]);

  return (
    <iframe
      style={{width: '100%', height: '100vh', border: 'none', minWidth: '391px'}}
      srcDoc={`<html><body onload="document.challenge.submit()">
        <form method="post" name="challenge" action="${url}">
          <input type="hidden" name="creq" value="${creq}" />
        </form>
      </body></html>`}
    />
  );
}
```

**After challenge completes**, retry the same endpoint with the transactionId:
```javascript
const response = await axios.post(`${API_URL}/api/checkout/card/${merchantId}`, {
  subtotal: {cents: 198}, // MUST match original
  authentication3DS: {transactionId},
  card: {...sameCardData},
});
```

### 3DS in Angular

Same flow, but use `DomSanitizer` for the iframe URL and a service pattern:
```typescript
@Injectable()
export class ThreeDSService {
  get3DSParams() {
    return {
      colorDepth: window.screen.colorDepth,
      screenHeight: window.screen.height,
      screenWidth: window.screen.width,
      timeZone: -new Date().getTimezoneOffset(),
    };
  }
}
```

### 3DS Testing

**By test card:**
| Card | Result |
|------|--------|
| 5204 2477 5000 1471 | Frictionless |
| 4000 0200 0000 0000 | Friction Success |
| 4055 0111 1111 1111 | Friction Failure |
| 4264 2815 1111 2228 | Rejection |
| 2303 7799 5100 0446 | Friction Success |
| 2303 7799 5100 0453 | Friction Failure |
| 2303 7799 5100 0347 | Rejection |

**By amount (simpler):**
- Subtotal ending in `98` → Friction Success
- Subtotal ending in `97` → Friction Failure
- Subtotal ending in `96` → Rejection
- Anything else → Frictionless

---

## Saved Cards & Card on File

### Saving a card

Cards are saved during initial checkout. The `paymentId` returned becomes the reference.

### Refreshing a saved card token (CVV re-entry)

Before using a saved card, you must refresh the token with a new CVV:

```jsx
import {CoinflowCvvForm} from '@coinflowlabs/react';

const cvvRef = useRef(null);

<CoinflowCvvForm
  ref={cvvRef}
  merchantId="your-merchant-id"
  env="sandbox"
  token="411111YJM5TX1111" // saved card token
/>

// On submit:
const {token} = await cvvRef.current.tokenize();
// Use refreshed token with POST /api/checkout/token/{merchantId}
```

### Card on File (Customer Initiated Transactions)

For repeat purchases where the customer is present but doesn't re-enter card details:

```bash
POST /api/checkout/card-on-file
{
  "subtotal": {"cents": 2500, "currency": "USD"},
  "originalPaymentId": "uuid-from-first-purchase"
  // OR "token": "411111YJM5TX1111"
}
```

**Configuration limits:**
- `maxCount`: Max CoF payments per period
- `period`: Time window in seconds (86400 = 24 hours)
- `maxMultiple`: Max amount as multiplier of original (e.g., 3x)
- `expiration`: Window original payment is valid as reference

**COMMON ERRORS:**
| Status | Error | Cause | Fix |
|--------|-------|-------|-----|
| 403 | Not Enabled | CoF not enabled | Contact Coinflow |
| 400 | Mobile Wallets | Apple/Google Pay as original | Use regular card first |
| 400 | Chained Transactions | CoF payment as original for another CoF | Reference initial CVV payment |
| 410 | Max Payments Reached | `maxCount` exceeded | Show Coinflow UI for re-verification |
| 410 | Original Payment Expired | `expiration` exceeded | Show Coinflow UI for re-verification |
| 410 | Amount Exceeds Limit | > original × `maxMultiple` | Show Coinflow UI at full amount |

### Merchant Initiated Transactions (MIT)

Charges without customer present (usage-based billing, account top-ups):

```bash
POST /api/checkout/merchant-initiated-transaction
{
  "subtotal": {"cents": 5000, "currency": "USD"},
  "originalPaymentId": "uuid-from-zero-auth-or-first-purchase",
  "settlementType": "USDC"
}
```

If 3DS is required: HTTP 412 response. Present 3DS challenge, then retry with `threeDsAuthenticationData`.

---

## Zero Authorization

Validate a card without charging ($0.00 auth). Creates a reference for future Card on File or MIT.

### SDK:
```jsx
<CoinflowPurchase
  {...baseProps}
  zeroAuthorizationConfig={{disableSavedPaymentMethods: true}}
  onSuccess={({paymentId}) => saveReference(paymentId)}
/>
```

Config options:
- `{disableSavedPaymentMethods: true}` — New card only
- `{disableSavedPaymentMethods: false}` — Show saved cards too
- `{cardToken: "token_abc123"}` — Pre-select a specific card

### API:
```bash
POST /api/checkout/zero-authorization/YOUR_MERCHANT_ID
{
  "card": {"cardToken": "6400XXXXXX1111_bt", "expMonth": "12", "expYear": "26", "firstName": "John", "lastName": "Doe", "email": "john@example.com", "address1": "123 Main St", "city": "New York", "state": "NY", "zip": "10001", "country": "US"}
}
# Returns: {"paymentId": "uuid"}
```

The `cardToken` is obtained from the `CoinflowCardNumberInput` component — never send raw card numbers.

---

## Chargeback Protection

**REQUIRED** for most merchants. Provides device fingerprinting and session tracking for fraud detection.

### Setup

Add protection to **every page** (not just checkout):

**React (recommended):**
```jsx
import {CoinflowPurchaseProtection} from '@coinflowlabs/react';

// In your app layout (renders nothing visible):
<CoinflowPurchaseProtection merchantId="your-merchant-id" coinflowEnv="sandbox" />
```

**Non-React:** Contact Coinflow for setup instructions specific to your stack. A script tag integration is available — your Coinflow representative will provide the required configuration values.

### Including protection data in checkout

```javascript
const chargebackProtectionData = [
  {
    productName: 'Premium Subscription',
    productType: 'subscription', // see product types below
    quantity: 1,
    rawProductData: {description: 'Monthly premium access'}
  }
];

// Pass to CoinflowPurchase or include in API request body
```

**Product types:** `inGameProduct`, `gameOfSkill`, `dataStorage`, `computingResources`, `sportsTicket`, `eSportsTicket`, `musicTicket`, `conferenceTicket`, `virtualSportsTicket`, `virtualESportsTicket`, `virtualMusicTicket`, `virtualConferenceTicket`, `alcohol`, `DLC`, `subscription`, `fundACause`, `realEstate`, `computingContract`, `digitalArt`, `topUp`

### Travel bookings

Merchants selling flights and hotel reservations must declare each booking with a travel-specific `itemClass` so fraud detection evaluates travel-specific signals (itinerary, lodging, PNR, trip, and flight details) instead of treating the purchase as a generic product:

| `itemClass` | Use for | Required fields (besides `sellingPrice`) |
| --- | --- | --- |
| `travel` | Combined flight + hotel package | `lodging`, `pnr`, `trip`, `firstFlight` |
| `flightTicket` | Flight ticket on its own | `pnr`, `trip`, `firstFlight` |
| `flightUpgrade` | Upgrade to an existing flight booking | `upgradeType` (`changes` \| `fareIncrease` \| `baggage` \| `seats` \| `insurance` \| `other`; `description` required when `other`), optional `pnr` |
| `lodging` | Hotel reservation on its own | The fields of the `lodging` object below, flattened at the top level of the item |

Example combined package (`itemClass: 'travel'`) — all of `sellingPrice`, `lodging`, `pnr`, `trip`, and `firstFlight` are required:

```javascript
const chargebackProtectionData = [
  {
    itemClass: 'travel',
    sellingPrice: {valueInCurrency: 1250.0, currency: 'USD'},
    lodging: {
      reservationId: 'X36Q9C',
      nights: 3,
      checkInDate: '2026-09-02',
      checkInHour: '15', // '00'-'23'
      hotel: {name: 'The Plaza Hotel', country: 'US', city: 'New York'},
      wereExtrasPurchased: false,
      isRefundable: true,
      highestRoomType: 'Deluxe King',
      numberOfRooms: 1,
      numberOfGuests: 2,
      numberOfKids: 0
    },
    pnr: 'X36Q9C',
    trip: {
      numberOfFlights: 2,
      numberOfCheckedBags: 1,
      firstWayDestination: 'JFK', // three-letter IATA code
      isRoundTrip: true,
      highestClass: 'economy', // 'economy' | 'luxurious' (business/first)
      isFlexibleTicket: false,
      wereExtrasPurchased: false
    },
    firstFlight: {
      numberOfTravelers: 2,
      departureTime: 1788373800000, // milliseconds since unix epoch
      departureAirport: 'SFO', // three-letter IATA code
      duration: 5.5, // hours
      flightNumber: 'AA1234'
    }
  }
];
```

Flight-only bookings (`itemClass: 'flightTicket'`) use the same `pnr`, `trip`, and `firstFlight` shapes without `lodging`. Hotel-only bookings (`itemClass: 'lodging'`) use the `lodging` fields flattened at the top level of the item. Flight upgrades (`itemClass: 'flightUpgrade'`) need only `sellingPrice` and `upgradeType`:

```javascript
{
  itemClass: 'flightUpgrade',
  sellingPrice: {valueInCurrency: 45.0, currency: 'USD'},
  pnr: 'X36Q9C',
  upgradeType: 'seats' // 'changes' | 'fareIncrease' | 'baggage' | 'seats' | 'insurance' | 'other'
}
```

**Pitfall:** omitting `itemClass` on a travel booking makes it score as a generic product, which hurts approval rates for travel transactions.

### Protection headers for API calls

```jsx
import {useCoinflowProtectionHeaders} from '@coinflowlabs/react';

const getHeaders = useCoinflowProtectionHeaders();
const headers = getHeaders(); // {x-device-id: "...", x-session-id: "..."}
```

### React Native setup

Follow the Coinflow React Native setup guide for chargeback protection. Native SDKs are available for both iOS and Android.

Pass `deviceId` to `<CoinflowPurchase>` or include as `x-device-id` header.

---

## Payer Events (Fraud Detection)

Send lifecycle events to improve fraud scoring:

```bash
POST /api/events
{
  "eventType": "SignUp", // SignUp | SignIn | SignInFailure | BuyerChallenge
  "customerId": "user-123",
  "country": "US",
  "username": "therock72",
  "email": "user@example.com",
  "firstName": "Dwayne",
  "lastName": "Johnson"
}
```

---

## Settlement

Where do merchant funds go after a successful payment?

### Option 1: Coinflow In-App Wallet (recommended)
- Simplest setup
- Non-custodial wallet
- Withdraw to bank account anytime
- Configure in Settings > Earnings Settlement Settings

### Option 2: Your Own Wallet (BYO)
- Bring your own crypto wallet (Solana, EVM, Stellar)
- Full custody
- **IMPORTANT**: Do NOT pass the `transaction` prop when settling to BYO wallet

### Option 3: Smart Contract
- Settle directly to on-chain contract
- Requires contract whitelisting by Coinflow

**Settlement types:** `USDC` (instant), `Credits` (in-app credits, 1:1 dollar), `Bank` (ACH to merchant bank)

---

## Currency Presentment

Show prices in the customer's local currency:

```javascript
subtotal: {cents: 200, currency: "USD"},
presentment: "SGD" // Customer sees SGD equivalent
```

**Rule:** Base currency must be USD or same as presentment. You cannot do `{cents: 200, currency: "PHP"}` with `presentment: "SGD"`.

---

## Mobile Implementation

### iOS (Swift/WKWebView)

**CRITICAL:** The WKWebView message handler **MUST be named exactly `"ReactNativeWebView"`**. Without this, the checkout UI will NOT render.

```swift
let userContentController = WKUserContentController()
userContentController.add(context.coordinator, name: "ReactNativeWebView")
configuration.userContentController = userContentController
```

Listen for messages:
```swift
func userContentController(_ userContentController: WKUserContentController,
                           didReceive message: WKScriptMessage) {
    guard let messageString = message.body as? String else { return }
    // "success" or "authDeclined"
}
```

### Mobile Card Tokenization

Native mobile apps on consumer devices are out of PCI scope. Tokenize directly:

```bash
# 1. Get config
GET /api/checkout/mobile-tokenize-config

# 2. Tokenize card + CVV
POST /api/tokenize/mobile/card-and-cvv
{"data": "4111111111111111", "cvv": "123", ...config}

# 3. Use token in checkout
POST /api/checkout/card/YOUR_MERCHANT_ID
```

For saved cards, re-link CVV (single-use):
```bash
# Get config with existing token
GET /api/checkout/mobile-tokenize-config?token=PREVIOUS_TOKEN

# Link CVV
POST /api/tokenize/mobile/cvv-only
```

---

## Auth Only (Authorize without Capture)

```javascript
authOnly: true // in CoinflowPurchase props or API request
```

- Default 7-day window before auto-void
- Use capture endpoint when ready to charge

**PITFALL:** Cannot use `authOnly: true` together with `transaction` prop. Remove one or the other.

---

## Testing

### Test Cards

| Card | Number | CVV |
|------|--------|-----|
| Visa | 4242 4242 4242 4242 | 123 |
| Mastercard | 5555 5555 5555 4444 | 123 |
| Discover | 6011 0100 0000 0003 | 123 |
| Amex | 3782 822463 10005 | 1234 |

**To force a decline:** Use zip code `99999`.

### Production Testing Rules

- Minimum **$2** per transaction
- Max **1 purchase per 15 minutes** with same card
- Max **3 test transactions** per card per day
- **Failure to follow = account suspension**

### Sandbox Limits

- **5 failed checkouts per 10 minutes** rate limit
- Tokens have indefinite validity (vs 5 min in prod)

---

## Common Errors & Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| "User is blocked" | Customer has/had chargeback in Coinflow ecosystem | Cannot override — platform-wide fraud prevention |
| "No Customer Associated with Customer ID" | First-time payer | Expected — record created after first purchase |
| "Invalid merchant configuration - credit settlement not supported" | Buying Credits but not configured | Check wallet headers for correct blockchain |
| "Please sign a message and get a JWT token" | Missing/invalid session key | Call GET /api/auth/session-key, pass as x-coinflow-auth-session-key |
| "Address: call to non-contract" (EVM) | Settling to contract on sandbox | Ensure `transaction.to` is testnet contract address |
| "Insufficient allowance" (EVM) | Non-whitelisted contract | Ensure contract is whitelisted, pass correct address |
| "Transfer amount exceeds allowance" (EVM) | Approval amount too low | Check Coinflow contract approval amount |
| "Payments not configured correctly" | `authOnly: true` with `transaction`, or `transaction` with non-Credits settlement | Remove transaction OR remove authOnly; use Credits settlement with transaction |

---

## Blockchain-Specific Notes

### Solana
- Recommended chain (Coinflow covers gas fees)
- Only USDC movement on-chain; everything else off-chain
- Use `Connection` from `@solana/web3.js`
- Supports versioned transactions via `supportsVersionedTransactions` prop

### EVM (Ethereum, Base, Polygon, Arbitrum)
- Pass `usePermit: true` for ERC-20 permit-based approvals
- Transaction types: `safeMint`, `returned`, `token`, `NormalRedeem`, `DecentRedeem` (cross-chain)
- Testnet contract addresses differ from mainnet — always verify

### Stellar
- Use G-prefixed public key wallet
- Establish USDC trustline before receiving funds
- Transaction format: base64 XDR

---

## Wallet Generation (for payers without wallets)

If your users don't have crypto wallets, generate deterministic ones:

**Solana — from email:**
```javascript
const wallet = await CoinflowUtils.getWalletFromEmail({
  email: 'user@email.com',
  merchantId: 'your-merchant-id',
  env: 'sandbox'
});
```

**Solana — from user ID:**
```javascript
const wallet = await CoinflowUtils.getWalletFromUserId({
  userId: 'user123',
  merchantId: 'your-merchant-id',
  env: 'sandbox'
});
```

**EVM — from user ID:**
```javascript
// This generates a Coinflow customer identifier, NOT a real crypto wallet.
// No private key exists for this address — do not send funds to it.
const hash = crypto.createHash('sha256').update(userId).digest('hex').substring(0, 40);
const pubkey = `0x${hash}`;
```

Or use `sessionKey` authentication (no wallet needed):
```jsx
<CoinflowPurchase sessionKey={jwtToken} merchantId="id" env="sandbox" />
```

---

## Custom Branding

```javascript
theme: {
  font: 'Inter',           // Any Google Font
  fontSize: '14px',
  style: 'rounded',        // 'rounded' | 'sharp' | 'pill'
  primary: '#0066cc',
  background: '#ffffff',
  textColor: '#111827',
  ctaColor: '#0066cc',
  cardNumberPlaceholder: 'Card number',
  cvvPlaceholder: 'CVV',
  expirationPlaceholder: 'MM / YY',
  showCardIcon: true
}
```

---

## Refunds

- Default limit: $5,000 per merchant
- Initiated via dashboard (Payments tab > Select payment > Refund Transaction)
- If settlement has sufficient USDC/credits, refund is direct
- If not, Coinflow covers and invoices end of month

---

## Transaction Stages (Card Payments)

1. **Card Authorization** — Processor validates, shows as pending on statement
2. **Fraud Monitoring** — Fraud detection systems analyze the transaction; credits minted if applicable
3. **Payment Capture** — If fraud check passes, charge finalized. If fails, refund within 1 business day.

---

## Checkout Limits

- Minimum purchase: **$0.50**
- Card: **$5,000/transaction**, **$10,000/card** max
- ACH: **$50,000**
- Higher limits available — contact Coinflow sales

## Fee Calculation

Use `POST /api/checkout/totals/{merchantId}` to calculate fees dynamically. The response breaks down fees by payment method (card, ACH, etc.) including processing fees, chargeback protection fees, and gas fees.
