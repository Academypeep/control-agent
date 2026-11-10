# Mobile App Production Readiness Audit (iOS & Android)

This audit documents production readiness requirements and remediation procedures for shipping and operating apps on the Apple App Store and Google Play (covering Native Swift/Kotlin, Flutter, and React Native).

---

## 1. Security (Auth, Tokens, Storage, Network, Secrets, Third-Party SDKs)

### 1.1 Unprotected API Secrets & Private Keys in the Binary
- **Severity:** Blocker
- **Exact Problem:** Hardcoding private API keys, backend server secrets, or cloud credentials (e.g., OpenAI, Stripe secret keys, AWS access tokens) directly in client source files, embedded `.env` bundles, or asset folders.
- **Why it causes crashes, rejections, or bad reviews:** App binaries (`.ipa`, `.apk`, `.aab`) are public client artifacts easily decompiled or inspected via `jadx`, `apktool`, or strings dump. Automated scanners on Google Play and Apple flag exposed secrets, leading to quota theft, account suspension, and backend compromise.
- **Concrete Fix:**
  - Route all privileged operations through an authenticated Backend-For-Frontend (BFF) proxy. The client app must never hold private server secrets.
  - Restrict public client keys (Firebase, Google Maps) in their developer consoles by Bundle ID, Android SHA-256 fingerprint, and HTTP referrer.
  - Inject environment variables via CI/CD into native build files (`BuildConfig` on Android, `Secrets.xcconfig` ignored by git on iOS).

### 1.2 Insecure Auth Token and PII Storage in Plaintext Preferences
- **Severity:** Blocker
- **Exact Problem:** Storing JWT refresh tokens, access tokens, passwords, or PII in unencrypted storage:
  - iOS: `UserDefaults`
  - Android: `SharedPreferences`
  - React Native: `AsyncStorage` / `localStorage`
  - Flutter: `shared_preferences`
- **Why it causes crashes, rejections, or bad reviews:** Plaintext preferences are backed up in unencrypted device backups (iTunes/iCloud and Android ADB), and can be extracted directly from rooted/jailbroken devices or physical forensic dumps. Security audits and compliance regulations mandate secure hardware-backed storage.
- **Concrete Fix:**
  - **iOS:** Use the Keychain Services API with `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`.
  - **Android:** Use `EncryptedSharedPreferences` / Android Jetpack Security backed by the Android Keystore (TEE / StrongBox).
  - **React Native:** Use `react-native-keychain` or `expo-secure-store`.
  - **Flutter:** Use `flutter_secure_storage` with `AndroidOptions(encryptedSharedPreferences: true)` and `IOSOptions(accessibility: KeychainAccessibility.first_unlock_this_device)`.

### 1.3 Permissive Network Security (Cleartext HTTP & Missing SSL Policies)
- **Severity:** High
- **Exact Problem:** Allowing cleartext HTTP connections or permitting arbitrary loads without strict TLS enforcement.
- **Why it causes crashes, rejections, or bad reviews:** Apple App Transport Security (ATS) rejects apps with `NSAllowsArbitraryLoads = true` without detailed justification (Guideline 2.5.4). Google Play flags `cleartextTrafficPermitted="true"` as a high-severity security risk. User traffic on public Wi-Fi is vulnerable to MITM interception.
- **Concrete Fix:**
  - **iOS `Info.plist`:** Enforce ATS (`NSAllowsArbitraryLoads = false`).
  - **Android `res/xml/network_security_config.xml`:** Explicitly set `cleartextTrafficPermitted="false"`.
  - Implement SSL certificate public key pinning on critical authentication and transaction endpoints.

### 1.4 Missing Apple Privacy Manifests for Third-Party SDKs
- **Severity:** Blocker
- **Exact Problem:** Utilizing third-party SDKs (Firebase, Sentry, Meta, AppsFlyer) without bundling a valid `PrivacyInfo.xcprivacy` manifest or using Required Reason APIs without explicit declarations.
- **Why it causes crashes, rejections, or bad reviews:** Apple automatically rejects binary uploads in App Store Connect that invoke Required Reason APIs (file timestamp, disk space, user defaults, system boot time) without a corresponding privacy manifest and valid declared reason.
- **Concrete Fix:**
  - Update all dependencies to versions containing verified Apple Privacy Manifests.
  - Create a root `PrivacyInfo.xcprivacy` declaring all `NSPrivacyAccessedAPITypes` and data collection categories corresponding to the app’s telemetry.

