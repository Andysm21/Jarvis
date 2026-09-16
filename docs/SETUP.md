# Setup — what you need to do

Accounts and credentials only you can create. Nothing here requires coding. Work through
it and JARVIS will have everything it needs when the build starts.

Rough total: **60–90 minutes**, most of it waiting on verification emails.

---

## 1. Oracle Cloud — the server
*~20 min · free forever · card required for identity verification, not charged*

1. Sign up at [cloud.oracle.com](https://cloud.oracle.com/) — choose **Always Free**.
2. Pick your home region carefully — **you cannot change it later**, and free ARM capacity varies by region. Less-congested regions have better availability.
3. Create a compute instance:
   - Shape: **VM.Standard.A1.Flex** (Ampere ARM) — ask for 2 OCPU / 12 GB to start
   - Image: **Ubuntu 22.04**
   - Save the SSH private key it gives you — you cannot download it twice
4. Open ports 80 and 443 in the VCN security list (default is closed).

> **If you hit "Out of host capacity"** — common for free ARM — either retry over a couple
> of days, or create a **VM.Standard.E2.1.Micro** (AMD) instead. It's smaller but always
> available, and we'll use hosted Whisper instead of local transcription.

**Hand over:** public IP, SSH key, region.

---

## 2. A domain name
*~10 min · free options available*

WhatsApp requires an HTTPS webhook with a valid certificate. An IP address won't do.

- **Free:** a DuckDNS subdomain, or Cloudflare's free tier on any domain you own
- **Cheap:** a `.xyz` or `.top` from Namecheap, roughly $1–3/year

Point an A record at the Oracle instance's public IP. Caddy handles the certificate
automatically once DNS resolves.

**Hand over:** the domain.

---

## 3. WhatsApp Cloud API
*~20 min · free*

1. Create a Meta developer account at [developers.facebook.com](https://developers.facebook.com/).
2. **Create App** → type **Business**.
3. Add the **WhatsApp** product. Meta provisions a free test phone number automatically.
4. Under **API Setup**, add your own mobile number as a verified recipient and confirm the code Meta sends you.
5. Collect:
   - **Phone number ID**
   - **WhatsApp Business Account ID**
   - **Temporary access token** (24h — we'll swap it for a permanent System User token in Phase 1)
   - **App secret** (for webhook signature verification)
6. Invent a **webhook verify token** — any random string. Keep it.

> Leave the webhook URL blank for now. It gets configured in Phase 1 once the server is up.

**Hand over:** phone number ID, business account ID, app secret, verify token.

---

## 4. Google Cloud — Gmail and Calendar
*~20 min · free*

1. Create a project at [console.cloud.google.com](https://console.cloud.google.com/).
2. Enable the **Gmail API** and the **Google Calendar API**.
3. Configure the **OAuth consent screen**:
   - User type: **External**
   - Publishing status: leave in **Testing**, and add your own Google account as a test user. This avoids Google's app verification process entirely — testing mode is fine indefinitely for personal use.
4. Add these scopes:
   ```
   https://www.googleapis.com/auth/gmail.modify
   https://www.googleapis.com/auth/gmail.send
   https://www.googleapis.com/auth/calendar
   ```
5. **Credentials** → **Create OAuth client ID** → **Desktop app**. Download the JSON.

> Testing-mode refresh tokens expire after 7 days **if** the consent screen stays
> unverified *and* the app is in testing. We handle this by publishing the app to
> "In production" without verification — permitted for sensitive scopes when you're the
> only user, and it stops the 7-day expiry. Documented properly in Phase 3.

**Hand over:** the OAuth client JSON (never commit it).

---

## 5. ntfy.sh — push notifications
*~2 min · free · no account*

1. Install the **ntfy** app (iOS / Android).
2. Invent an unguessable topic name — treat it as a password, e.g. `jarvis-a7f3k9x2-andy`.
3. Subscribe to it in the app.

Anyone who knows the topic can push to it. Make it long and random.

**Hand over:** the topic name.

---

## 6. The model provider
*~5 min · see [COSTS.md](COSTS.md) before choosing*

Pick one — this is the decision from [PLAN.md §7](PLAN.md#7-open-questions-to-settle-before-phase-1):

- **A — Claude subscription** *(likely best for you)*: if you have Claude Pro or Max, the Agent SDK may authenticate against it with no extra cost. Needs confirming.
- **B — Hybrid** *(recommended if A doesn't work out)*: Gemini API key from [aistudio.google.com](https://aistudio.google.com/) (free tier) for routine work, plus a small Anthropic API balance for the heavy lifting.
- **C — Claude API only**: simplest, highest quality, ~$5–15/month.

**Hand over:** whichever key(s) apply.

---

## 7. Groq — free speech-to-text *(optional)*
*~3 min · free tier*

Sign up at [console.groq.com](https://console.groq.com/) for a free Whisper API key. It's
very fast and very generous. Skip this if the Oracle ARM instance came through — we'll run
Whisper locally instead.

**Hand over:** Groq API key (optional).

---

## Handing credentials over safely

**Do not paste secrets into chat, and do not commit them.** When the time comes:

1. SSH into the Oracle instance
2. Create `/opt/jarvis/.env` there directly
3. `chmod 600 .env`

The repo will carry a `.env.example` listing every variable with dummy values. Secrets live
on the server and nowhere else.

---

## Checklist

- [ ] Oracle Cloud instance running, ports 80/443 open, SSH access confirmed
- [ ] Domain pointing at the instance IP
- [ ] WhatsApp app created, your number verified as a recipient
- [ ] Google Cloud project with Gmail + Calendar APIs enabled and OAuth client downloaded
- [ ] ntfy topic chosen and subscribed on your phone
- [ ] Model provider decided and key obtained
- [ ] *(optional)* Groq key
