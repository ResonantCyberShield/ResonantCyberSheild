# VeriHuman — Base44 Build Prompt (v2)

Rewritten from the original Expo/React Native Claude Code prompt so it can be built in Base44 (React web app, wrappable for Google Play later). The prompt itself starts at the `---` line below; everything above it is notes for you.

## What changed from the Expo version, and why

| Problem in v1 | Fix in v2 |
|---|---|
| Written for Expo (expo-sqlite, expo-secure-store, react-native-iap, expo-dev-client). None of these exist in Base44. | Re-targeted to Base44: entities with per-user access instead of SQLite, a backend function for entitlements, and browser APIs for the checks. |
| The audio check was "the hard technical part" because React Native has no `AudioContext`. | In a web app the Web Audio API is native, so the website's FFT approach ports directly. The main technical risk is gone. |
| The heuristics only said "same as the website". A fresh builder can't see the website, so it would invent them. | The marker lists, scam-pattern categories, audio thresholds and scoring formulas are written out in full. |
| "Confidence Score" vs. a ring that is coral (bad) at 67+. A higher "confidence" number was worse, which contradicts itself. | Renamed to **Concern level**. 0–100, where higher means more warning signs, and the thresholds and colors stay the same. Every result is phrased the same way. |
| The contact's score was "computed live" but the formula was never given. | Formula defined (recency-weighted, Pro scans weighted higher, with the scan count shown). |
| "Dark mode swaps these, don't just invert" was asked for, but no dark values were given. | A full dark token set is provided. |
| The Pro "server-side deepfake classifier" was never named, which invites a fake result. | A provider-adapter backend function. Messages use the Base44 LLM. Photo/voice use an external detection API if a key is configured, and otherwise show "Verified Scan unavailable". It never shows a made-up number. |
| The device trial record relied on iOS Keychain surviving reinstall, which the web can't do. | A server-side TrialLedger keyed by a hashed email, writable only by a backend function. It is still lightweight and has no fingerprinting. |
| Subscription status could be edited client-side. | `subscriptionStatus` is written only by a backend function, so the client can only read it. Payment provider: Stripe on the web, with a note that Play Billing is required once the app is wrapped for Play. |
| No guidance for when a check comes back high-concern. | A calm "What you can do" block: reverse image search, ask for a live video call, never send money or gift cards, and report links (FTC, IC3). |
| Play Store compliance was left until the end. | Back-stack, optimistic UI, safe-area, pull-to-refresh and dark-mode rules are built into the behavior notes from day one. |

**Things to set up after the build:**
1. Add a `DEEPFAKE_API_KEY` secret (and the provider name) if you want Verified Scan for photo/voice. Without it, those two show as unavailable, which is honest.
2. Connect Stripe for web subscriptions.
3. Before submitting to Play, switch the purchase flow to Play Billing (Google requires it for digital subscriptions inside the app).

---

Build a mobile-first web app called "VeriHuman" that gives people a second opinion before they trust someone they have only met online: a dating-app match, a new social media contact, a "wrong number" texter. It runs three checks: (1) a photo check for signs of AI generation and missing camera metadata, (2) a voice-clip check for the spectral signs of cloned or synthetic speech, and (3) a message check against known romance-scam and pig-butchering script patterns. Unlike reverse-image tools or one-off detector websites, VeriHuman keeps a running history per contact, so a person's pattern across several photos, clips and messages builds up over time. Free checks run entirely in the browser and the input never leaves the device. Pro adds server-side "Verified Scans", a full-thread "Deep Analyze", and trend history per contact.

Require user accounts (email login). All data is private to the user who created it. Nothing is ever shared between users.

## Core concept: the Concern level

Every scan produces a **Concern level** from 0 to 100. Higher means more warning signs. It is never a probability that someone is fake. Bands, used everywhere:
- 0–33 **Low concern**: teal
- 34–66 **Some concern**: gold
- 67–100 **High concern**: coral

A Contact's **Concern level** is computed live from its scans, never stored or user-edited:
`weighted average of the contact's scan scores, where weight = recency × tier`. Recency is 1.0 for the newest scan, then multiplied by 0.85 for each older scan. Tier weight is 1.5 for Pro "Verified" scans and 1.0 for free "Quick" scans. Always show the scan count beside it ("based on 4 scans"). With only one scan, label it "Early read".

