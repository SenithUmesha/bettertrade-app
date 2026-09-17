# BetterTrade engineering notes

This document is the technical side of the public BetterTrade showcase.

The production repository is private, so this is not a source dump. It is a high-level walkthrough of the architecture, data flow, product constraints, and engineering decisions behind the app.

## 1. the shape of the app

BetterTrade is a Flutter application, but the interesting part is not really Flutter itself. The app has several systems that all need to agree on the same trade data:

```text
manual entry
CSV / JSON import
MT5 import
MT5 Auto Sync
        ↓
normalized trade model
        ↓
cloud persistence + local cache
        ↓
analytics / reviews / reports / insights
        ↓
mobile UI
```

Once multiple ingestion paths exist, the trade model becomes the contract for almost everything else in the product.

That pushed the project toward explicit boundaries instead of allowing Firebase models, widgets, imports, and analytics code to share whatever shape happened to be convenient on a given screen.

## 2. architecture

The app uses a feature-first Clean Architecture setup.

At a high level:

```text
Flutter UI
   ↓
Riverpod state
   ↓
use cases
   ↓
domain repository contracts
   ↓
data-layer implementations
   ↓
Firebase / local cache / Cloudflare / device APIs
```

The important rule is that domain code stays plain Dart.

That means the analytics engine, use cases, entities, and repository contracts do not need to know whether the data came from Firestore, a test fake, local cache, or an import parser.

A simplified production structure looks like this:

```text
lib/
├── core/
│   ├── ads/
│   ├── cache/
│   ├── error/
│   ├── monetization/
│   ├── platform/
│   ├── router/
│   ├── services/
│   ├── update/
│   ├── usecase/
│   ├── utils/
│   └── widgets/
│
├── features/
│   ├── access/
│   ├── analytics/
│   ├── auth/
│   ├── retention/
│   ├── settings/
│   ├── shell/
│   ├── subscription/
│   ├── support/
│   └── trade_journal/
│
├── injection_container.dart
└── main.dart
```

Each large feature owns its domain, data, and presentation concerns instead of putting every repository in one folder and every page in another.

## 3. state and dependency boundaries

### Riverpod

Riverpod drives the presentation state.

For journal and analytics flows, UI widgets observe providers while notifiers coordinate use cases, loading states, errors, refreshes, filters, and mutations.

Generated providers keep the boilerplate lower while Freezed is used for immutable state where richer state objects are useful.

### get_it + injectable

Dependency injection sits below the UI state layer.

Repositories, services, and use cases are resolved through `get_it`, with `injectable` generating the registration setup.

This lets production implementations and test doubles satisfy the same domain contracts without coupling feature code to constructors full of Firebase services.

## 4. journal data model

A trade is more than entry and exit price in BetterTrade.

The journal needs enough context for later analysis, so a trade can include information such as:

- symbol
- direction
- entry, stop loss and take profit
- lot size
- date / time
- open or closed state
- session
- notes
- screenshots
- tags
- behavior tags
- playbook / setup context
- pre-trade plan context
- review state
- post-trade review data

That context is what allows the analytics layer to answer questions that basic P&L summaries cannot.

For example, the same outcome can be grouped later by session, direction, playbook, behavior, checklist discipline, or risk pattern.

## 5. cloud persistence + local cache

Cloud Firestore is the durable cloud data store for core account data and journal records.

For the mobile runtime, BetterTrade also keeps a user-scoped local trade cache using Hive.

The cache is deliberately much simpler than the cloud data layer:

```text
Firestore
   ↓ sync / refresh
TradeEntity
   ↓ wire codec
Hive cache
   ↓
fast local reads
```

The local cache stores serialized trade entities under user-scoped keys and tracks the last successful sync time separately.

A freshness window prevents every navigation event from becoming another Firestore read.

When the cache is useful, the app can render journal data quickly and refresh the cloud copy in the background. When the cache is stale or empty, the repository can pull the current data and replace the local snapshot.

This is not a fully local-first conflict-resolution engine like Orderly. BetterTrade's cache is mainly about responsiveness, reduced read pressure, and keeping journal views useful through temporary connectivity problems.

## 6. analytics as derived data

One design choice I wanted to keep is: **the trade history is the source data; most analytics are derived from it.**

The app calculates things such as:

- win rate
- P&L and R-multiple performance
- expectancy
- streaks
- rolling performance
- symbol breakdown
- direction breakdown
- session breakdown
- playbook performance
- market-condition breakdowns
- risk metrics
- behavior cost
- checklist impact
- review coverage
- weekly summaries
- trader-health style scores

