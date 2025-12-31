# Privacy Policy

_Last updated: 2025-12-31_

Crush (“we”, “us”, “our”) is a college-only dating app that uses SMS login, `.edu` email verification, daily Crush Drop windows (three scheduled reveal minutes per day hashed by time zone), and in-app chat on Firebase + Cloud Functions. There is no swipe deck in the current release. This Privacy Policy explains what data we collect, how we use and share it, and the choices and rights available to you. See `docs/privacy/data-inventory.md` for a field-by-field breakdown; if anything conflicts, this policy controls.

## Information We Collect

We collect the following categories of data when you use the app:

1. **Account & Identity Data:** Display name, phone number, `.edu` email, school and state, graduation year, gender + “looking for”, age, Greek-life affiliation, majors, clubs/athletics, relationship intent, religion, height + optional preferred range, pet preference/has pets, smoking/drinking/going-out style, deal breakers, theme preference, and Firebase Auth identifiers.
2. **Verification Data:** Phone verification state from Firebase Auth plus `.edu` email OTP attempts (hashed with a ~10-minute expiry, 1-minute resend cooldown, and school/time zone mapping) stored in the `edu_verification` collection.
3. **Profile Content (UGC):** Bio and profile photos you upload to Firebase Storage, including any optional Greek organization details or interests you add.
4. **Discovery, Location & Presence Data:** Scope selection (campus/state/nationwide) and radius sliders, Crush Drop opt-in, time zone + school time zone, GPS coordinates (with source + timestamp) or campus fallback, Crush Drop window metadata (`dateKey`, status, scheduled minute, expiry), per-day pairing flags (`crushDropPairedDateKey`, `crushDropPairedIds`), and presence (`online`, `lastActiveAt`). Public profiles store coarse (~3 decimal) location and presence bucketed to 15-minute intervals.
5. **Engagement & Messaging Data:** Crush Drop activity feed entries, match records (participants, participantInterest, chatUnlocked, matchWindow labels/timestamps), match tombstones (participants + archivedAt) used to prevent rematching, chat messages, drop notification queue entries (ready/reminder/expire), matching event logs (drop impressions/responses/outcomes) stored in `mlEvents`, push tokens + last-seen metadata, messaging rate-limit counters, badge counts, block lists, and account deletion audit logs.
6. **Derived Matching Features:** Precomputed Crush Drop shortlists (candidate IDs) stored briefly to speed matching, plus numeric embeddings derived from profile photos and matching outcomes that help rank candidates and train matching models.
7. **Safety & Support Data:** Reports, support requests, SafeSearch moderation results and actions for photos, and removal logs for deleted photos.
8. **Device & Usage Data:** App version, device/OS type, Firebase Analytics event identifiers, and basic diagnostics used to monitor delivery of notifications and drops. No marketing trackers are present.

We do not knowingly collect information from individuals under 18; using the app requires a qualifying `.edu` address and college enrollment.

## How We Use Your Information

- **Service delivery:** Authenticate accounts, verify college enrollment, prepare Crush Drop windows (deterministic shared minute per time zone), surface spotlight reveals, process Like/Pass responses with double opt-in before chat unlocks, and deliver chat via a server-side function with rate limits.
- **Personalization & discovery:** Apply school themes, render profile cards, and respect gender/height preferences, distance filters, location-based scopes, and the compatibility signals you choose to share (majors/clubs, Greek life, relationship intent, religion, pets, lifestyle habits).
- **Intelligent matching:** Score potential Crush Drop partners using profile completeness, shared interests, recency/presence, distance, and chemistry factors (including optional lifestyle + relationship signals). We may precompute shortlists to reduce drop latency and use derived photo embeddings to improve match ranking while enforcing deal breakers and block lists.
- **Safety & integrity:** Enforce eligibility, deal breakers, and block lists; round public location/presence data; scan photos with Google Cloud Vision SafeSearch and delete/flag when needed; and throttle message sends.
- **Communications:** Send SMS codes, `.edu` verification emails (via SendGrid), push notifications for drops (ready/reminder/expire), matches, messages, and support responses. No marketing email is sent today.
- **Analytics & diagnostics:** Monitor aggregate usage, notification delivery, and Crush Drop performance, including matching quality signals used to improve the pairing model.
- **Compliance:** Maintain deletion audit logs and verification evidence needed to operate a college-only network.

