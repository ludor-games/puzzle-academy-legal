# Google Play Data Safety mapping — Puzzle Academy Android test

Audit date: 2026-09-13  
Package: `com.ludor.puzzleacademy`

This is an implementation-backed worksheet for the Google Play Data Safety form, not a submission export. It covers the current Android test build on `ludor-games/puzzle-academy` `origin/main` and the bundled Appodeal configuration. Google Play defines “collected” as data transmitted off device, including by SDKs. “Shared” below follows Google Play’s third-party transfer definition and its service-provider exceptions.

## Form-level answers

| Question | Current answer | Basis / action |
|---|---|---|
| Does the app collect or share required user data types? | Yes | Firebase Analytics/Crashlytics and Appodeal transmit data off device. |
| Is all collected user data encrypted in transit? | Yes, subject to vendor confirmation | Firebase documents HTTPS in transit; verify Appodeal/BidMachine’s current Play declaration in the release dashboard before submission. |
| Can users request deletion? | Yes, by email | `bambagamesbcn@gmail.com`; device-generated IDs may require information from the device to locate. Confirm that the public Play form accepts this process and enter its required deletion URL if prompted. |
| Does the app support account creation? | No | No account registration/login flow was found; `AuthService.UserId` is `SystemInfo.deviceUniqueIdentifier`. |

## Data-type mapping

| Google Play category / type | Collected | Shared | Ephemeral | Required / optional | Purposes | SDK / service | Repository evidence and caveats |
|---|---:|---:|---|---|---|---|---|
| Location / Approximate location | Yes | Yes | No | Required while the advertising SDK is enabled | Advertising or marketing; Personalization; Fraud prevention, security and compliance | Appodeal / BidMachine / Bidon ad delivery | No `ACCESS_COARSE_LOCATION` or `ACCESS_FINE_LOCATION` use was found, so GPS/network-location permission data is not collected. Appodeal documents that ads are targeted by IP without those permissions; Google Play notes that IP-derived location may be Approximate location. Declare approximate location on that basis. Verify this row against the exact Appodeal 4.3.0 and enabled adapter declarations in Play Console. |
| Location / Precise location | No | No | N/A | N/A | N/A | N/A | No precise-location permission or API use was found. Appodeal states location collection requires a declared and granted location permission. |
| Personal info / User IDs | Yes | No under service-provider exception | No | Required | Analytics; App functionality; Fraud prevention, security and compliance | Firebase Analytics | `AuthService.UserId` returns `SystemInfo.deviceUniqueIdentifier`; analytics event bases send it as `user_id`. No account is created. The game does not call `Appodeal.setUserId`. Confirm whether Play Console expects this pseudonymous device-derived value under both User IDs and Device or other IDs; conservative submission is to declare both. |
| App activity / App interactions | Yes | Yes | No | Required | Analytics; App functionality; Advertising or marketing; Personalization | Firebase Analytics; Appodeal / BidMachine / Bidon | Session, level, progression, reward, feature, purchase-result and rewarded-ad events are sent through `AnalyticsService` to Firebase. Appodeal documents collection of ad impressions/clicks. The SDK initializes automatically when enabled, although viewing a rewarded ad is user-initiated. |
| App info and performance / Crash logs | Yes | Requires vendor verification | No | Required | Analytics; App functionality; Fraud prevention, security and compliance | Firebase Crashlytics; potentially BidMachine adapter | `CrashReportsService` initializes Crashlytics, reports uncaught exceptions as fatal, and sends handled exceptions. Firebase documents automatic stack traces and application state. BidMachine’s current SDK documentation says crash data can be collected; verify whether the enabled Appodeal BidMachine adapter does so and whether Play treats that transfer as sharing. |
| App info and performance / Diagnostics | Yes | Yes | No | Required | Analytics; App functionality; Advertising or marketing; Fraud prevention, security and compliance | Firebase Crashlytics; Appodeal / BidMachine / Sentry analytics adapter | Crash breadcrumbs/non-fatals are sent to Crashlytics. Appodeal documents diagnostics, device/network information and ad measurement; Sentry analytics adapter `8.44.1.0` is resolved in `AndroidResolverDependencies.xml`. Verify the adapter’s exact payload and sharing declaration in the vendor dashboard. |
| App info and performance / Other app performance data | Yes | Yes | No | Required | Analytics; App functionality; Advertising or marketing; Fraud prevention, security and compliance | Appodeal / BidMachine / Sentry analytics adapter | Appodeal documents device model, memory, storage and user-agent collection for advertising/analytics; BidMachine documents technical device/performance fields. |
| Device or other IDs | Yes | Yes | No | Required | Analytics; App functionality; Advertising or marketing; Fraud prevention, security and compliance; Personalization | Firebase; Appodeal / BidMachine / Bidon | Firebase installation/session/Crashlytics identifiers and the game’s device-derived `user_id` are used. Appodeal documents Advertising ID, IP address, MCC-MNC and similar identifiers for targeting/tracking. No code-level Appodeal user ID is set. |
| Financial info / Purchase history | Yes | No under service-provider exception; verify | No | Optional | App functionality; Analytics; Fraud prevention, security and compliance | Google Play Billing / Unity IAP; Firebase Analytics | `IapService` receives SKU, transaction ID, receipt/payload and status from Google Play. `PurchaseEvent` sends product/type/cost/SKU/payload/result metadata to Firebase. The app does not call `Appodeal.trackInAppPurchase`. Verify Google Play Payments’ current treatment and whether the event `payload` can contain receipt or transaction identifiers in the released catalog flow. |
| App activity / Advertising data | Yes | Yes | No | Required while the advertising SDK is enabled | Advertising or marketing; Analytics; Personalization; Fraud prevention, security and compliance | Appodeal / BidMachine / Bidon; Firebase Analytics | Rewarded ads are enabled; interstitials are disabled in `AppodealSdkConfig.json`. Appodeal documents ad impression/click and device identifier processing. `RewardedCompleteEvent` sends placement, reward type/amount and game context to Firebase. |
| Personal info / Email address | No through the app | No | N/A | Optional outside the app | Developer communications | User’s email provider / Bamba Games support mailbox | The app has no working support submission flow; users may voluntarily email support outside the app. Do not mark as app collection unless a later build adds an in-app support form or email collection. |

