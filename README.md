# AP Connect

A cross-platform Flutter app to manage the relationship between sales personnel and fabricators. Built for Android, iOS, and web. Currently deployed as a browser version and used by regional and national sales heads at aluplast India.

Screenshots available in ```/screenshots```.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Flutter (Dart) |
| State management | Riverpod (`StreamProvider`, `FutureProvider`, `StateNotifier`) |
| Navigation | GoRouter with role-based redirect guards |
| Backend | Firebase (Auth, Firestore, Storage, Hosting) |
| Email | Firebase Trigger Email extension (Gmail SMTP) |
| Charts | fl_chart |
| Backup | Google Cloud Storage + Cloud Scheduler (cron) |

---
## Architecture

### Role-based access

Four roles with distinct data visibility, enforced at both the router and Firestore rules level:

| Role | Designation | Scope |
|---|---|---|
| `sales_person` | `nsm` | All India — all regions, all channels |
| `sales_person` | `rsm` | Their assigned region only |
| `sales_person` | `sales_person` | Their assigned fabricators only |
| `fabricator` | — | Their own data only |

### State management

Three chained Riverpod providers handle auth state end-to-end:

- `authStateProvider` — `StreamProvider` wrapping `FirebaseAuth.authStateChanges()`
- `currentUserProvider` — async generator that bridges Firebase auth state into a live Firestore profile stream; holds the router on the splash screen until both resolve
- `appRouterProvider` — GoRouter created once using a `ChangeNotifier` refresh pattern; `ref.listen` (not `ref.watch`) prevents the router from being recreated on every auth state change

### Navigation

GoRouter redirect guard enforces role and designation routing. On login, users are routed to one of five shell screens (`NsmShell`, `RsmShell`, `SalesPersonShell`, `FabricatorShell`, `QualityShell`). Cross-role navigation is blocked at the redirect level.

---

## Authentication Flow

Two-step registration gated by an email allowlist:

1. User enters email → app checks `allowed_users/{email}` in Firestore
2. On success, form reveals remaining fields; OTP is sent to the phone number stored in the allowlist
3. Phone sign-in via `FirebaseAuth.verifyPhoneNumber()` → `signInWithCredential(PhoneAuthCredential)`
4. Email+password linked to the phone account via `user.linkWithCredential(EmailAuthProvider.credential(...))`
5. Firestore `users/{uid}` doc written on completion

Session expiry is tracked locally via `SharedPreferences` (30-day inactivity window).

---

## Firestore Data Model

```
allowed_users/{email}     — registration allowlist; role, designation, region, phone
users/{uid}               — user profile; role, designation, region, companyName
claims/{claimId}          — fabricator quality claims with photo URLs and status lifecycle
promotions/{promoId}      — promotional material uploaded by NSM
sales_records/{invoiceNo} — individual sales transactions (written by Apps Script)
monthly_summaries/{id}    — pre-aggregated monthly figures by region/channel (Apps Script)
fabricators/{id}          — fabricator master list (Apps Script)
settings/{month}          — monthly sales targets set by NSM/RSM
mail/{docId}              — outbound email queue consumed by Trigger Email extension
```

### Firestore security rules highlights

- `allowed_users` is publicly readable (allowlist check happens before registration, before auth exists)
- `claims` write access restricted to fabricators for their own claims; update restricted to quality role
- `sales_records`, `monthly_summaries`, `fabricators` are read-only from the client — writes blocked at the rules level; data is written exclusively by Apps Script
- `users` doc prevents client-side mutation of `role`, `designation`, and `region` fields via `affectedKeys()` check

---

## Backup

Firestore data is backed up daily to a **Google Cloud Storage bucket** using a **Cloud Scheduler cron job** that triggers a Firestore managed export. 

---

## Firebase Services

| Service | Usage |
|---|---|
| Firebase Auth | Email+password + Phone OTP; credential linking |
| Cloud Firestore | Primary database |
| Firebase Storage | Claim photos (`claims/{claimId}/`), promotion images (`promotions/{promoId}/`) |
| Firebase Hosting | Web build (`flutter build web` → `firebase deploy`) |
| Trigger Email extension | Invitation emails on user creation, via Gmail App Password |

---

## Key Packages

```yaml
flutter_riverpod, riverpod_annotation  # state management
go_router                               # navigation
firebase_auth, cloud_firestore          # backend
firebase_storage, firebase_messaging    # storage + push
fl_chart                                # dashboard charts
image_picker, file_picker               # claim photo upload
cached_network_image                    # promotion images
intl                                    # date/number formatting
```