---

## 2. Privacy & Compliance (Privacy Policy, Consent, Account Deletion, ATT / Data Safety)

### 2.1 Missing In-App Account Deletion
- **Severity:** Blocker
- **Exact Problem:** The application supports account registration but does not offer an accessible, self-service in-app deletion flow.
- **Why it causes crashes, rejections, or bad reviews:** Guaranteed App Store rejection under Apple Guideline 5.1.1(v) and Google Play Account Deletion Policy. Providing only an email mailto link or contact form is strictly rejected if account creation is handled in-app.
- **Concrete Fix:**
  - Add an in-app "Delete Account" button within the user's account/security settings.
  - Require re-authentication, explain data consequences, and trigger an automated API call that purges or schedules a purge while revoking all active sessions immediately.
  - Provide an external web deletion URL on Google Play Console as required by Play Store policy.

### 2.2 Tracking Users Prior to ATT Consent & Store Questionnaire Discrepancies
- **Severity:** Blocker
- **Exact Problem:** Calling advertising or tracking APIs (IDFA access, attribution SDKs) before the user grants permission via App Tracking Transparency (ATT), or publishing App Store Privacy Nutrition Labels and Google Play Data Safety forms that do not match network payloads.
- **Why it causes crashes, rejections, or bad reviews:** Apple Guideline 5.1.2 rejection and Google Play Data Safety compliance takedowns. Store reviewers inspect outbound traffic for undeclared identifiers.
- **Concrete Fix:**
  - Gate tracking SDK initialization behind `ATTrackingManager.requestTrackingAuthorization()`. If denied, run the SDK in anonymous/non-tracking mode.
  - Proxy outbound traffic in release builds using Proxyman/Charles to audit every collected parameter.
  - Align App Store Privacy Labels and Google Play Data Safety forms 1-to-1 with actual network transmissions.

### 2.3 Inaccessible or Incompliant Privacy Policy Link
- **Severity:** Blocker
- **Exact Problem:** The privacy policy URL provided in store metadata or in-app settings returns a 404/500 error, requires a login, or lacks third-party processor disclosures.
- **Why it causes crashes, rejections, or bad reviews:** Reviewers test privacy links across desktop and mobile browsers. Non-functional URLs cause immediate rejection under App Store Guideline 5.1.1 and Google Play Policy.
- **Concrete Fix:**
  - Ensure the privacy policy is hosted on a high-availability HTTPS URL without authentication gates.
  - Enumerate third-party sub-processors (analytics, crash reporting, cloud providers) and specify user rights and contact info.

---

## 3. Reliability (Crash Handling, Offline Mode, Error States, Background Tasks, Push)

### 3.1 Missing Global Uncaught Exception Handlers & Symbolicated Crash Reporting
- **Severity:** Blocker
- **Exact Problem:** Uncaught JavaScript promise rejections, Flutter isolate errors, or native fatal signals crashing directly to the OS home screen without reporting or with un-symbolicated hex stack traces.
- **Why it causes crashes, rejections, or bad reviews:** Exceeding Google Play's Bad Behavior threshold (>1.09% crash rate) penalizes organic search discovery and risks app removal. High crash rates lead directly to 1-star reviews.
- **Concrete Fix:**
  - Integrate Sentry or Firebase Crashlytics.
  - Automate symbolication file uploads in CI/CD (upload Android Proguard/R8 `mapping.txt` and iOS dSYM bundles).
  - Register global error boundaries and top-level error hooks (`ErrorUtils.setGlobalHandler` in React Native, `FlutterError.onError` / `PlatformDispatcher.instance.onError` in Flutter).

### 3.2 Infinite Loading Spinners on Network Drops (Missing Offline States)
- **Severity:** High
- **Exact Problem:** When a device experiences spotty connectivity or server 502/503 errors, screens hang indefinitely on loading skeletons or blank views.
- **Why it causes crashes, rejections, or bad reviews:** App Store Guideline 2.1 (Performance: App Completeness). Reviewers test apps on throttled networks and airplane mode; apps that hang without actionable error states are rejected.
- **Concrete Fix:**
  - Monitor network reachability via `NWPathMonitor`, Android `ConnectivityManager`, or community NetInfo libraries.
  - Provide distinct UI states across all views: Loading, Error (with a retry button), and Empty.
  - Implement network timeouts (10-15s) and cache recent critical read data locally.

