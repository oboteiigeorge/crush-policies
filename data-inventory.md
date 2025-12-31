# Crush Data Inventory

_Last updated: 2025-12-31_

This inventory lists every place the product stores or processes user data so engineering, product, and legal can reason about consent, retention, access, and safety requirements. Update it whenever you add a new field, collection, queue, or third-party processor.

## How to maintain this file
- When adding a field/collection/service, document **what** you store, **why**, **who can access it**, and **how long** it lives. Link to the relevant code or spec.
- Record both the **current behavior in code** and the **policy you intend to uphold** (e.g., “deleted 30 days after account closure”). If no policy exists, call it out as a TODO.
- Cross-check that privacy policy text, consent copy, and user-facing controls reference the same fields/categories found here.
- Pair every retention or access guarantee with an automated control (Cloud Function, scheduled job, Firestore TTL, etc.).

### Sensitivity legend
- `Basic PII` – identifiers a user expects to share in a social app (display name, school, email, phone).
- `Sensitive PII` – data covered by stricter policies (location, age, gender, verification codes, safety reports).
- `UGC` – user-generated content (photos, bios, chat messages).
- `Operational` – metadata, tokens, or logs required to run the service.

## Data store index
| Store | Location | Primary contents | Sensitivity | Current access surface | Notes / decisions |
| --- | --- | --- | --- | --- | --- |
| `users` collection | Firestore | Profile, preferences, verification state, notifications, location, tokens | Basic + Sensitive + UGC | Owner (per-user), backend functions, admin tooling | Source of truth; see detailed table below. |
| `system` collection | Firestore | Backfill state docs (e.g., `schoolTimeZoneBackfill` last processed user ID + timestamps) | Operational + Basic | Backend only | Internal state for backfills and migrations; delete when jobs complete. |
| `publicProfiles` collection | Firestore | Public-facing profile + compatibility fields (name, bio, photos, age, major, school/state, Greek life, gender/lookingFor, intent/lifestyle/pets, height + prefs, deal breakers, verification flags, coarse location/presence, `blockedUserIds`) | Basic + Sensitive | Signed-in clients, backend | Derived view for drop cards/chat presence; omits phone, email, tokens. |
| `users/{uid}/blocked` | Firestore subcollection | Block list entries | Basic | Owner + backend cleanup | Used for safety + match filtering. |
| `crushDropWindows` (fields on `users/{uid}`) | Firestore | Per-day window metadata (`dateKey`, `status`, `processedAt`), `timeZone`, `schoolTimeZone`, scope overrides | Operational | Owner + backend scheduler + drop UI | Drives the three local-time drops, fuels the campus-first intelligent matcher, and informs client countdowns/spotlights. |
| `crushDropShortlists` | Firestore | Precomputed Crush Drop candidate shortlists (`userId`, `windowId`, `dateKey`, `timeZone`, `candidates`, `builtAt`, `version`) | Operational + Basic | Backend only | Scheduled cleanup removes entries after ~3 days; built to speed drop matching. |
| `swipes` | Firestore | Legacy viewer/target decisions | Operational | Read-only to the viewer; backend retains for audit | Writes disabled; keep until retention policy finalized. |
| `matches` | Firestore | Match metadata, participant info, unread counters, `participantInterest`, `chatUnlocked`, `matchWindow` | Basic + Operational | Client (participants only), backend notifications | Includes double opt-in state, window labels for daily drops, and powers the spotlight reveal shown when a new drop arrives. |
| `match_tombstones` | Firestore | Tombstones for deleted matches (`participants`, `archivedAt`) | Operational + Basic | Backend only | Prevents rematching; retention policy TBD. |
| `messages/{matchId}/thread` | Firestore | Chat messages | UGC | Only participants + moderation tooling | Needs retention + abuse workflow. |
| `reports` | Firestore | Safety reports | Sensitive | Trust & Safety tooling only | Contains free-form text; restrict access. |
| `message_limits` | Firestore | Per-user messaging rate-limit windows (`windowStart`, `count`) | Operational | Backend functions only | Used by server-side messaging guardrails. |
| `support_requests` | Firestore | Help-center tickets | Basic | Support tooling | Same storage story as reports. |
| `account_deletions` | Firestore | Deletion audit trail | Operational + Sensitive | Admin only | Needed for compliance evidence. |
| `edu_verification` | Firestore | `.edu` email verification attempts | Sensitive | Cloud Functions only | Holds hashed codes; TTL required. |
| `activity/{uid}/feed` | Firestore | Drop-ready notifications, interest confirmations, match/message events | Operational | Read-only to the owner; writes by Cloud Functions | No TTL yet; entries remain until account deletion or future cleanup job. |
| `crushDropNotifyQueue` | Firestore | Drop notification jobs (`type`, `matchId`, `runAt`, status/attempts timestamps) | Operational + Basic | Backend only | No TTL yet; entries removed on account deletion for related matches. |
| `mlEvents` | Firestore | Crush Drop impressions, responses, and outcome logs with score feature snapshots | Operational + Basic + Sensitive (derived) | Backend only | TTL planned via `expiresAt` (target 180 days). |
| (deprecated) `feeds` | Firestore | Pre-ranked swipe deck entries per viewer (candidate IDs, scores, scope metadata) | Basic + Operational | Backend only | Legacy artifact from swipe deck; references removed from the client. |
| `Firebase Storage` | `gs://<project>/users/{uid}/photos/*` | Uploaded profile photos | UGC | Authenticated users only; owner write/delete | URLs stored in `users.photos`, scanned by SafeSearch, deleted on account deletion/photo removal, and swept daily for unreferenced files. Still need lifecycle rules for failed uploads/other buckets. |
| `photo_moderation` | Firestore | SafeSearch results + actions per upload | Operational | Backend/admin tooling | Audit trail for removed/flagged photos. |
| `photo_embeddings` | Firestore | Derived photo embeddings (numeric vectors) for ML training and matching | Sensitive (derived) | Backend only | Deleted when the related photo is removed; used to build user-level embeddings. |
| `user_embeddings` | Firestore | Aggregated per-user photo embeddings for real-time matching | Sensitive (derived) | Backend only | Rebuilt on photo updates; removed on account deletion. |
| Firebase Auth | Managed service | UID, phone number, auth factors | Basic | Firebase Admin SDK | Must be in privacy policy. |
| Push messaging | FCM/APNs | Device tokens + notification payloads | Operational | Firebase Messaging & Apple/Google | Payload contains match/user display names. FCM tokens that bounce with “not registered” are pruned immediately; scheduled cleanup based on `lastSeen` is still pending. |
| Email delivery | SendGrid | `.edu` address + school metadata | Basic | Cloud Function, SendGrid logs | DPAs + unsubscribe flow required. |

