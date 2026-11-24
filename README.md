# Forá – Complete Technical Analysis
## Encryption Architecture · Pro Plan · Usage Limit Management

> **Scope:** Read-only codebase investigation. No code was changed. All findings are traced to actual source files.

---

## 1. Overall System Architecture

Forá is a **React + TypeScript SPA** (Vite) backed entirely by **Firebase**:

| Layer | Technology |
|---|---|
| Frontend | React 18 + TypeScript, Vite, Tailwind, Radix UI |
| Auth | Firebase Authentication (ID Tokens, JWT) |
| Backend | Firebase Cloud Functions v2 (Node.js / TypeScript) |
| Database | Firestore (NoSQL) |
| File Storage | Firebase Storage |
| Payment | Razorpay (India) |
| Moderation | OpenAI Moderation API |
| Email | Resend |
| Secrets | Google Cloud Secret Manager |
| Crypto | Web Crypto API (browser-native — zero external crypto library) |

The application is a **time-capsule** platform: users write encrypted messages (capsules) addressed to themselves or friends, locked until a future date or recurring event.

---

## 2. Encryption Architecture

### 2A. What Cryptographic Techniques Are Used

The app implements a **hybrid end-to-end encryption (E2EE)** scheme using only the **browser-native Web Crypto API** (`window.crypto.subtle`). Zero external cryptographic libraries are used.

Three cryptographic primitives are combined:

| Primitive | Purpose |
|---|---|
| **PBKDF2 (SHA-256, 310,000 iterations)** | Derives the master key from `uid + serverPepper + salt` |
| **AES-GCM 256-bit** | Encrypts the RSA private key (at rest in Firestore) + encrypts capsule content |
| **RSA-OAEP 2048-bit (SHA-256)** | Wraps the per-capsule AES content key for each recipient |

This is a classic **envelope encryption** pattern:
- Content is encrypted with a symmetric AES key (fast, any size).
- That AES key is then encrypted with each recipient's RSA public key (asymmetric wrapping).

### 2B. Encryption Workflow — Step by Step

#### On First Login / New Account

```
User logs in with Firebase Auth
        ↓
AuthContext.onAuthStateChanged fires
        ↓
userHasKeys(uid) → Firestore: check users/{uid}.publicKey
  → missing → initUserKeys(uid) called
        ↓
1. crypto.getRandomValues(16 bytes) → salt
2. Firestore: setDoc users/{uid} { salt: base64(salt) }
3. Cloud Function getKeyMaterial() called (Firebase Callable)
   - Backend: returns { keyMaterial: uid + FORA_PEPPER }
   - FORA_PEPPER = secret stored in Google Cloud Secret Manager
4. PBKDF2: deriveMasterKey(uid+pepper, salt, 310000 iterations, SHA-256)
   → 256-bit AES-GCM CryptoKey (non-extractable)
5. generateRSAKeyPair() → RSA-OAEP 2048-bit { publicKey, privateKey }
6. AES-GCM encrypt(privateKey_PKCS8_bytes, masterKey) → { encrypted, iv }
7. exportPublicKey(publicKey) → base64(SPKI format)
8. Firestore updateDoc users/{uid} {
     publicKey:           base64_spki,
     encryptedPrivateKey: base64_ciphertext,
     privateKeyIV:        base64_iv,
     keysGeneratedAt:     serverTimestamp
   }
9. privateKey returned as in-memory CryptoKey → stored in React state only
```

#### On Subsequent Logins

```
onAuthStateChanged fires
        ↓
userHasKeys(uid) → true
        ↓
loadPrivateKey(uid):
  1. Firestore: getDoc users/{uid} → { salt, encryptedPrivateKey, privateKeyIV }
  2. Cloud Function getKeyMaterial() → uid + FORA_PEPPER
  3. PBKDF2: re-derive masterKey (same salt, same pepper → same key)
  4. AES-GCM decrypt(encryptedPrivateKey, masterKey, privateKeyIV)
     → raw PKCS8 private key bytes
  5. importKey("pkcs8", bytes, RSA-OAEP, non-extractable, ["decrypt"])
     → CryptoKey stored in React state (memory only)
```

#### Capsule Creation (Encryption)

Triggered inside `handleFinalSubmission()` in [`CreateCapsule.tsx`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/src/pages/CreateCapsule.tsx).

```
User submits capsule form
        ↓
1. allRecipientIds = [creatorUid, ...selectedFriends]
2. getRecipientPublicKeys(allRecipientIds)
   → Firestore: getDoc users/{uid} for each → { uid: base64_publicKey }
3. generateContentKey()
   → AES-GCM 256-bit (extractable, for wrapping)
4. Upload attachments to Firebase Storage (plain, unencrypted)
5. encryptText(title, contentKey) → { ciphertext: base64, iv: base64 }
   encryptText(content, contentKey) → { ciphertext: base64, iv: base64 }
   (each call generates a fresh random 12-byte IV)
6. wrapContentKeyForRecipients(contentKey, publicKeys):
   For each recipient:
     - importPublicKey(base64_spki) → RSA CryptoKey
     - RSA-OAEP encrypt(raw_contentKey_bytes, publicKey)
     → encryptedKeys = { uid1: base64_wrapped, uid2: base64_wrapped, ... }
7. POST /createCapsule {
     encryptedTitle, titleIV,
     encryptedContent, contentIV,
     encryptedKeys,         // per-recipient wrapped AES key
     version: "v2",
     plaintextTitle,        // ≤200 chars, for moderation ONLY, NOT stored
     plaintextPreview,      // ≤1000 chars, for moderation ONLY, NOT stored
     recipients, unlockDate, ... metadata
   }
```

