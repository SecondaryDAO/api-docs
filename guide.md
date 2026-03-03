# SecondaryDAO Trader API Documentation

**Version:** 1.0
**Base URL:** `https://api.secondarydao.com` (production) | `http://localhost:5000` (development)

This document covers everything a trader needs to build applications that interact with SecondaryDAO programmatically: creating API keys, authentication, available endpoints, rate limits, and code examples.

---

## Table of Contents

1. [Getting an API Key](#1-getting-an-api-key)
2. [Authentication](#2-authentication)
3. [How Trading Works](#3-how-trading-works)
4. [Rate Limits](#4-rate-limits)
5. [API Endpoints](#5-api-endpoints)
   - [Market Data (Public)](#51-market-data-public--no-auth-needed)
   - [Portfolio](#52-portfolio)
   - [Orders](#53-orders)
   - [Trading](#54-trading)
   - [Distributions](#55-distributions)
   - [Buyouts](#56-buyouts)
   - [API Key Management](#57-api-key-management)
6. [Scopes Reference](#6-scopes-reference)
7. [Error Codes](#7-error-codes)
8. [Code Examples](#8-code-examples)
9. [Security Best Practices](#9-security-best-practices)
10. [Websocket Events](#10-websocket-events)

---

## 1. Getting an API Key

### Prerequisites

Before you can create an API key, your account must meet these requirements:

1. **Email verified** - Complete email verification during registration
2. **KYC approved** - Complete identity verification through our KYC provider
3. **AML-compliant wallet** - Connect a wallet and pass AML screening

### Creating a Key

Log into the SecondaryDAO web application, navigate to Account Settings, and create an API key. You can also create keys via the API if you have an active JWT session.

When you create a key, you choose:
- **Label** - A name for this key (e.g., "My Trading Bot")
- **Scopes** - What the key can do (see [Scopes Reference](#5-scopes-reference))
- **IP Whitelist** (optional) - Lock the key to specific IP addresses

You will receive:
- **Key ID** (`keyId`) - Your public identifier, like `sd_live_a1b2c3d4e5f6`
- **Secret** (`secret`) - A 96-character hex string

**CRITICAL: The secret is shown exactly ONCE at creation time. Save it immediately. It cannot be retrieved later.** If you lose it, revoke the key and create a new one.

---

## 2. Authentication

### HMAC Request Signing (Recommended)

Every authenticated API request requires three headers and an HMAC-SHA256 signature. The secret **never** leaves your machine -- only the signature is sent over the wire.

| Header | Value | Description |
|--------|-------|-------------|
| `X-SD-KEY` | `sd_live_a1b2c3d4e5f6` | Your API key ID |
| `X-SD-SIGNATURE` | `<64-char hex HMAC>` | HMAC-SHA256 request signature |
| `X-SD-TIMESTAMP` | `1709500000000` | Current Unix timestamp in milliseconds |

**How to compute the signature:**

```
message  = timestamp + METHOD + path + body
signature = HMAC-SHA256(secret, message)
```

Where:
- `timestamp` = value of `X-SD-TIMESTAMP` (string, Unix ms)
- `METHOD` = uppercase HTTP method (`GET`, `POST`, `DELETE`, etc.)
- `path` = full request path including query string (e.g., `/api/portfolio/holdings?walletAddress=0x...`)
- `body` = raw request body as a string (empty string `""` for GET requests)
- `secret` = your 96-character hex API secret

### Timestamp Validation

The `X-SD-TIMESTAMP` must be within **30 seconds** of the server's time. If your clock is out of sync, you'll receive a `401` error with `serverTime` in the response body so you can calculate the offset.

### Example Request

```bash
# Variables
KEY_ID="sd_live_a1b2c3d4e5f6"
SECRET="your96charhexsecrethere..."
TIMESTAMP=$(date +%s000)
METHOD="GET"
PATH="/api/portfolio/holdings?walletAddress=0xYourWallet"
BODY=""

# Compute HMAC signature
MESSAGE="${TIMESTAMP}${METHOD}${PATH}${BODY}"
SIGNATURE=$(echo -n "$MESSAGE" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')

curl -X GET "https://api.secondarydao.com${PATH}" \
  -H "X-SD-KEY: ${KEY_ID}" \
  -H "X-SD-SIGNATURE: ${SIGNATURE}" \
  -H "X-SD-TIMESTAMP: ${TIMESTAMP}" \
  -H "Content-Type: application/json"
```

### Legacy Authentication (Deprecated)

Sending the raw secret via `X-SD-SECRET` header is still supported but **deprecated**. Requests using this method will receive an `X-SD-Deprecation` response header. Migrate to HMAC signing as soon as possible.

| Header | Value | Description |
|--------|-------|-------------|
| `X-SD-KEY` | `sd_live_a1b2c3d4e5f6` | Your API key ID |
| `X-SD-SECRET` | `<96-char hex string>` | Your API secret (deprecated) |
| `X-SD-TIMESTAMP` | `1709500000000` | Current Unix timestamp in milliseconds |

---

## 3. How Trading Works

### Architecture: Off-Chain Matching, On-Chain Settlement

SecondaryDAO uses a **hybrid order book**. Orders are matched off-chain in a centralized matching engine (fast, no gas fees), then settled on-chain by the platform (atomic token/stablecoin swap).

```
1. Bot places order via API  -->  MongoDB (off-chain)
2. Matching engine finds match -->  price-time priority
3. Platform settler executes   -->  on-chain atomic swap
4. Tokens move, USDC moves     -->  in one transaction
```

This means your bot places orders via HTTP and the platform handles all blockchain execution. You do not sign individual trades.

### One-Time On-Chain Setup (Required)

Before your bot can trade, the wallet must authorize the escrow contract to move tokens on its behalf. This is a standard ERC-20 approval -- the same pattern used by Uniswap, Aave, and every DeFi protocol.

**For buying:** Approve USDC (or USDT) to the property's escrow contract.
```javascript
// ethers.js v6
const usdc = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, signer);
await usdc.approve(ESCROW_ADDRESS, ethers.MaxUint256); // or a specific amount
```

**For selling:** Approve your property tokens to the escrow contract.
```javascript
const token = new ethers.Contract(PROPERTY_TOKEN_ADDRESS, ERC20_ABI, signer);
await token.approve(ESCROW_ADDRESS, ethers.MaxUint256);
```

These are one-time transactions per token per escrow contract. After approval, all subsequent trades settle automatically when orders match -- no further wallet interaction needed.

**To find contract addresses:** Use `GET /api/explorer/properties` (public) which returns escrow and token contract addresses for each property.

### What Your Bot Does vs. What the Platform Does

| Step | Who | How |
|------|-----|-----|
| Approve tokens/USDC to escrow | **Your bot** | One-time on-chain tx (ethers.js, web3.js, etc.) |
| Place buy/sell orders | **Your bot** | `POST /api/order-book/create` (HMAC-signed HTTP) |
| Match orders | **Platform** | Automatic, off-chain, price-time priority |
| Execute settlement on-chain | **Platform** | Settler wallet calls `settleTrade()` on escrow contract |
| Transfer tokens + USDC | **Platform** | Atomic swap using your pre-approved allowances |

### IPS Purchases (Initial Property Sale)

During an Initial Property Sale, the flow is different. Your bot signs an ERC-2612 permit locally (your private key never leaves your machine), then sends the signature to the API. The platform relays the purchase transaction gaslessly.

```
1. Bot signs ERC-2612 USDC permit (locally, off-chain)
2. Bot sends permit signature to POST /api/token-purchase/gasless
3. Platform relays the purchase on-chain (platform pays gas)
```

See [Section 5.4 Trading](#54-trading) for the endpoint details.

### Security Model

**Your API key is your trading authority.** Anyone with a valid API key that has `orders:write` scope can place orders against the associated wallet. This is the same security model as Binance, Coinbase Pro, and every exchange API -- the key authorizes trading, and on-chain approvals authorize token movement.

Protect your keys accordingly:
- **Restrict wallet scope** -- lock each key to one specific wallet, not "all wallets"
- **Whitelist IPs** -- if your bot runs on a fixed server, only allow that IP
- **Minimize scopes** -- a read-only dashboard key should never have `orders:write`
- **Set expiration** -- use time-limited keys for development and testing

See [Section 9: Security Best Practices](#9-security-best-practices) for the full list.

---

## 4. Rate Limits

| Category | Limit | Window |
|----------|-------|--------|
| General API calls | 300 requests | 1 minute |
| Order creation | 30 requests | 1 minute |
| Trade execution | 10 requests | 1 minute |

Rate limit information is returned in response headers:
- `RateLimit-Limit` - Maximum requests allowed
- `RateLimit-Remaining` - Requests remaining in current window
- `RateLimit-Reset` - Unix timestamp when the window resets

When you hit a rate limit, you'll receive a `429 Too Many Requests` response.

---

## 5. API Endpoints

### 5.1 Market Data (Public -- No Auth Needed)

These endpoints are publicly accessible. No API key required, but you can use one.

#### List Properties

```
GET /api/properties
```

Returns all available properties with market data.

**Response:**
```json
{
  "success": true,
  "properties": [
    {
      "_id": "64abc123...",
      "name": "Sunset Ridge Apartments",
      "location": "Austin, TX",
      "propertyValue": 2000000,
      "tokenPrice": 2000,
      "totalSupply": 1000,
      "tokenSymbol": "SRAX",
      "status": "trading",
      "bestBid": 1950,
      "bestAsk": 2050,
      "lastTradePrice": 2010
    }
  ]
}
```

#### Get Explorer Properties

```
GET /api/explorer/properties
```

Returns properties visible in the explorer with contract addresses and distribution history.

**Response includes:**
- Property metadata (name, location, value)
- Contract addresses (propertyCore, escrowContract, distributionManager, buyoutRegistry)
- Status (selling, trading, coming_soon, halted)
- Last trade prices

#### Check Trading Status

```
GET /api/trading-status/status/:propertyId
```

Returns the current trading state for a property.

**Response:**
```json
{
  "buttonState": "buy",
  "buttonText": "Buy Tokens",
  "message": "Trading is active",
  "timeRemaining": null
}
```

Possible `buttonState` values: `buy`, `launching`, `coming_soon`, `disabled`, `auto_launch_ready`

#### Get Order Book

```
GET /api/order-book/book/:propertyId?depth=10
```

Returns the current order book (buy and sell sides) for a property.

#### Get Token Statistics

```
GET /api/token-stats/total-supply
GET /api/token-stats/total-balance
```

#### Get Merkle Proof

```
GET /api/merkle/proof/:walletAddress
```

Returns the merkle proof needed for on-chain token purchases.

**Response:**
```json
{
  "address": "0x1234...",
  "isWhitelisted": true,
  "root": "0xabc...",
  "proof": ["0x...", "0x...", "0x..."]
}
```

#### Get Active Buyout Offers

```
GET /api/buyout/active-offers/:propertyId
```

Returns active buyout offers for a property.

---

### 5.2 Portfolio

**Required scope:** `portfolio:read`

#### Get Portfolio Summary

```
GET /api/portfolio/summary?walletAddress=0xYourWallet
```

Returns a cached portfolio summary for the given wallet.

#### Get Holdings

```
GET /api/portfolio/holdings?walletAddress=0xYourWallet
```

**Response:**
```json
{
  "success": true,
  "holdings": [
    {
      "propertyAddress": "0xabc...",
      "propertyName": "Sunset Ridge Apartments",
      "tokenSymbol": "SRAX",
      "tokenAmount": "50",
      "currentPrice": 2010,
      "usdValue": 100500,
      "percentageOwnership": 5.0
    }
  ]
}
```

#### Get Distribution History

```
GET /api/portfolio/distributions?walletAddress=0xYourWallet
```

Returns distribution payments received by the wallet.

#### Get Performance Data

```
GET /api/portfolio/performance?walletAddress=0xYourWallet&period=30d
```

Returns performance chart data over a time period.

#### Get Trade History

```
GET /api/portfolio/trades?walletAddress=0xYourWallet
```

Returns past trades for the wallet.

#### Get Income Summary

```
GET /api/portfolio/income?walletAddress=0xYourWallet
```

Returns rental income breakdown.

---

### 5.3 Orders

#### List Orders

**Required scope:** `orders:read`

```
GET /api/trading/orders
GET /api/trading/orders/buy
GET /api/trading/orders/sell
```

Query parameters: `type`, `status`, `propertyId`

#### List My Orders

**Required scope:** `orders:read`

```
GET /api/order-book/my-orders?status=open&propertyId=xxx&page=1&limit=20
```

Returns orders created by the authenticated user.

#### Create Order

**Required scope:** `orders:write`

```
POST /api/order-book/create
```

**Request body:**
```json
{
  "propertyId": "64abc123...",
  "orderType": "buy",
  "executionType": "limit",
  "tokenAmount": 10,
  "pricePerToken": 2000,
  "walletAddress": "0xYourWallet",
  "timeInForce": "GTC",
  "expirationHours": 168,
  "conditions": {
    "minFillAmount": 1,
    "maxSlippage": 2.0,
    "postOnly": false
  }
}
```

**Fields:**
| Field | Required | Description |
|-------|----------|-------------|
| `propertyId` | Yes | MongoDB ID of the property |
| `orderType` | Yes | `buy` or `sell` |
| `executionType` | Yes | `market` or `limit` |
| `tokenAmount` | Yes | Number of tokens |
| `pricePerToken` | Yes (limit) | Price per token in USD |
| `walletAddress` | Yes | Your connected wallet address |
| `timeInForce` | No | `GTC` (default), `IOC`, `FOK`, `GTD` |
| `expirationHours` | No | 1-720 hours (max 30 days) |
| `conditions.minFillAmount` | No | Minimum partial fill amount |
| `conditions.maxSlippage` | No | Max slippage % (0-50) |
| `conditions.postOnly` | No | Only place if maker (no taker fill) |

**Time in Force options:**
- `GTC` - Good Till Cancelled (stays until filled or cancelled)
- `IOC` - Immediate Or Cancel (fill what you can, cancel rest)
- `FOK` - Fill Or Kill (all or nothing, immediately)
- `GTD` - Good Till Date (expires at expirationHours)

**Response:**
```json
{
  "success": true,
  "message": "Order created successfully",
  "order": {
    "_id": "...",
    "orderId": "ORD-...",
    "propertyId": "...",
    "orderType": "buy",
    "tokenAmount": 10,
    "pricePerToken": 2000,
    "status": "open",
    "createdAt": "2026-03-03T..."
  }
}
```

---

### 5.4 Trading

#### Gasless Token Purchase (IPS)

**Required scope:** `trading:execute`

```
POST /api/token-purchase/gasless
```

Purchase tokens during an Initial Property Sale using ERC-2612 permit signatures. The platform pays gas.

**Request body:**
```json
{
  "propertyId": "64abc123...",
  "amount": "100000000",
  "isUSDT": false,
  "buyerAddress": "0xYourWallet",
  "signature": {
    "v": 28,
    "r": "0x...",
    "s": "0x...",
    "deadline": 1709500000
  }
}
```

**Fields:**
| Field | Description |
|-------|-------------|
| `propertyId` | MongoDB ID of the property |
| `amount` | Amount in stablecoin units (6 decimals, so 100 USDC = `100000000`) |
| `isUSDT` | `false` for USDC, `true` for USDT |
| `buyerAddress` | Your wallet address |
| `signature` | ERC-2612 permit signature (v, r, s, deadline) |

**Note:** Limited to 10 gasless purchases per day per wallet.

---

### 5.5 Distributions

**Required scope:** `distributions:read`

Rent distributions are currently **auto-pushed** to all token holders on-chain via the TrustlessDistributionManager. No action is required from your bot -- earnings arrive directly in your wallet. Use these endpoints to track distribution history and accrued amounts.

#### Get Unclaimed Earnings

```
GET /api/distributions/unclaimed/:walletAddress
```

**Response:**
```json
{
  "success": true,
  "totalUnclaimed": 150.50,
  "byProperty": [
    {
      "propertyId": "...",
      "propertyName": "Sunset Ridge",
      "unclaimed": 150.50
    }
  ]
}
```

#### Get Claim History

```
GET /api/distributions/history/:walletAddress
```

Returns past distribution payments received by the wallet.

#### Future: Gasless Claims

A gas-efficient claim-based distribution system (MerklePayoutDistributor) is built but not yet active. When enabled, it will allow bots to claim accrued earnings via EIP-712 signed gasless claims through the API. The endpoints (`/api/distributions/claim-data`, `/api/distributions/gasless-claim`, `/api/distributions/record-claim`) will be documented here once activated.

---

### 5.6 Buyouts

**Required scope:** `buyout:read`

#### Check Buyout Approval

```
GET /api/buyout/check-approval/:propertyId
```

Check if the authenticated user is approved for buyout offers.

#### Get Offer Requirements

```
GET /api/buyout/offer-requirements/:propertyId?walletAddress=0xYourWallet
```

Returns the minimum token amount, premium percentages, and fees required for a buyout offer.

#### Get My Token Position

```
GET /api/buyout/my-position/:propertyId
```

Returns the authenticated user's token balance and ownership percentage for a property.

---

### 5.7 API Key Management

**Requires JWT authentication (web login), not API key.**

These endpoints are for managing your API keys through the web UI or authenticated session.

#### Create API Key

```
POST /api/api-keys
```

**Request body:**
```json
{
  "label": "My Trading Bot",
  "scopes": ["portfolio:read", "orders:read", "orders:write"],
  "allowedIPs": ["203.0.113.50"],
  "expiresIn": 8760
}
```

**Response (SAVE THE SECRET!):**
```json
{
  "success": true,
  "message": "API key created. Save the secret -- it will not be shown again.",
  "keyId": "sd_live_a1b2c3d4e5f6",
  "secret": "a1b2c3d4e5f6...96 chars total...",
  "label": "My Trading Bot",
  "scopes": ["portfolio:read", "orders:read", "orders:write"],
  "allowedIPs": ["203.0.113.50"],
  "expiresAt": "2027-03-03T00:00:00.000Z",
  "createdAt": "2026-03-03T00:00:00.000Z"
}
```

#### List API Keys

```
GET /api/api-keys
```

Returns all your API keys (never includes secrets).

#### Revoke API Key

```
DELETE /api/api-keys/:keyId
```

Revokes the key immediately. Any in-flight requests with this key will fail.

#### Update API Key

```
PATCH /api/api-keys/:keyId
```

Update label or IP whitelist. Scopes cannot be changed after creation.

```json
{
  "label": "Updated Label",
  "allowedIPs": ["203.0.113.50", "198.51.100.0/24"]
}
```

---

## 6. Scopes Reference

| Scope | Access |
|-------|--------|
| `market:read` | Browse properties, trading status, token stats |
| `portfolio:read` | View holdings, distributions, performance, trades, income |
| `orders:read` | View open orders and order history |
| `orders:write` | Create and cancel buy/sell orders |
| `trading:execute` | Execute gasless token purchases |
| `distributions:read` | View unclaimed earnings, claim proofs, claim history, record claims |
| `buyout:read` | View buyout offers, approval status, position, requirements |

**Recommended scope combinations:**

| Use Case | Scopes |
|----------|--------|
| Read-only portfolio tracker | `portfolio:read`, `distributions:read` |
| Market data monitor | `market:read` |
| Trading bot | `orders:read`, `orders:write`, `market:read` |
| Full access bot | All scopes |

---

## 7. Error Codes

### HTTP Status Codes

| Status | Meaning |
|--------|---------|
| `200` | Success |
| `201` | Created (new resource) |
| `400` | Bad request (validation error) |
| `401` | Authentication failed (invalid key, expired timestamp) |
| `403` | Forbidden (insufficient scope, IP blocked, KYC/AML issue) |
| `404` | Resource not found |
| `429` | Rate limit exceeded |
| `500` | Internal server error |

### Application Error Codes

| Code | Description |
|------|-------------|
| `NO_AUTH_TOKEN` | Missing authentication headers |
| `ACCOUNT_SUSPENDED` | Account is suspended from trading |
| `KYC_REQUIRED` | KYC approval needed |
| `WHITELIST_REQUIRED` | AML/whitelist approval needed |
| `WALLET_BLACKLISTED` | Wallet is blocked for compliance |
| `API_KEY_RATE_LIMIT_EXCEEDED` | API key rate limit hit |
| `API_KEY_ORDER_RATE_LIMIT_EXCEEDED` | Order creation rate limit hit |
| `API_KEY_TRADE_RATE_LIMIT_EXCEEDED` | Trade execution rate limit hit |

---

## 8. Code Examples

### Python

```python
import requests
import time
import hmac
import hashlib
import json

class SecondaryDAOClient:
    def __init__(self, base_url, key_id, secret):
        self.base_url = base_url.rstrip('/')
        self.key_id = key_id
        self.secret = secret

    def _sign(self, method, path, body=''):
        """Compute HMAC-SHA256 request signature"""
        timestamp = str(int(time.time() * 1000))
        message = timestamp + method.upper() + path + body
        signature = hmac.new(
            self.secret.encode(),
            message.encode(),
            hashlib.sha256
        ).hexdigest()
        return {
            'X-SD-KEY': self.key_id,
            'X-SD-SIGNATURE': signature,
            'X-SD-TIMESTAMP': timestamp,
            'Content-Type': 'application/json'
        }

    def get_properties(self):
        """List all available properties (public, no auth needed)"""
        r = requests.get(f'{self.base_url}/api/properties')
        r.raise_for_status()
        return r.json()

    def get_holdings(self, wallet_address):
        """Get token holdings for a wallet"""
        path = f'/api/portfolio/holdings?walletAddress={wallet_address}'
        r = requests.get(
            f'{self.base_url}{path}',
            headers=self._sign('GET', path)
        )
        r.raise_for_status()
        return r.json()

    def get_order_book(self, property_id, depth=10):
        """Get the order book for a property (public)"""
        r = requests.get(
            f'{self.base_url}/api/order-book/book/{property_id}',
            params={'depth': depth}
        )
        r.raise_for_status()
        return r.json()

    def get_my_orders(self, status=None, property_id=None):
        """Get your open orders"""
        params = []
        if status:
            params.append(f'status={status}')
        if property_id:
            params.append(f'propertyId={property_id}')
        qs = '?' + '&'.join(params) if params else ''
        path = f'/api/order-book/my-orders{qs}'
        r = requests.get(
            f'{self.base_url}{path}',
            headers=self._sign('GET', path)
        )
        r.raise_for_status()
        return r.json()

    def create_order(self, property_id, order_type, token_amount,
                     price_per_token, wallet_address,
                     execution_type='limit', time_in_force='GTC',
                     expiration_hours=168):
        """Create a buy or sell order"""
        body = json.dumps({
            'propertyId': property_id,
            'orderType': order_type,
            'executionType': execution_type,
            'tokenAmount': token_amount,
            'pricePerToken': price_per_token,
            'walletAddress': wallet_address,
            'timeInForce': time_in_force,
            'expirationHours': expiration_hours
        })
        path = '/api/order-book/create'
        r = requests.post(
            f'{self.base_url}{path}',
            data=body,
            headers=self._sign('POST', path, body)
        )
        r.raise_for_status()
        return r.json()

    def get_unclaimed_distributions(self, wallet_address):
        """Check unclaimed distribution earnings"""
        path = f'/api/distributions/unclaimed/{wallet_address}'
        r = requests.get(
            f'{self.base_url}{path}',
            headers=self._sign('GET', path)
        )
        r.raise_for_status()
        return r.json()

    def get_trading_status(self, property_id):
        """Check if a property is currently trading (public)"""
        r = requests.get(
            f'{self.base_url}/api/trading-status/status/{property_id}'
        )
        r.raise_for_status()
        return r.json()


# Usage
client = SecondaryDAOClient(
    base_url='https://api.secondarydao.com',
    key_id='sd_live_a1b2c3d4e5f6',
    secret='your96charhexsecrethere...'
)

# Browse properties (no auth needed)
properties = client.get_properties()
for prop in properties.get('properties', []):
    print(f"{prop['name']}: ${prop['tokenPrice']}/token, "
          f"status={prop['status']}")

# Check holdings (HMAC-signed)
holdings = client.get_holdings('0xYourWalletAddress')
for h in holdings.get('holdings', []):
    print(f"{h['propertyName']}: {h['tokenAmount']} tokens "
          f"(${h['usdValue']})")

# Place a limit buy order (HMAC-signed)
order = client.create_order(
    property_id='64abc123...',
    order_type='buy',
    token_amount=5,
    price_per_token=1950,
    wallet_address='0xYourWalletAddress',
    time_in_force='GTC',
    expiration_hours=72
)
print(f"Order created: {order['order']['orderId']}")
```

### JavaScript / Node.js

```javascript
const axios = require('axios');
const crypto = require('crypto');

class SecondaryDAOClient {
  constructor(baseUrl, keyId, secret) {
    this.baseUrl = baseUrl.replace(/\/$/, '');
    this.keyId = keyId;
    this.secret = secret;
  }

  /**
   * Compute HMAC-SHA256 signature for a request.
   * message = timestamp + METHOD + path + body
   */
  _sign(method, path, body = '') {
    const timestamp = String(Date.now());
    const message = timestamp + method.toUpperCase() + path + body;
    const signature = crypto
      .createHmac('sha256', this.secret)
      .update(message)
      .digest('hex');
    return {
      'X-SD-KEY': this.keyId,
      'X-SD-SIGNATURE': signature,
      'X-SD-TIMESTAMP': timestamp,
      'Content-Type': 'application/json'
    };
  }

  async getProperties() {
    const { data } = await axios.get(`${this.baseUrl}/api/properties`);
    return data;
  }

  async getHoldings(walletAddress) {
    const path = `/api/portfolio/holdings?walletAddress=${walletAddress}`;
    const { data } = await axios.get(
      `${this.baseUrl}${path}`,
      { headers: this._sign('GET', path) }
    );
    return data;
  }

  async getOrderBook(propertyId, depth = 10) {
    const { data } = await axios.get(
      `${this.baseUrl}/api/order-book/book/${propertyId}`,
      { params: { depth } }
    );
    return data;
  }

  async getMyOrders(params = {}) {
    const qs = new URLSearchParams(params).toString();
    const path = `/api/order-book/my-orders${qs ? '?' + qs : ''}`;
    const { data } = await axios.get(
      `${this.baseUrl}${path}`,
      { headers: this._sign('GET', path) }
    );
    return data;
  }

  async createOrder({ propertyId, orderType, tokenAmount, pricePerToken,
                      walletAddress, executionType = 'limit',
                      timeInForce = 'GTC', expirationHours = 168 }) {
    const path = '/api/order-book/create';
    const body = JSON.stringify({
      propertyId, orderType, executionType,
      tokenAmount, pricePerToken, walletAddress,
      timeInForce, expirationHours
    });
    const { data } = await axios.post(
      `${this.baseUrl}${path}`,
      body,
      { headers: this._sign('POST', path, body) }
    );
    return data;
  }

  async getUnclaimedDistributions(walletAddress) {
    const path = `/api/distributions/unclaimed/${walletAddress}`;
    const { data } = await axios.get(
      `${this.baseUrl}${path}`,
      { headers: this._sign('GET', path) }
    );
    return data;
  }

  async getTradingStatus(propertyId) {
    const { data } = await axios.get(
      `${this.baseUrl}/api/trading-status/status/${propertyId}`
    );
    return data;
  }
}

// Usage
const client = new SecondaryDAOClient(
  'https://api.secondarydao.com',
  'sd_live_a1b2c3d4e5f6',
  'your96charhexsecrethere...'
);

(async () => {
  // Browse properties (no auth needed)
  const { properties } = await client.getProperties();
  properties.forEach(p =>
    console.log(`${p.name}: $${p.tokenPrice}/token, status=${p.status}`)
  );

  // Check holdings (HMAC-signed)
  const { holdings } = await client.getHoldings('0xYourWallet');
  holdings.forEach(h =>
    console.log(`${h.propertyName}: ${h.tokenAmount} tokens ($${h.usdValue})`)
  );

  // Place a limit buy order (HMAC-signed)
  const order = await client.createOrder({
    propertyId: '64abc123...',
    orderType: 'buy',
    tokenAmount: 5,
    pricePerToken: 1950,
    walletAddress: '0xYourWallet'
  });
  console.log(`Order created: ${order.order.orderId}`);
})();
```

---

## 9. Security Best Practices

### API Key Security

Your API key is your trading authority. Anyone with a valid `orders:write` key can place trades against the associated wallet. Treat keys like exchange API credentials.

1. **Use HMAC signing.** Always use `X-SD-SIGNATURE` (HMAC) instead of `X-SD-SECRET` (legacy). HMAC ensures your secret never leaves your machine and protects against request body tampering.

2. **Never share your API secret.** Treat it like a private key. If compromised, revoke immediately.

3. **Restrict wallet scope.** When creating a key, lock it to one specific wallet address instead of "all wallets." This limits the blast radius if a key is compromised.

4. **Use IP whitelisting.** If your bot runs from a fixed server, whitelist that IP address. A stolen key is useless if the attacker can't reach the API from the whitelisted IP.

5. **Use minimum required scopes.** A portfolio tracker only needs `portfolio:read`. A trading bot needs `orders:read` + `orders:write`. Don't grant `trading:execute` unless you need gasless IPS purchases.

6. **Set key expiration.** Use `expiresIn` for development/testing keys. For production bots, rotate keys on a regular schedule.

7. **Store secrets in environment variables.** Never hardcode secrets in source code or commit them to version control.

8. **Monitor usage.** Use `GET /api/api-keys` to check `lastUsedAt` and `totalRequests` for each key. Unexpected activity means a compromised key.

9. **Rotate keys periodically.** Create a new key, update your bot, then revoke the old key.

### On-Chain Approval Safety

10. **Approve specific amounts, not unlimited.** Instead of `ethers.MaxUint256`, approve only the amount you plan to trade. This limits exposure if the escrow contract is ever compromised.

11. **Revoke approvals when done.** If you stop trading a property, set the approval back to zero:
    ```javascript
    await usdc.approve(ESCROW_ADDRESS, 0);
    ```

12. **Verify contract addresses.** Always fetch escrow addresses from `GET /api/explorer/properties` rather than hardcoding them. Confirm they match on-chain before approving.

### Auto-Revocation Triggers

Your API keys will be automatically revoked if:
- You change your password
- Your AML status changes to 'failed'
- Your account is suspended
- An admin revokes your access

---

## 10. Websocket Events

The platform uses WebSocket connections for real-time updates. WebSocket access is available through the web application. API key users should poll the REST endpoints for updates.

**Recommended polling intervals:**
- Order book: Every 5-10 seconds
- Holdings: Every 30-60 seconds
- Distributions: Every 5 minutes
- Trading status: Every 30 seconds

---

## Appendix: Quick Reference

### Authentication Headers (HMAC -- Recommended)

```
X-SD-KEY: sd_live_<your_key_id>
X-SD-SIGNATURE: HMAC-SHA256(secret, timestamp + METHOD + path + body)
X-SD-TIMESTAMP: <unix_milliseconds>
```

### Authentication Headers (Legacy -- Deprecated)

```
X-SD-KEY: sd_live_<your_key_id>
X-SD-SECRET: <your_96_char_secret>
X-SD-TIMESTAMP: <unix_milliseconds>
```

### Most Common Endpoints

| Action | Method | Endpoint | Scope |
|--------|--------|----------|-------|
| Browse properties | GET | `/api/properties` | None (public) |
| View holdings | GET | `/api/portfolio/holdings?walletAddress=0x...` | `portfolio:read` |
| View orders | GET | `/api/order-book/my-orders` | `orders:read` |
| Place order | POST | `/api/order-book/create` | `orders:write` |
| Check distributions | GET | `/api/distributions/unclaimed/0x...` | `distributions:read` |
| Order book depth | GET | `/api/order-book/book/:propertyId` | None (public) |
| Trading status | GET | `/api/trading-status/status/:propertyId` | None (public) |
| Merkle proof | GET | `/api/merkle/proof/:address` | None (public) |

### Complete Endpoint Map

**Public (no auth):**
- `GET /api/properties`
- `GET /api/explorer/properties`
- `GET /api/trading-status/status/:propertyId`
- `GET /api/token-stats/total-supply`
- `GET /api/token-stats/total-balance`
- `GET /api/merkle/proof/:address`
- `GET /api/buyout/active-offers/:propertyId`
- `GET /api/buyout/premium-vote-status/:propertyId`
- `GET /api/buyout/voter-status/:propertyId/:walletAddress`
- `GET /api/buyout/property-financials/:propertyId`
- `GET /api/order-book/book/:propertyId`

**Authenticated (API key required):**
- `GET /api/portfolio/summary` - `portfolio:read`
- `GET /api/portfolio/holdings` - `portfolio:read`
- `GET /api/portfolio/distributions` - `portfolio:read`
- `GET /api/portfolio/performance` - `portfolio:read`
- `GET /api/portfolio/trades` - `portfolio:read`
- `GET /api/portfolio/income` - `portfolio:read`
- `GET /api/trading/orders` - `orders:read`
- `GET /api/trading/orders/buy` - `orders:read`
- `GET /api/trading/orders/sell` - `orders:read`
- `GET /api/order-book/my-orders` - `orders:read`
- `POST /api/order-book/create` - `orders:write`
- `POST /api/token-purchase/gasless` - `trading:execute`
- `GET /api/distributions/unclaimed/:walletAddress` - `distributions:read`
- `GET /api/distributions/history/:walletAddress` - `distributions:read`
- `GET /api/buyout/check-approval/:propertyId` - `buyout:read`
- `GET /api/buyout/offer-requirements/:propertyId` - `buyout:read`
- `GET /api/buyout/my-position/:propertyId` - `buyout:read`

---

## Support

If you encounter issues with the API:
1. Check the error response body for specific error codes and messages
2. Verify your API key is active using `GET /api/api-keys` (via web session)
3. Ensure your clock is synchronized (timestamp within 30 seconds)
4. Check rate limit headers to see if you're being throttled
