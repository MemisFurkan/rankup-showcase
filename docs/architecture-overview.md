# Architecture overview

[Back to the showcase](../README.md)

RankUp consists of a React Native mobile application, an Express backend, a PostgreSQL database accessed through Prisma, and small web pages for account workflows. This document describes the reviewed implementation at a conceptual level. It contains no application source, deployment topology or live service identifiers.

## Mobile application

The mobile application uses Expo and TypeScript. Expo Router organizes authentication screens, tabs and detail routes. Reusable components cover tasks, rewards, progress, social interactions, avatars and system states.

React Context providers coordinate authentication, onboarding, themes and mascot state. Typed service modules encapsulate API calls; domain types describe request/response data for features such as tasks, matches and progress. The central API client handles response parsing, timeouts, authentication refresh and rate-limit feedback.

Authentication tokens are stored through Expo SecureStore on the native client. AsyncStorage holds cached user state, token-expiry metadata and local preferences; it is not presented as encrypted storage for all profile data. Authentication epochs and serialized persistence help prevent stale asynchronous work from crossing account boundaries.

The interface combines reusable themed components, NativeWind configuration, React Native styling, animation and chart components. Loading, error, empty and retry states are represented in components. Their presence does not establish a completed accessibility or device-compatibility audit.

## Backend organization

The backend is a modular Express application using JavaScript/CommonJS. It is organized around route modules, middleware, controllers, domain services and shared helpers. The layering is pragmatic: task/progress logic uses service modules, while substantial duel/co-op orchestration lives in controllers and helpers.

Request handling includes token/session checks, verified-email requirements on protected application routes, input validation, rate limiting and error handling. Reward, price, eligibility and ownership decisions are made on the server rather than trusted from the mobile interface.

Prisma provides relational queries and interactive transactions. SQL migrations also define constraints and indexes, including rules that are not fully expressed in the Prisma model file. The migration chain therefore carries part of the integrity contract.

Scheduled jobs reconcile duel/co-op state, process notifications, expire authentication sessions and retry profile-image cleanup. They are registered by the backend and are configuration dependent. Their presence does not imply a separately deployed worker cluster.

## Main data relationships

| Domain | Stored relationships and responsibility |
| --- | --- |
| Identity | Users own revocable sessions and time-limited email-verification/password-reset records |
| Tasks | Templates define tasks, assignments connect them to a user, and completion records preserve the activity history |
| Progress | XP/rank balances coexist with activity-derived streaks, weekly summaries and a versioned weekly goal |
| Social | Friend-request records connect users; duels and co-op quests store participants, status and time bounds |
| Rewards | Challenge assignments/completions, earned badges, cosmetic ownership and avatar-purchase records represent distinct reward sources |
| Messaging | Notifications have deduplication identities; device/session bindings, deliveries and attempts track push processing |
| Profile | Streak-protection records and selected badge slots extend the profile without inventing task completions |
| Cleanup | Account deletion can leave a durable image-cleanup job for external media removal |

This is a domain summary, not a published database schema.

## Representative flows

### Task completion and reversal

1. The mobile client sends a completion request for an assigned task.
2. The API checks authentication and validates the request. The service obtains a user-row lock inside a transaction, verifies ownership and checks the current day.
3. Completion state, the completion record, primary XP/coin changes and rank update are committed together. SQL uniqueness backs up duplicate-completion checks.
4. Match reconciliation runs after the task transaction commits. A reconciliation failure is logged without turning the durable task completion into an apparent failure that invites duplicate submission.
5. Same-day undo removes the activity and reverses its primary rewards in a transaction. If the balance cannot cover reversal, the operation is rejected and rolled back.

Daily/weekly calculations use shared server date helpers with a default UTC+3 day boundary and Monday week start. Notification quiet hours and streak protection also use Istanbul day/time rules. The implementation does not establish arbitrary per-user time-zone support.

### Authentication lifecycle

Registration/login creates a persisted session and access/refresh tokens. Passwords are hashed with bcrypt. Refresh tokens and email/reset tokens have hashed representations in the database; refresh rotation uses a conditional update so a superseded token cannot silently replace the current session state.