This keeps the product from accumulating dozens of summary fields that can drift out of sync with the journal.

It also makes import and edit flows safer: change the underlying trade, recompute the view.

For heavier analytics, work can be moved away from the main UI execution path so large journals do not turn scrolling and navigation into computation bottlenecks.

## 7. the insight engine

BetterTrade's coaching/insight layer is intentionally rule-based.

It is not a market-prediction model and it is not meant to generate trade signals.

The input is the user's own journal. Rules inspect combinations of performance and behavior data and emit findings that can be ranked and shown as coaching-style feedback.

Examples of the kinds of things the engine can surface include:

- revenge-trading patterns
- risk drift
- weak trading sessions
- poor checklist follow-through
- behavior-related R cost
- setup / playbook quality
- recurring mistakes

Conceptually:

```text
journal data
    ↓
analytics snapshot
    ↓
rule evaluation
    ↓
insight candidates
    ↓
severity / relevance ranking
    ↓
coach UI
```

Keeping this deterministic has a few benefits:

1. The output can be explained.
2. The rules can be unit-tested.
3. The app does not pretend to know something about the market that is not present in the user's data.
4. Results stay stable enough to compare over time.

The current product ships with a set of built-in insight rules and a follow-up loop that connects an insight to the next thing the user is trying to improve.

## 8. reviews and habit loop

The journal is intentionally not finished when a trade closes.

BetterTrade has a separate review workspace so a user can see what still needs review without using the main trade list as a to-do list.

The wider loop includes:

```text
pre-session journal
        ↓
trade plan / context
        ↓
trade execution
        ↓
trade review
        ↓
daily journal
        ↓
weekly report
        ↓
next focus rule
```

This is also why reminders are tied to missing review/journal work rather than simply telling someone to "log a trade" every day.

## 9. import pipeline

Imports are one of the places where a journal can quietly corrupt its own analytics.

BetterTrade supports multiple input formats, including generic CSV, BetterTrade exports, and MetaTrader 5 history formats.

The flow is roughly:

```text
file picker
   ↓
format detection / selected parser
   ↓
parse raw rows
   ↓
normalize fields
   ↓
validate trades
   ↓
generate / compare fingerprints
   ↓
filter duplicates
   ↓
persist accepted records
```

Trade fingerprints are important because the same history can be imported twice, an MT5 export can overlap an earlier file, or an automatically synced trade can later appear in a manual import.

The importer therefore has to think about identity, not just parsing columns.

## 10. MT5 Auto Sync

MT5 Auto Sync turns BetterTrade into a small distributed system rather than just a mobile app.

The high-level path is:

```text
MetaTrader 5 terminal
        ↓
BetterTrade Expert Advisor
        ↓ HTTPS
Cloudflare Worker
        ↓
validation + account mapping
        ↓
BetterTrade backend data
        ↓
push / refresh path
        ↓
mobile journal
```

The mobile app handles pairing and setup. A generated credential/API-key flow connects the user's BetterTrade account to the MT5-side integration.

The EA sends closed-trade data through the Worker, which acts as the narrow public boundary rather than exposing Firebase internals directly to the MT5 client.

The mobile side can then treat automatically captured trades like any other normalized journal records.

This separation also means the MT5 integration can evolve without putting platform-specific MetaTrader logic into the Flutter codebase.

## 11. screenshot storage

Trade screenshots are user content, but they are also relatively large compared with journal fields.

Keeping them directly inside Firestore would be the wrong storage model, and moving all backend functionality to a heavier paid stack just for screenshots would be overkill.

The production path uses:

```text
mobile app
   ↓
Cloudflare Worker
   ↓
R2 object storage
```

Firestore keeps the metadata/reference needed by the journal while the image bytes live in object storage.

The Worker gives the app a controlled upload/delete boundary and keeps storage credentials out of the client.

## 12. access, Free and Pro

BetterTrade uses one product with centrally defined access limits rather than maintaining separate app variants.

The Free tier is intentionally useful enough for long-term manual journaling. Pro is focused on additional depth and automation such as advanced analytics, the full insight layer, MT5 Auto Sync, larger import/export access, additional customization/storage, and ad-free usage.

Product limits are centralized in the domain layer rather than copied into individual pages.

That matters because a rule such as an import limit may need to be respected by:

- the UI
- the import service
- backend enforcement
- upgrade messaging
- analytics / quota displays

