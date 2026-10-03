# VeriHuman — Claude Code Build Prompt (React Native / Expo), v2

This is the native-app version of the prompt, rewritten using what the Base44 build and its audit taught us. Paste everything below the line into Claude Code to start a new project. It is self-contained.

Differences from v1: the score is renamed and its formulas are spelled out, the scan heuristics are written in full with the false-positive fixes from testing, dark-mode values are added, there is a backend and security model, and the audit checklist now comes with concrete test cases.

---

## What to build

VeriHuman is a mobile app that gives people a second opinion before they trust someone they have only met online: a dating-app match, a new social media contact, a "wrong number" texter. It runs three checks:
- a **photo** check for AI-generator fingerprints and missing camera metadata,
- a **voice-clip** check for the spectral signs of cloned or synthetic speech,
- a **message** check against known romance-scam and pig-butchering scripts.

Free users get unlimited on-device Quick Scans. Pro users (subscription with a 14-day trial) get server-side Verified Scans, full-thread Deep Analyze, and per-contact trend history. A web version of the app already exists, built on Base44. This native app is its sibling, so carry over the tokens, rules and copy below exactly.

## Tech stack

- **Expo + TypeScript + Expo Router.** Use `expo-dev-client` and EAS Build from day one. Expo Go cannot run in-app purchases or the native audio decoding.
- **Local data:** `expo-sqlite` (Contacts, Scans) behind a typed data layer. `expo-secure-store` holds the device trial record, because the Keychain survives reinstall on iOS. On Android, SecureStore is wiped on uninstall, so the server-side ledger below is the real backstop there.
- **Purchases:** `react-native-iap` (StoreKit 2 / Play Billing). The store is the source of truth, but **always validate receipts on the server** (App Store Server API / Google Play Developer API) before granting Pro. Never trust a client-side purchase event alone.
- **Backend:** a small serverless API (any host) for receipt validation, the trial ledger, Verified Scan and Deep Analyze. See the security section.
- **Fonts:** `@expo-google-fonts/source-serif-4`, `inter`, `ibm-plex-mono`.
- **Gradients:** `expo-linear-gradient`. **Colored glow on Android:** `react-native-shadow-2`.
- **Concern Ring:** `react-native-svg`.
- **Audio:** try `react-native-audio-api` (Software Mansion) first, because its `decodeAudioData` lets the web FFT code port unchanged. If it can't decode m4a/mp3/ogg/opus, fall back to decoding to PCM with a native decoder and running the same FFT. Write down which path you took and why. Never drop or fake the voice check: if decoding fails, show "Couldn't read this audio format".
- Run voice analysis **off the JS thread** (a worklet or worker). In testing, a 60-second clip took 5.6 s on a server CPU on the main thread.

## Core concept: the Concern level

Every scan produces a **Concern level** from 0 to 100. Higher means more warning signs. It is never a probability that someone is fake, and it is never called a "confidence" or "trust" score.
- **0–33 Low concern:** cyan (safe).
- **34–66 Some concern:** amber (caution).
- **67–100 High concern:** magenta (risk).

**Contact Concern level** is always computed live, never stored:
- It is a weighted average of the contact's scan scores, where weight = recency × tier.
- Recency is 1.0 for the newest scan and ×0.85 for each step older.
- Tier weight is 1.5 for Verified and 1.0 for Quick.
- Show "based on N scans" alongside the level, or "Early read" when there is only one scan.

## Quick Scan engines (on-device; the input never leaves the phone)

**Message** (case-insensitive regex). Each category that matches adds its weight once, and the matched phrase becomes the signal value, capped at 40 characters.

| Category | Weight | Patterns |
|---|---|---|
| Move off-platform | 20 | WhatsApp, Telegram, Google Chat, "text me at", "my private number". Match Signal only as an app ("on signal", "signal app", "add me on signal"), never bare "signal". |
| Money / crypto / investment | 30 | crypto, USDT, bitcoin, trading platform, "my uncle/aunt taught me", guaranteed returns, mining pool |
| Scam payment methods | 30 | gift card, Western Union, wire, Zelle, Cash App, Bitcoin ATM, steam card |
| Unverifiable far-away job | 15 | oil rig, offshore, deployed, military doctor, UN peacekeeping, contractor overseas, ship engineer |
| Emergency or fee | 25 | hospital, customs fee, stuck at the airport, frozen account, lawyer fee, inheritance |
| Early love bombing | 15 | soulmate, destiny, "never felt this way", "my wife/queen" |
| Avoids video | 20 | "camera is broken", "can't/can’t video call", "bad connection" |
| Secrecy or urgency | 15 | "don't/don’t tell anyone", "keep this between us", "act now", "today only" |
| Wrong-number opener | 10 | "is this <Name>?" only at the start of the message, or "sorry, wrong number". Never any question. |

