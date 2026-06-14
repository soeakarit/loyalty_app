# Loyalty App — MVP Planning Document

**Project:** loyalty_app  
**Stack:** Flutter + Firebase (Firestore, Auth, Cloud Functions)  
**Platforms:** iOS & Android  
**Date:** 2026-06-15

---

## 1. MVP Scope

The MVP covers two user roles: **Customer** and **Staff**.

| Feature | Customer | Staff |
|---|---|---|
| Sign up / log in with email+password | Yes | Yes (pre-created accounts) |
| View current points balance | Yes | — |
| View points transaction history | Yes | — |
| Redeem available rewards | Yes | — |
| View available rewards list | Yes | — |
| Search customer by phone number | — | Yes |
| Add points to a customer account | — | Yes |
| View customer profile & balance | — | Yes |
| Mark a redemption as fulfilled | — | Yes |

Out of scope for MVP: social login, push notifications, admin dashboard, tier levels, expiry dates, QR-code scanning.

---

## 2. Folder Structure

```
lib/
├── main.dart                    # App entry point, Firebase init
├── app.dart                     # MaterialApp + GoRouter setup
│
├── core/
│   ├── constants.dart           # App-wide constants (colors, strings)
│   ├── theme.dart               # ThemeData
│   └── utils.dart               # Shared helpers (formatters, validators)
│
├── models/
│   ├── app_user.dart            # User model (uid, role, phone, points)
│   ├── transaction.dart         # Points transaction record
│   └── reward.dart              # Reward definition
│
├── services/
│   ├── auth_service.dart        # Firebase Auth wrapper
│   ├── user_service.dart        # Firestore CRUD for users
│   ├── points_service.dart      # Add/deduct points logic
│   └── reward_service.dart      # Reward list + redemption logic
│
├── providers/
│   ├── auth_provider.dart       # Riverpod: current auth state + role
│   ├── user_provider.dart       # Riverpod: current customer profile
│   └── reward_provider.dart     # Riverpod: available rewards stream
│
├── router/
│   └── app_router.dart          # GoRouter with role-based redirect guard
│
├── screens/
│   ├── auth/
│   │   ├── login_screen.dart
│   │   └── register_screen.dart
│   │
│   ├── customer/
│   │   ├── customer_home_screen.dart   # Points balance + quick actions
│   │   ├── history_screen.dart         # Transaction history list
│   │   └── rewards_screen.dart         # Browse & redeem rewards
│   │
│   └── staff/
│       ├── staff_home_screen.dart      # Search customer entry point
│       ├── customer_detail_screen.dart # Customer profile + add points
│       └── redemption_screen.dart      # Pending redemptions to fulfil
│
└── widgets/
    ├── points_card.dart         # Reusable balance display card
    ├── transaction_tile.dart    # Single history row
    └── reward_card.dart         # Reward display card

docs/
└── LoyaltyAppPlan.md

test/
└── widget_test.dart
```

---

## 3. Firebase Collections

### `users/{uid}`
```jsonc
{
  "uid": "string",
  "email": "string",
  "displayName": "string",
  "phone": "string",          // used by staff to search
  "role": "customer" | "staff",
  "points": 0,                // current balance (integer)
  "createdAt": "Timestamp"
}
```

### `transactions/{txnId}`
```jsonc
{
  "customerId": "string",     // uid of the customer
  "staffId": "string",        // uid of staff who performed the action
  "type": "add" | "redeem",
  "points": 0,                // positive for add, negative for redeem
  "note": "string",           // e.g. "Purchase $50", "Free Coffee reward"
  "createdAt": "Timestamp"
}
```

### `rewards/{rewardId}`
```jsonc
{
  "title": "string",          // e.g. "Free Coffee"
  "description": "string",
  "pointsCost": 0,
  "isActive": true,
  "imageUrl": "string | null"
}
```

### `redemptions/{redemptionId}`
```jsonc
{
  "customerId": "string",
  "rewardId": "string",
  "rewardTitle": "string",    // denormalised snapshot
  "pointsCost": 0,
  "status": "pending" | "fulfilled" | "cancelled",
  "createdAt": "Timestamp",
  "fulfilledAt": "Timestamp | null",
  "fulfilledBy": "string | null"  // staff uid
}
```

---

## 4. Authentication Flow

