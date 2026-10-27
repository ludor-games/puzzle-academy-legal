# Data deletion procedure — Puzzle Academy / Sort Masters

Drafted 2026-10-27 for ludor-games/puzzle-academy#2445. Internal procedure, not a public page.

The Game has no account. A player is identified by the **User ID** (`AuthService.UserId`, the Unity
`SystemInfo.deviceUniqueIdentifier`). Deletion requests arrive by email and are fulfilled by hand, provider by
provider, using that User ID and the identifiers it leads to.

## 1. Receiving a request

- **Channel:** email to `bambagamesbcn@gmail.com` (the Privacy Policy's *Your choices and rights*).
- **What the player sends:** the User ID. It is shown in **Settings** (tap it to copy User ID and version), and
  **Settings → Contact support** opens an email that already contains it below the message.
- **No User ID:** reply asking for it. Without it no provider record can be located, because nothing the Game sends
  is keyed to an email address. Do not act on a User ID taken from anywhere but the requester's own message.
- **Deadline:** one month from receipt (GDPR Art. 12(3)). Reply to acknowledge, then again when done.
- **Log:** record each request (date received, User ID, date completed per provider, reply sent) in a private
  place outside this public repository. Do not commit User IDs here.

## 2. Identifiers to collect first

Run in BigQuery (location EU) on `puzzle-academy-prod` — and on `puzzle-academy-dev` for a test device:

```sql
SELECT DISTINCT user_pseudo_id
FROM `puzzle-academy-prod.analytics_555377836.events_*`
WHERE user_id = @user_id;
```

`user_pseudo_id` is the Firebase app instance ID. Keep the list: steps 3.1 and 3.2 need it. The advertising ID is
not in our data; AppsFlyer and AppLovin requests use the User ID or, if the player supplies it, the advertising ID.

## 3. Per provider

### 3.1 Google Analytics for Firebase

- **Holds:** events with `user_id` = User ID; the app instance ID (`user_pseudo_id`).
- **Delete:** Analytics Admin API `properties/{property}:submitUserDeletion` (scope `analytics.edit`), once with
  `userId` = User ID and once per `appInstanceId` from step 2. Each request deletes only the ID it names.
- **Takes:** removed from reporting within days; permanently deleted from Analytics servers in the next deletion
  cycle (up to about two months).
- **Does not cover BigQuery** — Google states the API does not delete data already exported. Step 3.2 does.

### 3.2 BigQuery (Firebase export and the analytics model)

Needs a key with write access (BigQuery Data Editor); the shared reader keys cannot delete. Show the statements to
the maintainer before running them: a DML delete on prod is irreversible.

Per project (`puzzle-academy-prod` dataset `analytics_555377836`; `puzzle-academy-dev` dataset `analytics_531589322`):

```sql
-- raw export, every daily and intraday table
DELETE FROM `<project>.<analytics_dataset>.events_*`
WHERE user_id = @user_id OR user_pseudo_id IN UNNEST(@pseudo_ids);
DELETE FROM `<project>.<analytics_dataset>.users_*` WHERE user_id = @user_id;
DELETE FROM `<project>.<analytics_dataset>.pseudonymous_users_*` WHERE pseudo_user_id IN UNNEST(@pseudo_ids);

-- incremental model tables keep history, so delete there too
DELETE FROM `<project>.analytics_model.stg_events` WHERE user_pseudo_id IN UNNEST(@pseudo_ids);
DELETE FROM `<project>.analytics_model.player_attributes` WHERE user_pseudo_id IN UNNEST(@pseudo_ids);
```

BigQuery does not allow DML on a wildcard table: run each `DELETE` per daily table (generate the list from
`INFORMATION_SCHEMA.TABLES`), or skip tables older than the player's first event. The other `analytics_model`
tables are rebuilt in full from `stg_events` on each daily Dataform run, so they drop the player on the next run.

Crashlytics export (prod `firebase_crashlytics.com_ludor_puzzleacademy_ANDROID`, and the iOS table when it exists):

```sql
DELETE FROM `puzzle-academy-prod.firebase_crashlytics.com_ludor_puzzleacademy_ANDROID`
WHERE user.id = @user_id;
```

Verify: rerun the step-2 query and a `COUNT(*)` per table; all must return 0.

### 3.3 Firebase Crashlytics (console)

- **Holds:** crash reports carrying `user.id` = User ID, only for players who granted analytics consent.
- **Delete:** there is no per-user deletion in the console. Crash data is deleted after 90 days (stated in the
  policy). If the request needs it sooner, file a Firebase support request naming the project and User ID.

### 3.4 AppsFlyer

- **Holds:** install, in-app event and revenue data with Customer User ID = User ID, plus the advertising ID and
  AppsFlyer ID of the install.
- **Delete:** an erasure request through AppsFlyer's GDPR (OpenDSR) Requests API or the dashboard's privacy
  request page. **Before the first real request, confirm in the current AppsFlyer API reference which identity
  types it accepts** — if Customer User ID is not one of them, look up the install's AppsFlyer ID by Customer User
  ID (raw data / user lookup) and submit that. Requests for Limit Ad Tracking users cannot go through the API;
  contact AppsFlyer support instead.
- **Takes:** AppsFlyer processes erasure within its GDPR deadline; record its request ID in the log.

### 3.5 AppLovin MAX (and the networks it mediates, including Google AdMob)

- **Holds:** data keyed to the advertising ID. The Game sends AppLovin no User ID.
- **Delete:** ask the player for their advertising ID only if they want ad-side deletion (Android: Settings →
  Google → Ads; iOS: the IDFA is not available to the player), or point them to the provider's own request
  routes, which the policy already names: resetting or deleting the advertising ID, and
  `https://www.applovin.com/privacy/` for AppLovin. Google deletes ad data tied to a reset/deleted advertising ID
  under its own policy.
- AppLovin passes deletion requests to its demand partners through its GDPR deleted-device-IDs feed.

### 3.6 Local data

Game progress lives on the device. Uninstalling the Game, or clearing its storage, deletes it. Tell the player so in
the reply.

## 4. Reply template

> We have deleted the data associated with User ID `<id>` from Google Analytics for Firebase, our analytics
> warehouse and AppsFlyer [and submitted the request to AppLovin]. Crash reports are deleted automatically within
> 90 days. Game progress stored on your device is removed when you uninstall the Game. If you keep playing on this
> device, new data will be collected under the same User ID.

## 5. Dry run (acceptance for #2445)

Before declaring the path working, run steps 2–3 end to end on a test device's User ID against
`puzzle-academy-dev` and the dev Firebase project, and record on the issue: the IDs found, each provider's request
or job ID, and the zero-row verification.