---

## Firestore `users` collection
Profiles are created/updated via `ProfileRepository` (`lib/services/profile_repository.dart`) and augmented by notification, presence, and safety services.

Public-facing data is copied into `publicProfiles/{uid}` by `functions/index.js:syncPublicProfile`, so clients read a derived view that includes profile and compatibility fields used in drop cards (display name, bio, photos, age, major, school/state, Greek-life info, gender/lookingFor, intent/lifestyle/pets, height + preferences, deal breakers, verification flags, and `blockedUserIds`). The canonical `users/{uid}` document stays owner-only. During this copy, GPS coordinates are rounded to ~100 m precision (and include a `precisionMeters` hint) while presence timestamps are bucketed to 15-minute intervals and `online` only reports activity within the last 2 minutes.

| Group | Fields | Purpose / feature | Sensitivity | Current retention | Notes / gaps |
| --- | --- | --- | --- | --- | --- |
| Identity & enrollment | `displayName`, `phoneNumber`, `eduEmail`, `major`, `gender`, `age`, `graduationYear`, `schoolId`, `stateCode`, `inGreekLife`, `greekOrganizationId`, `greekOrganizationLetters` | Build trust in profiles, enforce campus eligibility, filter by org | Basic + Sensitive | Stored until profile deletion; no automatic pruning | Need user-facing edit + delete controls; document lawful basis (consent vs legitimate interest). |
| Profile content | `bio`, `athletics`, `clubs`, `photos` (+ `photosCount`), `blockedUserIds` | UGC that powers cards and safety filtering | UGC | Same as above | Cloud Function now removes Storage objects + moderation logs when photos are removed; monitor for failures. |
| Intent & lifestyle | `lookingFor`, `relationshipIntent`, `dealBreakers`, `drinkingHabit`, `smokingHabit`, `goingOutPreference`, `religion`, `petPreference`, `hasPets`, `heightInInches`, `preferredHeightMinInches`, `preferredHeightMaxInches` | Compatibility inputs for discovery/matching and profile context | Basic + Sensitive (religion/lifestyle choices) | Stored until profile deletion; no automatic pruning | Ensure UI copy clarifies optionality; add deletion controls and review whether any of these need stronger consent. |
| Discovery preferences | `scope`, `campusRadiusMiles`, `stateRadiusMiles`, `crushDropOptIn`, `autoMatchOptIn` (legacy), `timeZone`, `schoolTimeZone` | Controls matching radius, Crush Drop participation, and local drop schedule | Operational + Sensitive (time zone implicitly reveals region) | Until user edits; time zone updates when device/school change | Surface in privacy dashboard with explanations. Document how to disable drops when traveling. |
| Crush Drop windows | `crushDropWindows.{windowId}.dateKey/status/label/scheduledAt/scheduledMinute/readyAt/readyNotifiedAt/noMatchNotifiedAt/retryAfter/retryCount/expiresAt/reason`, `autoMatchWindows` (legacy), `crushDropReadyDateKey`, `crushDropReadyWindowId`, `crushDropNoMatchKeys`, `crushDropPairedDateKey`, `crushDropPairedIds`, `lastCrushDropAttempt`, `lastAutoMatchAttempt`, `lastCrushDropAt` | Tracks scheduling, readiness, and outcomes for each daily drop window | Operational | Per-window fields overwrite daily; readiness/no-match keys currently persist without cleanup | Clients render countdowns/status/ready states from this set; add TTL or periodic cleanup for historical no-match/ready keys. |
| Notifications | `notifyMatches`, `notifyMessages`, `notifyCrushDrop`, `notifyAutoMatch`, `fcmTokens`, `fcmTokenLastSeen`, `badgeUnreadMessages`, `badgePendingDrops`, `badgeTotal` | Delivery controls for push notifications | Operational | Tokens linger until explicitly removed; last-seen map not pruned | Add scheduled cleanup using `fcmTokenLastSeen` and remove stale tokens automatically. |
| Verification state | `phoneVerified`, `eduVerified`, `updatedAt`, `lastCrushDropAt` | Gate access to features and Crush Drop cadence | Sensitive | Indefinite | Consider logging verification timestamp for audit. |
| Safety & presence | `online`, `lastActiveAt`, `location` (`latitude`, `longitude`, `source`, `updatedAt`) | Presence indicator, location-based feed, harassment mitigation | Sensitive | Location refreshed max every 30 min; stored indefinitely | Add TTL or precision-reduction policy; make opt-out explicit. |
| App preferences | `themePreference`, `brightnessPreference` | Persist theme selection (school vs standard) and system/forced brightness choice | Operational | Until user edits or resets settings | Keep consistent between platforms and surface in settings/privacy dashboard. |