## Data model

- **Contact**: per user: label (text, e.g. "Mike from Hinge"), platform (enum: dating app, social media, messaging app, email, other), notes (text, optional), last_scan_at (date, updated when a scan is attached). The concern level is computed from Scans, not stored.
- **Scan**: per user: contact_id (reference to Contact, nullable, because a scan can run standalone and be attached later), type (enum: photo, voice, message, thread), tier (enum: quick, verified), score (number 0–100), band (enum: low, some, high, derived from score), signals (array of {name, value, weight, direction: "raises" | "lowers"}), summary (text, one plain sentence), engine (text, e.g. "On-device heuristics v1" or the server provider name), raw_input_ref (text, Verified scans only: the private file path sent to the server. Quick scans never store the input, only the result), created_at.
- **UserProfile**: per user, one record: display_name, theme_preference (enum: system, light, dark; default system), subscription_status (enum: none, trialing, active, expired, canceled; default none), trial_started_at (date, set once on the first gated-feature tap, not at signup), trial_ends_at (computed: trial_started_at + 14 days), subscription_platform (enum: web, android, ios), verified_upload_consent (boolean, default false). subscription_status and the trial fields are writable **only** by the backend function `entitlements`, never from the client.
- **TrialLedger**: service-role only, no client read or write: email_hash (SHA-256 of the lowercased email), trial_used (boolean, set true the moment a trial starts), first_seen_at. It exists only to stop delete-account-and-re-signup trial abuse. Store nothing else.

No seed contacts. The empty state does the onboarding (see Home).

## The three Quick Scan engines (run in the browser, nothing uploaded)

Each engine returns {score, signals[], summary}. Put each engine in its own pure module (`lib/engines/photo.js`, `voice.js`, `message.js`) so it can be tested and reused.

**Photo** (read the file as an ArrayBuffer with FileReader and decode it as latin1 text for string matching):
- AI-generator markers. Each one found adds 35, capped at 70: `c2pa`, `trainedAlgorithmicMedia`, `Stable Diffusion`, `stable-diffusion`, `Midjourney`, `DALL-E`, `DALL·E`, `Adobe Firefly`, `NovelAI`, `ComfyUI`, `InvokeAI`, `Imagen`, `Flux`, a PNG `tEXt` chunk named `parameters` or `prompt`, `Software: GPT-4o`.
- Camera markers. If none are found, add 25. If they are present, subtract up to 15: EXIF `Exif\0\0` header, `Make`/`Model` tags with a recognizable camera or phone brand (Apple, Samsung, Google, Canon, Nikon, Sony, Fujifilm, OnePlus, Xiaomi, Huawei), `FNumber`, `ExposureTime`, `ISOSpeedRatings`, GPS IFD.
- Re-save signals: add 10 if there is no EXIF at all and the dimensions are exactly 512, 768, 1024, 1536 or 2048 on a side (common generator outputs). Add 5 if the image is a screenshot-sized PNG with no metadata.
- Always add the caveat signal "Social apps strip metadata, so missing camera data alone means little" when the camera markers are absent.
- Clamp to 0–100.

**Voice** (Web Audio API `AudioContext.decodeAudioData`, mono mix, radix-2 Cooley–Tukey FFT, 2048-sample Hann windows with 50% overlap):
- Spectral flatness (geometric mean / arithmetic mean of the power spectrum, averaged over voiced frames). Very low variance of flatness across frames suggests synthesis: map a variance below 0.002 to +30.
- High-frequency energy ratio (energy above 8 kHz / total). Below 0.5% on a clip with a sample rate of 32 kHz or more suggests a band-limited vocoder: +25.
- Silence floor: perfectly digital silence (RMS < 1e-5) between words: +15 (real rooms have noise).
- Pitch-contour smoothness (autocorrelation pitch per frame, std-dev of frame-to-frame change). Unnaturally smooth: +20.
- Clips under 3 seconds: score capped at 50, with the signal "Clip too short for a strong read".
- Accept m4a, mp3, wav, ogg and webm. Also offer "Record a clip" via MediaRecorder for a voice note playing on speaker. If decoding fails, show a clear "Couldn't read this audio format" error. Never show a fake result.