```
App launch
  └─> Firebase Auth state stream
        ├─> null (not signed in)  ──> /login
        └─> User signed in
              └─> fetch users/{uid}.role from Firestore
                    ├─> "customer"  ──> /customer/home
                    └─> "staff"     ──> /staff/home
```

- Email + password auth via `firebase_auth`.
- On registration, a `users/{uid}` document is created in Firestore with `role: "customer"` and `points: 0`.
- Staff accounts are created manually in Firestore (or via a restricted admin script) with `role: "staff"`. They log in through the same login screen.
- The role is fetched once after sign-in and stored in the `AuthProvider`. The GoRouter redirect guard uses this role to enforce routing.

---

## 5. Customer Screens

### Login / Register (`/login`, `/register`)
- Email + password fields with validation.
- Register collects: display name, phone number, email, password.
- Error messages shown inline (wrong password, email in use, etc.).

### Customer Home (`/customer/home`)
- Large points balance card at the top.
- Two quick-action buttons: **History** and **Rewards**.
- Displays the customer's display name and a greeting.

### Transaction History (`/customer/history`)
- Chronological list of all `transactions` where `customerId == uid`.
- Each row shows: type icon, note, point delta (+/-), date.
- Paginated using Firestore `startAfter` cursor (20 items per page).

### Rewards (`/customer/rewards`)
- Grid or list of all active rewards from the `rewards` collection.
- Each card shows title, description, points cost.
- "Redeem" button disabled if customer's balance < pointsCost.
- Tapping Redeem triggers the redeem flow (see Section 9).

---

## 6. Staff Screens

### Staff Home (`/staff/home`)
- Search bar: enter customer phone number.
- Displays matching customer card on result.
- Tap customer card to navigate to Customer Detail.

### Customer Detail (`/staff/customer/:uid`)
- Shows customer's name, phone, current points balance.
- **Add Points** button — opens a bottom sheet (see Section 8).
- Recent transaction history for that customer (last 10 entries).

### Redemption Queue (`/staff/redemptions`)
- Lists all `redemptions` with `status: "pending"`.
- Each item shows: customer name, reward title, points cost, date.
- **Mark Fulfilled** button updates status and records `fulfilledBy` + `fulfilledAt`.
- Accessible from Staff Home via a bottom nav or app bar action.

---

## 7. Role-Based Routing

GoRouter is used with a `redirect` callback that fires on every route change.

```
/login           → public (redirect away if already signed in)
/register        → public
/customer/*      → requires role == "customer"
/staff/*         → requires role == "staff"
/                → redirect to /login (fallback)
```

**Guard logic (in `app_router.dart`):**
1. If auth state is loading → show splash/loading screen, no redirect.
2. If user is null → redirect to `/login` (unless already there).
3. If user role is `customer` and route starts with `/staff` → redirect to `/customer/home`.
4. If user role is `staff` and route starts with `/customer` → redirect to `/staff/home`.
5. Otherwise allow navigation.

The `AuthProvider` (Riverpod) exposes `AsyncValue<AppUser?>` so the router can react to auth state changes reactively.

---

## 8. Add-Points Flow (Staff)

1. Staff taps **Add Points** on the Customer Detail screen.
2. A `ModalBottomSheet` opens with:
   - Points amount input (numeric, min 1).
   - Note / reason text field (optional, e.g. "Purchase $50").
   - **Confirm** button.
3. On confirm, `PointsService.addPoints()` runs a **Firestore batch write**:
   - Increments `users/{customerId}.points` by the entered amount (`FieldValue.increment`).
   - Creates a new document in `transactions/` with `type: "add"`.
4. On success: bottom sheet closes, Customer Detail screen refreshes balance via stream.
5. On error: snackbar with error message, no partial writes (batch is atomic).

---

## 9. Redeem Reward Flow (Customer)

1. Customer taps **Redeem** on a reward card.
2. A confirmation dialog shows reward name and points cost.
3. On confirm, `RewardService.redeemReward()` runs a **Firestore transaction**:
   - Reads `users/{uid}.points` inside the transaction.
   - Aborts if balance < pointsCost (prevents race conditions).
   - Decrements `users/{uid}.points` by `pointsCost` (`FieldValue.increment(-cost)`).
   - Creates a document in `redemptions/` with `status: "pending"`.
   - Creates a document in `transactions/` with `type: "redeem"`.
4. On success: dialog closes, balance updates, a success snackbar shown.
5. On failure (insufficient points or network error): error dialog, no state changes.

