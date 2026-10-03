# VeriHuman — Claude Code Build Prompt (React Native / Expo)

Paste everything below into Claude Code as the opening prompt for a new project. It's self-contained — Claude Code won't have any memory of the conversation that produced it.

---

## What to build

VeriHuman is a mobile app that gives someone a second opinion before they trust a stranger they've only met online — a dating app match, a new social media contact, anyone. It checks photos for AI-generation and camera-metadata signals, analyzes voice clips for the spectral signatures of cloned/synthetic audio, and scans message text against known romance-scam script patterns. Free users get unlimited heuristic checks running on-device. Pro users (subscription) get a real server-side deepfake classifier ("Verified Scan"), full-thread message analysis ("Deep Analyze"), and a running trust history on a given contact instead of one-off scans.

A companion static website with the same three checks already exists and is live — this app should look and feel like its mobile sibling, not a different product. Carry over the exact design tokens below rather than inventing new ones.

## Tech stack

- **Expo (React Native) + TypeScript.** Use Expo Router for navigation (file-based routing, matches the 6 screens below).
- **This app needs a custom dev client / EAS Build, not Expo Go** — two of its requirements (in-app purchases, and likely the audio analysis library) need native modules Expo Go doesn't support. Set up `expo-dev-client` from the start rather than discovering this halfway through.
- **Local persistence:** `expo-sqlite` for Contacts and Scans (structured, queryable, and this is real user data worth a real schema rather than AsyncStorage blobs). `expo-secure-store` specifically for the device trial record (see below) since it needs to survive app deletion/reinstall on iOS, which plain storage does not.
- **Subscriptions:** `react-native-iap` against StoreKit (iOS) and Play Billing (Android). Treat the store as the source of truth for subscription status — don't re-derive entitlement independently in app code; listen for the store's purchase/restore events and webhook-equivalent callbacks to update `subscriptionStatus`.
- **Fonts:** `@expo-google-fonts/source-serif-4`, `@expo-google-fonts/inter`, and `@expo-google-fonts/ibm-plex-mono` (all real, published Expo font packages) — load via `expo-font` at app start, same type pairing as the website.
- **Confidence Ring component:** `react-native-svg` for the circular progress ring described below.
- **Gradients:** `expo-linear-gradient` for the header bands and primary CTA buttons described in the design tokens section.
- **Colored glow/shadow:** `react-native-shadow-2` (or an equivalent) for the Android colored-glow fallback described below — plain `elevation` won't produce a visible color tint on Android.

## The hard technical part — be upfront about this

