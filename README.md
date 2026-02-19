# John Thiga — Portfolio

Personal portfolio hosted on **GitHub Pages** with M-Pesa payment integration.

---

## 📁 File Structure

```
your-repo/
├── index.html        ← Main portfolio (all-in-one)
├── 1.png             ← Your profile photo (add this yourself!)
├── api/
│   └── mpesa.js      ← Vercel serverless function (M-Pesa backend)
└── README.md
```

---

## 🚀 Deploy to GitHub Pages

### Step 1 — Create your repo
1. Go to [github.com](https://github.com) → **New repository**
2. Name it exactly: `john11e.github.io`
3. Set to **Public** → **Create repository**

### Step 2 — Upload files
Drag and drop `index.html`, `1.png`, and `README.md` into the repo, then click **Commit changes**.

Or use Git:
```bash
git init
git add .
git commit -m "Initial portfolio"
git remote add origin https://github.com/john11e/john11e.github.io.git
git push -u origin main
```

### Step 3 — Enable GitHub Pages
1. Repo → **Settings** → **Pages**
2. Source: **main** branch, **/ (root)**
3. Click **Save**
4. Visit: `https://john11e.github.io` (live in ~2 minutes)

---

## 🖼️ Adding Your Photo

1. Rename your photo to `1.png`
2. Upload to the **root** of your repo (same level as `index.html`)
3. It auto-appears in the hero section

> Best results: square crop, at least 500×500px, face near top

---

## 📬 Contact Form — Formspree Setup

1. Sign up free at [formspree.io](https://formspree.io)
2. **+ New Form** → name it "Portfolio Contact"
3. Copy your form ID (e.g. `xyzabcde` from `https://formspree.io/f/xyzabcde`)
4. In `index.html`, find and update:
   ```js
   const FORMSPREE_ID = 'xyzabcde';  // ← paste your ID here
   ```
5. Save and re-upload to GitHub. Done ✅

---

## 🟢 M-Pesa — Daraja API Setup (Full STK Push)

Currently the form shows a WhatsApp fallback. When you're ready for full automation:

### Step 1 — Get Daraja Credentials
1. Register at [developer.safaricom.co.ke](https://developer.safaricom.co.ke)
2. Create an App → note down:
   - **Consumer Key**
   - **Consumer Secret**
   - **Shortcode** (your M-Pesa number/Till number: `254748903818`)
   - **Passkey** (given by Safaricom)

### Step 2 — Deploy the Backend to Vercel (Free)

Because GitHub Pages is static HTML, you need a tiny server to make the Daraja API call (the browser can't call it directly due to CORS and security).

**Vercel** lets you deploy serverless functions for free.

#### Create `/api/mpesa.js` in your repo:

```js
// /api/mpesa.js
// Vercel Serverless Function — handles M-Pesa STK Push

export default async function handler(req, res) {
  // Allow CORS from your GitHub Pages site
  res.setHeader('Access-Control-Allow-Origin', 'https://john11e.github.io');
  res.setHeader('Access-Control-Allow-Methods', 'POST, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type');

  if (req.method === 'OPTIONS') return res.status(200).end();
  if (req.method !== 'POST') return res.status(405).json({ message: 'Method not allowed' });

  const { phone, amount, accountReference, transactionDesc } = req.body;

  // ── Validate ──
  if (!phone || !amount || !accountReference) {
    return res.status(400).json({ message: 'Missing required fields' });
  }

  try {
    // ── Step 1: Get OAuth token ──
    const consumerKey    = process.env.DARAJA_CONSUMER_KEY;
    const consumerSecret = process.env.DARAJA_CONSUMER_SECRET;
    const credentials    = Buffer.from(`${consumerKey}:${consumerSecret}`).toString('base64');

    const tokenRes = await fetch(
      'https://api.safaricom.co.ke/oauth/v1/generate?grant_type=client_credentials',
      { headers: { Authorization: `Basic ${credentials}` } }
    );
    const { access_token } = await tokenRes.json();

    // ── Step 2: Generate password ──
    const shortcode  = process.env.DARAJA_SHORTCODE;  // e.g. 174379 for sandbox, your real shortcode for prod
    const passkey    = process.env.DARAJA_PASSKEY;
    const timestamp  = new Date().toISOString().replace(/[-T:.Z]/g, '').slice(0, 14);
    const password   = Buffer.from(`${shortcode}${passkey}${timestamp}`).toString('base64');

    // ── Step 3: Initiate STK Push ──
    const stkRes = await fetch(
      'https://api.safaricom.co.ke/mpesa/stkpush/v1/processrequest',
      {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${access_token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          BusinessShortCode: shortcode,
          Password:          password,
          Timestamp:         timestamp,
          TransactionType:   'CustomerPayBillOnline',
          Amount:            parseInt(amount),
          PartyA:            phone,            // customer phone e.g. 254748903818
          PartyB:            shortcode,
          PhoneNumber:       phone,
          CallBackURL:       process.env.DARAJA_CALLBACK_URL,  // optional — see note below
          AccountReference:  accountReference,
          TransactionDesc:   transactionDesc || 'Payment to John Thiga'
        })
      }
    );

    const stkData = await stkRes.json();

    if (stkData.ResponseCode === '0') {
      return res.status(200).json({ message: 'STK push sent', data: stkData });
    } else {
      return res.status(400).json({ message: stkData.errorMessage || 'STK push failed' });
    }

  } catch (err) {
    console.error('M-Pesa error:', err);
    return res.status(500).json({ message: 'Server error. Please try again.' });
  }
}
```

#### Deploy to Vercel:
1. Install Vercel CLI: `npm i -g vercel`
2. In your project folder: `vercel`
3. Follow the prompts — it will give you a URL like `https://john11e.vercel.app`

#### Add Environment Variables in Vercel:
Go to your Vercel project → **Settings** → **Environment Variables** and add:

| Key | Value |
|-----|-------|
| `DARAJA_CONSUMER_KEY` | your consumer key |
| `DARAJA_CONSUMER_SECRET` | your consumer secret |
| `DARAJA_SHORTCODE` | `254748903818` (or your Till number) |
| `DARAJA_PASSKEY` | your Safaricom passkey |
| `DARAJA_CALLBACK_URL` | `https://john11e.vercel.app/api/callback` (or any URL) |

### Step 3 — Connect to Your Portfolio

In `index.html`, find the commented Daraja block and uncomment it:

```js
// Change this line:
const res = await fetch('https://YOUR-VERCEL-URL.vercel.app/api/mpesa', {
// To your actual Vercel URL:
const res = await fetch('https://john11e.vercel.app/api/mpesa', {
```

Then comment out or delete the `// FALLBACK` block below it.

---

## 📞 Contact Details

| Channel | Detail |
|---------|--------|
| Email | johnthigamureithi@gmail.com |
| WhatsApp | +254 748 903 818 |
| Phone | +254 748 903 818 |
| GitHub | github.com/john11e |
| M-Pesa | 0748 903 818 |
