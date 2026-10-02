# VeriHuman (Base44): audit and hardening log, 2 Oct 2026

Live app: https://check-human-first.base44.app. Published from checkpoint `6ac016796b11fe8a15b3e32c`.

## Engine tests

16 of 16 pass. The tests ran in the Base44 sandbox against the actual app code.

| Area | Case | Result |
|---|---|---|
| Message | Sample scam message | 100 (High) |
| Message | Harmless text | 5 |
| Message | "I had bad signal on the train" | 5 (no longer flagged as off-platform) |
| Message | "is this your dog in the photo?" | 5 (no longer flagged as a wrong-number opener) |
| Message | "Is this Sarah?" | Wrong-number opener detected |
| Message | "add me on signal" | Off-platform detected |
| Message | Curly apostrophes | Matched |
| Photo | Stable Diffusion PNG with a `parameters` chunk | 100 |
| Photo | iPhone JPEG with EXIF | 0 |
| Photo | Apple ICC colour profile only | No camera credit |
| Photo | Garbage file | No crash |
| Voice | 5 s synthetic tone | 75 (High) |
| Voice | Noisy, pitch-varying clip | 0 |
| Voice | 2 s clip | Capped at 50 |
| Voice | Noise-gated pause | +5 only, with a context note |
| Voice | 60 s clip | About 2.1 s, in a Web Worker (was 5.6 s on the main thread) |

## Bugs found and fixed

- **Pro could be self-granted, twice.** First by editing the user profile record, then through the `activate` action. Pro is now set only by the signed Stripe webhook.
- **Verified Scan had no server-side Pro check.**
- **Deep Analyze was advertised but not built.**
- **Private uploads weren't readable by the AI model or the detection provider.** Fixed with short-lived signed URLs.
- **Every voice scan would have failed.** The voice worker called the audio APIs, which browsers don't allow inside a Web Worker.
- **Privacy copy overclaimed.** It said files were "deleted afterward" and that "nothing is uploaded".
- **Engine false positives:** "signal", "is this …?", camera credit from an Apple ICC profile, and the silence floor on voice notes.
- **Failed payments kept Pro.**
- **Light-mode contrast:** gold and coral text failed WCAG.

## Security controls now in place

- **Entitlement, TrialLedger, Upload, UsageLog and WebhookEvent:** only the service role can write them.
- **Scan:** browsers can create only Quick, non-thread scans. Verified and Deep Analyze results are written by the server.
- **Verified Scan and Deep Analyze:**
  - check Pro and enforce rate limits (5 per minute, 30 per day);
  - validate input limits;
  - reject files the caller didn't upload;
  - guard the AI analysis against prompt injection.
- **Unauthenticated calls:** a clean 401, with generic error messages and no stack traces.
- **Stripe:** checks the webhook signature, ignores replayed events and other prices, and builds checkout URLs from a fixed app URL.

## Accepted residual risks

- A user can update their own Scan records, so they could relabel one of their own results as Verified. This only affects their own data.
- A client could set `owner_id` on a Quick scan to another user's ID, injecting a junk scan into that user's list. It requires knowing an internal user ID, which the app never exposes.

## Not yet testable (needs keys)

- Verified photo and voice scans need `DEEPFAKE_API_KEY` and `DEEPFAKE_PROVIDER`.
- Stripe checkout and the webhook need `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` and `STRIPE_PRICE_ID`.
- A wrapped Google Play build must replace Stripe with Play Billing.
