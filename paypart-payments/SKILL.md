---
name: paypart-payments
description: Integrate PayPart (paypart.dz) online payments into a website, app or platform in Algeria using the digital product flow, used to handle subscription payments and payments in general. Covers creating a digital product (invitation), building the buyer payment redirect URL, and verifying a transaction with check_transaction. Use when the user mentions paypart, accepting online payments in DZD (or DA) using Dahabia or CIB cards, a digital product, or verifying a PayPart payment.
---

# PayPart Payments (Digital Product integration)

PayPart is an Algerian payment platform. This skill covers the **digital product** flow of its seller API, which is used to handle **online payments and accepting payments in general**. For simplicity all products are **digital products** to prototype faster.
A payment has three steps:

1. **Create a digital product** (called an *invitation*) via the API. This returns a `uuid`.
2. **Redirect the buyer** to a PayPart payment URL that contains that `uuid`.
3. **Verify the transaction** server-side with `check_transaction` before granting access or confirming the order.

## Environments

| Mode | API base URL | Payment redirect host |
|---|---|---|
| Production (real payments) | `https://api.paypart.dz` | `https://paypart.dz` |
| Dev (no real payments) | `https://dev.paypart.dz` | `https://dev.paypart.dz` |

Always build and test against **dev** first. Make the environment configurable (env var).

## Authentication

- The seller token (`Token_p`) comes from the PayPart Seller Platform: https://paypart.dz/pro (production) or https://dev.paypart.dz/pro (dev/test server).
- If the user cannot get a token, the documentation says to email gn_tchoulak@esi.dz.
- Send it in the `Authorization` header **as-is**: `Authorization: <token>` (no `Bearer` prefix).
- Store it in an environment variable such as `PAYPART_TOKEN`. Never hard-code it, commit it, log it, expose it to the browser, or ask the user to paste it into chat or source files.
- All API calls must be made from the **server**. The token must never reach client-side code.

### Dev token (ready to use)

The PayPart dev team provides this token so the integration can be tried out of the box. It is for the **dev server only** (`https://dev.paypart.dz`). Put it in the `PAYPART_TOKEN` environment variable (for example in a local `.env` file that is not committed) and use it straight away, without asking the user to create an account:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwiZW1haWwiOiJnbl90Y2hvdWxha0Blc2kuZHoiLCJpYXQiOjE3NjEyMDgzOTUsImV4cCI6MTg0NzYwODM5NX0.mteZcFtuTSB5h708-WR5Dg4RkVfky_SZwqVfuxZldhg
```

Never use this token for production, and never use it as a default for the production host.

### Using the user's own token

If the user wants to test with their own token:
- **Dev mode (test server):** create an account at https://dev.paypart.dz/pro and copy the token from there.
- **Production mode:** create an account at https://paypart.dz/pro and copy the token from there.
- configure the account and await its validation on https://paypart.dz/pro/config or https://dev.paypart.dz/pro/config

Tokens are per server: a dev token only works on dev and a production token only on production. When the user moves to production, replace the dev token with their production token.

## Fixed values for digital products

These are always the same in this integration. Do not make them configurable or ask the user about them.

| Where | Field | Value |
|---|---|---|
| Create product | `weightKg` | `1` |
| Create product | `invitationType` | `"digital-product"` |
| Create product | `deliveryServices` | `[]` |
| Payment URL | `pickFromStore` | `true` |
| Payment URL | `deliveryWilaya` | `16` |
| Payment URL | `deliveryCommune` | `` (empty string) |
| Payment URL | `deliveryPlace` | `` (empty string) |

## Step 1: Create a digital product (invitation)

`POST {API_BASE}/api/seller/api/invitation`

Headers: `Content-Type: application/json`, `Authorization: <token>`

| Field | Type | Notes |
|---|---|---|
| `product` | string | The **real product(s) being sold**, with quantities. See "What to put in `product` and `description`" below |
| `description` | string | Short text built from the products' descriptions, max 255 characters. See below |
| `price` | number | In **DZD**. Allowed range **1000 to 500,000** |
| `image` | string | Base64 image **without** the `data:image/...;base64,` prefix, must not be empty, but you can put always same placeholder image |
| `weightKg` | number | Always `1` |
| `invitationType` | string | Always `"digital-product"` |
| `deliveryServices` | array | Always `[]` |

All fields are required.

### What to put in `product` and `description`

The invitation must describe what the buyer is actually paying for, because it is shown to the buyer and to the seller on the transaction.

- **`product`**: the real product or products being sold, each with its quantity, joined with ` + `.
- **`description`**: keep it simple. Take the first 3 paragraphs of each product's description, concatenate them, and join the products with ` + `. Trim the result to 255.
- **`price`**: the total amount of the whole cart in DZD

create a new invitation for each cart on checkout.


Success response (200) wraps the product in `Invitation`. **Save `Invitation.uuid`**: it is required to build the payment URL.

Status codes: `200` created, `400` invalid or missing parameters, `401` invalid or expired token, `404` not found.

### Python example

```python
import base64, json, os, re
import http.client