### 3.3 Cold-Start Push Notification & Deep Link Routing Race Conditions
- **Severity:** High
- **Exact Problem:** Opening a push notification or Universal Link when the app is completely terminated causes a cold launch that fails to navigate, crashes, or drops the target route because authentication state is unhydrated.
- **Why it causes crashes, rejections, or bad reviews:** Broken user journeys, 1-star reviews, and App Review rejections during deep-link verification.
- **Concrete Fix:**
  - Buffer incoming notification intents or deep link URLs until the root navigation container and auth state are ready.
  - Implement a centralized router that checks authentication: if unauthenticated, save the pending target route, complete login, and resume navigation.

---

## 4. Performance (Startup Time, Memory, Battery, Large Lists, Image Handling)

### 4.1 Blocking the Main Thread on Cold Startup
- **Severity:** High
- **Exact Problem:** Initializing all analytics, remote config, ad SDKs, and heavy local database migrations synchronously on the main thread during app launch.
- **Why it causes crashes, rejections, or bad reviews:** iOS Watchdog kills apps that block the main thread on startup for more than 5–20 seconds (`0x8badf00d` crash). Android Vitals penalizes cold start times exceeding 5 seconds.
- **Concrete Fix:**
  - Defer non-critical SDK initializations to background threads or after the initial frame renders.
  - Implement native splash screens (`androidx.core.splashscreen` on Android; LaunchScreen storyboard on iOS) to immediately display brand UI during initialization.
  - Ensure Hermes is enabled on React Native (`hermesEnabled = true`).

### 4.2 Out-Of-Memory (OOM) Crashes Due to Unbounded Image Loading
- **Severity:** High
- **Exact Problem:** Loading full-resolution remote images or camera photos directly without specifying target dimensions or memory bounds.
- **Why it causes crashes, rejections, or bad reviews:** OOM crashes (`EXC_RESOURCE RESOURCE_TYPE_MEMORY` on iOS; `java.lang.OutOfMemoryError` on Android), especially on budget Android devices and older iPhones.
- **Concrete Fix:**
  - Downsample images to the exact target render size before memory allocation.
  - Use memory-managed image loaders:
    - **iOS:** Kingfisher or Nuke.
    - **Android:** Coil or Glide.
    - **React Native:** `react-native-fast-image` or `expo-image`.
    - **Flutter:** `cached_network_image` with `memCacheWidth` / `memCacheHeight`.

### 4.3 Un-Recycled Large Lists Causing Frame Drops
- **Severity:** Medium
- **Exact Problem:** Rendering large lists with standard `ScrollView` and `.map()`, or un-recycled views without view recycling or fixed item keys.
- **Why it causes crashes, rejections, or bad reviews:** Scrolling stutters below 30fps, causing sluggish UI feedback and poor store reviews.
- **Concrete Fix:**
  - **React Native:** Use `@shopify/flash-list` in place of `FlatList`.
  - **Flutter:** Use `ListView.builder` or `SliverList` with `prototypeItem`.
  - **Android Compose:** Use `LazyColumn` with stable, unique `key` parameters.
  - **iOS:** Use `UICollectionViewDiffableDataSource` or SwiftUI `List` with `Identifiable` elements.

---

## 5. Environment & Configuration (Separation, Flags, Logs)

### 5.1 Missing Staging vs Production Separation
- **Severity:** High
- **Exact Problem:** Mixing staging API URLs or test sandbox credentials in production builds, or using a single Bundle ID across development and release.
- **Why it causes crashes, rejections, or bad reviews:** Risk of corrupting production data during QA testing or accidentally releasing test endpoints to the live App Store.
- **Concrete Fix:**
  - Configure distinct schemes and bundle identifiers:
    - Dev: `com.company.app.dev`
    - Staging: `com.company.app.staging`
    - Prod: `com.company.app`
  - Separate `.env` and `.xcconfig` configs per flavor in CI/CD pipelines.

### 5.2 Verbose Console Logging & Debug Tools Left in Release Builds
- **Severity:** High
- **Exact Problem:** Leaving `console.log`, `print`, or debug loggers (e.g. Flipper, React DevTools) enabled in release binaries.
- **Why it causes crashes, rejections, or bad reviews:** Logs can expose sensitive PII and auth tokens via system logs (readable by external debugging tools or malicious apps). Console logging slows down the JS/main thread significantly.
- **Concrete Fix:**
  - Strip console logs in production via `babel-plugin-transform-remove-console` (React Native) or Proguard/R8 rules (Android):
    `-assumenosideeffects class android.util.Log { *; }`
  - Remove all Flipper / development middleware from production builds.