Protected requests check both the token and persisted user/session state. Password changes/resets and logout paths revoke the relevant sessions or account token version. The client shares refresh work and uses generation checks to keep old responses from restoring stale authentication. After refresh following a 401 response, the API client only automatically repeats GET requests; writes require an explicit retry.

The supporting web pages call account endpoints and support verification/reset links. Token-fragment parsing and URL cleanup are implemented, but hosting associations and actual email delivery still require environment acceptance.

### Duels and co-op quests

Duels support XP, category and streak races. Invitations carry configuration, including an in-app coin stake; acceptance reserves balances transactionally. Scores are computed from stored activity within the match window. Finalization checks current state under participant locks, then applies the terminal transition and eligible rewards/refunds together.

Co-op progress counts task completions by the two participants within the quest window, with optional category filtering. The server derives the reward from the target. Completion and failure finalizers re-read state and use conditional transitions to make repeated reconciliation safe.

The mobile match service supplies idempotency keys when creating invitations. Database uniqueness and payload checks distinguish a repeat of the same request from conflicting reuse. These are request/state protections, not a claim of a real-time socket-based system or general exactly-once delivery.

### Coin purchases and the ledger boundary

Coin spending checks the catalog, ownership and authoritative balance. Purchase paths use transactions and conditional balance updates; durable ownership and avatar-purchase records support duplicate/concurrency checks. Coins are an application reward unit, not a payment integration.

An economy ledger implementation adds event identities, balance baselines and reconciliation checks. Its activation helper explicitly restricts it to an isolated test database and rejects production activation. It is therefore documented as implemented work under controlled validation, not the active production accounting path. No test-database addresses or activation values are published here.

### Notification processing

The notification service derives a stable identity for supported reminder/social/reward events. Database uniqueness prevents duplicate inbox records. Push bindings connect an installation and generation to a user/session. Delivery processing checks that the binding is still current before a provider attempt.

Durable delivery/attempt records and claim leases support retry processing and reject stale worker results. Explicit temporary failures are eligible for bounded retry. Uncertain outcomes are retained as unknown rather than automatically replayed. Ticket acceptance, provider receipt and visible notification on a physical device are separate evidence levels.

Notification preferences use a versioned update to detect edits from another device. Delivery-time policy checks apply category preferences, weekday frequency and quiet hours. Policy-blocked sends are skipped rather than queued until quiet hours end; the inbox identity remains. Implemented account-security notification types bypass these user preference filters.

### Profile features without changing earned activity

Weekly summaries aggregate task activity for the current and previous week. A weekly goal counts active days rather than total tasks or a separate reward claim. Version checks prevent silent overwrites when two devices edit the goal.

Badge showcase editing checks ownership and unique slots in a transaction. Free streak protection uses durable per-day/per-week usage and only bridges an eligible missed day in the profile view. Real activity remains the input for task rewards, challenge metrics and leaderboards.

### Account deletion and external cleanup

Deletion rechecks authorization, coordinates participant state, handles affected match/refund records and removes account data transactionally. Managed profile-image deletion is separated into durable cleanup work so a storage-provider failure does not require pretending that external cleanup already succeeded. Provider cleanup acceptance remains separate from database deletion.

## Integrations and verification boundaries

Nodemailer supports account email workflows, Expo APIs support push delivery, and Cloudinary supports managed profile images. Mobile Sentry initialization is conditional on production configuration. These integrations are present in code; no live provider account, credentials, service address or successful production operation is asserted here.

The source includes unit/regression tests and PostgreSQL integration tests. GitHub Actions defines backend migration/schema checks, regression execution and dependency audit; the mobile workflow defines TypeScript, tests, Expo Doctor, lint and dependency audit. Existing project records describe unresolved lint/audit and release gates. No workflow result, coverage percentage or benchmark was freshly verified during this documentation-only task.

Remaining acceptance work includes physical-device flows, external-provider behavior and environment-specific release/migration readiness. The public showcase is deliberately limited to implementation evidence and engineering decisions.
