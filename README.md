# WorthIt iOS App

WorthIt helps people decide whether a YouTube video deserves their time. It pulls the video's transcript and community feedback, distills the content into summaries and scores, and offers follow-up Q&A in a single SwiftUI experience. The app ships as both a standalone iOS app and a share extension so users can trigger an analysis directly from the YouTube app.

---

## Highlights
- **Instant summaries & "Worth It" score** - Transcript analysis produces highlights, actionable takeaways, and a Gems of Wisdom section. A blended score (content depth + community sentiment) tells users at a glance whether to watch.
- **Comment intelligence** - The top 50 comments are clustered into themes, scored for sentiment, and categorized (humor/insightful/etc.) to surface the crowd's perspective.
- **Ask Anything assistant** - Users can chat with the app to pull specific facts from the transcript with conversational context.
- **Usage tracking & paywall** - A daily quota allows five free analyses with StoreKit subscriptions and complimentary grants lifting limits.
- **Seamless sharing** - The share extension mirrors the main UI, reusing caches and quickly returning results when invoked from the iOS share sheet.

---

## Technology Stack
- **Platform**: SwiftUI, Swift Concurrency (async/await), Combine, StoreKit 2
- **Bundled Targets**: `WorthIt` (main app) and `Share` (share extension)
- **Persistence**: App Group `UserDefaults`, `NSCache`, on-disk JSON/text blobs
- **AI/Backend**: Custom proxy endpoints (`/transcript`, `/comments`, `/openai/responses`) wrapping OpenAI Responses API (GPT-5 Nano)
- **Instrumentation**: Custom `Logger` (OSLog + optional file logging) and `AnalyticsService` (Firebase-ready console logging)

---

## Architecture Overview
```
WorthItApp (SwiftUI App)
 |- AppState (global view routing)
 |- SubscriptionManager (StoreKit 2)
 |- MainViewModel (business logic + UI state)
 |   |- APIManager (networking + AI prompts)
 |   |- CacheManager (memory/disk/shared caches)
 |   |- UsageTracker (daily quota actor)
 |   |- AnalyticsService / Logger
 |   \- Models (ContentAnalysis, CommentInsights, ScoreBreakdown, etc.)
 \- RootView (navigation shell + feature screens)
```

The same `MainViewModel` is injected into the share extension so both targets share business logic and persistence.

---

## End-to-End Flow
1. **URL intake**  
   - Paste a link/ID in the main app or invoke the share extension.  
   - `URLParser` validates supported YouTube hostnames or 11-character IDs.
2. **Usage guard & paywall**  
   - `UsageTracker` (App Group defaults) ensures free-tier limits (5/day).  
   - Subscribers or complimentary-access users bypass the quota.  
   - When the quota is exhausted a `PaywallContext` is issued to the UI.
3. **Cached fast path**  
   - `CacheManager` looks up transcript, summaries, comment insights, Q&A history, and categorized comments.  
   - Cache hits instantly populate the UI while background tasks refresh stale data.
4. **Fresh analysis pipeline**  
   - `APIManager` fetches the transcript (`/transcript`) with language fallbacks.  
   - Top comments are retrieved (`/comments?limit=50`).  
   - The transcript runs through a JSON-only summarization prompt; comments run through classification and insight prompts.  
   - A lightweight "Essentials" pass produces quick scores & suggested questions while full analysis completes.
5. **Results & actions**  
   - `MainViewModel` merges transcript and comment outputs into `ContentAnalysis`, computes the Worth-It score, persists to cache, and logs analytics.  
   - Users can open the Essentials tab, view categorized comments, launch the share overlay, or enter the Ask Anything chat.
6. **Ask Anything**  
   - Conversational questions are answered against the transcript using the same proxy (`/openai/responses`), chaining `previous_response_id` to retain context.

---

## Backend & AI Contracts
- **Endpoints** (`API_PROXY_BASE_URL` from the Info.plist):
  - `GET /transcript?videoId=...&languages=...` - returns `{ text }`. Handles 404 with retries over preferred languages.
  - `GET /comments?videoId=...&limit=50` - returns `{ comments:[String] }`.
  - `POST /openai/responses` - wraps OpenAI Responses API. Requests enforce JSON-only output, low verbosity, and minimal reasoning to cut token usage.
- **Prompts** live in `APIManager`:
  - Transcript prompt produces `longSummary`, `takeaways`, and `gemsOfWisdom` with strict formatting rules.
  - Comment prompt returns sentiment summary, comment themes, and per-comment categories (humor/insightful/controversial/spam/neutral) with validity checks.
  - Comment insight prompt scores `contentDepthScore`, `overallCommentSentimentScore`, and suggested user questions.
  - Q&A prompt yields `{ "answer": "..." }` responses and is lightly retried on network hiccups.
- **Model Selection**: defaults to `gpt-5-nano-2025-08-07`, with reasoning effort set to `minimal` and responses stored for observability.

---

## Caching & Persistence
- **CacheManager** (`actor`)  
  - Memory (`NSCache`) + disk (JSON/TXT) under App Group container `WorthItAICache/`.  
  - Persists `ContentAnalysis`, comment insights, transcripts, comment lists, and Q&A messages.  
  - Tracks "recent analyses" metadata (used by Recent Videos UI).  
  - Provides helpers to clear memory/disk caches independently.
- **UsageTracker** (`actor`)  
  - Daily limit snapshots with bonus credit handling and per-video deduping.  
  - Persists usage counters and bonus allowances in App Group defaults.
- **Time Saved Ledger**  
  - Stores cumulative minutes saved per video, streak data, and contextual banner history in shared defaults for celebratory toasts.