The website's audio check uses the browser's Web Audio API (`AudioContext.decodeAudioData` + a hand-rolled FFT) to measure spectral flatness and high-frequency energy in a voice clip. **React Native has no equivalent to `AudioContext`.** The most realistic path is `react-native-audio-api` (Software Mansion's RN port of Web Audio API concepts) — evaluate it first since it would let the existing FFT/spectral-flatness logic port over almost unchanged. If it doesn't support decoding the file formats you need, the fallback is reading raw PCM via a dedicated audio-decoding library and running the FFT (reuse a radix-2 Cooley-Tukey implementation, same approach as the website) directly against the decoded samples. Flag which path you took and why — don't silently drop the audio check or fake its output if neither library works cleanly.

The photo check and message check port over directly: the photo scan reads file bytes via `expo-file-system` and scans for the same AI-generator string markers and EXIF/camera markers as the website (plain byte/string matching, no special library needed); the message scan is pure JS regex against the same scam-pattern list and ports with no changes.

## Design tokens — "Night Scanner, Bold" (carry over exactly — don't reinterpret)

Dark mode is the primary/hero presentation for this app. The intensity level is deliberate: this is NOT a dark app with small color accents — color should cover a meaningful portion of each screen (gradient headers, glowing card borders, filled gradient buttons), closer to Cash App/Robinhood-style boldness than a muted dark-mode app with a single accent color.

```
Dark (primary):
  bg:             #0B0F1A
  surface (card): #141A2C   — distinctly lifted/colorful, not just a slightly lighter gray
  ink (text):     #EAF6FF
  ink-soft:       #7C94AA
  safe/cyan:      #00E5FF   (low concern)      soft-fill #07312F
  caution/amber:  #FFC24B   (mid concern)      soft-fill #3A2A0A
  risk/magenta:   #FF3D71   (high concern)     soft-fill #3A0E1C
  violet:         #6C4CFF   (decorative gradient bridge color ONLY — never a status color)
  gradient:       linear-gradient(135deg, #00E5FF 0%, #6C4CFF 55%, #FF3D71 100%)
  gradient-ink:   #061018   (text color ON TOP of the gradient — near-black, not white)

Light (secondary):
  bg:             #F7F9FB
  surface:        #EDF1F5
  ink (text):     #0E1420
  ink-soft:       #5B6578
  safe/cyan:      #008299   (low concern)      soft-fill #E0F7FA
  caution/amber:  #B9770E   (mid concern)      soft-fill #FDF1DD
  risk/magenta:   #D81B60   (high concern)     soft-fill #FCE4EC
  gradient:       linear-gradient(135deg, #00E5FF 0%, #6C4CFF 55%, #D81B60 100%)
  gradient-ink:   #061018
```

Use React Native's `useColorScheme()` plus a theme context, three states (system / explicit light / explicit dark), same as the website.

**Where the gradient goes:** the Dashboard header/top band, every primary CTA button (filled with the gradient, not outlined — `expo-linear-gradient` for the fill, `gradient-ink` for the button text), and the Upgrade/Pro screen's hero area. This should NOT be subtle — these are the highest-visibility surfaces in the app and are supposed to look bold.

**Card treatment:** every Contact card and signal row gets a visible border tinted toward its own status color (roughly 20-30% opacity of cyan/amber/magenta depending on that contact's concern level), not a neutral gray border. A "safe" card should look subtly cyan-tinted at a glance, a "risk" card subtly magenta-tinted, even before reading any text.

**The signature detail — don't skip this:** in dark mode, give the Confidence Ring's active arc a strong two-layer glow — a tight inner glow (~`0 0 16px`) plus a wider soft outer glow (~`0 0 32px`), both in the arc's own color. On iOS use shadowColor/shadowRadius/shadowOpacity; on Android, elevation alone won't produce a colored glow, so use a blurred colored View behind the ring or `react-native-shadow-2`. Apply the same two-layer glow treatment to the primary gradient CTA buttons in dark mode. In light mode, drop the glow (it doesn't read against a light background) but keep the gradient fills and the colored card borders — light mode should still look colorful, just without the glow effect specifically.

Type pairing: Source Serif 4 for headlines and score readouts, Inter for body/UI text, IBM Plex Mono for every numeric value (scores, signal values) — numbers should always render in the mono face.

## Data model

**User** — `email`, `displayName`, `subscriptionStatus` (`none` | `trialing` | `active` | `expired` | `canceled`), `trialStartedAt` (set once, lazy-started on first gated-feature tap, not at signup), `trialEndsAt` (computed from `trialStartedAt` + 14 days, never independently editable), `subscriptionPlatform` (`ios` | `android`), `createdAt`.

**DeviceTrialRecord** — keyed by device, not account: `deviceId` (opaque, generated once and stored in `expo-secure-store`/iOS Keychain specifically so it survives reinstall), `trialUsed` (boolean, set true the moment a trial *starts*, not at trial end), `firstSeenAt`. Stores nothing else — this exists only to stop delete-and-reinstall trial abuse, not to fingerprint the user.

**Contact** — `label` (user-chosen, e.g. "Mike from the dating app"), `createdAt`, `confidenceScore` (always computed live from its Scan rows — never a manually-entered field), `lastScanAt`.

**Scan** — `contactId` (nullable — a scan can run standalone before being attached to a contact), `type` (`photo` | `voice` | `message`), `tier` (`free` | `pro`), `score` (0–100, computed), `signals` (array of `{name, value}`, same shape as the website's signal list), `rawInputRef` (Pro scans only — a pointer to what was actually sent to the server classifier; free scans never store the input itself, only the result, matching the website's "nothing is uploaded" promise for the free tier), `createdAt`.

## Screens (Expo Router, 6 routes)

**1. Home / Dashboard** — list of Contacts as cards, each showing a small Confidence Ring and last-scan date. Prominent "New Scan" action that doesn't require picking a Contact first. Empty state shows the three check types directly rather than a blank list.

**2. New Scan** — three entry points: Photo, Voice clip, Message, each with one line explaining what it checks before the user commits anything (same "second opinion, not a verdict" framing as the website). Free tier runs the on-device heuristic engines; Pro tier routes the same input to the server classifier and visibly labels the result "Verified Scan" vs. free "Quick Scan" so the user always knows which engine produced a given number.

**3. Scan Result** — the Confidence Ring is the signature visual element: a circular SVG progress ring, cyan under 34 / amber 34–66 / magenta 67+ (identical thresholds to the website), glowing in its concern color in dark mode, numeric score centered inside it in IBM Plex Mono. Below it, the named-signal list and the same disclaimer block as the website ("This is a signal, not a verdict"). An "Attach to a Contact" action rolls the scan into that Contact's running Confidence Score instead of leaving it a one-off.

**4. Contact Detail** — full scan history as a timeline, with the Confidence Score's trend over time, not just the latest number — this is the concrete Pro-tier advantage over the free website (tracking a person across multiple scans vs. one-off checks).

**5. Upgrade / Pro** — paywall screen shown when a free user taps a gated feature (Verified Scan, Deep Analyze, or trend history beyond the latest scan). Reuses the app's existing tokens rather than a separate paywall visual style. Shows a trial-status banner when `trialing` ("13 days left in your trial" — plain text, no countdown animation, no red urgency styling). Shows both billing options from the Pricing section below ($9.99/mo and $79.99/yr, annual visually favored) side by side, plus a plain free-vs-Pro feature comparison mirroring the website's pricing section.

**6. Settings / Account** — subscription status, "Manage subscription" deep link to the relevant store's subscription page, "Restore purchases," sign-out, and a data export/delete action. Every user can export or delete their own Contact and Scan data regardless of subscription state — this is a free-tier right, not a Pro feature.

## Pricing — Pro tier

Single Pro tier, two billing options for the same entitlement:

- **Monthly: $9.99/month**
- **Annual: $79.99/year** (works out to ~$6.67/month — present this next to the monthly price as "save ~33%" on the Upgrade/Pro screen, not just as a second unexplained number)

This was benchmarked against the closest real competitor — Bitdefender's RealCheck, a consumer deepfake-detection app, prices its tiers at $4.99/mo and $12.99/mo. VeriHuman sits above RealCheck's entry tier because it does more (Deep Analyze full-thread scans, persistent per-contact trust history that RealCheck doesn't offer), but stays well under "people-search/background-check" apps like Social Catfish ($27–80/mo) and BeenVerified (~$33/mo) — those apps charge more because they're paying for data-broker lookups per search, which is a different cost structure than VeriHuman's on-device heuristics plus occasional server-side classifier calls. Don't price this like a background-check product; it isn't one.

Set up both as actual store products, not a single SKU with a client-side discount:

- `react-native-iap` needs two separate product identifiers registered in both App Store Connect and Google Play Console — one for the monthly auto-renewing subscription, one for the annual auto-renewing subscription, both mapped to the same entitlement/`subscriptionStatus` on the User record. Suggested product IDs: `com.verihuman.pro.monthly` and `com.verihuman.pro.annual` (swap in the real bundle ID).
- The Upgrade/Pro screen shows both options side by side (annual visually favored, e.g. a "Best value" badge, since that's the one worth nudging toward), not monthly-only with annual buried in Settings.
- The 14-day free trial (see Behavior rules below) applies to whichever option the user picks first — don't make the trial monthly-only or require monthly before upgrading to annual.
- Trial-to-paid conversion uses whichever product ID the user actually tapped; `subscriptionPlatform`/store webhook logic doesn't need to know the price, just the product ID and resulting entitlement state.

## Behavior rules (cross-cutting)

- Core actions (running a free Quick Scan, viewing Contacts, viewing past scans) always work regardless of subscription state — never gate the fundamental loop behind an entitlement check.
- A lapsed trial or subscription never deletes or hides historical Scan/Contact data — it only locks forward-looking gated features.
- Trial eligibility checks the store's platform account first; the `DeviceTrialRecord` in Secure Store is only a lightweight backstop against reinstall abuse — no IP correlation, ad-ID fingerprinting, or phone verification. This app is privacy-positioned; don't add tracking beyond what's specified here.
- Every result screen states plainly that this is a signal, not a verdict — don't let any copy imply certainty the underlying check can't back up.
- Free-tier scans never leave the device. Only Pro "Verified Scan" input is transmitted to a server, and that should be disclosed in-app at the moment it happens, not buried in settings.

## Build order suggestion

1. Scaffold the Expo project with `expo-dev-client`, set up theme tokens + font loading, confirm dark/light switching works before building any real screen.
2. Build the SQLite schema and a typed data-access layer for User/Contact/Scan/DeviceTrialRecord.
3. Port the message pattern scan first (pure JS, no native dependency risk) and get the Scan Result screen + Confidence Ring working end-to-end on that one check type.
4. Port the photo metadata scan (file-system read, same byte-matching logic).
5. Resolve the audio-analysis library question (`react-native-audio-api` or the PCM-decode fallback) before wiring up the voice-clip check — this is the one piece with real technical risk, so surface problems here early rather than late.
6. Wire up `react-native-iap`, the trial/paywall logic, and the Settings screen last, once the core scanning loop is solid.

## Audit checklist — run this before calling it done

Don't just confirm the app builds and screens render. Verify these specifically:

- **Free-tier privacy promise holds in practice.** Inspect actual network traffic (or just grep the free-scan code paths) and confirm a free Quick Scan makes no network call at all — the UI copy says "nothing leaves your device," so prove that's literally true in the code, not just in the text.
- **Confidence Score can't be set directly anywhere** — it must only ever be derived from a Contact's Scan rows. Check the SQLite layer for any write path that sets it independently.
- **Trial only lazy-starts on a real gated-feature tap**, not at signup or app open — write a quick test that creates a fresh user, leaves the app idle, and asserts `trialStartedAt` is still null.
- **DeviceTrialRecord actually survives reinstall.** Confirm it's written to `expo-secure-store`/Keychain, not AsyncStorage — AsyncStorage is wiped on uninstall on both platforms, which would silently defeat the entire trial-abuse backstop.
- **A lapsed trial/subscription never deletes or hides historical Contacts/Scans** — simulate expiry and check old data is still fully visible and exportable, with only new Pro actions blocked.
- **Data export/delete works on a free account**, unauthenticated-to-Pro — it's a free-tier right, test it as one, not just on a Pro test account where it might be bundled in with other export features.
- **Every Scan Result screen renders the "signal, not a verdict" disclaimer**, including on high-concern results — that's the screen where it's most tempting for copy to drift into sounding like a confident diagnosis.
- **Dark mode keeps the cyan/amber/magenta concern colors visually distinct from each other**, and the glow effect actually renders on both iOS and Android — not just "inverted colors with a shadow prop that silently does nothing on one platform." Check a real Scan Result screen in both modes on both platforms.
- **This actually runs via the dev client, not Expo Go** — confirm whoever picks this project up next knows `expo start` alone won't work once `react-native-iap` and the audio library are wired in.