**Message** (pure JS regex, case-insensitive, over pasted text). Each category that matches adds its weight once, with the matched phrase shown as the signal value:
- Move off-platform (WhatsApp, Telegram, Signal, Google Chat, "text me at", "my private number"): 20
- Money, crypto or investment (crypto, USDT, bitcoin, trading platform, "my uncle/aunt taught me", guaranteed returns, mining pool): 30
- Payment methods scammers use (gift card, Western Union, wire, Zelle, Cash App, Bitcoin ATM, "steam card"): 30
- Unverifiable far-away job (oil rig, offshore, deployed, military doctor, UN peacekeeping, contractor overseas, ship engineer): 15
- Emergency or fee (hospital, customs fee, stuck at the airport, frozen account, lawyer fee, inheritance): 25
- Love bombing early (soulmate, destiny, "never felt this way", "my wife/queen" plus fast timelines): 15
- Avoids video ("camera is broken", "can't video call", "bad connection"): 20
- Secrecy or urgency ("don't tell anyone", "keep this between us", "act now", "today only"): 15
- Wrong-number opener ("is this ___?", "sorry, wrong number" followed by continued chat): 10
- Clamp to 0–100. If there are no matches, the score is 5 with the summary "No known script patterns found. That's good, but it doesn't prove anything."

## Pro features

- **Verified Scan** (photo, voice, message): backend function `verifiedScan`. Before the first upload, show an in-context disclosure sheet: "This scan sends your file to our server and a detection provider. It is deleted after analysis." Require a tap on "I understand" and store verified_upload_consent. Upload with UploadPrivateFile, never as a public file. Provider adapter:
  - message: Base44 InvokeLLM with a JSON schema response {score, signals[{name, value, direction}], summary}, told to act as a cautious romance-scam analyst and never claim certainty.
  - photo and voice: call the external detection API configured in the `DEEPFAKE_API_KEY` and `DEEPFAKE_PROVIDER` secrets. If no key is configured, return `{unavailable: true}`. The UI then says "Verified Scan for photos/voice isn't set up yet. Your Quick Scan still works." Never fabricate a score.
  - Delete the uploaded file after the result comes back. Keep only raw_input_ref as a path label.
- **Deep Analyze** (thread): paste a whole conversation (or upload screenshots, read with InvokeLLM vision). The LLM returns a timeline of escalation stages (contact, rapport, isolation, crisis, ask) with quoted lines. It is shown as a vertical stage tracker plus the Concern ring.
- **Trend history**: the full timeline and trend chart on Contact Detail. Free users see the latest scan plus a blurred trend preview with an "Unlock trend history" button.

## Entitlements and trial (backend function `entitlements`)

- Actions: `status`, `startTrial`, `activate`, `cancel`, `expire`. Only this function writes subscription fields.
- `startTrial` is called lazily the first time a free user taps a gated feature, and only if both UserProfile.trial_started_at is empty and TrialLedger has no trial_used for this email hash. Otherwise go straight to the paywall.
- `status` recomputes live: trialing past trial_ends_at becomes expired.
- Payments: Stripe Checkout and webhook on the web, calling `activate`/`cancel`. Isolate this in a `billing` module, because Google Play Billing must replace it in the Android wrapper.
- No IP correlation, device fingerprinting, ad IDs or phone verification.

## Pages

1. **Home** (`/`): a header with the wordmark and theme toggle. A large "New Scan" button that doesn't require picking a contact. A list of Contact cards, each with a small Concern ring, label, platform chip, "last scanned 3 days ago" and scan count. Sort by most recently scanned. Search by label. Empty state: three large tiles (Photo, Voice, Message), each with its one-line explanation, plus "Try a sample message" which pre-fills an obvious crypto-scam example so the user sees a real result instantly. Pull to refresh.
2. **New Scan** (`/scan`, optional `?contact=id`): a three-segment picker (Photo / Voice / Message), each with one line on what it checks, shown before any input:
   - Photo: "Looks for AI-generator fingerprints and missing camera data."
   - Voice: "Listens for the flat, too-clean sound of cloned or synthetic voices."
   - Message: "Compares the text to common romance-scam scripts."
   Then the input (file picker, camera, record, or paste box) and a Quick / Verified toggle. Verified shows a Pro badge, and free users who tap it go to the trial/paywall flow. A one-line privacy note under the button: Quick says "Runs on your device, nothing uploaded." Verified says "Sent to our server for analysis, deleted afterward."
