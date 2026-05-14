# UGC Ad Generator

A Telegram bot that turns a product photo into a polished short-form video ad — image variants, AI video with synced audio, and a ready-to-paste caption — in under 4 minutes.

Built on n8n. Drop the workflow in, connect 5 credentials, DM your bot a product photo. It does the rest.

---

## What it does

You send the bot a product photo. It walks you through 5 quick decisions and delivers a finished ad:

1. **Photo arrives** → bot acknowledges, GPT-4o analyzes the product (brand, category, colors, use case)
2. **Pick a style** → 4 options: B-Roll, UGC With People, Cinematic, Custom
3. **Concept preview** → bot drafts a creative direction + caption, you approve or send revision notes
4. **3 image variants** → NanoBanana Pro generates 3 versions of your product image in parallel
5. **Pick your favorite** → tap 1, 2, 3, or hit Regenerate
6. **Confirm video** → costs ~$2 to generate, so this gate prevents accidental spend
7. **Veo 3.1 generates an 8-second vertical video** with native audio
8. **Done** → bot sends the video + caption ready to copy-paste to TikTok/Reels

Every run is logged to a Google Sheet (auto-created on first run) so you can browse your history.

---

## What's under the hood

| Layer | Model | Provider |
|---|---|---|
| Photo analysis | GPT-4o | OpenAI |
| Prompt generation + revisions | Claude Sonnet 4.6 | OpenRouter |
| 3 image variants | NanoBanana Pro (Gemini 3 Pro Image) | Kie.ai |
| 8s video with audio | Veo 3.1 (Quality or Fast) | Kie.ai |
| Sheet logging | Google Sheets API | Google |
| Bot interface | Telegram Bot API | Telegram |

---

## Cost per ad

The video generation dominates the cost. You control which video tier to use via the **veo_model** field in the Config node.

### Veo 3.1 Quality (`veo3`) — premium tier
| | |
|---|---|
| GPT-4o photo analysis | ~$0.003 |
| Claude Sonnet 4.6 (prompt + revision) | ~$0.02 |
| 3× NanoBanana Pro 1K | ~$0.27 |
| Veo 3.1 Quality 8s vertical | ~$2.00 |
| **Total** | **~$2.30** |

### Veo 3.1 Fast (`veo3_fast`) — budget tier
| | |
|---|---|
| (same upstream costs) | ~$0.29 |
| Veo 3.1 Fast 8s vertical | ~$0.30 |
| **Total** | **~$0.60** |

**Recommendation:** start every project on `veo3_fast` to iterate cheaply. Switch to `veo3` once you have a winning concept and want the polished version. Toggle in `⚙️ Config` → `veo_model`.

> Pricing on Kie.ai changes occasionally. Verify current rates at https://kie.ai/pricing before committing to high volume.

---

## Customization

You edit values in **3 nodes**, never logic:

### ⚙️ Config
- `bot_token` (REQUIRED) — your Telegram bot token from @BotFather
- `veo_model` — `veo3` or `veo3_fast`
- `aspect_ratio` — `9:16` (vertical for Reels/TikTok) or `16:9` (horizontal for YouTube)
- `default_style` — fallback style if user skips the picker
- `sheet_id` — auto-fills on first run
- `max_revisions` — how many "No, change it" attempts before bot resets the conversation
- `max_poll_iterations` — how many times to check on a slow generation before timing out

### 🎨 Brand Kit (optional — leave blank for generic output)
- `brand_voice` — e.g. "playful, sarcastic, Gen-Z energy"
- `target_audience` — e.g. "men 25-40 into fitness"
- `brand_colors` — e.g. "navy, off-white, mustard"
- `banned_words` — comma-separated words the AI must never use
- `cta_default` — e.g. "Link in bio 👇"

When any of these are filled, they get injected into every prompt.

### 📋 Style Packs
The 4 system prompts that drive each style. Each is a complete creative brief sent to the agent. Edit any of them, or add new ones — but if you add a new style, also add it to the **Ask Style** node's dropdown options.

---

## Built-in casting diversity (UGC With People style)

When the user picks "UGC With People," the workflow randomly assigns ethnicity, age, and gender BEFORE the AI agent runs — instead of asking the agent to "be diverse" (which doesn't work because each run is stateless).

Default mix:
- **75%** Caucasian European (Slavic, Mediterranean, Northern, Central, Romanian/Balkan rotation)
- **25%** other (Latina, Black, East Asian, South Asian, Middle Eastern, Southeast Asian, Mixed heritage rotation)
- **Age 22-37** (rotated across 7 specific values for variation)
- **Gender:** woman (hardcoded — change in Pick Prompt code if you want men or mixed)

To change the mix, edit the casting roulette in the **Pick Prompt** node.



## Privacy and security

- The workflow processes images in your n8n instance. Photos are sent to OpenAI for analysis and Kie.ai for generation. Both are deleted from those services after processing per their respective retention policies.
- Generated image and video URLs are short-lived (typically 14 days on Kie.ai). Save anything you want to keep.
- Your API keys are stored in n8n's credential vault — never copy them into prompts, captions, or shared screenshots.
- The Google Sheet log is private to your Google account. No data leaves it.

---

## Limitations and known quirks

- **First-run sheet creation requires you to manually paste the sheet ID into Config.** The bot DMs you the ID after the first successful run. If you skip this step, every subsequent run will create a new sheet.
- **Variant generation is racy under load.** The workflow uses a staticData accumulator to wait for all 3 images, but if Kie.ai's queue is congested and one image takes >4 minutes, that variant will time out and the workflow stalls. The fix is to retry — the failed variants don't carry over.
- **The webview "Respond" link.** When the bot asks you to pick a style or approve a concept, it sends a "Respond" button that opens a tiny web form. This is n8n's native UX for sendAndWait, not a custom thing. Most users adapt within 2-3 messages.
- **Cost surprises.** A single accidental click on Veo Quality is $2. Run on Fast until you trust the workflow, then switch.

---

## Updating the workflow

Periodically the underlying APIs change. Most likely changes you'll need to track:
- **Kie.ai pricing** — check https://kie.ai/pricing if costs feel off
- **Veo model names** — `veo3` and `veo3_fast` are current; future models may use new identifiers
- **NanoBanana model identifier** — currently `nano-banana-pro`, may evolve

If the bot stops working with no other change on your end, an upstream model identifier likely shifted. Check Kie.ai's documentation and update the relevant HTTP node.

---

## Need help?

See [SETUP.md](./SETUP.md) for the step-by-step install guide, including how to create the Telegram bot from scratch and where to grab every API key.

Hit a wall? Common errors and fixes are at the bottom of SETUP.md under **Troubleshooting**.
