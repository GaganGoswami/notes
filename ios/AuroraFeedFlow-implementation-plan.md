## Plan: AuroraFeedFlow — Full FEEED-style RSS Reader

**TL;DR:** Build a production-grade iOS/iPadOS RSS reader from the existing SwiftUI boilerplate, using zero third-party dependencies — only Apple frameworks (SwiftUI, Combine, CoreData, CloudKit, XMLParser, URLSession, WebKit, BackgroundTasks, NaturalLanguage). 14 ordered phases, fully testable, multi-device iCloud sync.

---

### Architecture Overview (ASCII)

```
┌────────────────────────────────────────────────────────────────────┐
│                        AuroraFeedFlow App                          │
├─────────────────────────────────────────────────────────────────-──┤
│  UI Layer (SwiftUI)                                                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │FeedList  │ │Article   │ │Sidebar   │ │Settings  │             │
│  │  View    │ │ Reader   │ │  Menu    │ │  View    │             │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘             │
├────────────────────────────────────────────────────────────────────┤
│  ViewModel Layer (MVVM · @Observable · Combine)                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │ FeedVM   │ │ArticleVM │ │ MenuVM   │ │SettingsVM│             │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘             │
├────────────────────────────────────────────────────────────────────┤
│  Service Layer                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │FeedParser│ │CloudKit  │ │Network   │ │ Search   │             │
│  │ Engine   │ │  Sync    │ │ Client   │ │ Engine   │             │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘             │
├────────────────────────────────────────────────────────────────────┤
│  Persistence (CoreData + NSPersistentCloudKitContainer)            │
│  CDFeed · CDFeedItem · CDFolder · CDCustomMenu                     │
└────────────────────────────────────────────────────────────────────┘
```

---

### Complete Project Structure

```
AuroraFeedFlow/
├── App/
│   ├── AuroraFeedFlowApp.swift        ← CloudKit + env injection
│   ├── AppCoordinator.swift           ← NavigationSplitView root
│   └── AppEnvironment.swift           ← global EnvironmentObject
├── Core/
│   ├── Models/         Feed · FeedItem · FeedCollection · CustomMenu · Bookmark · UserPreferences
│   ├── Protocols/      FeedParseable · Syncable · Searchable
│   └── Extensions/     Date+Formatting · String+HTML · URL+Feed
├── Modules/
│   ├── FeedParser/     FeedParserEngine · RSSParser · AtomParser · JSONFeedParser · OPMLParser
│   ├── Networking/     NetworkClient · NetworkReachability · ImageLoader
│   ├── Persistence/    PersistenceController · .xcdatamodeld · CD*+CoreDataClass.swift files
│   ├── Sync/           CloudKitSyncEngine · SyncConflictResolver · SyncStatusMonitor · ChangeTracker
│   ├── Search/         SearchEngine · FuzzyMatcher
│   ├── BackgroundRefresh/  BackgroundRefreshScheduler · RefreshOperation
│   └── Theme/          ThemeManager · ColorPalette · Typography · AnimationConstants
├── Features/
│   ├── FeedList/       FeedListView · FeedListViewModel · FeedCard · ArticleRowView · SectionHeaderView
│   ├── ArticleReader/  ArticleReaderView · ArticleReaderViewModel · ArticleWebView · ReaderToolbar
│   ├── Sidebar/        SidebarView · SidebarViewModel · SidebarMenuRow
│   ├── CustomMenu/     MenuBuilderView · MenuBuilderViewModel · FilterEditorView
│   ├── Bookmarks/      BookmarksView · BookmarksViewModel
│   ├── Settings/       SettingsView · FontSettingsView · ThemeSettingsView · SyncSettingsView
│   └── OPML/           OPMLImportView · OPMLExportView
├── Shared/
│   ├── Components/     ProgressiveImageView · UnreadBadge · LoadingView · EmptyStateView · IconColorPicker
│   └── Modifiers/      CardShadowModifier · SpringAnimationModifier · AdaptiveLayoutModifier
└── Tests/
    ├── FeedParserTests.swift
    ├── SyncLogicTests.swift
    ├── ViewModelTests.swift
    └── DatabaseIndexingTests.swift
```

---

### Implementation Phases

