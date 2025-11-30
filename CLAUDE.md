# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

HSTracker is an automatic deck tracker and deck manager for Hearthstone on macOS. It monitors the Hearthstone game in real-time and provides overlay windows showing deck state, card tracking, Battlegrounds information, and game statistics.

## Build Commands

### Initial Setup
```bash
# Install SwiftLint (required)
brew install swiftlint

# Download card data
./scripts/cards_download.sh

# Bootstrap dependencies (downloads card data)
./scripts/bootstrap.sh
```

### Building
```bash
# Build with Xcode
xcodebuild -scheme HSTracker -project HSTracker.xcodeproj

# Or open in Xcode
open HSTracker.xcodeproj

# Build configurations: Debug, Release
# Default scheme: HSTracker
```

### Code Signing for Local Development
HSTracker **must** be code signed to function properly. For local development:
- Edit `Config.xcconfig` and uncomment the debug identity lines:
  ```
  CODE_SIGN_IDENTITY = -
  DEVELOPMENT_TEAM =
  ```
- **Never commit changes to Config.xcconfig**

### Linting
```bash
# SwiftLint runs automatically on build
# Manual run:
swiftlint
```

### Testing
```bash
# Run tests with fastlane
bundle exec fastlane scan

# Or in Xcode
xcodebuild -scheme HSTrackerTests test
```

## Architecture Overview

### Event-Driven MVC Hybrid Pattern

HSTracker uses a sophisticated event-driven architecture layered on top of MVC:
- **Model**: `Game` class (central state holder) with `Player`, `Entity`, and domain objects
- **View**: `WindowManager` orchestrates UI overlays and trackers
- **Controller**: `CoreManager` coordinates the entire system
- **Event Stream**: `PowerEventHandler` protocol drives game state from Hearthstone logs

### Core Data Flow

```
Hearthstone.exe → Log Files → LogReaderManager → PowerGameStateParser
→ PowerEventHandler (Game) → Game Model → WindowManager → UI Overlays
```

### Three Parallel Communication Channels with Hearthstone

1. **Log File Tailing** (primary mechanism):
   - `LogReaderManager` spawns multiple `LogReader` threads that tail different log files
   - `Power.log`, `Arena.log`, `LoadingScreen.log`, `Rachelle.log`, etc.
   - `PowerGameStateParser` uses regex patterns to parse events
   - Events dispatched to `PowerEventHandler` (implemented by `Game` class)

2. **Memory Reading** (via HearthMirror):
   - Direct process memory access using native C library wrapper
   - Retrieves BattleTag, log session directory, account info
   - Validates log-based state with real-time memory state

3. **Mono Runtime Inspection** (via HearthWatcher):
   - Hooks into Hearthstone's .NET/Mono runtime
   - Multiple watcher classes: `ArenaWatcher`, `BaconWatcher` (Battlegrounds), `DeckPickerWatcher`, etc.
   - Detects scene changes, UI states, and triggers callbacks
   - Managed by central `Watchers` class

### Key Architectural Components

