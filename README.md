# FinFit - Smart Lost and Found Manager (AI-Powered)

A full-stack, beginner-friendly mobile application and REST API system designed to connect people who lost items with those who found them. Featuring automated, rule-based matching combined with **NLP Text Similarity (TF-IDF)**, **Computer Vision Auto-Categorization** powered by the Anthropic API, **Email OTP Account Verification**, **Real-Time Socket.io Chat**, **FCM Push Notifications**, **Interactive Map Location & GeoJSON Nearby Queries**, and **Privacy-Preserving Belonging QR Code Tags**.

---

## 🔒 Phase 1: Security & Trust

### 1. Email OTP Verification System
- **Registration OTP:** Upon registering via `POST /api/auth/register`, a cryptographically random 6-digit numeric OTP is generated and stored with a **10-minute expiration** window.
- **Nodemailer SMTP Delivery:** Transmitted using Nodemailer configured via `.env` (`SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM`).
- **Dev Mode Fallback:** If SMTP credentials are not configured or email delivery fails, the OTP is printed directly to the server console in a highlighted ASCII box (`[DEV MODE] OTP FOR user@example.com: 123456`), guaranteeing effortless local testing without an active email server.
- **Login Blocking:** Unverified users are rejected on `POST /api/auth/login` with `403 Forbidden` (`requiresVerification: true`), preventing unauthorized or spam accounts from logging in.
- **Resend with Cooldown:** `POST /api/auth/resend-otp` allows requesting a fresh OTP with an enforced **60-second cooldown** to prevent email spamming (`429 Too Many Requests`).
- **Flutter OTP Screen:** Dedicated `OtpVerificationScreen` featuring 6-digit PIN input, a live 60s countdown timer for resending, and seamless login token persistence upon success.

### 2. Secret Claim Verification Questions
- **Found Item Secret Questions:** When posting a `FOUND` item, the finder can specify **1 to 3 secret verification questions** with expected answers (e.g. *"What is the phone lock screen wallpaper?"*, *"What brand of earphones are in the pocket?"*).
- **Hashed & Lowercased Storage:** Answers are trimmed, lowercased, and salted/hashed using `bcryptjs` before storage in MongoDB. **The raw answers and hashes are NEVER sent to the client** (`select: false` and strict JSON serialization transformations).
- **Claimant Challenge:** Anyone attempting to submit an ownership claim (`POST /api/claims`) for a found item with verification questions must submit their answers.
- **Automated Verification Score:** The backend compares lowercased claimant answers against the stored bcrypt hashes and computes a percentage `verificationScore` (0–100%) and details (`totalQuestions`, `correctAnswers`, `passed`).
- **Admin Review:** Administrators see the calculated verification score and individual question checkmarks (✓ / ✗) on claim cards and review dialogs before approving or rejecting ownership claims.

### 3. Report Fake/Spam & Automated Moderation
- **Community Reporting Endpoint:** `POST /api/reports` allows authenticated users to flag suspicious, fraudulent, or spam listings with categorized reasons.
- **3-Report Auto-Hide Threshold:** Upon receiving **3 or more reports**, the item is automatically flagged (`isFlagged: true`) and hidden (`isHidden: true`), disappearing from public feeds.
- **Admin Moderation:** Admins can review all reports, dismiss reports to unhide items, or permanently delete spam items.

---

## ⚡ Phase 2: Real-Time & Notifications

### 4. Real-Time Chat with Socket.io
- **Private Room Gating:** Chat rooms (`claim_${claimId}`) are strictly gated to the item owner and the item finder (and admins). An unauthorized intruder attempting to join is blocked with `403 Forbidden`.
- **JWT-Authenticated Socket Connection:** Socket handshakes verify the user's JWT token and associate `socket.user`.
- **Persistent Message History:** All chat messages are stored in MongoDB under the `Message` collection (`claim`, `item`, `sender`, `recipient`, `content`, `read`, timestamps) and can be retrieved anytime via `GET /api/chat/claim/:claimId/messages`.
- **Live Typing Indicator:** Real-time `typing` events broadcast live typing status ("Alice is typing...") to the other participant.
- **Flutter Integration:**
  - `ChatScreen`: Full real-time message exchange powered by `socket_io_client`, with custom bubble styling, read status checkmarks, auto-scroll, and debounced typing indicator.
  - `MyChatsScreen`: Lists all active conversations with item previews and unread message badges. Accessible from the home app bar and drawer.