#### Phase 1 — Foundation & Data Layer
1. Rewrite AuroraFeedFlowApp.swift: inject `PersistenceController`, `AppEnvironment`, `ThemeManager`; configure CloudKit container ID.
2. `AppEnvironment` — global `ObservableObject` holding all service references.
3. Define CoreData schema with 4 entities (see schema below); use `NSPersistentCloudKitContainer`.
4. `PersistenceController` — automatic merge, background write context, `mergeByPropertyObjectTrump` policy.
5. CoreData subclasses + value-type bridging structs (`Feed`, `FeedItem`, etc.)

#### Phase 2 — Networking & Feed Parsing
6. `NetworkClient` — `async/await` URLSession, ETag/Last-Modified header caching.
7. `NetworkReachability` — `NWPathMonitor` → Combine publisher → offline flag.
8. `FeedParserEngine` — type detection (Content-Type + XML root element) → dispatch.
9. `RSSParser` — `XMLParserDelegate`, RSS 2.0 + MediaRSS.
10. `AtomParser` — Atom 1.0.
11. `JSONFeedParser` — `Codable` JSON Feed 1.1.
12. `OPMLParser` — import/export OPML 2.0.
13. `ImageLoader` — progressive download, `NSCache` + disk cache.

#### Phase 3 — Theme System
14. `ColorPalette` — semantic tokens (background, surface, primary, accent `#5E7CE2`, unreadIndicator).
15. `Typography` — scale (headline SF Pro 17pt semibold → article serif 28pt body 17pt 1.6× line height).
16. `ThemeManager` — `@Observable`, persists font family/size/density/theme mode.
17. `AnimationConstants` — spring presets, easing curves.

#### Phase 4 — Core Data Services
18. `SearchEngine` — `NSFetchRequest` + `NSPredicate` + `FuzzyMatcher` scoring, Combine publisher.
19. `BackgroundRefreshScheduler` — `BGTaskScheduler`, per-feed refresh, background context writes.
20. `ChangeTracker` — Combine `debounce(1.5s)` coalescing of read/bookmark mutations.

#### Phase 5 — iCloud Sync
21. `CloudKitSyncEngine` — monitors `NSPersistentCloudKitContainer` notifications, push subscriptions.
22. `SyncConflictResolver` — server-wins for read status; union-merge for bookmarks; server-wins for feed list.
23. `SyncStatusMonitor` — publishes `SyncStatus` enum → sync indicator in UI.
24. Handoff — `NSUserActivity` carrying `articleURL` + `readingProgress`; `onContinueUserActivity` handler.

#### Phase 6 — Navigation Shell
25. `AppCoordinator` — `NavigationSplitView`: sidebar | feed list | article. iPad: 3-column. iPhone: stack.
26. `SidebarView` — All Feeds / Unread / Bookmarks / Trending / Folders / Custom Menus. Expand animation.
27. `SidebarMenuRow` — icon + color swatch + badge; long-press context menu.

#### Phase 7 — Feed List & Article Reader
28. `FeedListView` — `LazyVStack` cards, section headers by recency, pull-to-refresh, swipe actions.
29. `FeedCard` — thumbnail, title (2-line headline), feed name, time-ago, unread dot, card shadow.
30. `FeedListViewModel` — Combine pipeline from `@FetchRequest` filtered by active menu.
31. `ArticleReaderView` — full-screen sheet, spring pull-to-dismiss, reading progress tracking.
32. `ArticleWebView` — `WKWebView` + injected CSS matching app typography, tracker blocking.
33. `ReaderToolbar` — bookmark, share, open-in-Safari, font size slider.

#### Phase 8 — Custom Menu Builder
34. `MenuBuilderView` — list of saved menus, drag-to-reorder, "New Menu" button.
35. `FilterEditorView` — compose: feed selector, tag picker, date range, read status, min word count; live preview count.
36. `MenuBuilderViewModel` — serializes `FilterSpec` → `filterJSON`; validates; saves `CDCustomMenu`.
37. `IconColorPicker` — SF Symbol search grid + color wheel.

#### Phase 9 — Settings, OPML, Bookmarks, Search
38. Settings: grouped form; font, theme, refresh interval, sync status, data controls.
39. OPML import: `fileImporter` → parse → preview → confirm. Export: share XML.
40. Bookmarks: same card layout, filtered by `isBookmarked`. Swipe-to-unbookmark.
41. Global search: `.searchable()` in toolbar → `SearchEngine` → grouped results.

