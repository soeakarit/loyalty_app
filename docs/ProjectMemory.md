# ProjectMemory.md — Loyalty App

> **Purpose:** This is the single source of truth for any Claude Code session (or human developer) picking up this project. Update it whenever a major task is completed.
>
> **Do not store:** API keys, Firebase private keys, passwords, tokens, or any secrets.

---

## 1. Project Overview

A **loyalty points MVP app** for iOS and Android built with Flutter + Firebase.

Customers earn points at a business. Staff add points after purchases. Customers redeem points for rewards. Staff fulfil redemptions in person.

Two roles: **Customer** and **Staff**. No admin dashboard in MVP.

---

## 2. Current Goal

Set up Firebase (project creation, Auth, Firestore, config files) and then implement the Flutter app phase by phase following `docs/LoyaltyAppPlan.md`.

---

## 3. Tech Stack

| Layer | Choice |
|---|---|
| Framework | Flutter (Dart), targets iOS & Android |
| Auth | Firebase Authentication (email + password) |
| Database | Cloud Firestore |
| State management | flutter_riverpod |
| Navigation | go_router (role-based redirect guard) |
| Backend logic | Firestore batch writes + transactions (MVP); Cloud Functions post-MVP |
| IDE | Android Studio / VS Code |
| Version control | Git (main branch) |

---

## 4. Current Project Status

**Planning Phase: Done. Setup Phase: In Progress. Architecture decision added: Firebase MVP now, Node.js API + MySQL possible later.**

- Android emulator setup is **still pending** — do not run `flutter run` until confirmed ready.
- iOS Simulator may be used for early testing if on macOS.
- No Firebase project has been created yet.
- No packages beyond Flutter defaults have been added to `pubspec.yaml`.
- App code is still the default Flutter counter boilerplate.

---

## 5. Completed Work

| Date | Task | Notes |
|---|---|---|
| 2026-06-15 | Project scaffolded | `flutter create loyalty_app` — standard boilerplate |
| 2026-06-15 | Initial git commit | Branch: `main` |
| 2026-06-15 | `docs/LoyaltyAppPlan.md` created | Full MVP plan — 12 sections |
| 2026-06-15 | `docs/ProjectMemory.md` created | This file |

---

## 6. Pending Work

- [ ] Create Firebase project in Firebase Console
- [ ] Enable Email/Password auth in Firebase Console
- [ ] Create Firestore database (production mode)
- [ ] Download `google-services.json` → place in `android/app/`
- [ ] Download `GoogleService-Info.plist` → place in `ios/Runner/`
- [ ] Add both config files to `.gitignore`
- [ ] Seed one `rewards` document and one `staff` user in Firestore Console
- [ ] Update `pubspec.yaml` with all required packages
- [ ] Run `flutter pub get`
- [ ] Configure Firebase in `main.dart`
- [ ] Build model classes (`AppUser`, `Transaction`, `Reward`, `Redemption`)
- [ ] Build services (`AuthService`, `UserService`, `PointsService`, `RewardService`)
- [ ] Build Riverpod providers
- [ ] Build GoRouter with role-based guard
- [ ] Build Auth screens (Login, Register)
- [ ] Build Customer screens (Home, History, Rewards)
- [ ] Build Staff screens (Home, Customer Detail, Redemption Queue)
- [ ] Write and deploy Firestore Security Rules
- [ ] QA both role flows end-to-end
- [ ] Complete Android emulator setup and test on Android

---

## 7. Important Decisions

| Decision | Reason |
|---|---|
| Riverpod (not Provider or Bloc) | Modern, compile-safe, works well with async Firebase streams |
| GoRouter (not Navigator 2.0 raw) | Declarative, supports redirect guards cleanly |
| Firestore batch write for add-points | Atomic: increments balance AND writes transaction record together |
| Firestore transaction for redeem | Prevents race condition if two redemptions fire simultaneously |
| Role stored in Firestore `users/{uid}.role` (not Custom Claims) | Simpler for MVP; Custom Claims deferred to post-MVP |
| No Cloud Functions in MVP | Reduces setup complexity; Firestore Security Rules enforce write constraints instead |
| Staff accounts pre-created manually | No self-registration for staff in MVP; avoids open role escalation |
| Single `docs/ProjectMemory.md` as live context file | Survives across Claude sessions and can be shared with any AI or dev |
| Backend Replacement Strategy | Firebase is MVP backend; screens/widgets must never call Firebase directly; all access via service layer; Node.js + MySQL may replace Firebase later |

