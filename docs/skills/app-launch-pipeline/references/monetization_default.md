# Default monetization: every app in this pipeline

Unless the owner says otherwise, every Daybreak Labs app ships with this model. For the reasoning behind it, see the `subscription-trial-model` skill.

## Tiers

- **Free, forever.** The core loop works with no time limit. Raw data export and account deletion are always free.
- **Free tier carries ads** in compliant ad slots, described below.
- **Pro, with a 14-day free trial.**
  - The trial starts lazily, the first time someone taps a Pro feature, never at signup.
  - One trial per person: tracked by a server-side ledger keyed to a hashed email, and on mobile by a device record (Keychain on iOS).
  - A plain banner reads "N days left in your trial". No countdown, no red.
  - One reminder goes out at about 70–80% of the way through, referring to the user's own data where possible.
- **Pro is ad-free.** Ads disappear the moment a trial or subscription starts.
- **When a trial or subscription lapses**, only forward-looking Pro features lock. History is never hidden or deleted.
- **Entitlement is written only by the server**, from the store's verified purchase events: Play Billing or StoreKit in the apps, Stripe on the web. The client can never set it.

## Ad slots (free tier)

- **One reusable `<AdSlot placement="...">` component.**
  - It renders nothing for Pro or trialing users.
  - It reserves its height up front so there's no layout shift.
  - It is labelled "Advertisement", and mounts only after the screen's real content has rendered.
- **Approved placements: content screens only.**
  - The public landing page, below the fold.
  - Resources and guide articles, mid-article and at the end.
  - The FAQ.
  - The main dashboard/list, below the first content block.
  - Result screens, only below the result *and* its explanation.
- **Never** on login, register, onboarding, loading, empty, error, confirmation or thank-you screens, the paywall/upgrade or checkout, settings, account deletion, consent dialogs, modals, or print/export.
- **Web (AdSense):**
  - Store the client ID in config: `ADSENSE_CLIENT`, plus per-placement slot IDs.
  - Serve a correct `ads.txt` at the domain root, and verify it by fetching the live URL.
  - Load the script once, lazily.
- **Android/iOS:**
  - AdSense is *not* allowed inside an app's WebView unless it goes through Google's WebView API for Ads.
  - Use AdMob natively, or the WebView API for Ads, when the app is wrapped.
  - Flag this to the owner at the Stage 5 check; don't ship AdSense inside a plain WebView.
- **Privacy and consent:**
  - Default to non-personalized ads for privacy-positioned apps.
  - Use a Google-certified consent tool for users in the EEA, UK and Switzerland.
  - Disclose ads and ad partners in the Privacy Policy, and in Play's Data safety form ("contains ads").
- **Block ad categories that conflict with the app's purpose.** For example, a scam-detection or finance app blocks crypto, dating, gambling, get-rich-quick and loan ads; a health app blocks diet pills and similar.
- **Run the "Ad-supported free tiers" checklist** in the `subscription-trial-model` skill against every ad-bearing screen before launch, and again whenever a new screen is added.

## In the Base44 prompt

Every Stage 3 prompt includes a "Monetization" section stating the tiers above, the app's own Pro feature list, the 14-day lazy trial, the AdSlot rules and placements for that app's screens, and the blocked ad categories for its subject.