Clamp the score to 0–100. With no matches the score is **5**, with the summary "No known script patterns found. That's good, but it doesn't prove anything."

**Photo** (read the bytes and scan them as latin1 text):
- **AI markers:** +35 each, capped at 70. Markers: `c2pa`, `trainedAlgorithmicMedia`, `Stable Diffusion`, `stable-diffusion`, `Midjourney`, `DALL-E`, `DALL·E`, `Adobe Firefly`, `NovelAI`, `ComfyUI`, `InvokeAI`, `Imagen`, `Flux`, `Software: GPT-4o`, and a PNG `tEXt` chunk keyed `parameters` or `prompt`.
- **Camera credit:** −5 each, −15 at most, and **only from a real EXIF block**. That means an `Exif\0\0` header with Make/Model *inside the EXIF block* naming a camera or phone brand, plus FNumber, ExposureTime, ISOSpeedRatings and the GPS IFD. Brand names that appear in ICC or XMP data (for example "Apple Computer Inc." in a Display P3 profile) never count.
- **No camera data:** +25, plus the context signal "Social apps strip metadata, so missing camera data alone means little".
- **No EXIF and a generator-sized side** (512, 768, 1024, 1536 or 2048): +10.
- **Screenshot-sized PNG with no metadata:** +5.

**Voice** (mono mix; radix-2 FFT; 2048-sample Hann windows with 50% overlap; analyse the first 60 s at most; run pitch detection on every 4th frame):
- **Spectral-flatness variance below 0.002:** +30.
- **Energy above 8 kHz below 0.5%**, when the sample rate is at least 32 kHz: +25.
- **Perfect digital silence** (RMS below 1e-5): +5 only, plus the context signal "Messaging apps often gate silence, so this alone means little".
- **Pitch contour smoother than 8 Hz std-dev:** +20.
- **Clips under 3 s:** the score is capped at 50, with the signal "Clip too short for a strong read".

## Data model

The app keeps data locally in SQLite. A server mirror is needed only for Pro and entitlement state.
- **Contact:** id, label, platform (dating app / social media / messaging app / email / other), notes, created_at, last_scan_at. It has no score column.
- **Scan:** id, contact_id (nullable), type (photo / voice / message / thread), tier (quick / verified), score, band, signals[] ({name, value, weight, direction}), stages[] (thread only), summary, engine, raw_input_ref (Verified only, a label and never the content), created_at.
- **User (server):** id, email, display_name, created_at.
- **Entitlement (server; writable only by the server):** user_id, status (none / trialing / active / expired / canceled), trial_started_at, trial_ends_at (computed as +14 days), platform, original_transaction_id. The app can only read it.
- **TrialLedger (server-only):** email_hash, store_account_hash, trial_used, first_seen_at. It exists to stop delete-and-reinstall abuse and stores nothing else.
- **DeviceTrialRecord (SecureStore):** an opaque device_id, trial_used, first_seen_at. It is a lightweight backstop only.

## Screens

1. **Home:**
   - Contact cards, each with a small Concern Ring, the platform, "last scanned …", and the scan count.
   - A prominent "New Scan" button.
   - Empty state: three tiles (Photo, Voice, Message) plus "Try a sample message".
2. **New Scan:**
   - A Photo / Voice / Message / Thread picker, with one line under it explaining each check.
   - A Quick / Verified toggle.
   - Privacy line for Quick: "Your file or text stays on your device. Only the result is saved."
   - For Verified, show the consent sheet before *any* upload, for every type, message text included.
3. **Scan Result:**
   - The Concern Ring, with the score in IBM Plex Mono, then a "Quick Scan · on-device" or "Verified Scan · server" chip, then the summary and the signal list.
   - The disclaimer is always shown: "This is a signal, not a verdict. A low level doesn't prove someone is real, and a high level doesn't prove they're fake."
   - For Some or High concern, a "What you can do" card: ask for a live video call; reverse image search (Google Lens, TinEye); never send money, gift cards or crypto; talk it over with someone you trust; report at reportfraud.ftc.gov or ic3.gov.
   - Actions: attach to a contact, scan again, delete.
   - For Deep Analyze results, a vertical stage tracker: contact → rapport → isolation → crisis → ask, each with a quoted line.