## How We Share Information

We do not sell personal data. We share it only with:

- **Service providers:** Firebase (Auth, Firestore, Storage, Analytics, Cloud Messaging, Cloud Functions), Google Cloud Vision, Vertex AI (image embeddings), and SendGrid for `.edu` emails.
- **Other users:** Signed-in users can view `publicProfiles` with profile and compatibility fields you choose to share (display name, bio, photos, age, major, school/state, Greek life, gender/lookingFor, relationship intent, lifestyle/pets, height + preferences, deal breakers), verification flags, coarse location/presence, and block-list metadata used to enforce filtering. Match/chat data is visible only to participants, consistent with Firestore security rules.
- **Legal & safety recipients:** We may disclose information to comply with law, enforce our Terms, or protect the rights and safety of users.

All processors are bound by confidentiality and data protection agreements. We remain responsible for their handling of your data.

## Retention

- `.edu` verification attempts store hashed codes with a ~10-minute expiry and 1-minute resend cooldown; entries are deleted on successful verification or when the code is rejected/expired during a confirm attempt. There is no scheduled sweep for stale entries yet.
- Crush Drop shortlists are cleaned by a scheduled job after ~3 days.
- Drop notification queue entries persist until processed or related matches are deleted; there is no TTL yet.
- Match tombstones remain to prevent rematching; retention policy is still being defined.
- Profile, discovery, drop windows, matches, messages, activity entries, reports/support tickets, and token metadata persist while the account is active; there is no automated TTL cleanup today. Invalid push tokens are removed when FCM marks them unregistered, photo removals trigger Storage cleanup, and a daily sweep deletes unreferenced photos from Storage.
- Derived photo embeddings are removed when the underlying photo is deleted or when the account is deleted; no separate TTL exists yet.
- Account deletion removes the user document, blocked list, matches + chat threads, drop notifications, activity feed, public profile, and Storage photos, logs the request in `account_deletions`, deletes the Firebase Auth user, and leaves match tombstones to prevent rematching.

## Your Choices & Rights

- **Access & export:** A privacy dashboard with export/delete tooling is on our roadmap. Until then, contact us at privacy@crushso.com to request data access or removal.
- **Delete:** You can delete your account from the Safety & privacy screen; this triggers the `deleteAccount` Cloud Function described in the repository.
- **Notifications:** Toggle match, message, and Crush Drop notifications under Settings → Alerts or through system-level push controls.
- **Location:** Update GPS or campus fallback in Discovery settings. Revoking OS-level location stops GPS updates; if you stay opted in to Crush Drop we still use your school/campus details for time zone and scope.

## International Transfers

Data is hosted in Firebase’s U.S. regions. If you access the app from outside the United States, you consent to transferring your data to the U.S., where privacy laws may differ.

## Security

We rely on Firebase’s encryption at rest/in transit, hashed verification codes, Firestore security rules that require authentication, coarse public location/presence, SafeSearch photo moderation, and server-side messaging with rate limits. Admin/audit logging for sensitive reads is planned.

## Children

Crush is for college students 18+ and requires `.edu` verification. We do not knowingly collect data from minors; if we learn we have, we will delete it promptly.

## Changes

We will update this policy when we add new fields, processors, or retention schedules. The “Last updated” date reflects the latest change. Significant updates will be announced in-app or via email.

## Contact

Email privacy@crushso.com for questions, data requests, or privacy complaints. If you are in the EU/UK, you may also contact your local supervisory authority.

## Open Gaps & Roadmap

Per `docs/privacy/data-inventory.md`, the following improvements are in progress:

- Automated retention for verification attempts, drop activity, drop notification queue entries, match tombstones, reports, tokens, and inactive matches/messages.
- Privacy dashboard for export/delete/location precision controls.
- Lifecycle rules for orphaned Storage uploads.
- Admin access logging and monitoring for token purge jobs.

We will update this policy once those controls are live.