> **Important:** `plaintextTitle` and `plaintextPreview` are sent to the backend **only for OpenAI content moderation** and are explicitly **never stored** in Firestore. The backend stores only the encrypted fields.

#### Capsule Reading (Decryption)

Handled by [`useCapsuleDecryption.ts`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/src/hooks/useCapsuleDecryption.ts).

```
User views a capsule
        ↓
decryptCapsule(capsule) called
  1. If capsule.version !== "v2": return plain text (legacy v1 capsules)
  2. If !privateKey or !user: return locked fallback message
  3. Find: capsule.encryptedKeys[user.uid] → myWrappedKey
     If missing: return "no access" fallback
  4. unwrapContentKey(myWrappedKey, privateKey):
     - RSA-OAEP decrypt(wrappedKey, privateKey) → raw AES key bytes
     - importKey("raw", bytes, AES-GCM, non-extractable) → CryptoKey
  5. decryptText(encryptedTitle, titleIV, contentKey) → plaintext title
     decryptText(encryptedContent, contentIV, contentKey) → plaintext content
  6. Return { title, content }
```

### 2C. Key Management

| Key | Where Stored | Protection |
|---|---|---|
| RSA Public Key | Firestore `users/{uid}.publicKey` (base64 SPKI) | Public — by design |
| RSA Private Key (raw) | **Never stored** — memory only (React state) | Wiped on logout/refresh |
| RSA Private Key (encrypted) | Firestore `users/{uid}.encryptedPrivateKey` | AES-GCM encrypted with master key |
| Private Key IV | Firestore `users/{uid}.privateKeyIV` | Not secret; required for AES-GCM decryption |
| Salt (per-user) | Firestore `users/{uid}.salt` | Not secret; unique per user |
| `FORA_PEPPER` | Google Cloud Secret Manager | Never in client code or Firestore |
| AES Content Key | **Never stored** — derived in memory per session | Wiped on logout/refresh |
| Wrapped Content Keys | Firestore `capsules/{id}.encryptedKeys.{uid}` | RSA-OAEP encrypted — only owner can decrypt |
| Razorpay Key ID | `VITE_RAZORPAY_KEY_ID` env var (public) | Public key — safe to expose |
| Razorpay Key Secret | Google Cloud Secret Manager (`RAZORPAY_KEY_SECRET`) | Never in client |

**Key derivation path:**
```
uid (Firebase UID) + FORA_PEPPER (server secret)
    ──── PBKDF2(SHA-256, 310_000 iterations) ────
salt (per-user random 128-bit)
    ──────────────────────────────────────────────
                    Master Key (AES-GCM 256-bit, non-extractable)
```

**Key rotation:** Not implemented. There is no key rotation mechanism. The RSA key pair generated at account creation is permanent.

**Same key across users:** No. Each user gets their own RSA key pair and their own random salt. The FORA_PEPPER is shared (it is a server-side secret), but the salt is unique per user, ensuring each user's master key is distinct.

### 2D. Security Analysis

#### What Is Actually Implemented

| Security Property | Status | Mechanism |
|---|---|---|
| Capsule content confidentiality | ✅ Implemented | AES-GCM 256-bit |
| Per-recipient access control | ✅ Implemented | RSA-OAEP wrapping per recipient |
| Private key never stored raw | ✅ Implemented | Only AES-encrypted blob in Firestore |
| Private key memory-only | ✅ Implemented | React state, cleared on logout |
| Server-side key material (pepper) | ✅ Implemented | Google Cloud Secret Manager |
| Token-based API authentication | ✅ Implemented | Firebase ID tokens (JWT, RS256) |
| HTTPS transport | ✅ Firebase default | TLS 1.2/1.3 enforced by Firebase |
| Capsule integrity | ✅ Implicit | AES-GCM provides authenticated encryption (AEAD) — tampered ciphertext fails decryption |
| Password hashing | ✅ Firebase Auth | Firebase handles bcrypt/scrypt — not custom code |
| Firestore security rules | ✅ Implemented | Server-enforced read/write access control |
| Storage security rules | ✅ Implemented | Only owner + recipients can read capsule files |
| Payment signature verification | ✅ Implemented | HMAC-SHA256 on `orderId|paymentId` |

#### Distinctions (encryption vs hashing vs TLS)

- **Encryption (AES-GCM):** Applied to capsule title and content. Reversible with the content key.
- **Wrapping (RSA-OAEP):** Applied to the content key itself. Reversible with the recipient's RSA private key.
- **Key derivation (PBKDF2):** Applied to `uid+pepper` to produce the master key. Computationally expensive; not reversible.
- **Password hashing:** Entirely delegated to Firebase Authentication. The app never sees user passwords.
- **TLS:** Firebase Cloud Functions and Firestore enforce HTTPS. This is transport-level protection, not application-level encryption.

#### Attachment Encryption Status

> **⚠️ Attachments (images, video, audio) are uploaded to Firebase Storage UNENCRYPTED.**