**Access assumptions:** Firestore security rules allow users to read their own full document and signed-in clients to read `publicProfiles` only; all writes for matches/messages/drops go through Cloud Functions, not the client.

## Subcollection `users/{uid}/blocked`
| Fields | Purpose | Sensitivity | Access | Retention |
| --- | --- | --- | --- | --- |
| `displayName`, `photoUrl`, `blockedAt` | Shows who the user blocked and enforces filtering in feed/matches | Basic | Owner + backend cleanup jobs | Lives until user unblocks or deletes account. |

When blocking, the app also updates `users.blockedUserIds` and deletes overlapping swipes/matches (`SafetyRepository`).

## Collection `swipes`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `viewerId`, `targetId`, `decision` (`like`/`pass`), `createdAt` | Legacy record of swipe activity | Operational | Viewer read-only; writes disabled globally | Currently only retained for audit/backfill. Plan TTL (e.g., 90 days) or migration to delete after final analytics review. |

## Collection `crushDropShortlists`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `userId`, `windowId`, `dateKey`, `timeZone`, `candidates` (array of user IDs), `builtAt`, `version` | Precomputed candidate shortlist for a user + drop window | Operational + Basic | Backend only | Cleaned by scheduled job after ~3 days; ensure version bumps invalidate old entries. |

## Collection `crushDropNotifyQueue`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `type`, `matchId`, `windowId`, `windowLabel`, `windowDateKey`, `runAt`, `status`, `attempts`, `createdAt`, `startedAt`, `finishedAt`, `error` | Queue for Crush Drop ready/reminder/expire notifications | Operational + Basic | Backend only | No TTL yet; entries are removed on account deletion for related matches. |