---

## 6. Store Requirements (App Store Guidelines & Play Store Policies)

### 6.1 Non-Compliance with Apple Guideline 3.1.1 (In-App Purchases)
- **Severity:** Blocker
- **Exact Problem:** Unlocking digital features via external payment links (e.g., Stripe, web checkout) without using StoreKit In-App Purchases, or omitting a "Restore Purchases" button on the paywall screen.
- **Why it causes crashes, rejections, or bad reviews:** Instant App Store rejection under Guideline 3.1.1. Reviewers test every digital purchase link and check for the "Restore Purchases" button.
- **Concrete Fix:**
  - Route all digital subscriptions and goods through StoreKit 2 (or RevenueCat/Adapty) on iOS, and Google Play Billing v6+ on Android.
  - Include an explicit, functioning **"Restore Purchases"** button on paywalls.
  - Place links to Terms of Use (EULA) and Privacy Policy directly on the subscription paywall.

### 6.2 Missing Demo Account Credentials for App Review
- **Severity:** Blocker
- **Exact Problem:** Submitting an app that requires authentication without providing functional login credentials in the App Review Information box, or using SMS 2FA that reviewers cannot complete.
- **Why it causes crashes, rejections, or bad reviews:** App Reviewers cannot log in and immediately reject the build under Guideline 2.1 (Information Needed).
- **Concrete Fix:**
  - Provide dedicated demo account credentials with pre-seeded data in App Store Connect and Google Play Console.
  - Disable SMS 2FA on the demo account or configure a hardcoded OTP code (e.g., `123456`) in your auth backend for review accounts.

### 6.3 Apple Guideline 4.8 (Missing "Sign in with Apple")
- **Severity:** Blocker
- **Exact Problem:** Offering third-party social logins (Google, Facebook, Twitter) without offering "Sign in with Apple" as an equivalent option.
- **Why it causes crashes, rejections, or bad reviews:** Mandatory App Store rejection under Guideline 4.8.
- **Concrete Fix:**
  - If social auth is enabled, integrate `AuthenticationServices` ("Sign in with Apple") using standard Apple design specifications.

### 6.4 Android Target SDK & 64-bit Architecture Compliance
- **Severity:** Blocker
- **Exact Problem:** Android build targeting an API level lower than the annual Google Play mandate (API level 34 for new submissions and updates), or missing 64-bit native libraries (`arm64-v8a`).
- **Why it causes crashes, rejections, or bad reviews:** Google Play Console will reject `.aab` uploads that do not target API level 34 or do not include 64-bit libraries.
- **Concrete Fix:**
  - Update `build.gradle`: `compileSdk = 34` and `targetSdk = 34`.
  - Build and submit using **Android App Bundle (`.aab`)** format.

---

## 7. Polish & Missing Details

### 7.1 UI Clipping Under Notches, Safe Areas, and Dynamic Island
- **Severity:** Medium
- **Exact Problem:** Text or buttons overlapping the status bar, bottom Home Indicator, or camera cutouts.
- **Why it causes crashes, rejections, or bad reviews:** Rejections under design guidelines and negative reviews on modern devices.
- **Concrete Fix:**
  - Wrap top and bottom view hierarchies in platform safe area views (`SafeAreaView`, `SafeArea`, or native safeAreaInsets).

### 7.2 Text Truncation with Dynamic Type / Large Accessibility Fonts
- **Severity:** Medium
- **Exact Problem:** Using hardcoded container heights (`height: 48px`) on text elements, causing text truncation or unreadable overlapping when accessibility font scaling is enabled.
- **Why it causes crashes, rejections, or bad reviews:** Poor accessibility ratings and customer complaints.
- **Concrete Fix:**
  - Use dynamic, flexible container heights (`minHeight` instead of fixed `height`).
  - Test layouts in simulators with font scaling set to the maximum accessibility size.

---

## 8. Testing Gaps (Critical Edge Cases)

