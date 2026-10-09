# Push notifications — plan

Goal: when staff change an order's status in the admin portal, the customer's (and driver's) phone gets a push notification:
- **App closed or in the background:** a system tray notification. Tapping it opens the order.
- **App open (foreground):** a visible banner that opens the order when tapped. The open screen also refreshes.

Tested end to end on the **Android emulator** (no Apple account needed). iOS display can be checked on the simulator with `simctl push`.

---

## What already exists (no work needed)

| Piece | Where | Status |
|---|---|---|
| FCM sender with order pushes and catalogue topic pings | `laundry_admin/src/server/push/index.ts` | ✅ Sends after the DB commit; deletes dead tokens |
| Push on status change (customer + driver) | `laundry_admin/src/server/orders/transition.ts:105,120,207` | ✅ |
| EN/AR push copy | `laundry_admin/src/domain/push-copy.ts` | ✅ |
| Service account | `laundry_admin/.env` → `FIREBASE_SERVICE_ACCOUNT_JSON` | ✅ Set |
| Token registration, refresh, unregister on sign-out | `laundry_app/lib/core/push/push_service.dart` | ✅ |
| Tap → deep link (warm start and cold start) | `push_service.dart` + `lib/app.dart` `_PushListener` | ✅ |
| `google-services.json` / `GoogleService-Info.plist` | `android/app`, `ios/Runner` | ✅ |
| iOS `UIBackgroundModes: remote-notification` | `ios/Runner/Info.plist` | ✅ |
| iOS `aps-environment` entitlement | `ios/Runner/Runner.entitlements` | ✅ |
| iOS foreground banner | `setForegroundNotificationPresentationOptions` | ✅ |
| Podfile | `ios/Podfile` | ✅ `platform :ios, '15.0'` meets the Firebase minimum. Nothing to change |

## Gaps to fix

1. **Android foreground shows only a SnackBar.** It isn't a real notification and doesn't appear in the tray.
2. **No Android notification channel.** Pushes land in the plugin's fallback "Miscellaneous" channel at default importance, so they don't pop up as heads-up banners.
3. **No notification icon.** Android uses the launcher icon, which shows up as a white square in the status bar.
4. **`POST_NOTIFICATIONS` (Android 13+) isn't declared in our manifest.** The plugin merges it in, but declaring it makes it explicit.
5. **No background message handler.** Notification messages still show without one, but the plugin logs a warning, and data-only messages in the background are silently dropped.
6. **No way to send a test push** without walking an order through the whole flow.

---

## Changes

### laundry_app — Android

**`android/app/src/main/AndroidManifest.xml`**
```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
...
<application>
  <!-- FCM: channel + icon for notifications shown while the app is in the background / killed -->
  <meta-data android:name="com.google.firebase.messaging.default_notification_channel_id"
             android:value="order_updates" />
  <meta-data android:name="com.google.firebase.messaging.default_notification_icon"
             android:resource="@drawable/ic_notification" />
  <meta-data android:name="com.google.firebase.messaging.default_notification_color"
             android:resource="@color/notification_accent" />
</application>
```

**New resources**
- `res/drawable/ic_notification.xml`: a monochrome vector icon (a white glyph on a transparent background, as Android requires).
- `res/values/colors.xml`: `notification_accent`, set to the brand primary.

**`android/app/build.gradle.kts`**: turn on core library desugaring. `flutter_local_notifications` requires it.

### laundry_app — Dart

**Add dependency:** `flutter_local_notifications`.

**New `lib/core/push/local_notifications.dart`**, a small wrapper used **only on Android**:
- At startup, creates the `order_updates` channel with **high importance** (heads-up, sound) and localized name and description.
- `show(title, body, payload)` posts a real system notification. The payload is the push `data` encoded as JSON.
- On tap, decodes the payload and reuses `routeFor(...)`, so it follows the same routing as FCM taps.
- On iOS it isn't initialized. iOS already shows the system banner, and initializing it there would take over the `UNUserNotificationCenter` delegate from `firebase_messaging`.

**`push_service.dart`**
- `_onMessage`: on Android, call `LocalNotifications.show(...)` for messages that have a `notification`. Silent catalogue pings still only refresh. The `foregroundPushes` stream stays for in-app use.
- Taps on local notifications are fed into the existing `openedRoutes` stream, so `_PushListener` doesn't change. Cold starts from a local notification go through the same path as `getInitialMessage`.

**`lib/app.dart` `_PushListener._banner`**: remove the Android SnackBar, since the system notification replaces it. Keep the method as a no-op hook, or drop the subscription.

**Background handler**, in `lib/main.dart` before `runApp`:
```dart
@pragma('vm:entry-point')
Future<void> firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  // Tray notifications are shown by the system; nothing to do. Data-only
  // catalogue pings are irrelevant while closed — the app refetches on open.
}
FirebaseMessaging.onBackgroundMessage(firebaseMessagingBackgroundHandler);
```
Only registered when Firebase is ready (not in mock mode).

**l10n**: add `pushChannelName` / `pushChannelDescription` to `app_en.arb` / `app_ar.arb`.

### laundry_app — iOS

No code changes. Already correct: Info.plist background mode, entitlement, Podfile, and foreground presentation options.
One thing to know: the `aps-environment` entitlement plus a **free personal team** makes signing fail **on a real iPhone**. That's not a problem on the simulator. Once the paid account exists, enable the Push Notifications capability for the team and upload the APNs `.p8` key to Firebase → Project settings → Cloud Messaging.

### laundry_admin

**`src/server/push/index.ts`**: add the channel to order pushes so background notifications also use the high-importance channel:
```ts
android: { priority: "high", notification: { channelId: "order_updates", sound: "default" } },
```

**New script `scripts/push-test.ts`** and `"push:test"` in `package.json`:
```
pnpm push:test <phone-or-mobileUserId> [kind]
```
Looks up the user's device tokens and sends one order-style push through the real driver. It prints the token count and the FCM result. Use it to check FCM setup in isolation before testing the full status-change flow.

### Tests

- `laundry_app/test/core/push/`: test that encoding and decoding the local notification payload reaches the same route as `routeFor`, and that Android foreground messages call `show` while silent pings don't (local notifications faked).
- `laundry_admin`: unit test that the FCM message built for an order push includes `channelId: "order_updates"` and the `data` fields the app expects (`type`, `kind`, `orderId`).
- Run `flutter analyze`, `flutter test`, `pnpm typecheck`, `pnpm lint`, `pnpm test`.

---

## End-to-end test runbook (Android emulator)

1. **Emulator:** create an AVD with a **Google Play** or **Google APIs** system image (API 33+ so the permission prompt is tested too).
2. **Backend:** `cd laundry_admin && pnpm db:up && pnpm dev`. Make sure the mobile API answers on port 5080, which the app's default `http://10.0.2.2:5080` points to.
3. **App:**
   ```
   cd laundry_app
   flutter run --dart-define=USE_MOCK_API=false
   ```
   Sign in as a demo customer and allow notifications. Watch the admin logs for the `POST devices` call. The token is now stored.
4. **Check FCM alone:** `pnpm push:test <customer phone>`. A notification should appear.
5. **Full flow:** place an order in the app. In the admin portal, move it through the statuses (assign driver → picked up → invoice ready …). Check each one:
   - **App in foreground:** heads-up banner, order screen refreshes, tapping opens the order.
   - **App in background (Home button):** tray notification, tapping opens the order.
   - **App killed (swiped away):** tray notification, tapping launches the app on the order after the session restores.
   - **Arabic locale:** Arabic copy.
6. **Driver side:** sign in as the assigned driver on a second emulator, or sign out and back in. Assigning a pickup should send a "New pickup" push that opens the driver pickup screen.

## iOS simulator check (display only)

`laundry_app/tool/push.apns`, sent with `xcrun simctl push booted com.laundryapp.laundry tool/push.apns`. It includes `gcm.message_id` so `firebase_messaging` handles it, and `type/kind/orderId` so tapping it deep-links. Checks foreground, background and tap routing without FCM.

---

## Out of scope

- Real iOS delivery (needs the paid Apple account and APNs key).
- Notification grouping, images, action buttons.
- Badge counts.