#### Phase 10 — iPad & Platform Polish
42. 3-column `NavigationSplitView`, `.balanced` widths, always-visible sidebar on iPad landscape.
43. Keyboard shortcuts: `⌘R` refresh, `⌘F` search, `⌘B` bookmark, `⌘[`/`⌘]` next/prev, `⌘,` settings.
44. Drag-and-drop: feeds between folders via `onDrag`/`onDrop` + `NSItemProvider`.
45. Stage Manager: no hardcoded frames; adaptive `GeometryReader`-based layouts.

#### Phase 11 — Animations & Polish
46. Card appear: staggered `.spring(response: 0.4, dampingFraction: 0.7)` with index delay.
47. Pull-to-dismiss: `DragGesture` → scale + opacity, spring return on cancel.
48. Sidebar expand: `.easeInOut(0.25)` height transition.
49. Unread count: `.contentTransition(.numericText())`.
50. Loading skeleton: `.redacted(.placeholder)` + shimmer via `TimelineView`.

#### Phase 12 — Tests
51. `FeedParserTests` — mock RSS/Atom/JSON strings → assert correct `FeedItem` fields, date parsing, malformed input.
52. `SyncLogicTests` — in-memory context + conflict scenarios → assert resolved state.
53. `ViewModelTests` — in-memory CoreData store → `XCTestExpectation` on `@Published` outputs.
54. `DatabaseIndexingTests` — 1000-item insert → fetch benchmark < 100ms.

#### Phase 13 — Build & Run Instructions (generated as README)
55. Xcode requirements, signing setup, CloudKit container, entitlements, first-run steps.

#### Phase 14 — Optional AI Enhancements
56. On-device extractive summaries via `NaturalLanguage`.
57. Topic clustering via CoreML text embeddings + k-means.
58. Feed health monitor: track HTTP history per URL, alert on repeated failures.

---

### CoreData Schema

| Entity | Key Fields |
|---|---|
| `CDFeed` | id (UUID·idx), title, feedURL (idx), faviconURL, unreadCount, lastRefreshed, folderID, sortOrder |
| `CDFeedItem` | id (UUID·idx), feedID (idx), title (idx), articleURL, author, publishedAt (idx), summary, bodyHTML, imageURL, isRead (idx), isBookmarked (idx), readingProgress, wordCount, tags |
| `CDFolder` | id (UUID·idx), name, sortOrder |
| `CDCustomMenu` | id (UUID·idx), name, iconName, colorHex, filterJSON, sortOrder |

---

### Theme Spec

**Typography:**
- Feed list title: SF Pro Semibold 17pt
- Byline / timestamp: SF Pro Regular 12pt secondary
- Article title: New York / Georgia 28pt
- Article body: 17pt, 1.6× line height, max width 680pt
- Section header: SF Pro Medium 13pt all-caps

**Color Palette:**
- `background`: `systemBackground` / `#1C1C1E`
- `surface`: `secondarySystemBackground` / `#2C2C2E`
- `accent`: `#5E7CE2` (aurora blue — unread dots, highlights)
- `separator`: `separator` (system)
- Card shadow: light `rgba(0,0,0,0.08)` / dark `rgba(0,0,0,0.4)`, blur 12pt

---

### Verification Steps
1. `xcodebuild -scheme AuroraFeedFlow -destination 'iPhone 16 Pro'` — zero errors.
2. Add `https://feeds.feedburner.com/TechCrunch` — items appear in feed list.
3. Mark read → relaunch → remains read (CoreData persistence).
4. Two simulators with same iCloud → verify sync propagation.
5. XCTest suite — all pass, search benchmark < 100ms at 1000 items.
6. iPad: sidebar always visible in landscape, collapses in portrait.
7. Pull-to-dismiss article sheet — spring animation correct.
8. Custom menu "Unread + last 24h" — correct items filtered.
9. OPML import with BBC/Guardian sample → feeds added.
10. BGAppRefreshTask via debug → new items appear.

---

### Decisions
- **No third-party libraries** — pure Apple frameworks only.
- **CloudKit via `NSPersistentCloudKitContainer`** (not manual `CKRecord`) — less boilerplate, native conflict handling.
- **Swift 6 strict concurrency** — CoreData on dedicated `@ModelActor`-style actor, UI on `@MainActor`.
- **`@Observable` macro** (iOS 17+) for ViewModels; `@FetchRequest` in views needing live CoreData.
- **Minimum deployment: iOS 17.0 / iPadOS 17.0.**
- **Article rendering:** native `AttributedString` in `Text` for plain text; `WKWebView` + injected typography CSS for HTML.
- **Offline mode:** `NetworkReachability` gates all network calls; views always read from CoreData.

