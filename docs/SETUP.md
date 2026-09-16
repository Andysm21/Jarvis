# Setup — detailed walkthrough

Everything here is account creation and credentials — the things only you can do. No coding.

**Total: 60–90 minutes**, most of it waiting on verification emails. Do them in this order;
step 1 can stall for days, so start it first.

> **A note on screenshots and labels.** Oracle, Meta and Google redesign their consoles
> constantly. Button names below are correct as of writing, but if a label doesn't match
> exactly, look for the nearest equivalent — the *sequence* of what you're doing is stable
> even when the wording moves. Where a step is commonly renamed, I've noted the alternates.

---

# Step 1 — Oracle Cloud (the server)

**~20 min active · free forever · needs a card for identity verification (not charged)**

## 1.1 Create the account

1. Go to **[cloud.oracle.com](https://cloud.oracle.com/)** and click **Start for free**.
2. Enter your country and email. Click **Verify my email**.
3. Open the email from Oracle, click the verification link.
4. Set a password. For **Company Name**, put anything — your own name is fine.

5. **Choose your home region.** ⚠️ **This is permanent and cannot be changed.**

   Two competing pressures:
   - *Closer to you* = lower latency (barely matters for a chat assistant)
   - *Less popular* = far better odds of getting free ARM capacity (matters a lot)

   Heavily oversubscribed, expect capacity problems: **Frankfurt, London, Ashburn,
   Phoenix, São Paulo, Mumbai, Singapore, Tokyo**.

   Usually better availability: **Zurich, Amsterdam, Madrid, Marseille, Milan, Stockholm,
   Montreal, Osaka, Jeddah**.

   If you're in the UK or Europe, **Amsterdam, Zurich or Madrid** are good picks — near
   enough for low latency, far less contended than London or Frankfurt.

6. Enter card details. Oracle places a small temporary authorisation (about $1) to verify
   you're a real person. It's reversed within a few days. **Your account will not be
   charged** as long as you stay on Always Free resources.
7. Accept the agreement and submit.
8. Wait for the "Your Oracle Cloud account is ready" email — usually a few minutes, but it
   can take up to half an hour.

## 1.2 Create the instance

1. Sign in to the console. Click the **hamburger menu** (☰, top left) → **Compute** → **Instances**.
2. Click **Create instance**.
3. **Name:** `jarvis`
4. **Compartment:** leave as the default (root).
5. **Placement:** leave the default availability domain. *(If you hit capacity errors later, come back and try AD-2 or AD-3 if your region has them.)*

6. **Image and shape** → click **Edit**:
   - Click **Change image** → select **Canonical Ubuntu** → **22.04** → **Select image**
   - Click **Change shape** → select the **Ampere** tab → **VM.Standard.A1.Flex**
   - Set **OCPUs: 4** and **Memory: 24 GB** — this is the entire free ARM allowance in one machine
   - Click **Select shape**

7. **Networking:**
   - Leave **Create new virtual cloud network** selected
   - ✅ Confirm **Assign a public IPv4 address** is set to **Yes** — without this the server is unreachable

8. **Add SSH keys:**
   - Select **Generate a key pair for me**
   - Click **Save private key** *and* **Save public key** — download both now
   - ⚠️ **You cannot download the private key again.** If you lose it you must rebuild the instance.

9. **Boot volume:** the default 50 GB is plenty. *(Your free allowance is 200 GB total.)*
10. Click **Create**. Provisioning takes 1–2 minutes.
11. When status turns green (**Running**), **copy the Public IP address** from the instance page.

## 1.3 If you get "Out of host capacity"

This is the single most common Oracle free-tier frustration. It means no free ARM hardware
is available right now in that availability domain. It is not a problem with your account.

Try in order:

1. **Lower the request.** Try **1 OCPU / 6 GB** instead of 4/24 — small requests get placed
   far more often. You can resize upward later when capacity frees up.
2. **Try a different availability domain** (the Placement section), if your region has more than one.
3. **Retry at off-peak hours.** Capacity is released constantly. Early morning in your
   region's local time tends to be best.
4. **Just keep trying for a couple of days.** Persistence genuinely works here.

**If ARM never comes through**, create a **VM.Standard.E2.1.Micro** (AMD) instead:
- Found under the **Specialty and previous generation** shape tab
- 1/8 OCPU, 1 GB RAM — always available, never capacity-constrained
- Small, but adequate. We'd use hosted Whisper transcription instead of running it locally,
  which is free anyway via Groq (Step 7).

Tell me which shape you ended up with — it changes one decision in Phase 2, nothing else.

## 1.4 Connect over SSH

On your own machine, in a terminal:

```bash
# Move the key somewhere sensible and lock down its permissions
mv ~/Downloads/ssh-key-*.key ~/.ssh/jarvis.key
chmod 600 ~/.ssh/jarvis.key

# Connect (replace with your instance's public IP)
ssh -i ~/.ssh/jarvis.key ubuntu@YOUR_PUBLIC_IP
```

Accept the host fingerprint when prompted. You should land at an `ubuntu@jarvis:~$` prompt.

> **On Windows?** Use PowerShell (which has a built-in `ssh`), or install
> [Windows Terminal](https://aka.ms/terminal). `chmod` doesn't exist there — instead
> right-click the key file → Properties → Security → Advanced → Disable inheritance →
> remove every user except your own.

## 1.5 Open ports 80 and 443 — **both** layers

Oracle blocks these in two independent places. Miss either one and nothing works. This trips
up almost everyone, so do both.

**Layer 1 — Oracle's cloud firewall:**

1. ☰ → **Networking** → **Virtual Cloud Networks**
2. Click your VCN → **Security Lists** (left sidebar) → **Default Security List for ...**
3. Click **Add Ingress Rules**, and add these two:

   | Field | Rule 1 | Rule 2 |
   |---|---|---|
   | Stateless | No | No |
   | Source Type | CIDR | CIDR |
   | Source CIDR | `0.0.0.0/0` | `0.0.0.0/0` |
   | IP Protocol | TCP | TCP |
   | Destination Port Range | `80` | `443` |

4. Click **Add Ingress Rules** to save.

**Layer 2 — the instance's own iptables.** Oracle's Ubuntu images ship with local firewall
rules that drop everything except SSH. Over SSH on the instance, run:

```bash
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT
sudo netfilter-persistent save
```

*(If `netfilter-persistent` isn't found: `sudo apt update && sudo apt install -y iptables-persistent`, accept the save prompts, then re-run the save command.)*

**Verify it worked:**

```bash
# On the instance — start a throwaway listener
sudo python3 -m http.server 80
```

Then in a browser on your phone or laptop, visit `http://YOUR_PUBLIC_IP`. You should see a
directory listing. If you do, both firewall layers are open. Press `Ctrl+C` to stop it.

If it hangs instead, one of the two layers is still closed — recheck both.

**✅ Give me:** public IP, region, and which shape you got (ARM 4/24, ARM 1/6, or AMD micro).

---

# Step 2 — A domain name

**~10 min · free**

WhatsApp will only send webhooks to an HTTPS URL with a valid certificate. A bare IP address
will not work, and self-signed certificates are rejected.

## Option A — DuckDNS (free, 2 minutes)

1. Go to **[duckdns.org](https://www.duckdns.org/)**.
2. Sign in with GitHub, Google, Twitter or Reddit.
3. In the **domain** box type a name, e.g. `andy-jarvis`, and click **add domain**.
4. You now own `andy-jarvis.duckdns.org`.
5. In the **current ip** field for that domain, paste your Oracle **public IP** and click **update ip**.
6. Copy your **token** from the top of the page and keep it — it lets you refresh the IP later.

**Verify:**
```bash
nslookup andy-jarvis.duckdns.org
```
The answer should be your Oracle public IP. DNS may take 1–5 minutes to propagate.

## Option B — A real domain (~$2/year, nicer)

1. Buy a cheap `.xyz` or `.top` from [Namecheap](https://www.namecheap.com/) or [Porkbun](https://porkbun.com/).
2. Add the domain to [Cloudflare](https://dash.cloudflare.com/) (free plan) and update the nameservers at your registrar.
3. In Cloudflare **DNS**, add:
   - Type `A`, Name `jarvis`, Content = your Oracle public IP
   - **Proxy status: DNS only** (grey cloud) — ⚠️ the orange proxied cloud interferes with Let's Encrypt certificate issuance and with webhook delivery. Turn it off.

Both work identically. DuckDNS is faster to set up; a real domain looks better and is yours.

**✅ Give me:** the full domain (e.g. `andy-jarvis.duckdns.org`).

---

# Step 3 — WhatsApp Cloud API

**~25 min · free**

## 3.1 Developer account

1. Go to **[developers.facebook.com](https://developers.facebook.com/)** and log in with your Facebook account.
2. If you've never used it: click **Get Started** (top right), accept the terms, and verify
   your email or phone when asked.

## 3.2 Create the app

1. Click **My Apps** → **Create App**.
2. Meta asks **"What do you want your app to do?"** → choose **Other** → **Next**.
   *(If you don't see this question, skip to the app type list.)*
3. Select app type **Business** → **Next**.
4. Fill in:
   - **App name:** `Jarvis`
   - **App contact email:** your email
   - **Business portfolio:** if you have none, leave it unselected or create one when prompted
5. Click **Create app** and re-enter your Facebook password if asked.

## 3.3 Add WhatsApp

1. On the app dashboard, find **WhatsApp** in the product list → click **Set up**.
2. Meta will ask you to select or create a **Meta Business Account**. Create one — name it
   anything. This is free and requires no verification.
3. You'll land on the **API Setup** page (left sidebar: **WhatsApp** → **API Setup**).

## 3.4 Collect the identifiers

On the **API Setup** page, note down:

| What | Where |
|---|---|
| **Phone number ID** | Under the **From** dropdown — a long number |
| **WhatsApp Business Account ID** | Just below the phone number ID |
| **Temporary access token** | At the top — expires in 24 hours, we'll replace it in 3.6 |

Meta has given you a **free test phone number** — that's the "From" number. You don't need
to buy or verify a number of your own.

## 3.5 Verify your own phone as a recipient

1. In the **To** dropdown, click **Manage phone number list**.
2. Click **Add phone number**, enter **your own mobile** in full international format
   (e.g. `+447700900123`).
3. Meta sends you a WhatsApp or SMS code. Enter it.
4. Back on API Setup, with your number selected in **To**, click **Send message**.
5. **You should receive a "Hello World" template message on WhatsApp.** If it arrives,
   everything is wired correctly.

> The test number allows **up to 5 verified recipients**. You need one. This limit is the
> only meaningful restriction and it doesn't affect you.

## 3.6 Get a permanent access token

The temporary token dies in 24 hours. Replace it now so we're not re-doing it mid-build.

1. Go to **[business.facebook.com/settings](https://business.facebook.com/settings/)**.
2. Select your Business Account (top-left dropdown).
3. Left sidebar → **Users** → **System users** → **Add**.
4. Name it `jarvis-system`, role **Admin** → **Create system user**.
5. Click **Assign assets** → **Apps** → tick your **Jarvis** app → enable **Full control** → **Save changes**.
6. Click **Generate new token**:
   - **App:** Jarvis
   - **Token expiration:** **Never**
   - **Permissions:** tick **`whatsapp_business_messaging`** and **`whatsapp_business_management`**
7. Click **Generate token** → **copy it immediately**.

⚠️ **This token is shown exactly once.** If you lose it you must generate a new one. Treat
it like a password — it can send WhatsApp messages as your app.

## 3.7 App secret and verify token

1. Back in the app dashboard: **App settings** → **Basic**.
2. Next to **App secret**, click **Show** and copy it. *(This is what proves incoming
   webhooks genuinely came from Meta — it's a security-critical value.)*
3. **Invent a webhook verify token** — any random string you make up, e.g. run
   `openssl rand -hex 16` and use the output. Write it down. Meta will echo this back to us
   when it first calls our webhook, and we check it matches.

> **Webhook URL is configured in Phase 1**, not now — it needs the server running first.
> Leave that section alone.

**✅ Give me:** phone number ID, WhatsApp Business Account ID, app secret, verify token, and
the permanent access token. *(Not in chat — see "Handing credentials over" at the end.)*

---

# Step 4 — Google Cloud (Gmail + Calendar)

**~20 min · free**

⚠️ **Sign in with the Google account whose mail and calendar JARVIS should manage.** If you
use the wrong account here, you'll have to redo the whole step.

## 4.1 Create the project

1. Go to **[console.cloud.google.com](https://console.cloud.google.com/)**.
2. Click the project dropdown in the top bar → **New Project**.
3. **Project name:** `jarvis` → **Create**.
4. Wait for the notification, then **select the project** in the top bar. Confirm the top bar
   says `jarvis` before continuing — creating credentials in the wrong project is an easy mistake.

## 4.2 Enable the APIs

1. ☰ → **APIs & Services** → **Library**
2. Search **Gmail API** → click it → **Enable**
3. Go back to **Library**, search **Google Calendar API** → click it → **Enable**

## 4.3 Configure the consent screen

☰ → **APIs & Services** → **OAuth consent screen**

> Google renamed this area to **Google Auth Platform** in 2025, with sub-pages called
> **Branding**, **Audience**, **Clients** and **Data Access**. Both layouts are in the wild.
> I've given the new names in brackets.

1. **User Type:** **External** → **Create**
   *(External is correct even though you're the only user — Internal is only available to Workspace organisations.)*

2. **App information** *(new UI: **Branding**)*:
   - **App name:** `Jarvis`
   - **User support email:** your email
   - **Developer contact information:** your email
   - Leave logo and domains blank
   - **Save and continue**

3. **Scopes** *(new UI: **Data Access**)*:
   - Click **Add or remove scopes**
   - At the bottom there's a box labelled **Manually add scopes**. Paste these three, one per line:
     ```
     https://www.googleapis.com/auth/gmail.modify
     https://www.googleapis.com/auth/gmail.send
     https://www.googleapis.com/auth/calendar
     ```
   - Click **Add to table** → **Update** → **Save and continue**
   - Google will warn these are **sensitive/restricted scopes**. That's expected — read/send
     access to your mail is exactly what JARVIS needs, and exactly why it's classified that way.

4. **Test users** *(new UI: **Audience**)*:
   - Click **Add users**, enter **your own Google address** → **Add** → **Save and continue**

## 4.4 Publish the app (important — don't skip)

Still under **OAuth consent screen** *(or **Audience**)*, find **Publishing status**.

1. Click **Publish app** → **Confirm**.
2. Status should now read **In production**.

**Why this matters:** while the app sits in **Testing**, Google expires OAuth refresh tokens
after **7 days**. JARVIS would silently lose access to your mail every week. Moving to
production stops that.

**What you'll see because of it:** the app is unverified, so when you authorise it in Phase 3
Google shows a red *"Google hasn't verified this app"* warning. This is expected and safe —
it's your own app, authorising against your own account. Click **Advanced** → **Go to Jarvis
(unsafe)** to proceed.

> If refresh tokens still expire weekly despite this, tell me — there's a fallback (a
> scheduled silent re-auth) and I'll wire it into Phase 3.

## 4.5 Create the OAuth client

1. ☰ → **APIs & Services** → **Credentials** *(new UI: **Clients**)*
2. **Create credentials** → **OAuth client ID**
3. **Application type:** **Desktop app**
   *(Desktop, not Web — it gives us the loopback flow, which is far simpler to run once from a terminal.)*
4. **Name:** `jarvis-desktop` → **Create**
5. In the dialog, click **Download JSON**. It saves as `client_secret_....json`.

⚠️ **Never commit this file.** `.gitignore` already blocks `credentials.json`, but keep it
out of the repo directory entirely.

**✅ Give me:** the OAuth client JSON (via the secure method below — not pasted in chat).

---

# Step 5 — ntfy.sh (push notifications)

**~3 min · free · no account needed**

1. Install **ntfy** on your phone:
   - [iOS App Store](https://apps.apple.com/us/app/ntfy/id1625396347)
   - [Google Play](https://play.google.com/store/apps/details?id=io.heckel.ntfy)

2. **Generate a random topic name.** ⚠️ The topic name *is* the password — anyone who knows
   it can push notifications to your phone, and anyone who guesses it can read yours. Do not
   use anything guessable like `jarvis` or `andy`.

   ```bash
   echo "jarvis-$(openssl rand -hex 10)"
   ```
   Gives you something like `jarvis-8f3a91c4b7e25d0a6f18`.

3. In the ntfy app: tap **+** → paste the topic name → **Subscribe**.

4. **Test it** from any terminal:
   ```bash
   curl -d "JARVIS is listening." ntfy.sh/jarvis-8f3a91c4b7e25d0a6f18
   ```
   The notification should arrive on your phone within a second or two.

5. **iOS only:** open the notification once and allow notifications when prompted, otherwise
   they'll stay silent.

**✅ Give me:** the topic name.

---

# Step 6 — The model provider

**~5 min · read [COSTS.md](COSTS.md) first**

This is the only decision that affects what JARVIS costs to run. Start with 6.1.

## 6.1 Check your Claude subscription first

1. Go to **[claude.ai](https://claude.ai/)** → click your initials (bottom left) → **Settings** → **Billing**.
2. Note what it says: **Free**, **Pro**, **Max 5x**, or **Max 20x**.

- **Pro or Max** → there's a good chance JARVIS costs you **nothing extra**. Tell me which
  plan and I'll verify the Agent SDK can authenticate against it in Phase 0 before we commit
  to that path.
- **Free** → go to 6.2.

## 6.2 Gemini API key (the free tier workhorse)

Worth getting regardless — it's free and it absorbs the high-volume classification work.

1. Go to **[aistudio.google.com](https://aistudio.google.com/)** and sign in.
2. Click **Get API key** (left sidebar) → **Create API key**.
3. Select your existing `jarvis` Google Cloud project from Step 4, or let it create a new one.
4. Copy the key.

No card required. The free tier has daily request caps that are generous for one user.

## 6.3 Anthropic API key (only if 6.1 came back Free)

1. Go to **[console.anthropic.com](https://console.anthropic.com/)** and sign up.
2. **Settings** → **Billing** → add **$5** in credit to start. That's genuinely enough to
   run and test for weeks at personal volume.
3. **API keys** → **Create key** → name it `jarvis` → copy it.
4. *(Optional but recommended)* **Limits** → set a monthly spend cap so it can never surprise you.

**✅ Give me:** your Claude plan, plus whichever keys you obtained.

---

# Step 7 — Groq (free speech-to-text) *(optional)*

**~3 min · free**

**Skip this if you got the ARM instance** (4 OCPU / 24 GB) — that's ample to run Whisper
locally, which is faster and keeps your voice notes off third-party servers.

**Do it if you're on the AMD micro instance**, which can't run Whisper locally.

1. Go to **[console.groq.com](https://console.groq.com/)** and sign up (Google or GitHub login).
2. **API Keys** → **Create API Key** → name it `jarvis` → **Submit**.
3. Copy the key — shown once.

Groq's free tier is fast and generous. No card required.

**✅ Give me:** the Groq API key, if you got one.

---

# Handing credentials over safely

⚠️ **Do not paste any of these secrets into chat.** Chat transcripts are stored. A leaked
Google OAuth client plus token is full access to your email.

When we reach Phase 0, you'll do this over SSH on the server:

```bash
ssh -i ~/.ssh/jarvis.key ubuntu@YOUR_PUBLIC_IP

sudo mkdir -p /opt/jarvis
sudo chown ubuntu:ubuntu /opt/jarvis
nano /opt/jarvis/.env      # paste the values here
chmod 600 /opt/jarvis/.env
```

The repo will ship a `.env.example` listing every variable with dummy values, so you'll know
exactly what goes where. **Secrets live on the server and nowhere else** — not in the repo,
not in chat, not in a note.

**What you *can* safely tell me in chat:** your public IP, your domain, your region, which
instance shape you got, your Claude plan tier, and whether each step succeeded. None of those
are secrets.

---

# Checklist

Work down it. Tell me as each one lands and I'll start what it unblocks.

- [ ] **1.** Oracle account created, home region chosen
- [ ] **1.** Instance running, public IP noted, SSH key saved safely
- [ ] **1.** SSH connection confirmed working
- [ ] **1.** Ports 80/443 open in **both** the VCN security list **and** iptables — verified with the `python3 -m http.server` test
- [ ] **2.** Domain created and resolving to the instance IP (`nslookup` confirms)
- [ ] **3.** Meta app created, WhatsApp product added
- [ ] **3.** Your number verified as a recipient, "Hello World" test message received
- [ ] **3.** Permanent system-user token generated and saved
- [ ] **3.** App secret copied, verify token invented and saved
- [ ] **4.** Google project created, Gmail + Calendar APIs enabled
- [ ] **4.** Consent screen configured with all three scopes
- [ ] **4.** App **published to production** (stops the 7-day token expiry)
- [ ] **4.** Desktop OAuth client created, JSON downloaded
- [ ] **5.** ntfy topic chosen, subscribed on phone, test push received
- [ ] **6.** Claude plan checked; Gemini and/or Anthropic key obtained
- [ ] **7.** *(optional)* Groq key, if you're on the AMD micro instance

---

# When you get stuck

Tell me **which step number**, **the exact error text**, and **what you'd just clicked**. Most
of these have a known cause:

| Symptom | Almost always |
|---|---|
| "Out of host capacity" | Free ARM contention — see §1.3, request a smaller shape |
| SSH `Permission denied (publickey)` | Key permissions not `600`, or wrong username — it's `ubuntu`, not `root` |
| SSH times out entirely | Instance still provisioning, or wrong IP |
| Port 80 test hangs | You did the VCN security list but not iptables (or vice versa) — §1.5 needs **both** |
| WhatsApp test message never arrives | Number not verified, or not in full international format with `+` and country code |
| "App not active" from Meta | App is in development mode without the WhatsApp product properly set up |
| Google: "app is blocked" | Consent screen incomplete — a required field on the Branding page is empty |
| Google: "access_denied" | Your account isn't in test users, or the app isn't published |
| Certificate fails to issue later | Cloudflare proxy is on (orange cloud) — set it to DNS only, §2 Option B |