Central definitions reduce the chance of one screen saying an action is allowed while the backend says it is not.

Purchases are handled through RevenueCat on top of the platform stores, giving the app one entitlement concept across App Store and Google Play billing.

## 13. auth and account lifecycle

Authentication is Firebase-backed.

The production app supports platform-appropriate providers, including email-based auth and social sign-in paths.

Account deletion is treated as a product flow, not just an auth call. User-owned journal records, screenshots, feedback/support data, and relevant subcollections need to be cleaned up before the final auth account is removed.

That cleanup also has to account for reauthentication requirements from the current sign-in provider.

## 14. Firebase + Cloudflare split

The backend is intentionally mixed rather than trying to use one vendor for every problem.

### Firebase handles

- authentication
- Firestore journal/account data
- App Check
- analytics
- crash reporting
- messaging / notification infrastructure where needed

### Cloudflare handles

- public Worker endpoints for selected integrations
- R2-backed screenshot storage
- the MT5-facing boundary
- static website hosting

This keeps Firebase as the main application backend while moving object storage and external integration endpoints to infrastructure that fits those jobs better.

## 15. navigation and app shell

GoRouter handles declarative navigation and deep-linkable routes.

The main signed-in experience uses a shell route with persistent bottom navigation so high-frequency areas can share a stable app frame while detail flows push above it.

The route layer also gives notification taps, auth redirects, and integration setup screens a single place to map external intent into application navigation.

## 16. performance choices

Trading journals can become surprisingly data-heavy because analytics screens repeatedly slice the same history in different ways.

A few choices help keep the app responsive:

- local Hive cache for fast initial journal reads
- session-cached access/quota checks
- lazy refreshes for non-critical config data
- derived analytics separated from widget layout code
- loading skeletons for data-heavy screens
- compression / object storage for screenshots
- background or isolated computation where larger analysis work would block UI

The goal is not to precompute everything. It is to keep expensive work away from the frame-rendering path and avoid remote reads when the app already has a useful local answer.

## 17. observability

Production debugging uses Firebase Crashlytics together with application analytics/events.

The app also has platform and release-specific behavior — billing, sign-in providers, consent, ads, notifications, store links — so observability is especially useful for understanding failures that only happen in release builds or on one platform.

App Check adds another layer around Firebase-backed production traffic.

## 18. release engineering

The production repository uses GitHub Actions as a quality gate.

The CI flow includes:

```text
checkout
  ↓
install Flutter / Java
  ↓
flutter pub get
  ↓
code generation
  ↓
flutter analyze --fatal-infos --fatal-warnings
  ↓
flutter test --coverage
  ↓
Android debug build verification
  ↓
artifact / release steps on main
```

Store delivery still includes the platform-specific release work that CI alone cannot replace: signing, store metadata, subscription products, consent configuration, review assets, and device validation.

The same Flutter project ships to Android and iOS/iPadOS while keeping platform differences explicit where they actually matter.

## 19. testing strategy

The architecture makes several parts testable without a real production backend:

- use cases can run against repository fakes
- Firebase repositories can be tested with Firebase mocks/fakes where useful
- rule-based insights can be tested with deterministic trade sets
- analytics can be verified from known journals
- import parsers can be tested against fixtures
- state notifiers can be driven through success/error/loading transitions
- access limits can be tested as domain rules instead of widget conditions

The project uses generated code for Riverpod, Freezed and dependency injection, so CI runs generation before analysis/tests to catch stale generated output and integration errors.

## 20. what i would keep if i rebuilt it

A few decisions are especially worth keeping:

- **One normalized trade model.** Every ingestion path eventually needs to agree.
- **Derived analytics.** The journal should remain the source of truth.
- **Deterministic coaching rules.** Explainability is more useful here than pretending to have a market oracle.
- **A real local cache.** Mobile products should not feel broken because a connection is weak.
- **Cloudflare for the edges.** Object storage and MT5-facing endpoints did not need to become Firebase-shaped problems.
- **Centralized access rules.** Free/Pro logic belongs in the product domain, not scattered through UI code.
- **Feature-first boundaries.** Once the app became more than a journal, this stopped being optional.

## public-repo note

This repository intentionally contains documentation only.

The commercial BetterTrade codebase, production Firebase configuration, Worker configuration, signing material, store credentials, and environment values are not published here.

The goal of the public repo is to show the product and the engineering thinking behind it without turning production infrastructure into portfolio material.
