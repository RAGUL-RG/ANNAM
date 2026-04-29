# Food Donation & Volunteer App

A Flutter mobile app using Firebase Authentication, Cloud Firestore, and Provider.

If this folder was not created with `flutter create`, run this after installing Flutter to add mobile platform folders:

```bash
flutter create . --platforms=android,ios
```

## Project Structure

```text
pubspec.yaml
analysis_options.yaml
firestore.rules
storage.rules
vercel.json
functions/
  index.js
  package.json
lib/
  main.dart
  firebase_options.dart
  models/
    app_user.dart
    donation.dart
    user_role.dart
  providers/
    auth_provider.dart
    donation_provider.dart
  services/
    auth_service.dart
    donation_service.dart
    firestore_service.dart
    location_service.dart
    notification_service.dart
    storage_service.dart
  screens/
    auth/
      login_screen.dart
      register_screen.dart
    dashboards/
      donor_dashboard_screen.dart
      volunteer_dashboard_screen.dart
    donations/
      create_donation_screen.dart
      delivery_proof_screen.dart
  widgets/
    app_text_field.dart
    donation_map.dart
    donation_status_chip.dart
    primary_button.dart
    role_selector.dart
web/
  index.html
  manifest.json
```

## Firebase Setup Steps

1. Create a Firebase project at https://console.firebase.google.com.
2. Add Android and/or iOS apps to the Firebase project.
3. Enable Firebase Authentication:
   - Email/Password
   - Phone, if you want OTP login later
4. Create a Cloud Firestore database.
5. Enable Firebase Storage for food image uploads.
6. Enable Firebase Cloud Messaging for push notifications.
7. Enable Google Maps SDK:
   - Maps SDK for Android
   - Maps SDK for iOS
8. Install the Firebase CLI and FlutterFire CLI:

```bash
npm install -g firebase-tools
dart pub global activate flutterfire_cli
```

9. From the project root, configure Firebase:

```bash
flutterfire configure
```

This generates `lib/firebase_options.dart` with real Firebase config values. The checked-in file is a placeholder so the app structure is complete.

10. Install packages:

```bash
flutter pub get
```

11. Deploy Firestore and Storage rules:

```bash
firebase deploy --only firestore:rules,storage
```

12. Deploy Cloud Functions:

```bash
cd functions
npm install
npm run deploy
```

13. Run the app:

```bash
flutter run
```

## Deploy to Vercel

This project includes `vercel.json` for Flutter web hosting. Vercel will download the stable Flutter SDK during the build, run `flutter build web --release`, and serve `build/web`.

Before deploying, configure Firebase for web:

```bash
flutterfire configure
```

Make sure `lib/firebase_options.dart` contains real web values for:

```text
apiKey
appId
messagingSenderId
projectId
authDomain
storageBucket
measurementId
```

Replace the Google Maps key placeholder in `web/index.html`:

```html
REPLACE_WITH_GOOGLE_MAPS_WEB_API_KEY
```

Deploy with the Vercel CLI:

```bash
npm install -g vercel
vercel
```

Production deploy:

```bash
vercel --prod
```

You can also import this folder into the Vercel dashboard and let Vercel use the checked-in `vercel.json` build settings.

## Firestore User Document

Registered users are stored under:

```text
users/{uid}
```

Example document:

```json
{
  "uid": "firebase-auth-uid",
  "name": "Asha",
  "email": "asha@example.com",
  "phone": "+15551234567",
  "role": "donor",
  "fcmTokens": ["device-fcm-token"],
  "lastLocation": {
    "lat": 51.5072,
    "long": -0.1276
  },
  "createdAt": "server timestamp"
}
```

## Donation Document

Food donations are stored under:

```text
donations/{donationId}
```

Example document:

```json
{
  "donationId": "generated-firestore-id",
  "donorId": "firebase-auth-uid",
  "image": "https://firebasestorage.googleapis.com/...",
  "description": "Fresh rice and curry packs",
  "quantity": "20 meals",
  "location": {
    "lat": 51.5072,
    "long": -0.1276
  },
  "pickupTime": "firestore timestamp",
  "phoneNumber": "+15551234567",
  "status": "pending",
  "volunteerId": "assigned-volunteer-uid",
  "acceptedAt": "server timestamp",
  "pickedAt": "server timestamp",
  "deliveredAt": "server timestamp",
  "proofImage": "https://firebasestorage.googleapis.com/...",
  "deliveryNotes": "Delivered to the shelter reception.",
  "timestamp": "server timestamp"
}
```