### Backend Replacement Strategy

Firebase is chosen for MVP speed, not as a permanent dependency. The service layer must be built so the backend can be swapped later without rewriting any screen or widget.

1. **Firebase is the MVP backend** — fast to set up, not a long-term lock-in.
2. **Future backend may be Node.js REST API + MySQL** — build the code to allow this swap.
3. **Screens and widgets must never import Firebase** — no `FirebaseAuth`, `FirebaseFirestore`, or `DocumentSnapshot` in any screen or widget file.
4. **All backend access goes through repository classes:**
   - `AuthRepository` — sign in, sign out, register, auth state stream
   - `UserRepository` — get user, search by phone, stream user profile
   - `PointsRepository` — add points (atomic write)
   - `RewardRepository` — list rewards, get single reward
   - `RedemptionRepository` — create redemption, fulfil, list pending
5. **Services expose app-level method names** — e.g. `getUser(uid)`, not `FirebaseFirestore.instance.doc('users/$uid').get()`.
6. **Each repository has a contract and an implementation:**
   - `services/contracts/` — abstract Dart class (the interface)
   - `services/firebase/` — Firebase implementation (used in MVP)
   - `services/api/` — future Node.js API implementation
7. **Model classes are backend-neutral** — `AppUser`, `PointsTransaction`, `Reward`, `Redemption` use only Dart types.
8. **Firebase types must never leave the service layer** — convert `DocumentSnapshot` and Firebase `User` to model classes at the repository boundary.
9. **Points mutation and redemption logic stay inside service methods** — never computed in screens or providers.
10. **Folder structure supports swapping the implementation:**

```
lib/
  services/
    contracts/
      auth_repository.dart
      user_repository.dart
      points_repository.dart
      reward_repository.dart
      redemption_repository.dart
    firebase/
      firebase_auth_repository.dart
      firebase_user_repository.dart
      firebase_points_repository.dart
      firebase_reward_repository.dart
      firebase_redemption_repository.dart
    api/
      api_auth_repository.dart
      api_user_repository.dart
      api_points_repository.dart
      api_reward_repository.dart
      api_redemption_repository.dart
```

---

## 8. Firebase Plan

### Collections

**`users/{uid}`**
```
uid, email, displayName, phone, role ("customer"|"staff"), points (int), createdAt
```

**`transactions/{txnId}`**
```
customerId, staffId, type ("add"|"redeem"), points, note, createdAt
```

**`rewards/{rewardId}`**
```
title, description, pointsCost, isActive, imageUrl
```

**`redemptions/{redemptionId}`**
```
customerId, rewardId, rewardTitle, pointsCost,
status ("pending"|"fulfilled"|"cancelled"),
createdAt, fulfilledAt, fulfilledBy
```

### Security Rules Summary (to be written)
- Customers: read/write own `users` doc (except `points`); read own `transactions` and `redemptions`; read `rewards`
- Staff: read all `users`; create `transactions`; update `redemptions`
- Points field: only writable via `FieldValue.increment` by staff role (enforced via rules)
- `rewards` collection: read-only from client; written via Firebase Console only

---

## 9. Flutter Folder Structure

```
lib/
├── main.dart
├── app.dart
├── core/
│   ├── constants.dart
│   ├── theme.dart
│   └── utils.dart
├── models/
│   ├── app_user.dart
│   ├── transaction.dart
│   └── reward.dart
├── services/
│   ├── contracts/               # Abstract interfaces (backend-neutral)
│   │   ├── auth_repository.dart
│   │   ├── user_repository.dart
│   │   ├── points_repository.dart
│   │   ├── reward_repository.dart
│   │   └── redemption_repository.dart
│   ├── firebase/                # Firebase implementations (MVP)
│   │   ├── firebase_auth_repository.dart
│   │   ├── firebase_user_repository.dart
│   │   ├── firebase_points_repository.dart
│   │   ├── firebase_reward_repository.dart
│   │   └── firebase_redemption_repository.dart
│   └── api/                     # Future: Node.js API implementations
│       ├── api_auth_repository.dart
│       ├── api_user_repository.dart
│       ├── api_points_repository.dart
│       ├── api_reward_repository.dart
│       └── api_redemption_repository.dart
├── providers/
│   ├── auth_provider.dart
│   ├── user_provider.dart
│   └── reward_provider.dart
├── router/
│   └── app_router.dart
├── screens/
│   ├── auth/
│   │   ├── login_screen.dart
│   │   └── register_screen.dart
│   ├── customer/
│   │   ├── customer_home_screen.dart
│   │   ├── history_screen.dart
│   │   └── rewards_screen.dart
│   └── staff/
│       ├── staff_home_screen.dart
│       ├── customer_detail_screen.dart
│       └── redemption_screen.dart
└── widgets/
    ├── points_card.dart
    ├── transaction_tile.dart
    └── reward_card.dart
```