4. **Contact Detail:**
   - The live ring with "based on N scans", the trend chart with band zones, and the full scan timeline.
   - The timeline is always visible on every plan. Only the trend chart is Pro.
5. **Upgrade:**
   - Uses the same tokens as the rest of the app.
   - A plain "N days left in your trial" line, with no countdown and no red.
   - A Free vs Pro table.
   - Buttons: Start trial / Subscribe, Restore purchases, and a "Not now" that's just as visible.
6. **Settings:**
   - Theme (System / Light / Dark), subscription status, Manage subscription (a store deep link), Restore purchases.
   - Export data (JSON) and Delete all data. Both work on any plan, and delete is confirmed by typing DELETE.
   - Sign out, plus "How the checks work".

## Design tokens: "Night Scanner, Bold" (carry over exactly)

Dark mode is the main presentation. The boldness is deliberate: color should cover a large part of every screen (gradient headers, glowing card borders, filled gradient buttons). Think Cash App or Robinhood, not a muted dark app with one accent color.

Dark (primary):

| Token | Value | Use |
|---|---|---|
| bg | #0B0F1A | App background |
| surface (card) | #141A2C | Cards; clearly lifted from the background |
| ink | #EAF6FF | Text |
| ink-soft | #7C94AA | Secondary text |
| safe (cyan) | #00E5FF, soft fill #07312F | Low concern |
| caution (amber) | #FFC24B, soft fill #3A2A0A | Some concern |
| risk (magenta) | #FF3D71, soft fill #3A0E1C | High concern |
| violet | #6C4CFF | Decorative gradient bridge only; never a status color |
| gradient | linear 135°: #00E5FF 0% → #6C4CFF 55% → #FF3D71 100% | Header band, primary buttons, Upgrade hero |
| gradient-ink | #061018 | Text on the gradient |

Light (secondary):

| Token | Value | Use |
|---|---|---|
| bg | #F7F9FB | App background |
| surface | #EDF1F5 | Cards and sheets |
| ink | #0E1420 | Text |
| ink-soft | #5B6578 | Secondary text |
| safe (cyan) | #008299, soft fill #E0F7FA | Low concern |
| caution (amber) | #B9770E, soft fill #FDF1DD | Some concern |
| risk (magenta) | #D81B60, soft fill #FCE4EC | High concern |
| gradient | linear 135°: #00E5FF → #6C4CFF 55% → #D81B60 | Header band, primary buttons, Upgrade hero |
| gradient-ink | #061018 | Text on the gradient |

**Where the gradient goes:** the Dashboard header band, every primary button (filled with `expo-linear-gradient`, never outlined), and the Upgrade/Pro hero. These are the most visible surfaces in the app and should look bold.

**Cards:** every Contact card and signal row gets a border tinted with its own concern color at about 20–30% opacity. A safe card should read cyan at a glance and a risk card magenta, before any text is read.

**Signature glow (dark mode only):**
- The Concern Ring's active arc gets two layers of glow in its own color: a tight inner glow (about `0 0 16px`) plus a wide soft outer glow (about `0 0 32px`).
- Primary gradient buttons get the same two-layer glow.
- iOS: use `shadowColor` + `shadowRadius` + `shadowOpacity`.
- Android: use `react-native-shadow-2` or a blurred, colored view behind the element.
- Verify the glow on both platforms.
- Light mode drops the glow but keeps the gradient fills and the tinted borders.

**Contrast, measured; follow these rules:**
- **Dark:** concern colors on the surface pass for any text (cyan 12:1, amber 11.5:1, magenta 5.4:1).
- **Light:** concern colors fall below 4.5:1 on the surface. Use them for arcs, dots, borders and soft fills only. Text and numbers stay ink.
- **Gradient text:** `gradient-ink` on the gradient bottoms out around 3.7:1 (dark) and 3.3:1 (light) over the violet middle. Text on a gradient must therefore be large-text sized: at least 18px semibold, or 24px regular. Never put small captions on the gradient.
- **Accessibility:** a band is never shown by color alone. The ring's accessibility label reads like "Concern level 72 of 100, high concern".