## Not collected by the audited build

No repository evidence was found for collection of name, phone number, postal address, contacts, photos/videos, audio recordings, files/documents, calendar data, health/fitness data, messages, web browsing history, search history, or precise location. Local game saves and locally scheduled notification IDs remain on device and do not count as off-device collection. Firebase Messaging is present as a transitive notification dependency, but no remote push registration or token handling was found in game code; current gameplay reminders are local notifications only.

## Manual verification before Play Console submission

1. Confirm the exact Appodeal 4.3.0, BidMachine 3.7.1.0, Bidon 0.14.0.0, IAB 1.8.1.0, and Sentry analytics 8.44.1.0 Data Safety declarations in the vendor dashboards. Adapter behavior is not fully provable from dependency manifests.
2. Confirm **Crash logs → Shared** for the enabled BidMachine adapter. Firebase collection is certain; ad-partner sharing is vendor-dependent.
3. Confirm **Purchase history → Shared** under Google Play’s payment/service-provider exceptions and inspect the runtime value of analytics `payload` for real-money purchases.
4. Confirm that IP-derived ad geolocation is entered as **Approximate location → Collected and Shared** in the current Play form; no Android location permission data is collected.
5. Confirm Appodeal/adapter encryption-in-transit declarations and the consent configuration active in the Appodeal/Google UMP dashboard. Appodeal’s SDK includes automatic regional consent handling, but dashboard state is outside the repository audit.
6. Reconcile this worksheet against every active artifact/version on the Play track: Google Play requires the form to cover the union of data practices across distributed versions, not only the newest source tree.

## Audit references

- `PuzzleAcademy/Assets/Bundles/Common/Shared/RemoteConfig/AppodealSdkConfig.json`
- `PuzzleAcademy/Assets/Appodeal/Resources/Appodeal/AppodealDmChoices.asset`
- `PuzzleAcademy/ProjectSettings/AndroidResolverDependencies.xml`
- `PuzzleAcademy/Packages/com.ludor.sdk/Runtime/Appodeal/AppodealSdkInitializer.cs`
- `PuzzleAcademy/Packages/com.ludor.sdk/Runtime/Services/AnalyticsService.cs`
- `PuzzleAcademy/Packages/com.ludor.sdk/Runtime/Services/CrashReportsService.cs`
- `PuzzleAcademy/Packages/com.ludor.sdk/Runtime/Services/IapService.cs`
- `PuzzleAcademy/Packages/com.ludor.foundation/Runtime/Auth/AuthService.cs`
- `PuzzleAcademy/Packages/com.ludor.puzzleacademy/Runtime/Analytics/PurchaseEvent.cs`
- `PuzzleAcademy/Packages/com.ludor.puzzleacademy/Runtime/Analytics/RewardedCompleteEvent.cs`
- `PuzzleAcademy/Packages/com.ludor.puzzleacademy/Runtime/Notifications/LocalNotificationService.cs`

Vendor references:

- [Google Play Data Safety definitions](https://support.google.com/googleplay/android-developer/answer/10787469)
- [Firebase Android data disclosure guidance](https://firebase.google.com/docs/android/play-data-disclosure)
- [Appodeal Android Data Security](https://docs.appodeal.com/android/data-protection/app-privacy-details)
- [Appodeal Android permissions](https://docs.appodeal.com/faq-and-troubleshooting/troubleshooting/general/android-sdk-permissions)
- [BidMachine Android data privacy](https://developers.bidmachine.io/sdk/general/android/privacy)