## Collection `match_tombstones`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `participants` (array of UIDs), `archivedAt` | Prevent rematching after a match is deleted | Operational + Basic | Backend only | Retention policy TBD; consider TTL or anonymization strategy. |

## Collection `mlEvents`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `type`, `schemaVersion`, `createdAt`, `expiresAt`, `matchId`, `userId`, `candidateId`, `participants`, `windowId`, `windowLabel`, `dateKey`, `timeZone`, `source`, `groupId`, `passId`, `exploration`, `matcherVersion`, `score`, `scoreComponents`, `scorePenalties`, `distanceMiles`, `radiusMiles`, `radiusLimit`, `responseMs`, `timeToUnlockMs`, `timeToExpireMs` | Matching event log for Crush Drop impressions, responses, and outcomes; used for ML training and analytics | Operational + Basic + Sensitive (derived) | Backend only | TTL via `expiresAt` (target 180 days); ensure Firestore TTL configured. |

## Collection `photo_embeddings`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `userId`, `bucket`, `storagePath`, `gcsUri`, `contentType`, `model`, `modelPath`, `location`, `embedding` (array), `embeddingDim`, `createdAt`, `updatedAt` | Derived numeric vectors from profile photos for ML training and match ranking | Sensitive (derived) | Backend only | Deleted when the related photo is removed; no TTL yet; ensure downstream exports respect deletes. |

## Collection `user_embeddings`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `userId`, `embedding` (array), `embeddingDim`, `sourceCount`, `model`, `updatedAt` | Aggregated photo embeddings for real-time matching | Sensitive (derived) | Backend only | Rebuilt on photo changes and removed on account deletion. |

## Collection `matches`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `participants` (array of UIDs), `participantInfo.{uid}.displayName/schoolId`, `createdAt`, `lastUpdated`, `crushDrop`, `matchWindow`, `participantInterest.{uid}`, `chatUnlocked`, `lastMessage`, `lastMessageSenderId`, `participantUnread.{uid}`, `participantLastRead.{uid}`, `crushDropDeclinedBy` | Inbox list, notification source of truth, audit for harassment cases, double opt-in state | Basic + Operational | Participants + backend functions | Deleted when either participant deletes account or manually ends match (block). Need ability to delete stale inactive matches (policy TBD). |

## Collection `messages/{matchId}/thread`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `senderId`, `text`, `createdAt` | Chat between matches | UGC (may contain Sensitive details) | Participants; moderation tooling if needed | Deleted when match removed; no independent retention control yet. Consider message-level report workflow. |

## Collection `message_limits`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `windowStart`, `count` per `message_limits/{uid}` | Rate-limit guardrail for `sendMessage` Cloud Function | Operational | Backend functions only | Short-lived metadata; can be pruned automatically or left to TTL once defined. |

## Collection `reports`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `reporterId`, `reportedIdentifier` (userId, phone, or message id), `reason`, `details`, `status`, `createdAt` | Trust & Safety investigations | Sensitive | Restricted staff only | Needs lifecycle policy (e.g., keep 2 years) + tooling to update `status`. |

## Collection `support_requests`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `requesterId`, `subject`, `message`, `createdAt` | Customer support inbox | Basic | Support tooling | Add response tracking + retention rule (e.g., 1 year). |

## Collection `account_deletions`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `uid`, `reason`, `requestedAt` | Audit log proving deletion fulfillment | Sensitive | Admin only | Keep at least 2 years to satisfy regulatory inquiries. |

## Collection `edu_verification`
Maintained only by Cloud Functions, clients never read/write directly.

| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `email`, `domain`, `schoolId`, `stateCode`, `userId`, `codeHash`, `attempts`, `createdAt`, `lastSent` | Issue and validate 6-digit codes for `.edu` addresses | Sensitive (education status) | Cloud Functions; should never be exposed to clients | Records deleted on success/expiry; enforce TTL (firestore policies) for stale attempts. |

## Collection `system`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `schoolTimeZoneBackfill.lastDocId`, `lastProcessedAt`, `completedAt` | Track progress for school time zone backfill sweeps | Operational + Basic | Backend only | Delete backfill state when complete; avoid long-term retention of `lastDocId`. |