---

### Further Considerations
1. **Entitlements**: iCloud + CloudKit + Background Modes (fetch + remote-notifications + processing) must be enabled in the Xcode target — confirm if you have an Apple Developer account with a team/container ID ready to use, or if I should use a placeholder.
2. **AI Summaries**: On-device (`NaturalLanguage`) runs instantly; OpenAI API requires a key and network — include both or on-device only?
3. **Widget extension**: Should a HomeScreen widget (latest unread count + article) be a separate target in this plan, or deferred? 

Analyzing your answers...

Updated memory file [](file:///Users/gagan/Library/Application%20Support/Code/User/workspaceStorage/3d19de95f81da9c8075abc6a682dd026/GitHub.copilot-chat/memory-tool/memories/M2RhZGE5YTItZTA5Ny00NjQxLTgxNjYtNDBmZGVjYjIyMDFl/plan.md)

The answers are captured. Here is the **updated and finalized plan** with your decisions incorporated:

---

## Plan: AuroraFeedFlow — Complete iOS/iPadOS RSS Reader (Final)

**Confirmed decisions:**
- Placeholder personal team / container ID `iCloud.com.aurora.feedflow` (stub entitlements, easy to activate)
- AI summaries: on-device `NaturalLanguage` default + OpenAI abstractive (API key stored in Keychain)
- Widget target: `AuroraFeedFlowWidgets` (HomeScreen small/medium + Lock Screen)

---

### Phase Breakdown (14 phases → ~57 implementation steps)

| Phase | What Gets Built | Parallel? |
|---|---|---|
| 1 | Foundation: `AppEnvironment`, CoreData schema, `PersistenceController` | — |
| 2 | Networking + 4 parsers (RSS, Atom, JSON, OPML) + Image loader | parallel with 3 |
| 3 | Theme system: colors, typography, animation constants | parallel with 2 |
| 4 | Search engine, background refresh scheduler, change tracker | depends on 1+2 |
| 5 | iCloud sync engine, conflict resolver, Handoff | depends on 1 |
| 6 | Navigation shell: `AppCoordinator`, `SidebarView`, `SidebarMenuRow` | depends on 3 |
| 7 | `FeedListView`, `FeedCard`, `ArticleReaderView`, `WKWebView` reader | depends on 2+3+6 |
| 8 | Custom menu builder + `FilterEditorView` + `IconColorPicker` | depends on 6 |
| 9 | Settings, OPML import/export, Bookmarks, Global Search UI | depends on 6+7 |
| 10 | iPad: 3-column split, keyboard shortcuts, drag-and-drop | depends on 6+7 |
| 11 | Animations: staggered cards, pull-to-dismiss, skeleton loaders | depends on 7 |
| 12 | Widget target (`AuroraFeedFlowWidgets`) — WidgetKit small/medium/lock | depends on 1 |
| 13 | AI: `NaturalLanguage` summaries + optional OpenAI + topic clustering | depends on 7 |
| 14 | Tests: parser, sync, ViewModel, DB benchmarks | depends on 1–5 |

---

### Key Files to Create/Modify

**Modified:**
- AuroraFeedFlowApp.swift — rewire entry point with CloudKit + env injection
- ContentView.swift — replaced by `AppCoordinator` host

**Created (grouped by module):**

`App/` — `AppCoordinator.swift`, `AppEnvironment.swift`

`Core/Models/` — `Feed.swift`, `FeedItem.swift`, `FeedCollection.swift`, `CustomMenu.swift`, `Bookmark.swift`, `UserPreferences.swift`

`Core/Protocols/` — `FeedParseable.swift`, `Syncable.swift`, `Searchable.swift`

`Core/Extensions/` — `Date+Formatting.swift`, `String+HTML.swift`, `URL+Feed.swift`

`Modules/FeedParser/` — `FeedParserEngine.swift`, `RSSParser.swift`, `AtomParser.swift`, `JSONFeedParser.swift`, `OPMLParser.swift`

`Modules/Networking/` — `NetworkClient.swift`, `NetworkReachability.swift`, `ImageLoader.swift`

`Modules/Persistence/` — `PersistenceController.swift`, `AuroraFeedFlow.xcdatamodeld`, 4× `CD*+CoreDataClass.swift`

`Modules/Sync/` — `CloudKitSyncEngine.swift`, `SyncConflictResolver.swift`, `SyncStatusMonitor.swift`, `ChangeTracker.swift`

`Modules/Search/` — `SearchEngine.swift`, `FuzzyMatcher.swift`

`Modules/BackgroundRefresh/` — `BackgroundRefreshScheduler.swift`, `RefreshOperation.swift`

`Modules/Theme/` — `ThemeManager.swift`, `ColorPalette.swift`, `Typography.swift`, `AnimationConstants.swift`

`Features/FeedList/` — `FeedListView.swift`, `FeedListViewModel.swift`, `FeedCard.swift`, `ArticleRowView.swift`, `SectionHeaderView.swift`

`Features/ArticleReader/` — `ArticleReaderView.swift`, `ArticleReaderViewModel.swift`, `ArticleWebView.swift`, `ReaderToolbar.swift`

`Features/Sidebar/` — `SidebarView.swift`, `SidebarViewModel.swift`, `SidebarMenuRow.swift`

`Features/CustomMenu/` — `MenuBuilderView.swift`, `MenuBuilderViewModel.swift`, `FilterEditorView.swift`

`Features/Bookmarks/` — `BookmarksView.swift`, `BookmarksViewModel.swift`

`Features/Settings/` — `SettingsView.swift`, `FontSettingsView.swift`, `ThemeSettingsView.swift`, `SyncSettingsView.swift`

`Features/OPML/` — `OPMLImportView.swift`, `OPMLExportView.swift`

`Features/AI/` — `SummaryEngine.swift`, `OpenAISummaryClient.swift`, `TopicClusteringEngine.swift`

`Shared/Components/` — `ProgressiveImageView.swift`, `UnreadBadge.swift`, `LoadingView.swift`, `EmptyStateView.swift`, `IconColorPicker.swift`

`Shared/Modifiers/` — `CardShadowModifier.swift`, `SpringAnimationModifier.swift`, `AdaptiveLayoutModifier.swift`

`AuroraFeedFlowWidgets/` — `FeedWidget.swift`, `WidgetEntry.swift`, `AuroraFeedFlowWidgetsBundle.swift`

`Tests/` — `FeedParserTests.swift`, `SyncLogicTests.swift`, `ViewModelTests.swift`, `DatabaseIndexingTests.swift`

---

### CoreData Schema

```
CDFeed:       id·UUID(idx), title, feedURL(idx), faviconURL, unreadCount, lastRefreshed, folderID, sortOrder
CDFeedItem:   id·UUID(idx), feedID(idx), title(idx), articleURL, author, publishedAt(idx), summary,
              bodyHTML, imageURL, isRead(idx), isBookmarked(idx), readingProgress, wordCount, tags(idx)
CDFolder:     id·UUID(idx), name, sortOrder
CDCustomMenu: id·UUID(idx), name, iconName, colorHex, filterJSON, sortOrder
```

---

### Theme Spec Summary

| Token | Light | Dark |
|---|---|---|
| background | `systemBackground` | `#1C1C1E` |
| surface | `secondarySystemBackground` | `#2C2C2E` |
| accent / unread | `#5E7CE2` | `#5E7CE2` |
| card shadow | `rgba(0,0,0,0.08) blur 12pt` | `rgba(0,0,0,0.4) blur 8pt` |

| Element | Font |
|---|---|
| Feed title | SF Pro Semibold 17pt |
| Timestamp / byline | SF Pro Regular 12pt secondary |
| Article title | New York / Georgia 28pt |
| Article body | 17pt, 1.6× line height, max-width 680pt |
| Section header | SF Pro Medium 13pt caps |

---

### Verification Steps
1. Zero-error build on iPhone 16 Pro + iPad Pro simulators.
2. Add a real RSS URL → items parse and display in feed list.
3. Mark read → kill → relaunch → persisted.
4. Two simulators + iCloud → sync propagates.
5. Full XCTest suite passes; DB search < 100ms at 1000 items.
6. 3-column iPad split view — sidebar always visible in landscape.
7. Spring pull-to-dismiss on article reader plays correctly.
8. Custom menu with compound filter returns correct items.
9. OPML import with sample file → feeds added.
10. Widget renders on HomeScreen with current unread count.

---

**Ready to implement.** Approve this plan and the coding agent will execute all 14 phases in order, generating every file listed above.