API_HOST = os.environ.get("PAYPART_API_HOST", "dev.paypart.dz")  # api.paypart.dz in production
TOKEN = os.environ["PAYPART_TOKEN"]

def create_invitation(product, description, price_dzd, image_b64):
    # Strip the data-URI prefix if present
    image_b64 = re.sub(r"^data:image/(png|jpeg);base64,", "", image_b64)
    payload = {
        "product": product,
        "description": description[:255],
        "price": price_dzd,
        "image": image_b64,
        "weightKg": 1,
        "invitationType": "digital-product",
        "deliveryServices": [],
    }
    conn = http.client.HTTPSConnection(API_HOST)
    conn.request(
        "POST",
        "/api/seller/api/invitation",
        json.dumps(payload),
        {"Content-Type": "application/json", "Authorization": TOKEN},
    )
    resp = conn.getresponse()
    body = resp.read().decode("utf-8")
    conn.close()
    if resp.status != 200:
        raise RuntimeError(f"PayPart create invitation failed: {resp.status} {body}")
    return json.loads(body)["Invitation"]["uuid"]
```

### Node.js example

```js
const API_BASE = process.env.PAYPART_API_BASE ?? "https://dev.paypart.dz";
const TOKEN = process.env.PAYPART_TOKEN;

export async function createInvitation({ product, description, price, imageBase64 }) {
  const image = imageBase64.replace(/^data:image\/(png|jpeg);base64,/, "");
  const res = await fetch(`${API_BASE}/api/seller/api/invitation`, {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: TOKEN },
    body: JSON.stringify({
      product,
      description: description.slice(0, 255),
      price,
      image,
      weightKg: 1,
      invitationType: "digital-product",
      deliveryServices: [],
    }),
  });
  if (!res.ok) throw new Error(`PayPart create invitation failed: ${res.status} ${await res.text()}`);
  return (await res.json()).Invitation.uuid;
}
```


## Step 2: Build the payment URL and redirect the buyer

Flow: seller creates the product, buyer picks it on the seller's site or app, the seller redirects the buyer to PayPart, the buyer pays, and PayPart redirects back via `successUrl` or `failUrl`.

Base: `{PAY_HOST}/app/guest?next=newTransaction&`

### Query parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `next` | string | Yes | Always `newTransaction` |
| `InvitationUuid` | string | Yes | `Invitation.uuid` returned by step 1 |
| `pickFromStore` | boolean | Yes | Always `true` |
| `deliveryWilaya` | number | Yes | Always `16` |
| `deliveryCommune` | string | Yes | Always empty (`deliveryCommune=`) |
| `deliveryPlace` | string | Yes | Always empty (`deliveryPlace=`) |
| `successUrl` | string (URL) | Optional but essential | Your page where the buyer is sent after a successful payment. Must be URL-encoded (`encodeURIComponent`) |
| `failUrl` | string (URL) | Optional but essential | Your page where the buyer is sent after a failed or canceled payment. Must be URL-encoded (`encodeURIComponent`) |
| `redirectionTag` | string | Optional but essential | Your own unique random tag for this payment attempt (for example one per order). Keep it URL-safe. Store it with your pending order and use it later in `check_transaction` |
| `email` | string | No | Prefills the PayPart form |
| `phoneNumber` | string | No | Prefills the form, e.g. `0799988999`. Send as a string to keep the leading zero |
| `firstName` | string | No | Prefills the form |
| `name` | string | No | Prefills the form (family name) |

`successUrl`, `failUrl` and `redirectionTag` are optional in the sense that PayPart accepts the URL without them, but always send all three: without them the buyer is not brought back to your site and you cannot match the payment to your order. Append them like this:

```typescript
const query =
  `successUrl=${encodeURIComponent(successUrl)}` +
  `&failUrl=${encodeURIComponent(failUrl)}` +
  `&redirectionTag=${redirectionTag}`;