**Logging/** (Core game state model and log parsing):
- `Game.swift` (~4600 lines): Central state holder implementing `PowerEventHandler`
- `CoreManager.swift`: System orchestrator managing LogReaderManager and Game
- `LogReaderManager.swift`: Manages multiple LogReader instances for different log types
- `Parsers/PowerGameStateParser.swift`: Complex regex-based log event parser
- `Handlers/`: Event handlers for specific game events (Arena, Choices, LoadingScreen)

**HearthMirror/**: Native C library wrapper for memory reading
- Provides direct memory access to running Hearthstone process
- Used for account info and state validation

**HearthWatcher/**: Mono-based memory watchers
- Each watcher monitors specific game screens (Arena, Battlegrounds, Deck Picker, etc.)
- Triggers callbacks on state changes and scene transitions
- Parallel validation system to log-based tracking

**BobsBuddy/**: Battlegrounds combat simulator
- Mono proxy classes for calling into BobsBuddy.dll
- `BobsBuddyInvoker` orchestrates board simulation
- Invokes compiled C# code via Mono runtime

**Mono/**: Mono runtime integration
- C# bridge layer for accessing Hearthstone's .NET assemblies
- `MonoHelper` provides low-level Mono API calls
- Proxy classes wrap Mono objects to simulate Hearthstone state

**UIs/Trackers/**: Overlay window system
- `WindowManager.swift`: Factory for all overlay windows; manages positioning
- `Tracker.swift`: Player/opponent deck card list overlay
- Other overlays: Battlegrounds, BobsBuddy, Board damage, Timer, Counters
- Receives update callbacks from Game model, renders via NSWindow overlays

**Database/**: Card data and persistence
- Card definitions loaded from XML
- Realm-based persistence (Realm Swift ORM)
- Card metadata, mechanics, costs

**HSReplay/**: hsreplay.net integration
- OAuth2 authentication flow
- `LogUploader`: Sends game logs to HSReplay servers
- `HSReplayAPI`: REST API client for deck uploads and stats

### State Update Flow Example

When a card is played:
1. Hearthstone writes entry to `Power.log`
2. `LogReader` detects file change and reads new line
3. `PowerGameStateParser` regex matches event
4. `PowerEventHandler` callback fires (e.g., `Game.playerPlay()`)
5. `Game` model updates: `Entity`, `Player.cards`, deck state
6. `Game.updateTrackers()` dispatches to UI thread via GCD
7. `Tracker.update(cards: [...])` receives new card state
8. NSWindow renders updated card overlays on screen

### Key Architectural Patterns

- **Observer Pattern**: PowerEventHandler protocol; watchers with callbacks
- **Singleton Pattern**: AppDelegate, CoreManager, Database, Watchers
- **Proxy Pattern**: Mono proxy classes wrap C# objects
- **Queue-based Threading**: DispatchQueue for concurrent log reading and UI updates
- **Lazy Initialization**: Window overlays created on-demand in WindowManager
- **State Machine**: SceneHandler manages Hearthstone mode transitions

### Resilience Through Multiple Data Sources

- Logs alone might miss events; Mono watchers provide parallel validation
- Each game mode (Constructed, Battlegrounds, Arena, Mercenaries) has separate watchers
- Real-time performance via multiple concurrent threads prevents UI blocking
- Debounced updates (0.5s delay) for smooth UI rendering

## Common Development Patterns

### Adding a New Feature for a Game Mode

1. Create a new watcher in `HearthWatcher/` if you need to monitor Hearthstone UI state
2. Add log parsing logic in `Logging/Handlers/` if tracking log events
3. Update `Game.swift` to handle new game state
4. Create UI overlay in `UIs/` and register in `WindowManager`
5. Wire up callbacks from watcher/handler to Game to UI

### Working with Game State

- Central state is in `Game.swift` (`HSTracker/Logging/Game.swift:1`)
- All game state changes should go through `Game` class methods
- UI updates must be dispatched to main thread via `DispatchQueue.main.async`
- Use `Game.updateTrackers()` to trigger UI refresh

### Log Parsing

- Regex patterns in `PowerGameStateParser.swift`
- Events dispatched to `PowerEventHandler` protocol
- `Game` class implements `PowerEventHandler` to receive events
- Each log type (Power, Arena, LoadingScreen, etc.) has dedicated handler

### Mono/C# Integration

- Use proxy classes in `Mono/` to wrap Mono objects
- `MonoHelper` provides low-level Mono runtime API calls
- BobsBuddy uses Mono to invoke C# combat simulation code
- HearthWatcher uses Mono to inspect Hearthstone's .NET object state

## SwiftLint Configuration

SwiftLint runs automatically on build. Configuration in `.swiftlint.yml`:
- Excluded: `Carthage`, `LinqExtension.swift`, `NSData+GZIP.swift`, `CardIds/`, test directories
- Disabled rules: cyclomatic_complexity, function_body_length, file_length, type_body_length, identifier_name, function_parameter_count, line_length
- Max type name length: 128 chars (warning), 132 chars (error)

## Dependencies

Managed via Swift Package Manager (SPM). Key dependencies:
- **Realm**: Database/ORM for persistence
- **Sparkle**: Auto-update framework
- **PromiseKit**: Promise-based async operations
- **OAuthSwift**: OAuth2 authentication for HSReplay
- **Sentry**: Crash reporting
- **Mixpanel**: Analytics
- **Kanna**: XML/HTML parsing for card data
- **SwiftyBeaver**: Logging framework
- **Erik**: Web scraping utilities
- **FMDB**: SQLite wrapper

## Release Process

See `CONTRIBUTING.md` for full release flow. Tools used:
- **AppCenter**: Crash monitoring
- **Sparkle**: Auto-update delivery
- **GitHub Releases**: Release hosting
- **fastlane**: Automated release workflow (`bundle exec fastlane release`)

## Key Files to Know

- `HSTracker/AppDelegate.swift`: Application entry point
- `HSTracker/Logging/Game.swift`: Central game state model (~4600 lines)
- `HSTracker/Logging/CoreManager.swift`: System orchestrator
- `HSTracker/UIs/WindowManager.swift`: UI overlay factory
- `Config.xcconfig`: Code signing configuration (don't commit changes)
- `CHANGELOG.md`: Release notes (update for releases)
- `.swiftlint.yml`: Linting rules

## Project Conventions

- Follow existing commit message style (see `git log`)
- No more than one change per commit
- Every commit should pass all tests
- Keep commits focused and atomic
- HearthSim requires a signed CLA for contributions