## Firebase Storage `users/{uid}/photos`
| Data | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| Original uploads plus metadata (`contentType`, `storagePath`) | Profile photo gallery | UGC | Authenticated users only (Firebase Storage rules require ID token); owners can write/delete | Account deletion and per-photo removals trigger backend cleanup of Storage objects + moderation logs, a daily sweep deletes unreferenced files under `users/*/photos`, and SafeSearch auto-deletes blatant violations. Still need lifecycle rules for failed uploads/other orphaned files outside that prefix. |

## Collection `photo_moderation`
| Fields | Purpose | Sensitivity | Access | Retention / TODO |
| --- | --- | --- | --- | --- |
| `uid`, `storagePath`, `bucket`, `action` (`flag`/`delete`), `category`, `level`, `safeSearch`, `createdAt` | Audit trail for Vision SafeSearch outcomes and follow-up | Operational | Admin-only tooling | Keep for compliance + repeat-offender tracking; consider TTL once aggregated metrics exist. `flag` entries leave the object in Storage (for manual review) but remove it from the profile; `delete` entries remove the file entirely. |

## Firebase Auth (managed by Google)
| Data | Purpose | Sensitivity | Notes |
| --- | --- | --- | --- |
| `uid`, phone number (primary login), optional email, device metadata, auth history | Authentication + anti-abuse | Basic | Refer to Firebase Auth DPA. Deleting an account via `deleteAccount` Cloud Function also deletes Auth user. |

## Push notification ecosystem
| Component | Data | Purpose / Notes |
| --- | --- | --- |
| Firestore `users.fcmTokens` | Strings per device | Saved by `NotificationService`; used to address push notifications. Invalid tokens are pruned when FCM rejects them. `fcmTokenLastSeen` timestamps are stored but there is no scheduled cleanup yet. |
| FCM/APNs payloads | `title`, `body`, `matchId`, `notificationType`, `senderId` | Set by `functions/index.js` in `onMatchCreated` and chat message triggers; payload content must align with privacy policy. |
| Device OS | Receives payload + stores token | Ensure mobile apps surface notification settings and respect `notify*` toggles. |

## Third-party processors
| Vendor | Data shared | Purpose | Controls / notes |
| --- | --- | --- | --- |
| SendGrid | `.edu` email, school name, verification copy | Emailing verification codes | API key stored as secret; include processor in privacy policy & DPA. |
| Google (Firebase, Maps/Geolocator) | Location coordinates, device identifiers, crash logs | Location service + platform SDKs | Ensure OS-level permissions and in-app copy make usage clear. |
| Apple (APNs) / Google (FCM) | Push tokens + notification payload | Deliver alerts | Respect opt-in/out states. |

### Crush Drop scoring signals

The campus-first matcher evaluates each drop on the fly using existing fields already listed in this inventory (`photosCount`, `lastActiveAt`, `major`, `clubs`, `athletics`, `inGreekLife`, `scope`, `campusRadiusMiles`, `stateRadiusMiles`, and coarse `location`). We now log derived score components (not raw profile fields) into `mlEvents` for matching analytics and ML training, with a TTL target of 180 days. Documenting the signal list here ensures privacy/legal reviewers know which inputs feed the automated pairing logic.

## Open gaps / decisions
1. **Retention:** No automated TTL/cleanup jobs yet for `edu_verification`, matches/messages/activity, legacy `swipes`, drop notifications, or match tombstones; only account deletion clears data today. Shortlists are cleaned after ~3 days. Define retention windows and implement Firestore TTL/scheduled cleanups (including token aging).
2. **User controls:** Build a privacy dashboard (export/delete data, notification toggles, location precision choice) that maps 1:1 with the fields above.
3. **Access controls:** Document Firestore security rules per collection and ensure Trust & Safety tooling enforces least privilege when reading reports/support tickets.
4. **Storage cleanup:** Account deletion and per-photo removal now delete Storage objects, and a daily sweep removes unreferenced photos under `users/*/photos`. Still need lifecycle rules for failed uploads/other buckets and monitoring for the sweep.
5. **Audit logging:** Log admin access to sensitive collections (`reports`, `support_requests`, `account_deletions`) for compliance audits.
6. **Push token retention:** Add scheduled cleanup based on `fcmTokenLastSeen` timestamps and alerting so invalid/stale tokens are removed even without an FCM error.

Keep this document versioned. When product specs or backend schemas change, update the relevant section and tag legal/safety stakeholders for review.