Staff sees the new `redemptions` entry in their Redemption Queue and marks it fulfilled when the customer collects the reward in person.

---

## 10. Required Packages

Add to `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.8

  # Firebase
  firebase_core: ^3.6.0
  firebase_auth: ^5.3.1
  cloud_firestore: ^5.4.4

  # State management
  flutter_riverpod: ^2.5.1
  riverpod_annotation: ^2.3.5

  # Navigation
  go_router: ^14.3.0

  # Utilities
  intl: ^0.19.0           # date/number formatting
  uuid: ^4.4.2            # generate local IDs if needed

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^6.0.0
  build_runner: ^2.4.12
  riverpod_generator: ^2.4.3
  custom_lint: ^0.6.4
  riverpod_lint: ^2.3.10
```

> Version numbers are approximate; pin to the latest stable versions when adding.

---

## 11. Security Notes

### Firestore Security Rules (to be written before going live)

```
// users collection
- read: user can read own document; staff can read any document
- write: user can update own document only for display name / phone;
         points field must only be incremented via trusted server-side logic

// transactions collection
- read: customer reads own transactions; staff reads all
- create: only staff role (enforced via custom claim or role check)

// rewards collection
- read: authenticated users only
- write: deny all from client (managed via console or Cloud Functions)

// redemptions collection
- read: customer reads own; staff reads all
- create: customer (own document only, validated balance server-side)
- update: staff only (to mark fulfilled)
```

### Additional Notes
- **Never trust the client for points arithmetic.** The `FieldValue.increment` approach combined with Firestore Security Rules that deny direct overwrites of the `points` field is sufficient for MVP. For production, move point mutations to Cloud Functions.
- **Role is stored in Firestore, not in the Auth token** for MVP simplicity. For production, set Firebase Custom Claims so the role is available in Security Rules as `request.auth.token.role`.
- **Phone numbers** are stored as plain strings for search. Do not store sensitive PII beyond what is needed.
- **No API keys** should be committed. The `google-services.json` (Android) and `GoogleService-Info.plist` (iOS) files must be added to `.gitignore`.
- Validate all user input (points amount, phone format) on the client before writing to Firestore.

---

## 12. Step-by-Step Build Checklist

### Phase 1 — Firebase Setup
- [ ] Create a Firebase project in the Firebase Console
- [ ] Enable **Email/Password** authentication
- [ ] Create Firestore database in production mode
- [ ] Add Android app (`com.example.loyalty_app`) → download `google-services.json` → place in `android/app/`
- [ ] Add iOS app → download `GoogleService-Info.plist` → place in `ios/Runner/`
- [ ] Add `google-services.json` and `GoogleService-Info.plist` to `.gitignore`
- [ ] Manually seed at least one `rewards` document and one `staff` user in Firestore Console for testing

### Phase 2 — Package Integration
- [ ] Update `pubspec.yaml` with all packages from Section 10
- [ ] Run `flutter pub get`
- [ ] Configure `firebase_core` in `main.dart` (`Firebase.initializeApp`)
- [ ] Configure `android/build.gradle.kts` and `android/app/build.gradle.kts` for Google Services plugin
- [ ] Configure `ios/Runner/Info.plist` if needed for Firebase

### Phase 3 — Core Infrastructure
- [ ] Create `AppUser`, `Transaction`, `Reward`, `Redemption` model classes with `fromFirestore` / `toMap`
- [ ] Create `AuthService` (signIn, signOut, register, authStateStream)
- [ ] Create `UserService` (getUser, searchByPhone, stream of user doc)
- [ ] Create `PointsService` (addPoints using batch write)
- [ ] Create `RewardService` (getRewards stream, redeemReward using transaction)
- [ ] Create Riverpod providers for auth state and current user
- [ ] Set up `GoRouter` with role-based redirect guard (Section 7)

### Phase 4 — Auth Screens
- [ ] Build `LoginScreen` (email, password, sign-in button, link to register)
- [ ] Build `RegisterScreen` (name, phone, email, password, register button)
- [ ] Wire up form validation and error display
- [ ] Test: register new customer, log in, log out

### Phase 5 — Customer Screens
- [ ] Build `CustomerHomeScreen` with `PointsCard` widget
- [ ] Build `HistoryScreen` with paginated transaction list
- [ ] Build `RewardsScreen` with reward cards and Redeem button logic
- [ ] Implement redeem confirmation dialog and `redeemReward` call (Section 9)
- [ ] Test full customer redeem flow end-to-end