### 5. Push Notifications (Firebase Cloud Messaging - FCM)
- **FCM Token Registration:** Saves device push tokens on login (`fcmToken` in User model) or via `POST /api/auth/fcm-token`.
- **Automated Push Triggers:**
  - **Strong Match Found:** Dispatched when a new item has a match score of 70% or higher.
  - **Claim Status Updated:** Dispatched to the claimant when their claim is approved or rejected by an admin.
  - **New Chat Message:** Dispatched to the recipient when a private chat message is sent.
- **Graceful Dev Fallback:** Uses `firebase-admin`. If Firebase credentials are not provided in `.env`, the system seamlessly logs a fallback message without crashing, and in-app alerts continue to function 100%.

#### Firebase FCM Setup Instructions:
1. Go to the [Firebase Console](https://console.firebase.google.com/) and create a project.
2. In Project Settings > Service Accounts, click **Generate new private key**.
3. Save the downloaded JSON file as `backend/firebase-service-account.json`.
4. In `backend/.env`, set:
   ```env
   FIREBASE_SERVICE_ACCOUNT_KEY=./firebase-service-account.json
   ```

---

## 🗺️ Phase 3: Location and QR

### 6. Map Location & MongoDB 2dsphere Nearby Search
- **GPS Coordinates on Items:** Adds `latitude`, `longitude`, and GeoJSON `locationCoordinates: { type: 'Point', coordinates: [lng, lat] }` to the Item model.
- **MongoDB 2dsphere Indexing:** Items are indexed with a sparse `2dsphere` index, enabling high-performance geospatial queries.
- **Nearby Items API:**
  - `GET /api/items/nearby?lat=...&lng=...&maxDistance=10000&type=FOUND`
  - Uses MongoDB `$near` and `$geometry` to query items within a radius (default 10 km) sorted by distance.
- **Flutter Google Maps Integration:**
  - `LocationPickerScreen`: Interactive map where users tap or drag a pin to pinpoint exact item locations in the Add Item screen.
  - `ItemDetailsScreen`: Shows a Google Maps pin preview widget with exact coordinates for items with location data.
  - `ItemsMapScreen`: Interactive map showing all open lost (red pins) and found (green pins) items, with category chips, type filtering, item preview cards, and a "Search Nearby Items" action.

#### Google Maps API Key Setup Guide:
The Google Maps API key is configured safely without hardcoding:
1. Obtain an API key from the [Google Cloud Console](https://console.cloud.google.com/) with the **Maps SDK for Android** and **Maps JavaScript API** enabled.
2. **Android:** In `finfit_app/android/app/src/main/AndroidManifest.xml`, update the meta-data element:
   ```xml
   <meta-data
       android:name="com.google.android.geo.API_KEY"
       android:value="YOUR_ACTUAL_GOOGLE_MAPS_API_KEY_HERE" />
   ```
   Or pass it at build time using `--dart-define=MAPS_API_KEY=YOUR_KEY`.
3. **Web:** In `finfit_app/web/index.html`, add:
   ```html
   <script src="https://maps.googleapis.com/maps/api/js?key=YOUR_KEY"></script>
   ```

### 7. Belonging QR Code Tags
- **Generate Belonging Tags:** Authenticated users can generate QR tags for their personal belongings (laptops, keys, backpacks, wallets) via `POST /api/qr/tags`.
- **Privacy-Preserving Public Scan:**
  - Scanning the tag opens `GET /api/qr/:tagId`.
  - **Strict Privacy Protection:** Shows the belonging name, category, and custom note from the owner, but **NEVER leaks the owner's phone number or email address**. The owner name is displayed as safe verified pseudonym (e.g. *"Alice (Verified Owner)"*).
- **Finder "I Found This Item" Flow:**
  - Finder submits a report via `POST /api/qr/:tagId/found` with their message, location, and optional contact info.
  - The backend immediately logs the report, alerts the owner via in-app notification, and dispatches an FCM push notification.
- **Flutter UI Integration:**
  - `QrTagsScreen`: Manage belongings, view received finder reports, and delete tags.
  - `QrDisplayDialog`: Renders high-resolution QR codes using `qr_flutter` for viewing and printing.
  - `QrScannerScreen`: Live camera barcode scanner powered by `mobile_scanner`, with a manual Tag ID entry option for simulators/desktop.
  - `QrFoundReportScreen`: Safe finder form that alerts the owner directly.

---

## ⏰ Phase 4: Automation and Admin

### 8. Automated Item Expiry & Owner Expiry Alerts (node-cron)
- **Daily Midnight Cron Schedule:** Configured with `node-cron` running daily at midnight (`0 0 * * *`).
- **30-Day Expiry:** Open items created more than 30 days ago are automatically transitioned to `status: 'expired'` with `expiredAt: Date.now()`.
- **3-Day Advance Warning Alert:** When an item is 27 days old (3 days before expiration) and warning hasn't been dispatched yet (`expiryWarningSent: false`):
  - Sends an in-app notification alert to the item owner: *"Your item will expire in 3 days. If it has not been found or claimed, please renew or update your report."*
  - Automatically flags `expiryWarningSent = true` to prevent duplicate alerts.
- **Manual Admin Trigger Endpoint:**
  - `POST /api/admin/trigger-expiry` (Admin-only).
  - Allows administrators to instantly execute the expiry and warning cycle on demand without waiting for midnight, returning `{ expiredCount, warningCount }`.
- **Flutter UI Integration:** Admin Dashboard Quick Controls features a **"Run Auto-Expiry Check Now"** button with a real-time progress indicator and feedback toast.

### 9. Comprehensive Admin Dashboard & Analytics (fl_chart)
- **System Statistics API:** `GET /api/admin/stats` returns real-time aggregated metrics:
  - `totalUsers`: Total registered system users.
  - `itemsByType`: Breakdown between `LOST` and `FOUND` listings.
  - `itemsByStatus`: Complete lifecycle counts (`open`, `matched`, `claimed`, `returned`, `expired`).
  - `claimsByStatus`: Verification queue counts (`pending`, `approved`, `rejected`, `total`).
  - `monthlyReturns`: Historical aggregate of successfully returned items across the **past 6 calendar months**, dynamically grouped by year and month.
- **Interactive Flutter Visualizations (fl_chart):**
  - **Item Status Pie Chart:** Interactive color-coded donut/pie chart displaying percentages and item quantities for open (blue), matched (orange), claimed (purple), returned (green), and expired (red) items with a detailed legend.
  - **Monthly Returns Bar Chart:** High-resolution vertical bar chart showing successful item returns month-by-month over the past 6 months, complete with interactive touch tooltips, dynamic Y-axis scaling, and subtle grid lines.
  - **Overview Stat Cards:** High-level metric tiles for Total Items, Lost Reports, Found Reports, Returned Items, Expired Items, Total Claims, Pending Claims, and Registered Users.

---

## 🚀 AI Features

### 1. NLP Text Similarity Matching (TF-IDF & Cosine Similarity)
- Powered by the `natural` library.
- Extracts tokenized terms from item `title` and `description`, converts text to lowercase, and strips common English stop words.
- Computes Term Frequency–Inverse Document Frequency (TF-IDF) vectors across lost and found descriptions.
- Calculates Cosine Similarity between the vectors and yields a text match score from **0 to 30 points**.
- Combines with rule-based factors (Category: 40, Location: 30, Color: 20, Date: 10, Text: 30 = 130 max) and normalizes to **0–100**.
- Classifies match strength as:
  - 🟢 **Strong (70+)**
  - 🟡 **Possible (40–69)**
  - ⚪ **Weak (< 40)**
- Displays a visual progress bar for each individual factor in the Flutter Matches screen.

### 2. Auto-Category and Color from Photo (Anthropic API)
- Protected endpoint: `POST /api/ai/analyze-image`.
- Reads `ANTHROPIC_API_KEY` and `ANTHROPIC_MODEL` from `.env`.
- Sends the uploaded image as Base64 to Claude, requesting strict JSON output:
  ```json
  {
    "category": "Phone | Wallet | ID Card | Bag | Keys | Laptop | Books | Jewellery | Clothing | Other",
    "color": "Black | Blue | Red | ...",
    "title": "Short descriptive title",
    "description": "1-2 sentence description"
  }
  ```
- **Robust Fallback:** If the API key is missing or the external call fails, the endpoint returns an empty suggestion template and user-friendly notice. **The app never crashes.**
- **Flutter UI Integration:** When picking a photo on the Add Item screen, an AI analysis loading banner appears, and fields are automatically populated for the user to review and edit before submitting.

### 3. Automated Notification on Strong Match
- When an item is saved via `POST /api/items`, the backend immediately compares it against all opposing items (LOST ↔ FOUND).
- If any candidate achieves a score of **70 or above (Strong)**:
  - Both items' statuses are automatically updated to `"matched"`.
  - In-app notification alerts are generated for both the current user and the candidate item's owner.

---

## 🛠 Tech Stack

- **Frontend:** Flutter (Dart) - Modern Material 3 mobile UI
- **State Management:** Provider (`provider` package)
- **Networking:** HTTP (`http` package) + Multipart form data for image uploads
- **Local Storage:** `shared_preferences` (stores JWT securely)
- **Backend:** Node.js + Express.js REST API
- **NLP / AI Engine:** `natural` (TF-IDF & Cosine Similarity) + `@anthropic-ai/sdk`
- **Database:** MongoDB with Mongoose ODM
- **Authentication:** JSON Web Tokens (JWT) + bcryptjs password hashing
- **Image Uploads:** Multer (local `/uploads` directory)

---

## 📁 Project Structure

```
FindWhereYouLost/
├── backend/
│   ├── config/
│   │   └── db.js                 # MongoDB connection (supports MONGO_URI and MONGODB_URI)
│   ├── controllers/
│   │   ├── authController.js     # User registration, login, profile, user listing
│   │   ├── itemController.js     # CRUD for items, NLP match score calculation
│   │   ├── aiController.js       # Anthropic image analysis & category detection
│   │   ├── claimController.js    # Claim submission, review & approve/reject
│   │   ├── reportController.js   # Community report creation, auto-hide, dismiss & delete
│   │   ├── notificationController.js # In-app notification alerts
│   │   └── adminController.js    # System metrics and statistics
│   ├── middleware/
│   │   ├── auth.js               # JWT verification & req.user attachment
│   │   ├── admin.js              # Administrator role check middleware
│   │   └── upload.js             # Multer diskStorage & file filters
│   ├── models/
│   │   ├── User.js               # User schema (roles: user, admin)
│   │   ├── Item.js               # Item schema (LOST/FOUND, status, isFlagged, isHidden)
│   │   ├── Claim.js              # Claim schema (item, claimant, proof, status, verificationAnswers)
│   │   ├── Report.js             # Community reports schema
│   │   └── Notification.js       # In-app alerts schema
│   ├── routes/
│   │   ├── authRoutes.js         # /api/auth
│   │   ├── itemRoutes.js         # /api/items
│   │   ├── aiRoutes.js           # /api/ai/analyze-image
│   │   ├── claimRoutes.js        # /api/claims
│   │   ├── reportRoutes.js       # /api/reports (report, dismiss, delete)
│   │   ├── notificationRoutes.js # /api/notifications
│   │   └── adminRoutes.js        # /api/admin
│   ├── utils/
│   │   ├── textSimilarity.js     # TF-IDF & Cosine Similarity NLP engine
│   │   └── emailService.js       # Nodemailer OTP email sender & console dev fallback
│   ├── uploads/                  # Uploaded item & proof images
│   ├── .env                      # Environment variables
│   ├── .env.example              # Template with MONGO_URI, JWT_SECRET, SMTP_*, ANTHROPIC_*
│   ├── server.js                 # Express server entry point
│   ├── seed.js                   # Database seed script
│   ├── test_otp.js               # Email OTP verification automated test suite
│   ├── test_claim_questions.js   # Secret verification questions automated test suite
│   ├── test_reports.js           # Fake/spam reporting and 3-report auto-hide test suite
│   ├── test_match.js             # Unit tests for textScore and matchScore
│   ├── test_api.js               # API integration tests (includes AI fallback test)
│   ├── requests.http             # Plain-text HTTP test requests (includes OTP flows & reports)
│   ├── FinFit_API.postman_collection.json # Postman API collection
│   └── package.json
│
├── finfit_app/
│   ├── lib/
│   │   ├── constants/
│   │   │   ├── api_constants.dart # Endpoints, allowed categories, colors
│   │   │   └── app_theme.dart    # Theme & color styles
│   │   ├── models/
│   │   │   ├── user_model.dart   # Includes isVerified attribute
│   │   │   ├── item_model.dart
│   │   │   ├── match_result_model.dart # Factor breakdown (category, location, color, date, text)
│   │   │   ├── claim_model.dart
│   │   │   ├── report_model.dart
│   │   │   └── notification_model.dart
│   │   ├── services/
│   │   │   ├── api_service.dart  # HTTP client & token persistence
│   │   │   ├── auth_service.dart # Register, login, verifyOtp, resendOtp
│   │   │   ├── item_service.dart # Item CRUD, AI image analysis call
│   │   │   ├── claim_service.dart
│   │   │   ├── report_service.dart
│   │   │   └── notification_service.dart
│   │   ├── providers/
│   │   │   ├── auth_provider.dart
│   │   │   ├── item_provider.dart
│   │   │   ├── claim_provider.dart
│   │   │   ├── report_provider.dart
│   │   │   └── notification_provider.dart
│   │   │   ├── claim_provider.dart
│   │   │   └── notification_provider.dart
│   │   ├── widgets/
│   │   │   ├── item_card.dart
│   │   │   ├── smart_match_card.dart # Visual progress bar per match factor
│   │   │   ├── status_badge.dart
│   │   │   ├── custom_text_field.dart
│   │   │   └── empty_state_view.dart
│   │   ├── screens/
│   │   │   ├── splash_screen.dart
│   │   │   ├── login_screen.dart
│   │   │   ├── register_screen.dart
│   │   │   ├── otp_verification_screen.dart # 6-digit PIN input & 60s resend timer
│   │   │   ├── home_screen.dart
│   │   │   ├── add_item_screen.dart  # AI Auto-fill loading & form review
│   │   │   ├── item_details_screen.dart
│   │   │   ├── matches_screen.dart   # Ranked matches with progress bars
│   │   │   ├── my_claims_screen.dart
│   │   │   ├── notifications_screen.dart
│   │   │   └── admin_dashboard_screen.dart
│   │   └── main.dart
│   ├── pubspec.yaml
│   └── test/widget_test.dart
│
└── README.md
```

---

## ⚡ Setup & Run Instructions

### 1. Environment Configuration

In `backend/.env`:
```env
PORT=5000
MONGO_URI=mongodb://127.0.0.1:27017/finfit
JWT_SECRET=finfit_super_secret_jwt_key_2026_xyz
JWT_EXPIRES_IN=7d

# Nodemailer SMTP Configuration (Optional: fallback prints 6-digit OTP directly to console)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
SMTP_FROM="FinFit Support" <no-reply@finfit.com>

# Anthropic API Configuration (Optional: image analysis uses graceful fallback if empty)
ANTHROPIC_API_KEY=your_anthropic_api_key_here
ANTHROPIC_MODEL=claude-3-5-sonnet-20241022
```

---

### 2. Run the Backend

```bash
cd backend
npm install
npm run seed
npm start
```
The server will run on `http://localhost:5000`.

---

### 3. Run the Flutter App

```bash
cd finfit_app
flutter pub get
flutter run
```

---

## 🧠 Technical Deep-Dive

### How TF-IDF Text Scoring Works
1. **Tokenization & Stop-Word Removal:** The lost and found descriptions (and titles) are cleaned into lowercase words. English stop words (`the`, `is`, `at`, `which`, `with`, etc.) and lost-and-found generic words (`lost`, `found`, `item`) are filtered out.
2. **Term Frequency (TF):** Measures how frequently a word appears in a description:
   $$\text{TF}(t, d) = \frac{\text{count}(t, d)}{\text{total terms in } d}$$
3. **Inverse Document Frequency (IDF):** Penalizes words that appear across both documents, emphasizing distinctive keywords (e.g., `Hilfiger`, `MacBook`, `Passcode`, `Lanyard`):
   $$\text{IDF}(t) = 1 + \ln\left(\frac{N}{\text{DF}(t) + 1}\right)$$
4. **Cosine Similarity:** Computes the cosine of the angle between the two TF-IDF vectors:
   $$\text{Cosine Similarity} = \frac{\mathbf{v_1} \cdot \mathbf{v_2}}{\|\mathbf{v_1}\| \|\mathbf{v_2}\|}$$
5. **Score Scaling:** Scaled to range **0 to 30 points**.

### Combined Scoring Formula & Normalization
$$\text{Raw Total} = \text{Category}(40) + \text{Location}(30) + \text{Color}(20) + \text{Date}(10) + \text{Text}(30)$$
$$\text{Normalized Score} = \min\left(100, \text{round}\left(\frac{\text{Raw Total}}{130} \times 100\right)\right)$$

### How Image Analysis (Anthropic API) Works
1. User captures or selects a photo in the Flutter app.
2. Flutter sends a multipart request to `POST /api/ai/analyze-image` with the JWT token.
3. Express reads the file into a base64 buffer and calls Anthropic API with model `ANTHROPIC_MODEL`.
4. Claude processes the image and responds with strict JSON containing `category`, `color`, `title`, and `description`.
5. The backend parses and validates the category against the allowed categories list (`Phone`, `Wallet`, `ID Card`, `Bag`, `Keys`, `Laptop`, `Books`, `Jewellery`, `Clothing`, `Other`).
6. **Graceful Fallback:** If `ANTHROPIC_API_KEY` is not provided or network fails, the backend catches the error and returns `{ success: true, fallback: true, data: { ... } }`, preventing any crash and letting the user complete the report manually.

---

## 🧪 Automated Testing

### 1. Email OTP Verification Suite
```bash
cd backend
node test_otp.js
```
*Validates registration OTP generation, login blocking for unverified users, incorrect OTP rejection, 60-second resend cooldown (`429`), successful OTP verification, and subsequent valid login.*

### 2. Claim Verification Questions Suite
```bash
cd backend
node test_claim_questions.js
```
*Validates posting a FOUND item with secret questions, verifies answerHash is never leaked over API, verifies claimant answer submission, calculates 100% and 50% verification scores, verifies Admin score visibility, and approves claim.*

### 3. Report Fake/Spam & Auto-Hide Suite
```bash
cd backend
node test_reports.js
```
*Validates community reporting, self-reporting block, duplicate reporting prevention, auto-hiding item on 3rd report, public item filtering, admin report review, admin report dismissal (unhiding item), and admin item deletion.*

### 4. Phase 2: Socket.io Real-Time Chat & FCM Notification Suite
```bash
cd backend
node test_chat_socket.js
```
*Validates Socket.io JWT authentication, private claim room gating (intruders rejected with 403), bidirectional real-time message broadcasting, live typing indicators, FCM token registration, graceful push notification fallback, and message history persistence.*

### 5. Phase 3: Location (2dsphere $near) & QR Tag Suite
```bash
cd backend
node test_phase3_location_qr.js
```
*Validates item creation with GPS coordinates, MongoDB 2dsphere indexing, geospatial $near query radius and distance sorting, QR tag generation, privacy-preserving public scanning (no owner contact leak), and finder "I found this item" report submission.*

### 6. Phase 4: Auto-Expiry Cron & Admin Dashboard Analytics Suite
```bash
cd backend
node test_phase4_autoexpiry_stats.js
```
*Validates 30-day item auto-expiry logic, 27-day (3 days prior) advance warning notification dispatch to owner, `POST /api/admin/trigger-expiry` manual administrator trigger, `GET /api/admin/stats` system statistics payload, and 6-month historical returned items aggregation.*

### 7. NLP Text Similarity & Match Scoring Unit Tests
```bash
cd backend
node test_match.js
```
*Validates identical text (30 pts), partial overlap, disjoint text (0 pts), and normalization into Strong, Possible, and Weak categories.*

### 8. End-to-End API Integration Tests
```bash
cd backend
node test_api.js
```
*Tests Auth, Items, Smart Matches, Claims, Notifications, Admin Stats, and the AI Image Analysis Fallback mechanism.*

### 9. Flutter Analysis & Widget Tests
```bash
cd finfit_app
flutter analyze
flutter test
```
*Ensures 0 analysis warnings and passes widget test suite.*