---

## Scoring & Insights
- **Worth-It Score**  
  - `finalScore = round(100 * (0.60 * depth + 0.40 * sentiment))` with edge-case nudges (bonus for high/high, penalty for low depth).  
  - Falls back to transcript depth alone when comments are missing, flagging that gap in `ScoreBreakdown`.
- **Categorized Comments**  
  - Up to 50 comments are normalized and matched back to AI-labeled categories to populate humor/insightful/etc. trays in the UI.  
  - Comment themes include an example comment that is validated against the source list.
- **Suggested Questions**  
  - Essentials output typically supplies 2-4 suggested Q&A prompts.  
  - Fallback heuristics generate language-aware defaults if the model returns none.

---

## Subscriptions & Paywall
- StoreKit 2 integration via `SubscriptionManager`:
  - Fetches products (ordered by `subscriptionProductAnnualID`, `subscriptionProductMonthlyID`).
  - Listens for `Transaction.updates`, persists entitlements, and exposes `.subscribed` state.
  - Refreshes products and entitlements at startup and when the scene becomes active.
- `MainViewModel` enforces free-tier usage:
  - Complimentary access windows (stored in defaults) override the daily quota.
  - `PaywallContext` packages stats (minutes saved today/week, active streak, total unique videos) and the calculated reset date for UI display.
  - Manual deep link (`worthitai://subscribe`) can show the paywall on demand.

---

## Share Extension
- `ShareExtensionEntry` acquires a cross-process lock (file-based in App Group) so duplicate invocations exit early.  
- `ShareHostingController` builds the same SwiftUI tree as the main app, injecting fresh service singletons (`CacheManager.shared`, new `APIManager`, `UsageTracker.shared`).  
- Notifications allow the extension to dismiss itself or open the main app if needed.

---

## Logging & Analytics
- **Logger** (`Utils/Logger.swift`)  
  - Category-aware OSLog wrapper with optional file logging in the App Group `Logs/` directory (enabled in DEBUG).  
  - Supports runtime category enable/disable for targeted debugging.
- **AnalyticsService** (`Utils/AnalyticsService.swift`)  
  - Currently logs events to the console but mirrors Firebase-style APIs.  
  - Events cover app launch, share extension usage, video analysis lifecycle, cache hits/misses, paywall interactions, onboarding funnels, and network errors.

---

## Project Structure
```
WorthIt/
 |- WorthItApp.swift        // SwiftUI entry point
 |- MainViewModel.swift     // Core business logic + state machine
 |- Models.swift            // Shared models, constants, decoding helpers
 |- Services/
 |   |- APIManager.swift        // Networking + AI orchestration
 |   |- CacheManager.swift      // Memory/disk cache actor
 |   |- SubscriptionManager.swift
 |   \- UsageTracker.swift      // Daily quota actor
 |- Utils/
 |   |- AnalyticsService.swift
 |   |- Logger.swift
 |   |- Theme.swift
 |   \- URLParser.swift
 \- Views/                  // Screen + component SwiftUI files

Share/                      // Share extension target (hosts RootView)
```

---

## Configuration & Setup
1. **App Group**  
   - Update `AppConstants.appGroupID` and ensure the App + Share targets use the same App Group entitlement.
2. **Info.plist**  
   - Set `API_PROXY_BASE_URL` to your backend proxy (https://... base URL).  
   - Confirm custom URL scheme (`worthitai`) is registered for deep links.
3. **StoreKit Products**  
   - Keep `subscriptionProductMonthlyID` and `subscriptionProductAnnualID` aligned with App Store Connect identifiers.  
   - Update `tuliai.worthit.premium.storekit` if you add StoreKit configuration files for local testing.
4. **Privacy URLs**  
   - `AppConstants.termsOfUseURL`, `.privacyPolicyURL`, `.supportURL` point to hosted legal/support pages.
5. **Backend Expectations**  
   - Proxy must mirror OpenAI Responses API semantics and accept the enforced JSON-only prompt structure.  
   - Transcript and comment endpoints should tolerate multiple simultaneous requests (share extension + app).

---

## Development Notes
- The app defaults to dark mode. Use Xcode previews or run on-device to validate animations (e.g., time-saved banners).
- `APIManager.preWarm()` fires on launch to keep the proxy container warm-safe to no-op if the backend ignores GET `/`.
- When debugging the share extension, monitor the lock file in the App Group `ProcessLocks/` directory if instances exit unexpectedly.
- `Logger.isVerboseLoggingEnabled` is true under DEBUG; logs are searchable via the `[worthitapp]` prefix in Console.app.
- If you adjust prompt formatting, keep the strict JSON/line-break requirements to avoid decoding failures downstream.

---

## Testing & Verification
- **Unit/UI Tests** - Targets `WorthItTests` and `WorthItUITests` exist for future coverage.
- **Manual checks** (recommended before release):
  - Transcript + comment fetch for a long video.  
  - Cache hit path (run the same video twice).  
  - Paywall trigger after five free analyses in a day.  
  - Share extension invocation with the app closed and open.  
  - Q&A follow-up in different languages (ensures language handling works).  
  - Time-saved banner streak progression.

---

## Useful Links
- Terms of Use: `https://worthit.tuliai.com/terms`  
- Privacy Policy: `https://worthit.tuliai.com/privacy`  
- Support: `https://worthit.tuliai.com/support`  
- Manage Subscriptions: Apple's subscription management page (`https://apps.apple.com/account/subscriptions`)

---

Have questions or want to extend the app? Start with `MainViewModel` for flow control, `APIManager` for backend contracts, and `RootView` for UI composition. Happy shipping!
