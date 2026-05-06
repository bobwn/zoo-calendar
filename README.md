# ZooCalendar

**ZooCalendar.com** is a single-page web application that helps users discover zoos and wildlife parks across the United States. It provides a searchable directory of 108 venues across all 50 states and Washington D.C., with direct links to official zoo websites.

---

## Architecture

ZooCalendar is a **static single-page application** — there is no backend server. The entire app runs in a single `index.html` file with embedded CSS and JavaScript. All cloud services are accessed directly from the browser via their respective SDKs.

### File Structure

| File | Purpose |
|---|---|
| `index.html` | Main application — zoo directory, search, auth, comments |
| `zooData.json` | Zoo dataset (108 venues, all 50 states + D.C.) |
| `about.html` | About page |
| `contact.html` | Contact page |
| `privacy.html` | Privacy Policy |
| `terms.html` | Terms of Service |
| `giraffe1-logo.PNG` | Primary logo |
| `giraffe2-logo.PNG` | Alternate logo |

---

## Why Firebase?

ZooCalendar uses **Firebase** (by Google) as its cloud backend. Because the app is a static HTML file with no server, Firebase provides the database and authentication infrastructure that would otherwise require a dedicated backend.

Firebase was chosen for the following reasons:

- **No server required** — Firebase Auth and Firestore operate directly from the browser, keeping the app fully static and simple to host.
- **Production-grade authentication** — Email/password sign-in, account creation, and password reset are handled securely by Google without any custom backend code.
- **Generous free tier** — The Firebase Spark (free) plan covers up to 1 GB Firestore storage, 50,000 reads/day, 20,000 writes/day, and 10,000 authenticated users — sufficient for early-stage growth.
- **Queryable for email targeting** — Firestore supports array-contains queries, making it straightforward to find all users interested in a specific topic (e.g., `interests` contains `"new-animals"`) when sending targeted email notifications.

### Firebase Project

- **Project ID:** `zoo-calendar-comments`
- **Auth Domain:** `zoo-calendar-comments.firebaseapp.com`
- **Services used:** Firebase Authentication, Cloud Firestore

---

## Firebase Services

### Authentication

Firebase Authentication manages all user accounts. Users can:

- Create an account with email and password
- Sign in and sign out
- Reset their password via email

User records are visible in the [Firebase Console](https://console.firebase.google.com/) under **Authentication → Users**.

### Firestore Database

Two collections are used:

#### `comments` collection

Stores user-submitted comments/feedback. Each document contains:

| Field | Type | Description |
|---|---|---|
| `text` | string | Comment content |
| `timestamp` | timestamp | Server-generated time |
| `created` | string | Human-readable date |
| `userId` | string \| null | Firebase Auth UID (null if not signed in) |
| `userEmail` | string \| null | User's email (null if not signed in) |
| `userAgent` | string | Browser user agent |
| `source` | string | `"firebase"` or `"local-storage"` |

If Firebase is unreachable, comments fall back to **browser localStorage** under the key `zooComments`. No email is stored in the localStorage fallback.

#### `users` collection

Created when a new user completes account registration. Each document is keyed by the user's Firebase Auth `uid` and stores their email notification preferences:

| Field | Type | Description |
|---|---|---|
| `email` | string | User's email address |
| `uid` | string | Firebase Auth UID |
| `interests` | array | List of selected interest tags (see below) |
| `preferencesSet` | boolean | `true` if user selected interests; `false` if skipped |
| `createdAt` | timestamp | Account creation time |
| `updatedAt` | timestamp | Last preferences update time |

Two additional fields are written on every sign-in (merged, not overwritten):

| Field | Type | Description |
|---|---|
| `loginCount` | number | Running total of all sign-ins |
| `lastLoginAt` | timestamp | Time of the most recent sign-in |

#### `users/{uid}/loginHistory` subcollection

A new document is added to this subcollection every time the user signs in. Each document is auto-ID'd and contains:

| Field | Type | Description |
|---|---|---|
| `timestamp` | timestamp | Server time of the login event |
| `userAgent` | string | Browser and device information |

This provides a full audit trail of login frequency and timing per user. Combined with `loginCount` on the parent document, you can query both summary stats and the full history.

**Note:** Login events are recorded only on fresh sign-ins, not on page-load session restores (i.e., when a user returns to the site while already logged in from a previous session).

---

**Interest tags** (values stored in the `interests` array):

| Tag | Label |
|---|---|
| `new-animals` | New animal births |
| `exhibits` | New exhibits |
| `kids` | Kids & family events |
| `conservation` | Conservation news |
| `animal-types` | Animal spotlights |
| `special-events` | Special events |
| `behind-scenes` | Behind the scenes |
| `deals` | Deals & discounts |

### Firestore Security Rules

The following rules should be set in the Firebase Console under **Firestore → Rules**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Users can only read and write their own preferences document
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }

    // Anyone can add a comment; only authenticated users can read all comments
    match /comments/{commentId} {
      allow create: if true;
      allow read: if request.auth != null;
    }
  }
}
```

---

## Email Notifications (Roadmap)

Firebase does not send emails natively. When the email notification feature is built out, the planned approach is:

1. A **Firebase Cloud Function** (triggered on a schedule or event) queries the `users` collection for all users whose `interests` array contains the relevant tag.
2. The function calls an email service API (e.g., **SendGrid**, **Resend**, or **Mailchimp**) to send targeted emails to matching users.
3. No additional server infrastructure is required — Cloud Functions run serverlessly within the Firebase project.

---

## Navigation Menu

The current menu items visible to users are:

- **Sign In / Sign Out**
- **About**
- **Leave Comment**

The following items are present in the code but hidden for future use:

- Team Collaboration (`collaboration.html`)
- Shop (`shop.html`)
- Social media links: Facebook, Instagram, YouTube (X/Twitter is visible)

---

## Zoo Dataset

The `zooData.json` file contains 108 venues across 50 states and D.C.

| Type | Count |
|---|---|
| Zoo | 95 |
| Wildlife Park | 3 |
| Aquarium | 3 |
| Theme Park Zoo | 2 |
| Wildlife Refuge | 1 |
| Safari Park | 1 |
| Farm Zoo | 1 |
| Wildlife Rescue | 1 |
| Nature Center | 1 |

Each record includes: `name`, `url`, `city`, `state`, `type`, `size`, `notable_features`, `admission_required`, `hours`, `phone`, `admission_price`, `website_events_url`, `annual_attendance`, and `founded_year`.