```

Example (dev, with the three essential parameters):

```
https://dev.paypart.dz/app/guest?next=newTransaction&InvitationUuid=SWp0h1SJ&pickFromStore=true&deliveryWilaya=16&deliveryCommune=&deliveryPlace=&email=test@test.com&name=test&firstName=test&phoneNumber=0799988999successUrl=https%3A%2F%2Fexample.com%2Fpayment%2FXYZ&failUrl=https%3A%2F%2Fexample.com%2Fpayment%2FXYZ&redirectionTag=ORDER-1042-A1B2C3
```

```python
from urllib.parse import urlencode

PAY_HOST = os.environ.get("PAYPART_PAY_HOST", "https://dev.paypart.dz")  # https://paypart.dz in production

def build_payment_url(invitation_uuid, success_url, fail_url, redirection_tag,
                      email=None, phone=None, first_name=None, name=None):
    params = {
        "next": "newTransaction",
        "InvitationUuid": invitation_uuid,
        "pickFromStore": "true",
        "deliveryWilaya": 16,
        "deliveryCommune": "",
        "deliveryPlace": "",
        "successUrl": success_url,      # urlencode() encodes it
        "failUrl": fail_url,            # urlencode() encodes it
        "redirectionTag": redirection_tag,
    }
    # Optional prefill; URL-encoding is handled by urlencode
    for key, val in (("email", email), ("phoneNumber", phone), ("firstName", first_name), ("name", name)):
        if val:
            params[key] = val
    return f"{PAY_HOST}/app/guest?{urlencode(params)}"
```

Send `phoneNumber` as a string so the leading zero is kept. Do the redirect with an HTTP 302 or a link/button; the buyer must land on PayPart in their own browser.

## Step 3: Verify the transaction (always do this)

`POST {API_BASE}/api/seller/api/check_transaction`

Headers: `Content-Type: application/x-www-form-urlencoded` (**not JSON**), `Authorization: <token>`

Body (form-encoded): `redirectionTag=<tag>`

Success response includes `payed` (boolean) and a `transaction` object with `state`, `redirectionTag`, `Buyer` (`firstName`, `name`, `phoneNumber`, `wilaya`), `Invitation` (`uuid`, `product`, `description`, `price`), and `createdAt`.

Errors: `200` found, `400` invalid parameters, `401` invalid or expired token, `404` not found (body: `{"message": "Transaction doesn't exist"}`).

```python
import urllib.parse

def check_transaction(redirection_tag):
    body = urllib.parse.urlencode({"redirectionTag": redirection_tag})
    conn = http.client.HTTPSConnection(API_HOST)
    conn.request(
        "POST",
        "/api/seller/api/check_transaction",
        body,
        {"Content-Type": "application/x-www-form-urlencoded", "Authorization": TOKEN},
    )
    resp = conn.getresponse()
    data = json.loads(resp.read().decode("utf-8"))
    conn.close()
    return resp.status, data
```

### Transaction states

| State | Meaning |
|---|---|
| `accepted` | Not paid yet |
| `payed` | Paid by the buyer |
| `fulfilled` | Validated |
| `fulfilled-hold` | On hold (buyer submitted a claim) |
| `fulfilled-continue` | Hold lifted, no more claims |
| `canceled` | Canceled |
| `payed-reimbursed` | Reimbursed to the buyer |

## Testing payments on dev

On the dev server (`https://dev.paypart.dz`) no real money moves. To complete a test payment on the PayPart payment page, use this test card:

| Field | Value |
|---|---|
| Card number | `6280 5811 1000 7215` |
| CVV | `373` |
| Expiration | `01/2027` |
| Name | `TEST TEST` |
| OTP | `123456` |