**Theme:** `useColorScheme()` plus a theme context with three states (system / light / dark), stored locally.

**Type:** Source Serif 4 for headlines and the score readout, Inter for body text, IBM Plex Mono for every number.

**Motion:** the arc fills once over 700 ms. The glow is static. Respect reduced motion.

## Pricing (Pro tier)

There is one Pro entitlement with two store products:

| Plan | Price | Product ID |
|---|---|---|
| Monthly | $9.99/month | `com.verihuman.pro.monthly` |
| Annual | $79.99/year (≈ $6.67/mo, "save ~33%") | `com.verihuman.pro.annual` |

- Swap in the real bundle ID for the product IDs.
- **Upgrade screen:** show both plans side by side, with Annual preselected and a "Best value" badge.
- **Trial:** the 14-day free trial applies to whichever plan the user picks first.
- **Entitlement:** the server maps both product IDs to the same entitlement after validating the receipt.
- **Ads:** the free tier carries ads in compliant slots; Pro is ad-free.

## Security model (assume someone will attack this)

- **Pro is granted only on the server**, after receipt validation. No client call can set it to active or trialing.
- **Server entitlement records can only be written by the server.**
- **Verified Scan and Deep Analyze check entitlement on the server.**
- **The server creates Verified and Deep Analyze Scan records itself**, so a client can never forge a Verified result.
- **Uploads:**
  - Store them privately, and record each one against its uploader.
  - The server refuses any file the caller didn't upload.
  - Hand files to the model or detector only through short-lived signed URLs, and never return those URLs to the client.
- **Rate limits** on Pro endpoints: 5 per minute and 30 per day per user. Return 429 with a friendly message.
- **Input limits:**

  | Input | Limit |
  |---|---|
  | Text | 20,000 characters |
  | Screenshots | 5 |
  | Images | 10 MB; jpeg / png / webp / heic |
  | Audio | 15 MB and 3 minutes; mp3 / m4a / wav / ogg / webm |

- **Prompt injection:** wrap user conversations in delimiters and tell the model they're untrusted data. Clamp and validate every model output (score 0–100, known stage names, strings capped at 500 characters).
- **Error responses:** unauthenticated calls get a 401. Never return stack traces or raw error messages.
- **Store notifications:** verify App Store Server Notifications and Play RTDN (signature or JWS), and process them idempotently by event id.
- **No tracking:** no IP correlation, ad IDs, fingerprinting or phone verification.

## Build order

1. Scaffold the app with the dev client. Add the tokens and fonts, and get theme switching working.
2. Build the SQLite schema and data layer, with the concern formula and its unit tests.
3. Message engine, then the Scan Result screen and Concern Ring end to end.
4. Photo engine.
5. Audio decode decision, then the voice engine off the JS thread.
6. Backend (entitlements, ledger, receipt validation, Verified, Deep Analyze), then IAP, trial, paywall and Settings.

## Acceptance tests (automate these and run them before calling it done)

**Message:**
- The sample scam message scores ≥ 67.
- Benign text scores exactly 5.
- "I had bad signal on the train" scores < 20.
- "is this your dog in the photo?" scores 5.
- Curly apostrophes (don’t, can’t) still match.

**Photo:**
- An SD PNG with a `parameters` chunk scores ≥ 67.
- An iPhone JPEG with EXIF scores ≤ 33.
- A 1024² JPEG whose only metadata is an Apple ICC profile gets **no** camera credit.
- A garbage file doesn't crash.

**Voice:**
- A 5 s pure tone scores ≥ 67.
- A noisy, pitch-varying clip scores < 67.
- A 2 s clip scores ≤ 50.
- An undecodable file shows the exact error.
- A 60 s clip never blocks the UI thread.

**Concern:**
- Older 80 and newer 20 (both Quick) give 48.
- Older Verified 80 and newer Quick 20 give 54.
- A single scan reads "Early read".
- Band edges: 33 is low, 34 some, 66 some, 67 high.

**Privacy and gating:**
- A Quick Scan makes no network call except saving its result.
- The trial is still unset after signup and idling.
- History stays visible and exportable after expiry.
- Export and delete work on a never-Pro account.
- The disclaimer renders on every result.

**Security:**
- Every client path to activate Pro is rejected.
- A Verified call by a non-Pro user returns 403.
- A file that isn't yours returns 403.
- The 31st Verified call in a day returns 429.
- An unsigned store notification is rejected.