3. **Scan Result** (`/result/:id`): the signature Concern ring (below). Under it is a chip saying **Quick Scan · on-device** or **Verified Scan · server**, the engine name, then the one-sentence summary. Then the signal list: each row has a name, a mono value, and an arrow showing whether it raised or lowered concern. Then the disclaimer block, always visible: "This is a signal, not a verdict. A low level doesn't prove someone is real, and a high level doesn't prove they're fake." For some/high concern, add a "What you can do" card: ask for a live video call, reverse image search (links to Google Lens and TinEye), never send money, gift cards or crypto to someone you haven't met, talk it over with someone you trust, and report links (reportfraud.ftc.gov, ic3.gov). Actions: "Attach to a contact" (pick an existing one or create one inline), "Scan something else", and "Delete this scan".
4. **Contact Detail** (`/contact/:id`): the label (editable inline), the large live Concern ring with "based on N scans", a trend line chart (Recharts) of scan scores over time with the band colors as background zones, and a timeline of scans (newest first, type icon, tier chip, score). Tap a scan to open its result. "Run a new scan for this contact" pre-selects it. Free users see the latest scan and a blurred, locked trend. Edit notes, delete the contact (asks whether to keep the scans unattached or delete them too).
5. **Upgrade** (`/pro`): uses the same tokens as the rest of the app, with no separate paywall look. If trialing, show a plain-text banner "13 days left in your trial", with no countdown and no red. A two-column Free vs Pro comparison: Quick Scans (unlimited / unlimited), Verified Scans (– / ✓), Deep Analyze (– / ✓), Contact trend history (latest only / full), Data export and delete (✓ / ✓). Buttons: Start free trial (if eligible) or Subscribe, then Restore purchase. Price shown in mono. No dark patterns, no pre-checked upsells, and a "Not now" link that's as visible as the main button.
6. **Settings** (`/settings`): display name, theme (System / Light / Dark segmented control), subscription status with trial end date, Manage subscription (Stripe customer portal on the web, the Play subscriptions deep link on Android), Restore purchase, "Export my data" (downloads JSON of all Contacts and Scans), "Delete all my data" (type DELETE to confirm, works on any plan), Sign out, and links to Privacy and to "How the checks work" (a plain-language page explaining each engine and its limits).

Bottom tab bar on mobile: Home, Scan, Settings. Upgrade opens as a full page with a back arrow.

## Design direction

The tone is a calm, trusted friend who knows forensics. It is not a security-alarm product. These tokens carry over exactly from the existing VeriHuman website:

Light:
- paper #F6F7FA (app background), fog #EBEDF2 (cards and surfaces)
- ink #1E2340 (headlines, primary buttons), ink-2 #2B3160 (pressed and hover)
- char #262A36 (body text), char-soft #5B6072 (secondary text)
- teal #2E8B7C / teal-soft #E3F1EE (low concern)
- gold #C98A1F / gold-soft #FAF0DC (some concern)
- coral #E4664B / coral-soft #FBE7E1 (high concern, labeled "attention", never "danger")

Dark (follows the system by default, with a manual override saved in UserProfile and in localStorage for instant load. No flash on load):
- paper #12142A, fog #1C1F3A, ink #E8EAF6 (headlines), ink-2 #C5C9E8, char #D9DBE5, char-soft #9AA0B8
- teal #4FB3A2 / teal-soft #16332F, gold #E0A847 / gold-soft #3A2E15, coral #F08A72 / coral-soft #3D221D

Type (Google Fonts): **Source Serif 4** for headlines and the big score readout, **Inter** for body and UI, **IBM Plex Mono** for every number (scores, signal values, dates in timelines, prices). Numbers always use the mono face.