### 8.1 Process Death & Activity Recreation
- **Severity:** High
- **Exact Problem:** OS kills backgrounded apps to reclaim RAM. When users reopen the app, it crashes due to uninitialized singleton objects or lost state.
- **Why it causes crashes, rejections, or bad reviews:** Frequent cause of unexplained field crashes.
- **Concrete Fix:**
  - Android: Test with "Don't keep activities" enabled in Developer Options. Preserve screen state with `SavedStateHandle`.
  - iOS: Handle state restoration or ensure lazy re-initialization of core services upon resume.

### 8.2 Biometric Invalidation on Enrollment Changes
- **Severity:** High
- **Exact Problem:** Relying on Face ID / Biometrics without invalidating sessions when new fingerprints or faces are enrolled in device settings.
- **Why it causes crashes, rejections, or bad reviews:** Security bypass vulnerability.
- **Concrete Fix:**
  - Use `kSecAccessControlBiometryCurrentSet` in Keychain on iOS.
  - Require authentication bound to the current enrollment set on Android.

---

## Top 7 Blockers to Fix Before Submitting

1. **In-App Account Deletion Flow:** Provide an in-app button that confirms and executes deletion (Apple Guideline 5.1.1(v) & Play Store Policy).
2. **Reviewer Demo Credentials:** Provide working credentials without SMS 2FA in the App Review Notes to prevent Guideline 2.1 rejections.
3. **Privacy Manifests & Data Safety Match:** Bundle `PrivacyInfo.xcprivacy` and ensure App Privacy Labels accurately reflect all network traffic.
4. **Hardcoded Secrets & Cleartext Traffic:** Ensure no backend private keys are in client binaries and enforce HTTPS everywhere.
5. **In-App Purchase Compliance & "Restore Purchases":** Use StoreKit/Google Play Billing for digital goods and include a "Restore Purchases" button on all paywalls.
6. **Sign in with Apple:** Provide "Sign in with Apple" alongside any third-party social logins (Guideline 4.8).
7. **Production Crash Reporting & Symbolication:** Integrate Sentry or Crashlytics with automated symbolication (dSYM / R8 mapping upload) in CI/CD.

---

## Prioritized Action List (What to Do This Week)

### Day 1: Store Compliance & Account Lifecycle (Prevent Instant Rejections)
- [ ] Implement and test the in-app "Delete Account" flow with backend revocation.
- [ ] Verify StoreKit / Google Play Billing integration and ensure the **"Restore Purchases"** button works on paywalls.
- [ ] Add working demo credentials with pre-loaded mock data into App Store Connect and Google Play Console review notes.
- [ ] Verify that public Privacy Policy and Terms of Service URLs return HTTP 200 OK and render cleanly on mobile.

### Day 2: Security & Credentials Hardening
- [ ] Audit codebase for hardcoded private API keys, secrets, and test tokens; route all private operations through your backend.
- [ ] Migrate auth tokens and sensitive PII out of unencrypted storage into iOS Keychain and Android Keystore.
- [ ] Disable cleartext HTTP traffic across all iOS and Android build configurations.
- [ ] Audit all third-party SDKs and generate Apple Privacy Manifests (`PrivacyInfo.xcprivacy`).

### Day 3: Error States, Reliability & Offline Grace
- [ ] Integrate and verify crash reporting (Sentry / Firebase Crashlytics) in release mode; trigger a test crash and verify symbolicated stack traces.
- [ ] Test the entire app in Airplane mode: ensure every screen renders a retryable error message instead of hanging on an infinite spinner or crashing.
- [ ] Verify cold-start deep links and push notification clicks when the app is completely killed.

### Day 4: Performance & Memory Optimization
- [ ] Enable Hermes (React Native) or verify Flutter AOT / Proguard-R8 optimization (Android).
- [ ] Audit image components: enforce downsampling and caching to prevent OOM crashes on large lists.
- [ ] Profile cold startup time: defer non-essential third-party SDK initializations until after the first frame renders.
- [ ] Strip all `console.log` and debug print statements from release builds.

### Day 5: Release Build Verification & Polish
- [ ] Verify Android Adaptive Icons and iOS App Icon sets on real physical devices (ensure no white borders or cut-offs).
- [ ] Test keyboard avoidance on all input forms (login, registration, checkout, search).
- [ ] Toggle Dark Mode / Light Mode and verify text contrast and accessibility readability across all screens.
- [ ] Test on a low-end Android device and older iPhone model to confirm smooth 60fps scrolling and responsive interaction.
- [ ] Tag the git release commit, auto-increment the build number, and upload to TestFlight / Google Play Internal Testing.