Food images are uploaded to:

```text
donation_images/{donorId}/{donationId}.jpg
```

Delivery proof images are uploaded to:

```text
delivery_proofs/{volunteerId}/{donationId}.jpg
```

## Full Donation Workflow

1. Donor creates a donation post.
   - Firestore status: `pending`
2. Volunteer accepts a nearby pending donation.
   - Firestore status: `accepted`
   - Saves `volunteerId`
3. Assigned volunteer marks the donation as picked.
   - Firestore status: `picked`
   - Saves `pickedAt`
4. Assigned volunteer completes delivery.
   - Uploads proof image to Firebase Storage.
   - Adds delivery notes.
   - Firestore status: `delivered`
   - Saves `proofImage`, `deliveryNotes`, and `deliveredAt`

The donor dashboard listens to the donor's donation documents in real time. The volunteer dashboard listens to pending donations and assigned donations in real time.

## Volunteer Dashboard

Volunteers fetch pending donations with:

```text
donations where status == "pending"
```

The app captures the volunteer GPS position and filters donations by distance using `Geolocator.distanceBetween`. The dashboard supports 5 km and 10 km radius filters.

When a volunteer accepts a donation, the app updates:

```json
{
  "status": "accepted",
  "volunteerId": "firebase-auth-uid",
  "acceptedAt": "server timestamp"
}
```

The accept action uses a Firestore transaction so a donation must still be `pending` before it is assigned.

Accepted assignments show donor and volunteer markers on Google Maps. The donor marker uses `donations/{donationId}.location`, and the volunteer marker uses the current GPS location captured by the app.

## FCM Notifications

The app requests notification permission at startup through `NotificationService`.

Token handling:

```text
users/{uid}.fcmTokens[]
```

- On login, register, or auth restore, the current FCM token is saved to the signed-in user document.
- On token refresh, the new token is added automatically.
- On logout, the current token is removed from the user document.
- Volunteers update `users/{uid}.lastLocation` when the Volunteer Dashboard loads or refreshes location.

Notification trigger:

```text
functions/index.js -> notifyNearbyVolunteers
```

When a new `donations/{donationId}` document is created with `status == "pending"`, the Cloud Function:

1. Reads the donation location.
2. Fetches users where `role == "volunteer"`.
3. Calculates distance from each volunteer `lastLocation`.
4. Sends FCM to volunteers within 10 km.
5. Includes food description, quantity, latitude, longitude, and donation ID in the payload.
6. Uses high-priority Android delivery with the `food_donation_alerts` channel, default sound, and vibration.

Foreground notifications are displayed through `flutter_local_notifications`, so volunteers still get a sound/vibration alert while the app is open.

## Location Permissions

Add location permission to `android/app/src/main/AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.CAMERA" />
```

Add location usage text to `ios/Runner/Info.plist`:

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>We use your location so volunteers can find donation pickups.</string>
<key>NSCameraUsageDescription</key>
<string>We use your camera so volunteers can upload delivery proof.</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>We use your photo library so donors can upload food images.</string>
<key>UIBackgroundModes</key>
<array>
  <string>remote-notification</string>
</array>
```

## Google Maps Setup

Android: add your Maps key in `android/app/src/main/AndroidManifest.xml` inside `<application>`:

```xml
<meta-data
  android:name="com.google.android.geo.API_KEY"
  android:value="YOUR_GOOGLE_MAPS_API_KEY" />
```

iOS: add your Maps key in `ios/Runner/AppDelegate.swift`:

```swift
GMSServices.provideAPIKey("YOUR_GOOGLE_MAPS_API_KEY")
```

Also import Google Maps in that file:

```swift
import GoogleMaps
```

## Firebase Rules

Firestore rules are in `firestore.rules`.

They enforce:

- Users can read/write only their own profile.
- Only the donor can create a donation for themselves.
- Only the donor can edit pending post details.
- Only a volunteer can accept a pending donation for themselves.
- Only the assigned volunteer can move `accepted -> picked -> delivered`.
- Delivery requires `proofImage` and `deliveryNotes`.

Storage rules are in `storage.rules`.

They enforce:

- Donors can upload only their own food images under `donation_images/{donorId}`.
- Volunteers can upload only their own proof images under `delivery_proofs/{volunteerId}`.
- Authenticated users can read uploaded images.