In [`CreateCapsule.tsx` line 280](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/src/pages/CreateCapsule.tsx#L280), `encrypted: false` is explicitly set in the attachment metadata. Only text (title + content) is encrypted. Media files are protected only by Firebase Storage security rules.

---

## 3. Pro Plan — Subscription Architecture

### 3A. Plan Tiers

Three tiers exist in the codebase (branded names defined in [`en.ts`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/src/locales/en.ts)):

| Tier Key | Branded Name | Capsule Limit | Reset Interval |
|---|---|---|---|
| `free` | Arki | 5 (lifetime total) | None |
| `starter` | Kyklos | 50 | Monthly |
| `pro` | Aion | Unlimited (`null`) | None |

### 3B. Pricing (Razorpay, INR, in paise)

Defined in [`payment-funtion.ts`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/fora/functions/src/payment-funtion.ts#L54) and overridable by Firestore `config/pricing`:

| Plan | Monthly | Yearly |
|---|---|---|
| Starter | ₹199 (19,900 paise) | ₹999 (99,900 paise) |
| Pro | ₹399 (39,900 paise) | ₹1,999 (199,900 paise) |

### 3C. Where Subscription Data Lives

Subscription data lives in a **dedicated Firestore collection** called `subscriptions`, keyed by user UID:

**Collection:** `subscriptions/{uid}`

| Field | Type | Description |
|---|---|---|
| `tier` | string | `"free"`, `"starter"`, or `"pro"` |
| `billingCycle` | string | `"monthly"` or `"yearly"` |
| `capsuleLimit` | number \| null | 5 (free), 50 (starter), null (pro = unlimited) |
| `usedCapsules` | number | Running count of capsules created |
| `resetInterval` | string \| null | `"monthly"` for starter, `null` otherwise |
| `lastResetAt` | Timestamp | When `usedCapsules` was last reset to 0 |
| `status` | string | `"active"`, `"cancelled"`, `"expired"` |
| `subscriptionStartDate` | Timestamp | Payment verification timestamp |
| `subscriptionEndDate` | Timestamp | +1 month or +1 year from start |
| `previousTier` | string | Set when subscription expires |
| `cancelledAt` | Timestamp | Set on cancellation |
| `willDowngradeAt` | Timestamp | = `subscriptionEndDate` (access until end) |
| `expiredAt` | Timestamp | Set by the daily scheduler |

> **Note:** Previously, some plan data may have been stored inside `users/{uid}.plan`. The `verifyRazorpayPayment` function explicitly runs `FieldValue.delete()` on `users/{uid}.plan` to migrate old records into `subscriptions/{uid}`.

### 3D. Subscription Activation Flow

```
Frontend: startPayment({ planTier, billingCycle })
        ↓
1. user.getIdToken() → Firebase JWT
2. POST /createRazorpayOrder { planTier, billingCycle }
   - Backend: verifyIdToken(token) → uid
   - Reads pricing from Firestore config/pricing (or hardcoded fallback)
   - Calls Razorpay API: POST https://api.razorpay.com/v1/orders
   - Creates Firestore payments/{auto} { status: "pending", ... }
   - Returns { orderId, amount, currency }
3. Frontend: open Razorpay checkout modal
4. User completes payment in Razorpay modal
5. Razorpay calls handler(response) with { razorpay_order_id, razorpay_payment_id, razorpay_signature }
6. user.getIdToken() (fresh) → Firebase JWT
7. POST /verifyRazorpayPayment { order_id, payment_id, signature, planTier, billingCycle }
   - Backend: verifyIdToken(token) → uid
   - HMAC-SHA256 verification: expectedSig = HMAC(orderId|paymentId, RAZORPAY_KEY_SECRET)
   - If expectedSig !== razorpay_signature → HTTP 400 "Invalid payment signature"
   - Fetches payment details from Razorpay API for extra confirmation
   - Firestore transaction:
     a. subscriptions/{uid}: set { tier, capsuleLimit, usedCapsules:0, ... }
     b. users/{uid}: delete plan field (migration cleanup)
     c. payments/{doc}: update to "completed"
```

### 3E. Subscription Cancellation

```
POST /cancelSubscription
        ↓
Backend verifies token → uid
        ↓
subscriptions/{uid} update:
  { status: "cancelled", cancelledAt: now, willDowngradeAt: subscriptionEndDate }
        ↓
payments/{auto} create:
  { type: "cancellation", status: "completed", note: "User cancelled subscription" }
```

Access continues until `subscriptionEndDate`. The plan tier is **not** immediately downgraded.

### 3F. Subscription Expiry (Automated)

Cloud Function `checkExpiredSubscriptions` runs **every 24 hours** via Cloud Scheduler:

```
Query: subscriptions where subscriptionEndDate <= now AND status in ["active","cancelled"]
        ↓
For each expired subscription:
  update { tier: "free", capsuleLimit: 5, status: "expired", previousTier: <old>,
            resetInterval: null, lastResetAt: null, expiredAt: now }
```

This is the **only mechanism** that enforces downgrade after cancellation or non-renewal. If this scheduler fails, users retain paid access until the next run.

### 3G. Payment Verification Security

- HMAC-SHA256 signature verification on the backend using `RAZORPAY_KEY_SECRET` (from Google Cloud Secret Manager).
- The Razorpay API is called again from the backend (`GET /v1/payments/{id}`) to independently confirm payment status.
- Firebase ID token is verified on every backend request via `admin.auth().verifyIdToken(token)`.
- The `RAZORPAY_KEY_SECRET` is **never** in frontend code; only `VITE_RAZORPAY_KEY_ID` (the public key) is in the env.

### 3H. Webhook (Partially Implemented)

The `razorpayWebhook` function exists and verifies the webhook signature, but the `switch(event)` cases for `payment.captured`, `payment.failed`, and `order.paid` are **empty**. The webhook receives and validates events but takes no action on them. This is a confirmed gap — the backend relies entirely on the client-initiated `/verifyRazorpayPayment` flow.

---

## 4. Usage Limits — Technical Implementation

### 4A. Plan-Based Limits

The only limit currently enforced is **capsule count**.

| Feature | Arki (Free) | Kyklos (Starter) | Aion (Pro) | Limit Type |
|---|---:|---:|---:|---|
| Capsule count | 5 | 50 | Unlimited | Lifetime (free) / Monthly (starter) |
| Media uploads | No separate limit | No separate limit | No separate limit | Per capsule (50 MB/file, Storage rule) |
| File size | 50 MB | 50 MB | 50 MB | Per file (Firebase Storage rule) |
| Avatar upload | 10 MB | 10 MB | 10 MB | Per upload (Firebase Storage rule) |
| Recipients | No limit in code | No limit in code | No limit in code | — |
| API requests | No rate limiting in code | No rate limiting in code | No rate limiting in code | — |
| AI moderation | Backend-side, transparent to user | Same | Same | Per capsule creation |

No daily/weekly limits, no AI response limits, no storage quota per user, no project limits exist in the implemented code.

### 4B. Usage Calculation — Exact Mechanism

**Tracking field:** `subscriptions/{uid}.usedCapsules` (integer)

**How it increments:** Inside a Firestore **transaction** in `createCapsule` (Cloud Function):

```typescript
// From index.ts lines 418-493
await db.runTransaction(async (tx) => {
  const subSnap = await tx.get(subRef);
  const plan = subSnap.exists ? subSnap.data() : { tier: "free", usedCapsules: 0 };

  const capsuleLimit = plan.capsuleLimit === undefined ? 5 : plan.capsuleLimit;
  let usedCapsules = plan.usedCapsules ?? 0;

  // Monthly reset check (Starter plan only)
  if (plan.resetInterval === "monthly" && plan.lastResetAt ...) {
    if (isNewMonth) {
      usedCapsules = 0;
      tx.set(subRef, { usedCapsules: 0, lastResetAt: now }, { merge: true });
    }
  }

  // Enforcement gate
  if (capsuleLimit !== null && usedCapsules >= capsuleLimit) {
    throw new Error("CAPSULE_LIMIT_REACHED");
  }

  // Create the capsule document
  tx.set(ref, { ...capsuleFields });

  // Increment usage atomically
  tx.set(subRef, { usedCapsules: FieldValue.increment(1) }, { merge: true });
});
```

Key properties:
- Uses a **Firestore transaction** — read and write are atomic. No race condition between checking limit and incrementing.
- Uses `FieldValue.increment(1)` — server-side atomic increment, not a client read-modify-write.
- If the capsule write fails (e.g., Firestore error), the entire transaction rolls back — usage is not incremented.

**How it decrements:** Inside `deleteCapsule` (Cloud Function):

```typescript
await subRef.set({ usedCapsules: FieldValue.increment(-1) }, { merge: true });
```

This is **outside** the main delete transaction (non-fatal error handling). If Storage cleanup fails, the decrement still runs. If the decrement fails, it logs a warning but doesn't block the delete response.

**How usage is read:** The `getMyProfile` endpoint returns the current `usedCapsules` and `capsuleLimit` from `subscriptions/{uid}`. The frontend `usePlan` hook calls this endpoint on mount and stores the result in React state.

### 4C. Limit Logic — The Actual Code Equivalent

```
Free Plan:
  capsuleLimit = 5
  usedCapsules = running count since account creation (never resets)
  Remaining = 5 − usedCapsules
  If usedCapsules >= 5 → BLOCKED

Starter Plan:
  capsuleLimit = 50
  usedCapsules = running count since last monthly reset
  Monthly reset: if current month ≠ month of lastResetAt → usedCapsules = 0
  Remaining = 50 − usedCapsules
  If usedCapsules >= 50 → BLOCKED

Pro Plan:
  capsuleLimit = null (unlimited)
  Enforcement: if (capsuleLimit !== null && ...) — condition is false → never blocked
```

### 4D. Frontend Limit Check (Pre-flight)

In [`CreateCapsule.tsx` line 321–328](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/src/pages/CreateCapsule.tsx#L321):

```typescript
const hasReachedLimit =
  plan && plan.capsuleLimit !== null && plan.usedCapsules >= plan.capsuleLimit;

const handleFormSubmit = () => {
  if (hasReachedLimit) {
    toast({ title: "Capsule limit reached", ... });
    return;    // blocks form submission before API call
  }
  if (validateForm()) setShowSealModal(true);
};
```

This is a **UI-only pre-flight check** using the plan data fetched on page load. It prevents the submit dialog from appearing when the user is already at their limit. It is **not** the security enforcement — it is UX convenience. The real enforcement is the server-side transaction.

### 4E. Reset Mechanism

| Event | What Happens |
|---|---|
| New calendar month (Starter) | `usedCapsules` reset to 0, `lastResetAt` updated — triggered lazily on next capsule creation |
| Subscription expires (scheduler) | Tier → free, capsuleLimit → 5, usedCapsules preserved, resetInterval → null |
| Subscription cancelled | Status → cancelled, no immediate limit change; downgrade at end of billing period |
| User deletes a capsule | `usedCapsules` decremented by 1 |
| User upgrades from Free → Starter | `usedCapsules` reset to 0 on payment verification |
| User upgrades from Free → Pro | `usedCapsules` reset to 0 on payment verification |

The monthly reset for Starter is **lazy** — it only fires when the user attempts to create a new capsule. There is no background job to reset counters at midnight on the 1st.

### 4F. What Happens When Limit Is Reached

1. Backend transaction detects `usedCapsules >= capsuleLimit` → throws `CAPSULE_LIMIT_REACHED`.
2. Cloud Function responds with `HTTP 403 { error: "Capsule limit reached. Upgrade your plan.", code: "LIMIT_REACHED" }`.
3. Frontend catches the error in the fire-and-forget async block → displays a toast notification.
4. No partial writes occur — the Firestore transaction rolls back atomically.

---

## 5. Database Architecture

### Collection: `users/{uid}`

| Field | Type | Purpose |
|---|---|---|
| `uid` | string | Firebase UID (same as doc ID) |
| `email` | string | User email |
| `username` | string | Display name |
| `name` | string | Full name |
| `photoURL` | string | Avatar URL |
| `bio` | string | Profile bio |
| `location` | string | User location |
| `publicKey` | string | Base64 SPKI RSA public key |
| `encryptedPrivateKey` | string | Base64 AES-GCM encrypted private key |
| `privateKeyIV` | string | Base64 IV for private key decryption |
| `salt` | string | Base64 random 128-bit salt for PBKDF2 |
| `keysGeneratedAt` | Timestamp | When keys were generated |
| `connections` | array | Friend UIDs |
| `stats` | map | Activity statistics |
| `settings` | map | User preferences |
| `onboardingCompleted` | boolean | Onboarding state |
| `privacyLevel` | string | Profile visibility |
| `dateOfBirth` | string | Date of birth |
| `updatedAt` | Timestamp | Last profile update |

### Collection: `subscriptions/{uid}`

| Field | Type | Purpose |
|---|---|---|
| `tier` | string | `"free"`, `"starter"`, `"pro"` |
| `billingCycle` | string | `"monthly"`, `"yearly"`, or absent |
| `capsuleLimit` | number \| null | Max capsules; null = unlimited |
| `usedCapsules` | number | Current capsule count |
| `resetInterval` | string \| null | `"monthly"` for starter |
| `lastResetAt` | Timestamp | Last reset timestamp |
| `status` | string | `"active"`, `"cancelled"`, `"expired"` |
| `subscriptionStartDate` | Timestamp | Activation date |
| `subscriptionEndDate` | Timestamp | Expiry date |
| `previousTier` | string | Set on expiry |
| `cancelledAt` | Timestamp | Cancellation timestamp |
| `willDowngradeAt` | Timestamp | = subscriptionEndDate |
| `expiredAt` | Timestamp | Set by scheduler on expiry |

### Collection: `payments/{auto}`

| Field | Type | Purpose |
|---|---|---|
| `userId` | string | Owner UID |
| `orderId` | string | Razorpay order ID |
| `razorpayPaymentId` | string | Razorpay payment ID |
| `razorpaySignature` | string | HMAC-SHA256 signature (stored for audit) |
| `planTier` | string | `"starter"` or `"pro"` |
| `billingCycle` | string | `"monthly"` or `"yearly"` |
| `amount` | number | Amount in paise |
| `currency` | string | `"INR"` |
| `status` | string | `"pending"`, `"completed"`, `"failed"` |
| `paymentMethod` | string | From Razorpay (upi, card, etc.) |
| `email` | string | From Razorpay |
| `contact` | string | From Razorpay |
| `createdAt` | Timestamp | Order creation |
| `completedAt` | Timestamp | Payment verification timestamp |
| `subscriptionStartDate` | Timestamp | Same as verification time |
| `subscriptionEndDate` | Timestamp | +1 month or +1 year |
| `nextDueDate` | Timestamp | Same as subscriptionEndDate |
| `type` | string | `"cancellation"` for cancellation records |
| `note` | string | Human-readable note (cancellations) |

### Collection: `capsules/{capsuleId}`

| Field | Type | Purpose |
|---|---|---|
| `id` | string | Same as doc ID |
| `version` | string | `"v2"` (encrypted); absent = v1 (legacy plain) |
| `encryptedTitle` | string | Base64 AES-GCM ciphertext of title |
| `titleIV` | string | Base64 12-byte IV for title decryption |
| `encryptedContent` | string | Base64 AES-GCM ciphertext of content |
| `contentIV` | string | Base64 12-byte IV for content decryption |
| `encryptedKeys` | map | `{ uid: base64_RSA_wrapped_AES_key }` per recipient |
| `userId` | string | Creator UID |
| `recipients` | string[] | All recipient UIDs (including creator) |
| `recipientType` | string | `"self"`, `"friends"`, `"public"` |
| `triggerType` | string | `"time"`, `"event"`, `"connection"` |
| `unlockDate` | Timestamp | When the capsule unlocks |
| `isUnlocked` | boolean | Current lock state |
| `status` | string | `"scheduled"`, `"unlocked"` |
| `attachments` | array | Media metadata (storagePath, type, name, size) |
| `category` | string | User-selected category |
| `signature` | string | Firebase Storage URL of drawn signature |
| `createdAt` | Timestamp | Server creation time |
| `unlockedAt` | Timestamp | When unlocked |
| `openedBy` | string[] | UIDs that have opened the capsule |
| `recurrencePattern` | string | `"none"`, `"daily"`, `"weekly"`, `"monthly"`, `"yearly"` |
| `isReminderCapsule` | boolean | Whether this is a recurring reminder |
| `reminderConfig` | map | Reminder schedule config |
| `priorReminderEnabled` | boolean | Whether pre-unlock notification is enabled |
| `priorReminderDays` | number | Days before unlock to send reminder |
| `priorReminderSent` | boolean | Whether reminder was already sent |
| `nextInstanceCreated` | boolean | Recurrence: whether next instance was spawned |

### Collection: `friends/{uid1_uid2}` (sorted, joined by `_`)

| Field | Type | Purpose |
|---|---|---|
| `requesterId` | string | Who sent the request |
| `receiverId` | string | Who received it |
| `participants` | string[] | Both UIDs (for array-contains queries) |
| `status` | string | `"pending"`, `"accepted"`, `"rejected"` |
| `createdAt` | Timestamp | Request timestamp |

### Collection: `notifications/{auto}`

| Field | Type | Purpose |
|---|---|---|
| `userId` | string | Recipient UID |
| `type` | string | `"capsule_unlocked"`, `"prior_reminder"`, `"capsule_collab"` |
| `message` | string | Notification text |
| `read` | boolean | Read state |
| `createdAt` | Timestamp | Creation time |
| `capsuleId` | string | Related capsule (collab type only) |

---

## 6. Important Source Files — Code Walkthrough

| File | Key Function/Class | Responsibility |
|---|---|---|
| [`src/utils/cryptoUtils.ts`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/src/utils/cryptoUtils.ts) | `deriveMasterKey`, `generateRSAKeyPair`, `encryptPrivateKey`, `decryptPrivateKey`, `generateContentKey`, `encryptText`, `decryptText`, `wrapContentKeyForRecipients`, `unwrapContentKey` | All Web Crypto API primitives. Pure functions. Zero external deps. |
| [`src/encryption/keyManager.ts`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/src/encryption/keyManager.ts) | `initUserKeys`, `loadPrivateKey`, `userHasKeys`, `getRecipientPublicKeys` | RSA key pair lifecycle. Coordinates Firestore + Cloud Function + cryptoUtils. |
| [`src/contexts/AuthContext.tsx`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/src/contexts/AuthContext.tsx) | `AuthProvider`, `useAuth` | Holds in-memory private key in React state. Triggers key init/load on login. |
| [`src/hooks/useCapsuleDecryption.ts`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/src/hooks/useCapsuleDecryption.ts) | `useCapsuleDecryption` → `decryptCapsule` | Unwraps content key with private key, decrypts title + content on demand. |
| [`src/pages/CreateCapsule.tsx`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/src/pages/CreateCapsule.tsx) | `handleFinalSubmission` | Full capsule creation: encrypt text, RSA-wrap key per recipient, POST to backend. |
| [`src/hooks/usePlan.ts`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/src/hooks/usePlan.ts) | `usePlan` | Fetches plan info from `/getMyProfile`, provides `{ tier, capsuleLimit, usedCapsules }` to any component. |
| [`src/lib/razorpay.ts`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/src/lib/razorpay.ts) | `startPayment`, `cancelSubscription`, `getPaymentHistory` | Razorpay checkout flow. Creates order, opens modal, posts payment IDs to backend for verification. |
| [`fora/functions/src/index.ts`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/fora/functions/src/index.ts) | `createCapsule`, `getMyProfile`, `getKeyMaterial`, `checkAndUnlockCapsules` | Core backend: capsule creation (with limit enforcement), profile+plan retrieval, pepper distribution, scheduled unlock job. |
| [`fora/functions/src/payment-funtion.ts`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/fora/functions/src/payment-funtion.ts) | `createRazorpayOrder`, `verifyRazorpayPayment`, `cancelSubscription`, `checkExpiredSubscriptions` | Complete payment lifecycle. Signature verification. Subscription state machine. |
| [`firestore.rules`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/firestore.rules) | Firestore security rules | Access control. Capsule create requires v2 encrypted fields. Users can only write whitelisted fields. |
| [`storage.rules`](file:///c:/Users/Rohan/Documents/forzaaaaaaaaa/storage.rules) | Firebase Storage rules | Only owner + recipients can read capsule files. 50 MB per write. |

---

## 7. Frontend-to-Backend Execution Flow

### Full Request Lifecycle: Create Capsule

```
User → React UI (CreateCapsule.tsx)
  │ handleFormSubmit()
  │ hasReachedLimit check (UI pre-flight using cached plan data)
  │ setShowSealModal(true)
  ↓
SealCapsuleModal
  │ User signs / confirms
  │ onSealComplete(signature)
  ↓
handleFinalSubmission()
  │ setShowSealModal(false)
  │ navigate("/dashboard")  ← UI returns immediately
  │
  └── Background async IIFE:
      ├── getRecipientPublicKeys(allRecipientIds) → Firestore (direct SDK)
      ├── generateContentKey() → AES-GCM CryptoKey (browser)
      ├── uploadAllAttachments() → Firebase Storage (direct SDK)
      ├── encryptText(title, contentKey) → { ciphertext, iv }
      ├── encryptText(content, contentKey) → { ciphertext, iv }
      ├── wrapContentKeyForRecipients(contentKey, publicKeys) → encryptedKeys map
      ├── user.getIdToken() → Firebase JWT
      └── POST /createCapsule (Cloud Function)
              │
              ├── verifyIdToken(token) → uid
              ├── Zod schema validation
              ├── runPipeline() → OpenAI moderation on plaintextTitle + plaintextPreview
              │   └── If flagged → HTTP 422
              └── db.runTransaction():
                  ├── tx.get(subscriptions/{uid}) → plan
                  ├── Monthly reset check (Starter)
                  ├── Limit check: usedCapsules >= capsuleLimit?
                  │   └── If true → throw CAPSULE_LIMIT_REACHED → HTTP 403
                  ├── tx.set(capsules/{newId}) → all encrypted fields + metadata
                  └── tx.set(subscriptions/{uid}, { usedCapsules: increment(1) })
```

### Full Request Lifecycle: View/Decrypt Capsule

```
User → Dashboard.tsx → useCapsuleDecryption hook
  │ decryptCapsule(capsule) called
  │
  ├── If capsule.version !== "v2" → return plain text (legacy)
  ├── If !privateKey → return "key not loaded" fallback
  ├── capsule.encryptedKeys[user.uid] → myWrappedKey
  │   └── If missing → return "no access" fallback
  ├── unwrapContentKey(myWrappedKey, privateKey)
  │   └── RSA-OAEP decrypt → AES content key (browser memory)
  ├── decryptText(encryptedTitle, titleIV, contentKey) → plaintext title
  └── decryptText(encryptedContent, contentIV, contentKey) → plaintext content
```

No network call is made to decrypt. All decryption happens in the browser using the in-memory private key.

---

## 8. Sequence Diagram

```mermaid
sequenceDiagram
    participant U as User Browser
    participant AC as AuthContext
    participant CF as Cloud Function
    participant SM as Secret Manager
    participant FS as Firestore
    participant RZ as Razorpay API

    Note over U,FS: === LOGIN / KEY SETUP ===
    U->>AC: Firebase Auth (email/password)
    AC->>FS: getDoc users/{uid} → has publicKey?
    alt No keys yet
        AC->>CF: getKeyMaterial() [Callable, JWT]
        CF->>SM: read FORA_PEPPER
        SM-->>CF: pepper value
        CF-->>AC: { keyMaterial: uid+pepper }
        AC->>U: PBKDF2(uid+pepper, salt) → masterKey
        AC->>U: generateRSAKeyPair() → { pub, priv }
        AC->>U: AES-GCM encrypt(priv, masterKey) → encryptedPriv
        AC->>FS: updateDoc users/{uid} { publicKey, encryptedPrivateKey, privateKeyIV, salt }
        AC->>AC: store privateKey in React state (memory only)
    else Keys exist
        AC->>CF: getKeyMaterial() [Callable, JWT]
        CF-->>AC: { keyMaterial: uid+pepper }
        AC->>FS: getDoc users/{uid} → salt, encryptedPrivateKey, privateKeyIV
        AC->>U: PBKDF2 → masterKey; AES-GCM decrypt → privateKey
        AC->>AC: store privateKey in React state
    end

    Note over U,FS: === CREATE CAPSULE ===
    U->>FS: getDoc users/{recipientId} → publicKey (for each recipient)
    U->>U: generateContentKey() [AES-256-GCM]
    U->>U: encryptText(title, contentKey) → { ciphertext, iv }
    U->>U: encryptText(content, contentKey) → { ciphertext, iv }
    U->>U: RSA-OAEP wrap contentKey for each recipient → encryptedKeys map
    U->>CF: POST /createCapsule { encryptedTitle, encryptedContent, encryptedKeys, plaintextPreview... }
    CF->>CF: verifyIdToken(JWT) → uid
    CF->>CF: OpenAI moderation on plaintextPreview
    CF->>FS: transaction { check limit, create capsule, increment usedCapsules }
    CF-->>U: { success: true }

    Note over U,RZ: === UPGRADE TO PRO ===
    U->>CF: POST /createRazorpayOrder { planTier, billingCycle } [JWT]
    CF->>RZ: POST /v1/orders { amount, currency }
    RZ-->>CF: { orderId, amount }
    CF->>FS: payments/{auto} { status: "pending" }
    CF-->>U: { orderId, amount, currency }
    U->>RZ: open Razorpay modal → user pays
    RZ-->>U: { razorpay_order_id, razorpay_payment_id, razorpay_signature }
    U->>CF: POST /verifyRazorpayPayment { order_id, payment_id, signature, planTier }
    CF->>CF: HMAC-SHA256 verify signature
    CF->>RZ: GET /v1/payments/{payment_id} [confirm]
    CF->>FS: transaction { subscriptions/{uid} = { tier, capsuleLimit, usedCapsules:0 }, payments/{doc} = completed }
    CF-->>U: { success: true }
```

---

## 9. Security Audit

### Confirmed Issues

#### Issue 1: Attachments Are Not Encrypted
- **What:** Media files (images, video, audio) are uploaded to Firebase Storage without encryption. The `encrypted: false` flag is explicitly set in `CreateCapsule.tsx` line 280.
- **Where:** `src/pages/CreateCapsule.tsx` → `uploadAttachment()`
- **Why it's a problem:** Anyone who obtains a Storage download URL can read the media content. The E2EE guarantees on text do not extend to media.
- **Impact:** A database breach or misconfigured Storage rules would expose media files in plaintext, while text content remains protected.
- **Fix:** Encrypt file bytes with the capsule's AES content key before uploading to Storage. Store the IV alongside the file metadata.

#### Issue 2: Razorpay Webhook Cases Are Empty (Stub Only)
- **What:** `razorpayWebhook` verifies the webhook signature but takes no action on `payment.captured`, `payment.failed`, or `order.paid` events.
- **Where:** `fora/functions/src/payment-funtion.ts` lines 521–529
- **Why it's a problem:** If a client-initiated payment completes but the user loses connectivity before `/verifyRazorpayPayment` is called, the subscription is never activated. Subscription activation is entirely dependent on the client calling back.
- **Impact:** Lost subscriptions on network failure. No automatic recovery.
- **Fix:** Implement the webhook cases to activate/update subscriptions when Razorpay confirms payment.

#### Issue 3: Usage Decrement Is Outside the Delete Transaction
- **What:** In `deleteCapsule`, the capsule document is deleted first, then `usedCapsules` is decremented in a separate `subRef.set()` call marked as non-fatal.
- **Where:** `fora/functions/src/index.ts` lines 737–745
- **Why it's a problem:** If the decrement call fails (network error, Firestore timeout), the capsule count is permanently over-counted. The user has fewer capsules than their counter suggests, effectively reducing their remaining quota.
- **Impact:** Low — capsule is deleted, counter is wrong. User may hit a "limit reached" error when they should not.
- **Fix:** Wrap both the capsule deletion and usage decrement in a single transaction.

#### Issue 4: Frontend Limit Check Uses Stale Data
- **What:** `usePlan` fetches plan data once on page mount. The `hasReachedLimit` pre-flight check in `CreateCapsule.tsx` uses this cached value.
- **Where:** `src/hooks/usePlan.ts`, `src/pages/CreateCapsule.tsx`
- **Why it's a problem:** If a user opens two browser tabs and creates capsules in both, the stale limit in one tab may allow the UI to show the submit dialog even when the actual server-side limit has already been reached. The server-side transaction will correctly block the second creation, but the UI experience is confusing.
- **Impact:** Low — server always enforces correctly. UX inconsistency only.
- **Fix:** Refresh plan data after each successful capsule creation, or add a re-fetch on focus.

#### Issue 5: `handlePasswordChange` Does Not Call Firebase Auth API
- **What:** The Settings page has a "Change Password" form that only validates client-side and shows a toast. It never calls Firebase Auth's `updatePassword()` or `reauthenticateWithCredential()`.
- **Where:** `src/pages/Settings.tsx` → `handlePasswordChange()`
- **Why it's a problem:** The password change UI is decorative only. It does not change the user's actual Firebase Auth password.
- **Impact:** High — users believe they changed their password but did not.
- **Fix:** Implement `reauthenticateWithCredential(currentPassword)` → `updatePassword(newPassword)` using Firebase Auth SDK.

#### Issue 6: Settings State Is Not Persisted
- **What:** All settings in `Settings.tsx` (notifications, privacy, security, session timeout) are stored in local React state only. The `handleSaveSettings` function only shows a toast — no API call, no Firestore write.
- **Where:** `src/pages/Settings.tsx`
- **Why it's a problem:** Settings are reset to defaults every time the page is reloaded.
- **Impact:** Medium — settings are non-functional.
- **Fix:** Write settings to `users/{uid}.settings` in Firestore on save.

### Potential Risks (Require Further Testing)

#### Risk 1: `getKeyMaterial` Returns Pepper to All Authenticated Users
The `getKeyMaterial` callable returns `uid + FORA_PEPPER` to any authenticated user. While this is by design (each user gets a unique key material string incorporating their own UID), if an attacker compromises a Firebase Auth token, they can retrieve their own key material. The security boundary here is correct — the pepper alone is insufficient; the user's salt (in Firestore) and the encrypted private key (in Firestore) are also required to reconstruct the private key.

#### Risk 2: CORS Origin Whitelist Is Localhost-Only
The CORS config in `index.ts` and `payment-funtion.ts` only allows `localhost:5173` and `localhost:8080`. The production domain is not listed. This would block all production API calls unless the CORS config was updated before deployment or the functions use a different mechanism in production.

#### Risk 3: `sendMarketingBlast` Is Protected by a Weak Admin Secret
The admin-only endpoint uses a `x-admin-secret` header compared against `process.env.ADMIN_SECRET`. This is a symmetric secret — if the value is guessed or leaked, anyone can send bulk emails to all users.

---

## 10. Final Technical Summary

### What Works Well

1. **True E2EE for capsule text.** The hybrid RSA-OAEP + AES-GCM scheme is cryptographically sound. The backend never sees plaintext content. Per-recipient key wrapping is correct.
2. **Private key never stored raw.** The PBKDF2 master key derivation using `uid + server pepper + per-user salt` is a solid design. The 310,000 PBKDF2 iterations are above OWASP minimum recommendations.
3. **Atomic usage enforcement.** The Firestore transaction for capsule creation + usage increment eliminates race conditions. The server always enforces limits regardless of frontend state.
4. **Payment signature verification.** The backend correctly verifies Razorpay HMAC-SHA256 signatures and independently confirms payment status with the Razorpay API.
5. **Firestore security rules.** The rules enforce ownership and recipient access at the database level. Capsule creation requires all v2 encrypted fields — schema enforcement at the security-rule level.

### What Is Missing or Incomplete

1. **Attachment encryption** — media files are stored unencrypted.
2. **Razorpay webhook handler** — empty stub; no auto-recovery on client failure.
3. **Settings persistence** — all settings are UI-only.
4. **Password change functionality** — form exists but does not call Firebase Auth.
5. **Key rotation** — no mechanism to regenerate RSA keys.
6. **API rate limiting** — no per-user or per-IP rate limiting on Cloud Functions.
7. **Subscription downgrade enforcement** — relies entirely on the daily scheduler; a scheduler failure means expired subscriptions retain paid access indefinitely.

---

*Report generated by codebase analysis on 2026-11-24. All findings are based on actual source code. No code was modified.*