**Signature element: the Concern ring.** A circular SVG progress ring with a 10px stroke on a fog track. The arc fills clockwise to the score and is colored by band, with the score in large IBM Plex Mono centered inside and the band label ("Some concern") under it in Inter. On a result it animates from 0 to the score over 700ms ease-out once, and respects prefers-reduced-motion. The same component is reused small (40px) on contact cards. It fits because the app's whole promise is a measured reading, not a stamp: a dial showing how far the needle moved, never a "FAKE" or "REAL" badge.

Avoid: hacker green-on-black, shield-and-padlock security clichés, red alert banners, generic purple-gradient SaaS, and stock photos of couples. No skull, warning-triangle or siren icons. Use Lucide icons at 1.5px stroke.

## Behavior notes

- The core loop is never gated: running Quick Scans, viewing contacts and viewing past results always works, whatever the subscription state.
- A lapsed trial or subscription never hides or deletes history. It only locks new Verified Scans, Deep Analyze and the full trend view. Past Verified results stay visible.
- Quick Scan input never leaves the device. Only the result (score, signals, summary) is saved to the account. Verified Scan uploads are disclosed at the moment they happen, not only in Settings.
- Every result screen shows the "signal, not a verdict" disclaimer. No copy anywhere says "fake", "real", "scammer", "verified human" or gives a percentage chance. Say "warning signs" and "concern level".
- Concern levels are always computed (per scan by its engine, per contact by the formula above). They are never a user-editable field.
- Per-user data scope on every entity (users only read and write their own records). TrialLedger is service-role only. Subscription fields are written only by the `entitlements` function.
- Mobile-first, built for Play Store wrapping: one consistent back stack (the hardware or browser back goes to the previous screen, and from a result goes back to where the scan started, never exiting the app from a nested screen). Respect safe-area insets. Tap targets of 44px or more. Pull-to-refresh on Home and Contact Detail. Optimistic updates for attach, rename, delete and theme changes, with rollback and a toast on failure. Skeleton loaders instead of spinners on lists.
- Scans finish visibly: show a short step-by-step progress line ("Reading file… Checking metadata… Scoring…") so the user can see work happened. Analysis must never be faked or artificially delayed.
- Accessibility: band colors always appear with a text label, never color alone. WCAG AA contrast in both themes. The ring has an aria-label like "Concern level 72 of 100, high concern".

## Audit checklist: run this before calling it done

Don't stop at "it builds and the screens render". Check each of these specifically:

- **Quick Scan input really stays on the device.** The engines in `src/lib/engines/` must not import the Base44 client or call fetch, upload or LLM functions. The only network write in the Quick path should be saving the *result* to the Scan entity, and the UI copy must say exactly that.
- **The Concern level can't be set directly.** Contact has no score field. The contact score comes only from its Scans through the weighted formula.
- **The trial starts only on a real gated-feature tap.** It must not start at signup, on page load or on the `status` call. A fresh user who stays idle must still have no `trial_started_at`.
- **The trial-abuse backstop survives account re-creation.** TrialLedger is keyed by a hashed email, can be written only by the service role, and isn't touched by "Delete all my data". It never relies on localStorage.
- **Subscription state can't be self-granted.** No client-callable function or entity RLS rule lets a user set their own status to active or trialing. Only the signed Stripe webhook (later, Play Billing) can activate. Verified Scan and Deep Analyze check Pro on the server.
- **A lapsed trial or subscription never hides history.** The full scan timeline and past Verified results stay visible and exportable. Only new Pro actions and the trend chart lock.
- **Export and delete work on a free account.** Test them on an account that has never been Pro.
- **Every Scan Result renders the "signal, not a verdict" disclaimer**, including high-concern and Deep Analyze results.
- **Every upload to the server is disclosed before it happens**, for all Verified and Deep Analyze types, including message text.
- **The concern colors stay distinct and readable in both themes.** In light mode, gold and coral fall below text contrast on fog, so they must carry the band on the ring, a dot or the chip, never on the text itself.
- **Play Billing before Play submission.** Stripe is web-only. A wrapped Android build must switch to Play Billing for in-app subscriptions.
