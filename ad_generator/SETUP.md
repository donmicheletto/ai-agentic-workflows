# Setup Guide

Step-by-step setup for the UGC Ad Generator workflow. If you've never used n8n before, that's fine — this guide assumes nothing.

**Total setup time:** about 25-40 minutes depending on how fast you move through OAuth screens.

**What you'll have at the end:** a working Telegram bot that turns product photos into AI-generated short-form video ads.

---

## What you need before starting

### Accounts (all free to create — most have free trial credits)

- **n8n instance** — either [n8n Cloud](https://n8n.io) (free tier works) or a self-hosted install. If you don't have one, n8n Cloud is the fastest path.
- **Telegram account** — for creating the bot and testing
- **OpenAI account** — https://platform.openai.com/signup
- **OpenRouter account** — https://openrouter.ai
- **Kie.ai account** — https://kie.ai (this is where the image and video generation runs)
- **Google account** — for the auto-created log spreadsheet

### Money on the table

You'll need to put **at least $5 of credit on Kie.ai** to test. Generating one full ad on the cheap (`veo3_fast`) tier costs ~$0.60. On the premium (`veo3`) tier, ~$2.30. OpenAI and OpenRouter usage for this workflow is negligible (<$0.05 per run combined) but they require an active account with billing enabled.

---

## Step 1 — Create your Telegram bot

This is what your audience will message. Each bot has a unique username (e.g. @YourCompanyAdsBot) and an API token used to control it.

1. Open Telegram and search for **@BotFather** (the official Telegram account that creates bots — it has a blue verified checkmark)
2. Tap **Start** (or send `/start`)
3. Send `/newbot`
4. BotFather will ask for a **name** — this is the display name shown at the top of the chat. Example: `My UGC Bot`
5. Then it asks for a **username** — must end in `bot`. Example: `mycompany_ugc_bot`
6. BotFather replies with a message containing your **API token**. It looks like: `1234567890:AAFsomethingsomethingsomething`

**Copy the token and save it somewhere safe.** You'll paste it into n8n in Step 7.

> ⚠️ Treat this token like a password. Anyone with it can control your bot. If you ever paste it in a screenshot, video, or shared chat, immediately revoke and regenerate it via BotFather → `/revoke`.

While you're still talking to BotFather, optionally:
- Send `/setdescription` to add a description shown when users open the bot
- Send `/setuserpic` to upload a profile picture

---

## Step 2 — Get your OpenAI API key

Used by the photo analyzer (GPT-4o vision).

1. Go to https://platform.openai.com/api-keys
2. Sign in or create an account
3. If this is a new account, you may need to add billing first: https://platform.openai.com/account/billing — add a payment method and at least $5 credit
4. Click **Create new secret key**
5. Name it `n8n UGC Bot` (just a label for you to remember)
6. Click **Create secret key**, then **Copy** — the key starts with `sk-...`

**Save the key** — OpenAI shows it only once.

---

## Step 3 — Get your OpenRouter API key

Used for the creative agent and revision agent (Claude Sonnet 4.6).

1. Go to https://openrouter.ai
2. Sign up or log in
3. Click your profile icon (top right) → **Keys**
4. Click **Create Key**
5. Name it `n8n UGC Bot`
6. Click **Create**, then copy the key — starts with `sk-or-v1-...`

**Add credit:** OpenRouter has free trial credit, but Claude calls cost ~$0.02 per run. Top up at https://openrouter.ai/credits if you plan to use this seriously.

---

## Step 4 — Get your Kie.ai API key

Used for both image generation (NanoBanana Pro) and video generation (Veo 3.1). This is the most important key — it's where most of the money is spent.

1. Go to https://kie.ai and sign up
2. Add credit at https://kie.ai/recharge — minimum $5 recommended for testing (~8 ads on `veo3_fast`)
3. Go to https://kie.ai/api-key
4. Click **Create New API Key**
5. Give it a name like `n8n UGC Bot`
6. Important: **leave the IP whitelist field blank** for now. If you add an IP whitelist that doesn't include n8n's IP, the bot will fail with 401 errors. You can add restrictions later once you know it works.
7. Click **Create**, then copy the key — looks like a 32-character hex string

**Save the key.**

---

## Step 5 — Set up Google Drive OAuth (the trickiest one)

The workflow auto-creates a Google Sheet to log your generations. To do this, n8n needs OAuth permission to your Google account.

This step has two phases: setting up a Google Cloud project (one-time, ~10 min) and connecting it to n8n.

### Phase A — Google Cloud Console

1. Go to https://console.cloud.google.com
2. Sign in with the Google account you want to own the log spreadsheet
3. Click the project dropdown at the top → **New Project**
4. Name it `n8n UGC Bot` and click **Create**
5. Wait for the project to be created (10-20 seconds), then make sure it's selected in the dropdown
6. In the search bar at top, type **Google Drive API** → click the result → click **Enable**
7. Search again for **Google Sheets API** → click → **Enable**
8. In the left sidebar, go to **APIs & Services → OAuth consent screen**
9. Choose **External** → click **Create**
10. Fill in:
    - App name: `n8n UGC Bot`
    - User support email: your email
    - Developer contact: your email
    - (Skip everything else)
11. Click **Save and Continue** through the next screens (Scopes, Test users, Summary). On the **Test users** screen, add your own Google email so you're allowed to use the app.
12. Back in the sidebar, go to **APIs & Services → Credentials**
13. Click **+ Create Credentials → OAuth client ID**
14. Application type: **Web application**
15. Name: `n8n UGC Bot`
16. Under **Authorized redirect URIs**, add this exactly:
    - `https://YOUR-N8N-URL/rest/oauth2-credential/callback`
    - Replace `YOUR-N8N-URL` with your actual n8n domain (e.g. `your-instance.app.n8n.cloud` for Cloud, or your self-hosted URL)
17. Click **Create**
18. A popup shows your **Client ID** and **Client Secret** — copy both. You'll need them in n8n.

### Phase B — Connect in n8n

You'll do this in Step 7 below. For now, just have the Client ID and Client Secret ready.

---

## Step 6 — Import the workflow

1. Open your n8n instance
2. Click **Workflows** in the sidebar
3. Click **Add workflow** dropdown (top right) → **Import from File**
4. Select the `ugc-ad-generator-CLEAN.json` file
5. The workflow opens. You'll see ~62 nodes connected together with some red error indicators — those are nodes missing credentials, which is expected at this point.

---

## Step 7 — Create the 5 credentials in n8n

In your n8n sidebar, click **Credentials**. Then create one of each:

### 7.1 Telegram Bot
1. Click **Add Credential**
2. Search for **Telegram API** → select it
3. **Access Token:** paste the token from Step 1 (the one starting with numbers)
4. Name it `Telegram - UGC Bot` or similar
5. Click **Save**

### 7.2 OpenAI
1. **Add Credential** → search **OpenAI API**
2. **API Key:** paste the `sk-...` key from Step 2
3. Save as `OpenAI - UGC Bot`

### 7.3 OpenRouter
1. **Add Credential** → search **OpenRouter API**
2. **API Key:** paste the `sk-or-v1-...` key from Step 3
3. Save as `OpenRouter - UGC Bot`

### 7.4 Kie.ai (HTTP Header Auth)
This one's not a dedicated integration — it's a generic HTTP auth credential.
1. **Add Credential** → search **Header Auth**
2. **Name** field: type exactly `Authorization` (capital A, no spaces)
3. **Value** field: type `Bearer ` then paste your Kie.ai key. Final value looks like:
   `Bearer 7f00d617e5ba9bd93b377d2462f2fc34`
   - Note the single space between `Bearer` and your key
   - Don't add quotes
4. Save as `Kie.ai`

### 7.5 Google Drive OAuth2
1. **Add Credential** → search **Google Drive OAuth2 API**
2. Paste the **Client ID** and **Client Secret** from Step 5
3. Click **Sign in with Google** — a popup opens
4. Choose your Google account, accept the permissions
5. If you get a "this app isn't verified" warning, click **Advanced → Go to n8n UGC Bot (unsafe)**. This warning appears because your OAuth app isn't published — it's normal for personal-use OAuth apps.
6. After successful auth, the credential window closes. Save it as `Google Drive - UGC Bot`.

---

## Step 8 — Attach credentials to nodes (THE GOTCHA STEP)

⚠️ n8n does NOT auto-attach credentials. Even though you created them, every node still needs you to manually select the credential from a dropdown.

Open your workflow. For every node listed below, click it open and select the appropriate credential from the **Credential to use** dropdown:

### Telegram credential — attach to ALL of these nodes
- Received
- Ask for Photo
- Ask Style
- Send Parse Error
- Get Approval
- Send Restart
- Generating Variants
- Variant Timeout
- Pick Variant
- Confirm Video
- No Video
- Generating Video
- Veo Error
- Send Video
- Send Caption
- Notify Sheet ID
- Telegram Trigger (the very first node)

### OpenAI credential — one node
- Analyze Photo

### OpenRouter credential — two nodes
- OpenRouter Chat Model
- OpenRouter Chat Model1

### Kie.ai credential (HTTP Header Auth) — four nodes
- Create Variant
- Poll Variant
- Create Veo
- Poll Veo

### Google Drive credential — two nodes
- Create Sheet
- Append Row

**Save the workflow** (Cmd/Ctrl+S) when done. The red error indicators should disappear.

---

## Step 9 — Configure your bot

Find the **⚙️ Config** node on the left side of the canvas (it has a gear emoji).

Click it open. You'll see a list of fields:

| Field | What to do |
|---|---|
| `bot_token` | **Replace `PASTE_YOUR_TELEGRAM_BOT_TOKEN_HERE` with the token from Step 1** |
| `default_style` | Leave as `UGC With People` (the most popular style) |
| `veo_model` | Set to `veo3_fast` for testing (cheap), `veo3` for premium output |
| `aspect_ratio` | `9:16` for Reels/TikTok, `16:9` for YouTube/horizontal |
| `sheet_id` | Leave empty — auto-fills on first run |
| `max_revisions` | `3` is fine |
| `max_poll_iterations` | `8` is fine |

Click outside the node to save. Then **save the workflow** (Cmd/Ctrl+S).

### Optional: Brand Kit

Click open the **🎨 Brand Kit** node and fill in any fields you care about (brand voice, target audience, banned words, etc.). Leaving them blank is fine — the workflow handles missing values.

### Optional: Style Packs

Click open the **📋 Style Packs** node to see the 4 system prompts. You can edit any of them, or replace the casting demographics in the Pick Prompt code if you want a different distribution.

---

## Step 10 — Activate and test

1. Toggle the workflow to **Active** using the switch at the top right of the canvas
2. n8n will automatically register the webhook with Telegram
3. Open Telegram, search for your bot by username (e.g. `@mycompany_ugc_bot`)
4. Hit **Start**
5. Send a product photo. Optionally add a caption like `morning routine` or `for fitness Instagram`
6. Within ~2 seconds the bot should reply: *"Got it! Analyzing your photo and crafting prompts... ⚡"*

If you see that reply, the trigger is wired correctly. The full first run takes 3-4 minutes through all the gates.

### What you'll see, in order

1. *"Got it! Analyzing your photo..."*
2. *"Got your photo! Pick a style:"* → tap **Respond** → pick a style from the dropdown → submit
3. *"STYLE: UGC With People — CONCEPT: ... — CAPTION: ..."* → tap **Respond** → choose Yes or No → optionally add comments → submit
4. *"✅ Approved! Generating 3 image variants..."* (60-90 second wait)
5. Three product images arrive as a media group
6. *"Pick your favorite variant..."* → tap **Respond** → pick 1, 2, 3, or Regenerate
7. *"Generate the video from this image?"* → tap **Respond** → Yes (uses the Veo credit) or No
8. *"🎬 Generating video with Veo 3.1..."* (90-120 second wait)
9. The 8-second video arrives, followed by the caption to copy-paste
10. **First run only:** *"📋 First-run setup: I created a Google Sheet..."* with a sheet ID

---

## Step 11 — First-run sheet setup

After your very first successful video delivery, the bot DMs you a Google Sheet ID like:

```
1aBcDe...XyZ12345
```

Copy that ID, paste it into **⚙️ Config → sheet_id**, then save the workflow. From then on, every ad you generate is automatically logged to that sheet (timestamp, brief, prompts, image URL, video URL).

If you skip this step, the workflow will create a new sheet on every run instead of appending to the existing one.

---

## Troubleshooting

### "Bad request - please check your parameters" on Analyze Photo
The OpenAI node's `imageUrls` field is empty. Open the node, click **Expression** in the URL field, paste:
```
=https://api.telegram.org/file/bot{{ $('⚙️ Config').first().json.bot_token }}/{{ $json.result.file_path }}
```

### "401 Unauthorized" from Kie.ai
Three possible causes, in order of likelihood:
1. **IP whitelist on your Kie.ai key.** Go to https://kie.ai/api-key, click your key, disable the IP whitelist (or add your n8n server's IP).
2. **Credential format wrong.** Open your "Kie.ai" credential in n8n Credentials → confirm Name is exactly `Authorization` (capital A) and Value is `Bearer ` + your key (with one space, no quotes).
3. **Out of credit.** Check https://kie.ai/recharge — top up if balance is low.

Test your key outside n8n with this PowerShell command (replace `YOUR_KEY`):
```powershell
$h=@{Authorization="Bearer YOUR_KEY";"Content-Type"="application/json"}
Invoke-RestMethod -Uri "https://api.kie.ai/api/v1/chat/credit" -Headers $h
```
If you get a credit balance back, the key works. If you get 401, the key itself is bad.

### Bot doesn't respond at all
- Make sure the workflow toggle says **Active** (top right of canvas)
- Open the **Telegram Trigger** node and confirm the credential is selected from the dropdown
- Wait 30 seconds after activating — Telegram's webhook registration sometimes takes a moment
- Make sure you're DM'ing the bot directly, not in a group (groups need additional setup)

### Bot replies "Send a product photo with a caption"
You sent text without an attached photo. Send an actual photo file. A caption is optional.

### Workflow runs but only 1 or 2 of the 3 variants appear in the media group
This is the variant race condition. The Aggregate Variants node uses a staticData accumulator that should hold all 3 — but if the Kie.ai queue is congested and one variant takes >4 minutes, that variant times out and the workflow stalls. Re-send the photo to start a fresh run.

### "Cannot read properties of undefined (reading 'taskId')"
Your Kie.ai HTTP request is being rejected before n8n can read the response. Same as the 401 above — check IP whitelist and credit balance.

### The video has a different person than the image
This is a Veo 3.1 limitation — it sometimes drifts from the reference frame even when given a clear character description. The workflow already includes the casting in both prompts to mitigate this, but if it keeps happening, try regenerating the variant first (the second image often anchors better).

### Cost surprises
The `veo3` model is $2 per video. If you're testing, switch to `veo3_fast` ($0.30/video) by editing **⚙️ Config → veo_model**. You can flip back to `veo3` once you have a winning concept.

### Audio is silent or garbled
Veo 3.1's native audio is experimental and occasionally produces dud audio. Regenerate. If it persists across multiple runs, try simplifying the dialogue in the video prompt — long sentences fail more often than short ones.

### "App isn't verified" warning during Google OAuth
Normal for personal OAuth apps. Click **Advanced → Go to n8n UGC Bot (unsafe)**. The "unsafe" warning is misleading — it just means Google hasn't reviewed the app, not that it's actually unsafe.

### Webhook errors after duplicating the workflow
If you duplicate the workflow, all the webhook IDs collide with the original. Delete the duplicates' webhook IDs (Telegram Trigger node + every Wait node) and re-save — n8n will regenerate them.

### "$('Node Name') hasn't been executed" errors
Means the workflow tried to read from a node that hasn't run yet in this execution. Usually caused by the Config nodes not being chained correctly. Make sure the connections are: **Telegram Trigger → ⚙️ Config → 🎨 Brand Kit → 📋 Style Packs → Load Config → Received → ...**

---

## What to do next

Once you have it running:

1. **Set the Brand Kit fields** to make outputs feel like your brand
2. **Try all 4 styles** with the same product to see which fits your aesthetic
3. **Generate a few ads on `veo3_fast` first** to learn the prompt patterns that work for your products
4. **Switch to `veo3` for your final keepers** — the quality difference is noticeable

If you want to extend the workflow:
- **Multi-language captions:** revert the `captionLang` change in Pick Prompt code (one line)
- **Different gender mix:** edit the casting roulette in Pick Prompt
- **Add new styles:** add a field to **📋 Style Packs** AND add the style name to the **Ask Style** dropdown
- **Add a watermark:** Veo 3.1 supports a `watermark` parameter — add it to the **Create Veo** body
- **Change to 16:9 horizontal:** edit `aspect_ratio` in Config

---

## Stuck?

If you've worked through troubleshooting and still can't get past a step, screenshot the error AND the input panel of the failing node, then look up the error message in n8n's community forum: https://community.n8n.io
