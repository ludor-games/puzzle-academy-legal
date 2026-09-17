# Google Play Data Safety mapping — Puzzle Academy Android test

Audit date: 2026-09-17 (supersedes the 2026-09-13 Appodeal audit)  
Package: `com.ludor.puzzleacademy`

This is an implementation-backed worksheet for the Google Play Data Safety form, not a submission export. It covers the current Android test build on `ludor-games/puzzle-academy` `origin/main`, after Appodeal was replaced by **AppLovin MAX** (Google AdMob mediated) and **AppsFlyer** was added (ludor-games/puzzle-academy#1379). Google Play defines “collected” as data transmitted off device, including by SDKs. “Shared” below follows Google Play’s third-party transfer definition and its service-provider exceptions.

**Keys:** builds from `main` ship with an empty AppLovin SDK key, empty ad units and an empty AppsFlyer dev key. In that state MAX skips initialization and AppsFlyer never starts, so neither transmits anything. The form must still describe the build that will run **with** keys, which is what the rows below assume. If an earlier artifact on the Play track still contains Appodeal, the 2026-09-13 declarations also apply until it is no longer distributed (see verification item 6).

## Form-level answers

| Question | Current answer | Basis / action |
|---|---|---|
| Does the app collect or share required user data types? | Yes | Firebase Analytics/Crashlytics, AppLovin MAX, the Google Mobile Ads SDK and AppsFlyer transmit data off device. |
| Is all collected user data encrypted in transit? | Yes, subject to vendor confirmation | Firebase documents HTTPS in transit; Google documents TLS for all Google Mobile Ads SDK data. Confirm AppLovin’s and AppsFlyer’s current Play declarations before submission. |
| Can users request deletion? | Yes, by email | `bambagamesbcn@gmail.com`; device-generated IDs may require information from the device to locate. Confirm that the public Play form accepts this process and enter its required deletion URL if prompted. AppsFlyer and AppLovin hold identifiers keyed to the advertising ID, so a deletion request may need forwarding to them. |
| Does the app support account creation? | No | No account registration/login flow was found; `AuthService.UserId` is `SystemInfo.deviceUniqueIdentifier`. |
| Does the app use the advertising ID? (separate Play Console declaration) | Yes | The AppLovin SDK and the Google Mobile Ads SDK read the Android advertising ID for advertising and fraud prevention, and AppsFlyer reads it for attribution. Declare purposes: Advertising or marketing; Analytics; Fraud prevention, security and compliance. |

## Data-type mapping

| Google Play category / type | Collected | Shared | Ephemeral | Required / optional | Purposes | SDK / service | Repository evidence and caveats |
|---|---:|---:|---|---|---|---|---|
| Location / Approximate location | Yes | Yes | No | Required while the advertising SDK is enabled | Advertising or marketing; Analytics; Fraud prevention, security and compliance | AppLovin MAX; Google Mobile Ads SDK; AppsFlyer | No `ACCESS_COARSE_LOCATION` or `ACCESS_FINE_LOCATION` use was found, so no permission-based location is collected. Google documents that the Mobile Ads SDK collects the IP address, “which may be used to estimate the general location of a device”; Google Play treats IP-derived location as Approximate location. AppLovin and AppsFlyer also receive the IP address. Verify both vendors’ declarations. |
| Location / Precise location | No | No | N/A | N/A | N/A | N/A | No precise-location permission or API use was found. |
| Personal info / User IDs | Yes | No under service-provider exception | No | Required | Analytics; App functionality; Fraud prevention, security and compliance | Firebase Analytics | `AuthService.UserId` returns `SystemInfo.deviceUniqueIdentifier`; analytics event bases send it as `user_id`. No account is created. No user ID is set on AppLovin (`MaxSdk.SetUserId` is not called) or AppsFlyer (`setCustomerUserId` is not called). Confirm whether Play Console expects this pseudonymous device-derived value under both User IDs and Device or other IDs; conservative submission is to declare both. |
| App activity / App interactions | Yes | Yes | No | Required | Analytics; App functionality; Advertising or marketing; Fraud prevention, security and compliance | Firebase Analytics; AppLovin MAX; Google Mobile Ads SDK; AppsFlyer | Session, level, progression, reward, feature, purchase-result and ad events are sent through `AnalyticsService` to Firebase. Google documents collection of “app launch, taps, and video views”. AppsFlyer records install and app-open (session) events for attribution. The ads SDK initializes at startup when configured, although viewing a rewarded ad is user-initiated. |
| App info and performance / Crash logs | Yes | No; verify | No | Required | Analytics; App functionality | Firebase Crashlytics | `CrashReportsService` initializes Crashlytics, reports uncaught exceptions as fatal, and sends handled exceptions. Firebase documents automatic stack traces and application state. The Appodeal-era BidMachine/Sentry crash-data caveat no longer applies. Verify that AppLovin’s current declaration does not list crash logs. |
| App info and performance / Diagnostics | Yes | Yes | No | Required | Analytics; App functionality; Advertising or marketing; Fraud prevention, security and compliance | Firebase Crashlytics; AppLovin MAX; Google Mobile Ads SDK | Crash breadcrumbs/non-fatals are sent to Crashlytics. Google documents diagnostic collection (“app launch time, hang rate, and energy usage”). AppLovin collects SDK and device performance data for ad delivery; verify its exact declaration. |
| App info and performance / Other app performance data | Yes | Yes; verify | No | Required | Analytics; Advertising or marketing; Fraud prevention, security and compliance | AppLovin MAX | Device model, memory, storage and similar technical fields used for ad delivery. Keep this row only if AppLovin’s current declaration lists it; Google’s SDK declares these under Diagnostics. |
| Device or other IDs | Yes | Yes | No | Required | Analytics; App functionality; Advertising or marketing; Fraud prevention, security and compliance | Firebase; AppLovin MAX; Google Mobile Ads SDK; AppsFlyer | Firebase installation/session/Crashlytics identifiers and the game’s device-derived `user_id`. Google documents the Android advertising ID and app set ID. AppLovin uses the advertising ID for targeting and frequency capping. AppsFlyer uses the advertising ID (and its own AppsFlyer ID) to attribute installs; `com.android.installreferrer:installreferrer:2.1` is resolved for install referrer data. |
| Financial info / Purchase history | Yes | No under service-provider exception; verify | No | Optional | App functionality; Analytics; Fraud prevention, security and compliance | Google Play Billing / Unity IAP; Firebase Analytics | `IapService` receives SKU, transaction ID, receipt and status from Google Play. `IapPurchaseEvent` sends product ID, localized price in micros, currency and bundle type to Firebase; it no longer sends a receipt payload. `com.appsflyer:purchase-connector:2.2.0` is bundled with the AppsFlyer plugin but **never initialized** in game code, so no purchase data reaches AppsFlyer. If it is enabled later, this row becomes Shared with AppsFlyer. |
| App activity / Advertising data | Yes | Yes | No | Required while the advertising SDK is enabled | Advertising or marketing; Analytics; Fraud prevention, security and compliance | AppLovin MAX; Google Mobile Ads SDK; AppsFlyer; Firebase Analytics | Rewarded ads are enabled; interstitials are disabled in `AdsConfig.json`. Each paid impression is forwarded to AppsFlyer (`logAdRevenue`: network, ad unit, format, placement, revenue) and to Firebase as `ad_impression` (ad platform, source, unit, format, placement, currency, value). `RewardedCompleteEvent` sends placement, reward type/amount and game context to Firebase. Google Play has no “Advertising data” type of its own; map these fields to App interactions / Other actions if the form requires it. |
| Personal info / Email address | No through the app | No | N/A | Optional outside the app | Developer communications | User’s email provider / Bamba Games support mailbox | The app has no working support submission flow; users may voluntarily email support outside the app. Do not mark as app collection unless a later build adds an in-app support form or email collection. |

## Consent

AppLovin MAX’s Terms & Privacy Policy flow is enabled (`ProjectSettings/AppLovinInternalSettings.json`). In the EEA, UK and Switzerland it shows Google’s consent form (UMP, `com.google.android.ump:user-messaging-platform:4.0.0`) during MAX initialization, and a refusal limits ads to non-personalized. AppsFlyer starts only after MAX initialization finishes and reads the resulting IAB TCF strings (`enableTCFDataCollection`). Firebase Analytics has no consent handling yet (ludor-games/puzzle-academy#1344). Consent does not change which data types are declared, since Play’s form describes collection that can occur, but it matters for the privacy policy and for the “optional” column if collection becomes consent-gated.

## Not collected by the audited build

No repository evidence was found for collection of name, phone number, postal address, contacts, photos/videos, audio recordings, files/documents, calendar data, health/fitness data, messages, web browsing history, search history, or precise location. Local game saves and locally scheduled notification IDs remain on device and do not count as off-device collection. Firebase Messaging is present as a transitive notification dependency, but no remote push registration or token handling was found in game code; current gameplay reminders are local notifications only.

## Manual verification before Play Console submission

1. Confirm the exact AppLovin SDK 13.6.4, Google AdMob adapter 25.4.0.0 (Google Mobile Ads SDK 25.4.0) and AppsFlyer Android SDK 6.18.1 Data Safety declarations on the vendors’ pages. AppLovin’s and AppsFlyer’s pages could not be read during this audit; the rows citing them are marked *verify*.
2. Confirm **Other app performance data** and **Crash logs** against AppLovin’s declaration, and drop or add rows to match.
3. Confirm **Purchase history → Shared** under Google Play’s payment/service-provider exceptions, and that the AppsFlyer purchase connector is still not initialized in the build being submitted.
4. Confirm that IP-derived ad and attribution geolocation is entered as **Approximate location → Collected and Shared**; no Android location permission data is collected.
5. Confirm the merged Android manifest of the submitted build carries `com.google.android.gms.permission.AD_ID` (added by the ads and attribution SDKs), matching the advertising-ID declaration above.
6. Reconcile this worksheet against every active artifact/version on the Play track: Google Play requires the form to cover the union of data practices across distributed versions, not only the newest source tree. An Appodeal build still on a track keeps BidMachine, Bidon and Sentry in scope.

## Audit references

- `PuzzleAcademy/Assets/Bundles/Common/Shared/RemoteConfig/AdsConfig.json`
- `PuzzleAcademy/Assets/Bundles/Common/Shared/RemoteConfig/AttributionConfig.json`
- `PuzzleAcademy/ProjectSettings/AppLovinInternalSettings.json`
- `PuzzleAcademy/ProjectSettings/AndroidResolverDependencies.xml`
- `PuzzleAcademy/Packages/com.ludor.sdk/Runtime/Ads/MaxAdsInitializer.cs`
- `PuzzleAcademy/Packages/com.ludor.sdk/Runtime/Services/AdsService.cs`
- `PuzzleAcademy/Packages/com.ludor.sdk/Runtime/Attribution/AppsFlyerAttributionService.cs`
- `PuzzleAcademy/Packages/com.ludor.sdk/Runtime/Services/AnalyticsService.cs`
- `PuzzleAcademy/Packages/com.ludor.sdk/Runtime/Services/CrashReportsService.cs`
- `PuzzleAcademy/Packages/com.ludor.sdk/Runtime/Services/IapService.cs`
- `PuzzleAcademy/Packages/com.ludor.foundation/Runtime/Auth/AuthService.cs`
- `PuzzleAcademy/Packages/com.ludor.puzzleacademy/Runtime/Analytics/AdImpressionEvent.cs`
- `PuzzleAcademy/Packages/com.ludor.puzzleacademy/Runtime/Analytics/AdRevenueAnalyticsReporter.cs`
- `PuzzleAcademy/Packages/com.ludor.puzzleacademy/Runtime/Analytics/IapPurchaseEvent.cs`
- `PuzzleAcademy/Packages/com.ludor.puzzleacademy/Runtime/Analytics/RewardedCompleteEvent.cs`
- `PuzzleAcademy/Packages/com.ludor.puzzleacademy/Runtime/Notifications/LocalNotificationService.cs`

Vendor references:

- [Google Play Data Safety definitions](https://support.google.com/googleplay/android-developer/answer/10787469)
- [Firebase Android data disclosure guidance](https://firebase.google.com/docs/android/play-data-disclosure)
- [Google Mobile Ads SDK: Google Play data disclosure](https://developers.google.com/admob/android/privacy/play-data-disclosure)
- [AppLovin MAX Android: Google Play data safety](https://support.applovin.com/en/max/android/overview/data-and-privacy/google-play-data-safety)
- AppsFlyer Android SDK Data Safety guidance: locate on the AppsFlyer support site before submission