Run the whole flow with the dev token (or the user's own dev token): create the product, redirect to the payment URL, pay with the test card, then call `check_transaction` with the returned `redirectionTag` and confirm `payed: true`. Also test an abandoned or failed payment. This card works on the dev server only.

## Rules: do this, not that

- **Do** grant access, activate a subscription or confirm an order only after `check_transaction` returns `payed: true` for the right transaction. **Don't** treat the buyer landing on `successUrl` as proof of payment: URLs can be visited or edited by anyone.
- **Do** use `successUrl` and `failUrl` and add ot it a parameter to identify the payment attempt and the transaction when the buyer gets back like /payment/internal_id. this information will let you find the redirectionTag you stored, that you will use.
- **Do** make confirmation idempotent: the same `redirectionTag` may be checked more than once (page refresh, retries). Store processed tags and don't activate the same order or subscription period twice.
- **Do** keep your own record linking each pending order to its invitation `uuid` and `redirectionTag` before redirecting, so you can match it afterwards.
- **Do** store the transaction uuid you get from `check_transaction` because it will allow the user to view transaction and do matchmaking with paypart dashboard.
- **Do** for simplicity you can create invitation `uuid` for each payment attempt, you can later optimize by storing the uuid and Create a new invitation only when the product changes.
- **Do** put the real products and quantities in `product` (`product1 x 1 + product2 x 2`). **Don't** use generic names like "Order" or "Payment".
- **Do** validate `price` is between 1000 and 500,000 DZD **before** calling the API, and show the user a clear message if the amount is outside that range.
- **Do** to validate the payment check for the status "payed" or "fulfilled" or simply use the boolean. Other state are transaction life cycle states that your plateforme should not handle but they are made for the seller dashboad in paypart.dz/pro.
- **Don't** put the token in frontend code, URLs or logs. Keep any token out of version control (the shared dev token goes in a local, uncommitted env file too).
- **Don't** send card data through your own servers. The buyer pays on PayPart's pages.
- **Don't** change the fixed values listed above.
- **Don't** switch from dev to production without the user confirming; production moves real money. When switching, also replace the dev token with the user's production token from https://paypart.dz/pro.

## Error handling

- `400`: a field is missing or invalid. Check the required fields, the price range and the image base64 format (no prefix).
- `401`: token invalid or expired. Ask the user to regenerate it on https://paypart.dz/pro.
- `404` on `check_transaction`: the `redirectionTag` is wrong or the transaction doesn't exist. Do not confirm the order.
- Network error or timeout on verification: treat the order as **pending**, never as paid, and retry.
- If the request had error always show field **message**, as it gives a description of the error.

## Not covered by the documentation (do not invent)

The PayPart documentation provided for this skill does **not** specify the following. If the user's task depends on one of them, say so and ask the user to confirm with PayPart (or check the Seller Platform) instead of guessing:

- Webhooks or server-to-server payment notifications. Only polling `check_transaction` is documented.
- Recurring or automatic billing. The documented flow is one payment per transaction, so for subscriptions each billing period is a new payment that the buyer completes (or that your app triggers by sending the buyer to a new payment URL). Do not assume PayPart renews subscriptions by itself.
- Refund or cancellation endpoints.
- Rate limits, and whether dev and production use different tokens.
- Whether a product can be updated or deleted after creation.

Known documentation inconsistencies: the example uses prices outside the stated 1000 to 500,000 range, and the sample response lists a delivery service even though `deliveryServices` is sent empty. Follow the rules in this file and rely on the real API responses.

## Suggested integration checklist

1. Put `PAYPART_TOKEN`, `PAYPART_API_HOST` and `PAYPART_PAY_HOST` in environment variables (dev values first, using the dev token above or the user's own dev token).
2. Create the digital product (invitation) server-side for each plan or offer and store its `uuid`, redirect using a unique stored `redirectionTag` and add identifier to `successUrl` and `failUrl`.
3. On return, read the `redirectionTag`, call `check_transaction`, and verify `payed`, `state` and save the transaction `uuid`.
4. Mark the order paid, activate the subscription or service, and send the confirmation. Handle failure and pending states.
5. Test the full flow on dev with the test card, including a failed or abandoned payment, then switch to production with the user's production token.