### Phase 6 — Staff Screens
- [ ] Build `StaffHomeScreen` with phone number search
- [ ] Build `CustomerDetailScreen` with balance display and Add Points bottom sheet
- [ ] Implement `addPoints` call and success/error feedback (Section 8)
- [ ] Build `RedemptionScreen` with pending queue and Mark Fulfilled action
- [ ] Test staff add-points and redemption fulfilment flow end-to-end

### Phase 7 — Security Rules & Cleanup
- [ ] Write and deploy Firestore Security Rules (Section 11)
- [ ] Test rules: confirm customers cannot write points directly
- [ ] Test rules: confirm staff can read all users but not customers cannot
- [ ] Review and tighten `.gitignore`
- [ ] Remove unused boilerplate (`MyHomePage` counter widget)
- [ ] Set proper app display name in `AndroidManifest.xml` and `Info.plist`

### Phase 8 — QA & Polish
- [ ] Manual regression: customer register → view balance → redeem reward → view history
- [ ] Manual regression: staff login → search customer → add points → fulfil redemption
- [ ] Test on physical iOS device or iOS Simulator
- [ ] Test on Android device or emulator (once setup is complete)
- [ ] Fix any layout issues on small screens (SE-size)
- [ ] Tag `v0.1.0-mvp` in git when stable

---

## 13. Project Phase Board

> **Phase 1 — Planning** · Status: `Done`
>
> Goal: Define MVP scope, architecture, Firebase schema, auth flow, routing, and build checklist.
>
> Output:
> - `docs/LoyaltyAppPlan.md` created
> - `docs/ProjectMemory.md` created

---

> **Phase 2 — Setup** · Status: `In Progress`
>
> Goal: Complete Flutter environment, Android emulator, Firebase project, Auth, Firestore, and config files.
>
> Output:
> - [ ] Android emulator ready
> - [ ] Firebase project created
> - [ ] `google-services.json` added to `android/app/`
> - [ ] `GoogleService-Info.plist` added to `ios/Runner/`
> - [ ] Firebase Auth (Email/Password) enabled
> - [ ] Firestore database created

---

> **Phase 3 — Foundation** · Status: `Not Started`
>
> Goal: Add packages, folder structure, theme, model classes, services, providers, and routing.
>
> Output:
> - [ ] `pubspec.yaml` updated with all packages
> - [ ] Firebase initialized in `main.dart`
> - [ ] Riverpod providers ready
> - [ ] GoRouter role guard ready

---

> **Phase 4 — Auth** · Status: `Not Started`
>
> Goal: Build customer register, shared login, logout, and role-based redirect.
>
> Output:
> - [ ] Customer registration screen
> - [ ] Shared login screen (staff + customer)
> - [ ] Redirect to customer home after login
> - [ ] Redirect to staff home after login

---

> **Phase 5 — Customer Features** · Status: `Not Started`
>
> Goal: Build customer home, points balance, rewards list, redeem flow, and transaction history.
>
> Output:
> - [ ] Customer home screen (points balance card)
> - [ ] Rewards screen (browse + redeem)
> - [ ] Redeem confirmation flow
> - [ ] Transaction history screen

---

> **Phase 6 — Staff Features** · Status: `Not Started`
>
> Goal: Build staff dashboard, customer search, add-points flow, and redemption fulfilment.
>
> Output:
> - [ ] Staff home screen (phone number search)
> - [ ] Customer detail screen
> - [ ] Add points bottom sheet flow
> - [ ] Redemption queue screen

---

> **Phase 7 — Security** · Status: `Not Started`
>
> Goal: Write and deploy Firestore security rules to protect points and role fields.
>
> Output:
> - [ ] Customers cannot write to `points` field directly
> - [ ] Only staff can create `transactions`
> - [ ] `rewards` collection is read-only from client
> - [ ] `redemptions` protected — only staff can update

---

> **Phase 8 — QA** · Status: `Not Started`
>
> Goal: Test full app flow on Android and iOS.
>
> Output:
> - [ ] Customer register → login → view balance tested
> - [ ] Staff login → search → add points tested
> - [ ] Redeem flow tested end-to-end
> - [ ] Android build tested (once emulator ready)

---

*Last updated: 2026-06-15 — Phase Board added. Planning Phase done. Setup Phase in progress.*

*End of MVP Plan*
