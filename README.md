> **Writing sample by Saransh Rayguru.** Northwind Pay is a made-up company and this API does not exist. I wrote it to show what my developer docs read like.

# Northwind Pay API: take your first test payment

Taking a payment with the Northwind Pay API has three pieces. Your server creates a payment, the customer pays on our checkout page, then we ping you with a webhook when the bank says yes. By the end of this page you will have charged a fake card 49 dirhams and watched your own server log say "paid". I've seen plenty of live shops get that last piece wrong, so please don't stop at the redirect just because the happy path looks fine.

Bring a free test account and Node 18+. Budget fifteen minutes, maybe twenty if ngrok is having a day.

## Get your test API keys

Copy two strings from **Developers → API keys**. The `sk_test_...` one is secret. Server only. Not the browser, not an app bundle some kid can unzip. The `whsec_...` one is for webhooks, later.

Test mode can't move real money. Go wild. Stick both in `.env` and gitignore it right now, before your first push, not after:

```bash
NWP_SECRET_KEY=sk_test_51Hx...
NWP_WEBHOOK_SECRET=whsec_9f2c...
```

## Create a payment

Your backend asks, we reply with a link. Amount, currency and the after-payment page are up to you.

Heads up, this one gets everybody once. We count in fils, not dirhams (cents, if you're billing in dollars). `4900` means 49.00 AED. Type `49` and your customer pays less than a sip of karak.

```javascript
// server.js
import express from "express";

const app = express();

app.post("/checkout", express.json(), async (req, res) => {
  const response = await fetch("https://api.northwindpay.example/v1/payments", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.NWP_SECRET_KEY}`,
      "Content-Type": "application/json",
      "Idempotency-Key": req.body.orderId,
    },
    body: JSON.stringify({
      amount: 4900,
      currency: "AED",
      reference: req.body.orderId,
      return_url: "https://yourshop.example/thanks",
    }),
  });

  const payment = await response.json();
  res.json({ checkoutUrl: payment.checkout_url });
});
```

Why the `Idempotency-Key`? Networks drop requests. Say your request times out and your code fires it again. Same key, so we hand back the original payment and nobody is charged twice. I just use the order ID.

The reply has an `id` (something like `pay_7Qm2Lx`), `status: "pending"` and that all-important `checkout_url`.

## Redirect to checkout

Redirect to `checkout_url`. Cards get typed into our page, so the PCI headache is mostly ours. Afterwards they land on your `return_url`.

And that redirect proves nothing. People shut the tab mid-payment. Banks decline after a 3-D Secure prompt. Mobile browsers lose redirects for no reason anyone has ever explained to me. Don't ship anything yet.

## Verify the webhook signature

Add your endpoint in **Developers → Webhooks**. Whenever a payment moves, pending to paid or pending to failed, we POST it a small JSON event. Trust that. Ignore the redirect.

Signature check before anything else, though. It's a public URL and anybody with curl can hit it. Forging the HMAC, on the other hand, needs your `whsec_` secret.

```javascript
import crypto from "node:crypto";

app.post("/webhooks/northwind", express.raw({ type: "application/json" }), (req, res) => {
  const signature = req.get("NWP-Signature");
  const expected = crypto
    .createHmac("sha256", process.env.NWP_WEBHOOK_SECRET)
    .update(req.body)
    .digest("hex");

  const valid =
    signature &&
    signature.length === expected.length &&
    crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected));

  if (!valid) return res.status(400).send("bad signature");

  const event = JSON.parse(req.body);
  if (event.type === "payment.succeeded") {
    markOrderPaid(event.data.reference); // your code
  }

  res.sendStatus(200);
});
```

Two things I've personally lost afternoons to. One: hash the raw bytes. Let `express.json()` touch the body first and the signature just won't match, ever. Two: reply fast. We wait ten seconds for your 200. No 200? We retry, on and off, for three days, so yes, you'll see duplicates. Make `markOrderPaid` shrug at an order that's already paid.

## Test with test card numbers

Fire up the server and the tunnel, then pay with `4000 0000 0000 0077`. Watch the terminal; `payment.succeeded` turns up within a second or two.

Then be mean. Card `4000 0000 0000 0002` is declined every time; expect `payment.failed` and an order that stays unpaid. If both work, you're done with test mode.

Refunds and partial captures? Different page. See the API reference.

## FAQ

### Webhook or redirect: which to trust

The webhook, every time. Reaching your `return_url` doesn't mean the money moved. Tabs get shut mid-payment, banks decline after 3-D Secure, mobile browsers drop redirects. Wait for a signed `payment.succeeded` before you mark anything paid.

### Why webhook signature verification fails

Check that you hash the raw bytes. If `express.json()` touches the body first, the signature won't match. Use `express.raw()` on the webhook route and your `whsec_` secret.

### What the Idempotency-Key header does

Say a request times out and your code retries it. Same key, so we return the first payment and the card isn't charged twice. I just use the order ID.

### Handling duplicate webhooks and retries

Send your 200 within ten seconds. No 200 and we retry, on and off, for three days, so the same event can turn up more than once. Make `markOrderPaid` shrug at an order that's already paid.

---

**Hire me for docs like this.** Fiverr gig: [I will write clear technical documentation and user guides](https://www.fiverr.com/your_solution_/write-clear-technical-documentation-and-user-guides) · Contra profile: [contra.com/saransh_rayguru_bcplkfab](https://contra.com/saransh_rayguru_bcplkfab)