---

## 10. Known Issues

| Issue | Status |
|---|---|
| Android emulator not set up | Pending — do not run `flutter run` on Android until resolved |
| `lib/main.dart` is still boilerplate counter app | Will be replaced in Phase 3 of build checklist |
| No Firebase config files present | Blocked until Firebase project is created |
| `pubspec.yaml` has no Firebase or Riverpod packages | Blocked until Firebase setup is done |

---

## 11. Commands Used

```bash
# Initial project was created with:
flutter create loyalty_app

# After adding packages to pubspec.yaml, run:
flutter pub get

# To verify Flutter/Dart environment:
flutter doctor

# To run on iOS Simulator (when ready):
flutter run -d ios

# To run on Android (when emulator is ready):
flutter run -d android

# Git workflow:
git add <files>
git commit -m "message"
```

---

## 12. Next Steps

**Immediate next action: Firebase Setup (Phase 1 of build checklist)**

1. Go to [https://console.firebase.google.com](https://console.firebase.google.com)
2. Create a new project named `loyalty-app` (disable Google Analytics for MVP)
3. Enable **Authentication → Sign-in method → Email/Password**
4. Create **Firestore Database** → start in production mode → choose nearest region
5. Add Android app with package name `com.example.loyalty_app` → download `google-services.json` → place in `android/app/`
6. Add iOS app with bundle ID from `ios/Runner/Info.plist` → download `GoogleService-Info.plist` → place in `ios/Runner/`
7. Add both files to `.gitignore`
8. Manually add a `rewards` document and a `staff` user document in Firestore Console for testing
9. Then tell Claude: **"Firebase is set up. Start Phase 2 — add packages to pubspec.yaml."**

---

## 13. Claude Working Rules

Rules for Claude Code sessions working on this project:

1. **Always read this file first** at the start of a session to restore context.
2. **Update this file** after every major task: update Completed Work, Pending Work, Known Issues, and Next Steps.
3. **Do not run `flutter run`** on Android until the user confirms the emulator is set up.
4. **Do not store secrets** in this file or any tracked file — Firebase config files go in `.gitignore`.
5. **Do not modify app code** unless the user explicitly says to start a build phase.
6. **Follow the build checklist** in `docs/LoyaltyAppPlan.md` phase by phase — do not skip ahead.
7. **Use Firestore batch writes** for add-points and **Firestore transactions** for redemptions — never plain single-doc writes for financial operations.
8. **Role guard in GoRouter** must always be verified before adding new routes.
9. **Prefer Riverpod streams** over one-time Future reads for live Firestore data.
10. **Keep this file short and useful** — bullet points over paragraphs.

---

## 14. Project Phase Board

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
> Goal: Add packages, folder structure, theme, model classes, service contracts/adapters, providers, and routing.
>
> Output:
> - [ ] `pubspec.yaml` updated with all packages
> - [ ] Firebase initialized in `main.dart`
> - [ ] Service contracts (abstract interfaces) created in `services/contracts/`
> - [ ] Firebase implementations created in `services/firebase/`
> - [ ] `api/` folder stubbed for future Node.js implementation
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
> Goal: Write and deploy Firestore security rules to protect points and role fields. Document future API security approach for post-MVP Node.js backend.
>
> Output:
> - [ ] Customers cannot write to `points` field directly
> - [ ] Only staff can create `transactions`
> - [ ] `rewards` collection is read-only from client
> - [ ] `redemptions` protected — only staff can update
> - [ ] Future API security plan documented (JWT bearer tokens, server-side point validation, role middleware)

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

*Last updated: 2026-06-15 — Backend Replacement Strategy added. Service layer abstraction (contracts/firebase/api) documented. Planning Phase done. Setup Phase in progress.*
