# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **iOS Architect Crash Course** (January 26 - February 1, 2026) project from Essential Developer. It's a **legacy iOS application** designed as a refactoring exercise to practice applying clean architecture patterns to tightly-coupled code.

**Technology Stack:**
- UIKit-based iOS app (not SwiftUI)
- Xcode 16.x or 26.x required (no older/newer versions, no betas)
- XCTest integration tests
- No external dependencies (no SPM, CocoaPods, Carthage)

## Essential Commands

### Building and Testing
```bash
# Open project (if not already open)
open iACC.xcodeproj

# Run all tests (use xcodebuildmcp skill for more control)
# In Xcode: CMD+U
# Via CLI: xcodebuild test -project iACC.xcodeproj -scheme iACC -destination 'platform=iOS Simulator,name=iPhone 16'

# Build project
# In Xcode: CMD+B
```

**Critical Rule:** Run the full test suite (CMD+U) after **every** code change. All tests must pass before committing.

### Development Workflow
1. Make a change
2. Run tests with CMD+U
3. If tests pass → commit immediately
4. If tests fail → revert to previous commit and try again

**Build Requirements:**
- Project must build without warnings
- Code must be well-organized with consistent indentation

## Architecture Overview

### Application Structure

The app uses a **tab-based navigation** with three main screens:

1. **Friends** - List of friends with caching (premium users only)
2. **Transfers** - Sent/Received transfers with segment navigation
3. **Cards** - User's payment cards

**Key Files:**
- [SceneDelegate.swift](iACC/SceneDelegate.swift) - Entry point, initializes `FriendsCache`
- [MainTabBarController.swift](iACC/MainTabBarController.swift) - Root tab bar setup
- [ListViewController.swift](iACC/ListViewController.swift) - Generic reusable list (Friends, Transfers, Cards)
- [SegmentNavigationViewController.swift](iACC/SegmentNavigationViewController.swift) - Custom nav for Sent/Received segments

### Current Architecture (Legacy Anti-patterns)

**Singleton Pattern (Heavy Use):**
- `User.shared` - Current user singleton
- `FriendsAPI.shared`, `CardAPI.shared`, `TransfersAPI.shared` - Global API instances
- Makes testing difficult; requires global state replacement

**Tight Coupling:**
- `ListViewController` handles multiple contexts via boolean flags and type erasure (`[Any]`)
- View controllers directly access global singletons
- No dependency injection
- `FriendsCache` managed in `SceneDelegate` and accessed via `UIApplication.shared.connectedScenes`

**Closure-based APIs:**
- All APIs use completion handlers (not async/await)
- Manual threading with `DispatchQueue`

**Type Safety Issues:**
- `ListViewController` stores items as `[Any]` with runtime casting
- Generic model handling creates fragile type checks

### API Layer

All APIs simulate network delays:
- [FriendsAPI.swift](iACC/FriendsAPI.swift) - 0.75s delay
- [CardAPI.swift](iACC/CardAPI.swift) - 0.5s delay
- [TransfersAPI.swift](iACC/TransfersAPI.swift) - 0.5s delay
- [FriendsCache.swift](iACC/FriendsCache.swift) - 0.25s delay (premium users only)

### Models
- [User.swift](iACC/User.swift) - `id`, `name`, `isPremium`
- Friend, Card, Transfer models (referenced from APIs)

## Testing Strategy

**Test Location:** [iACCTests/](iACCTests/)

### Integration Test Approach

The project uses **integration tests** as the primary testing strategy (appropriate for legacy code):

**Test Categories:**
- [FriendsIntegrationTests.swift](iACCTests/Friends/FriendsIntegrationTests.swift) - 12+ tests covering listing, caching, retry, premium/non-premium
- [CardsIntegrationTests.swift](iACCTests/Cards/CardsIntegrationTests.swift) - Card listing, selection, refresh
- [SentTransfersIntegrationTests.swift](iACCTests/Transfers/SentTranfersIntegrationTests.swift), [ReceivedTransfersIntegrationTests.swift](iACCTests/Transfers/ReceivedTransfersIntegrationTests.swift)

### Test Infrastructure

**Dependency Injection for Tests:**
- [SceneBuilder.swift](iACCTests/Helpers/SceneBuilder.swift) - Creates test scenes with injectable dependencies
- Resets global singletons between tests
- Allows stubbing APIs, cache, and user state

**API Test Doubles:**
- `FriendsAPI+TestHelpers`, `CardAPI+TestHelpers`, `TransfersAPI+TestHelpers`
- Support `.once()`, `.results()`, `.resultBuilder()` patterns for stubbing
- Prevent real network requests

**UI Test Helpers:**
- [TestHelpers.swift](iACCTests/Helpers/TestHelpers.swift) - Async helpers: `until()`, `existence()`, `Timeout()`
- `UIViewController+TestHelpers` - View hierarchy traversal
- `UIView+TestHelpers`, `UIView+Tap` - UI interaction simulation
- `UIBarButtonItem+TestHelpers` - Button simulation

**Testing Pattern:**
```swift
// Typical integration test structure
let friends = [Friend.make()]
api.stub(.once(.success(friends)))
let scene = SceneBuilder.scene(api: api)

scene.launch()
await until(friends.first?.name, in: scene)
```

## Refactoring Targets

This is legacy code **intentionally designed for refactoring**. Common improvements:

1. **Replace Singletons with Dependency Injection**
   - Inject APIs and cache instead of using `.shared`
   - Remove global `User.shared`

2. **Separate Generic Controllers**
   - Split `ListViewController` into `FriendsListViewController`, `CardsListViewController`, `TransfersListViewController`
   - Remove type erasure (`[Any]`)

3. **Modernize Concurrency**
   - Migrate completion handlers to async/await
   - Use the `swift-concurrency` skill for guidance

4. **Add Architectural Layers**
   - Introduce Presenters/ViewModels
   - Separate business logic from view controllers
   - Consider Clean Architecture or MVVM patterns

5. **Improve Caching**
   - Abstract cache behind protocol
   - Remove direct UIApplication scene access

## Available Skills

Use these specialized skills via the Skill tool:

- **xcodebuildmcp** - Build, test, run, debug iOS apps (official skill, preferred for Xcode operations)
- **swift-concurrency** - Migrate to async/await, fix data races, Swift 6 patterns
- **swiftui-expert-skill** - SwiftUI patterns (though this project is UIKit-based)

## Project Constraints

**Xcode Version:** Must use 16.x or 26.x (no older, newer, or beta versions)

**Testing Discipline:**
- All tests must pass before committing
- Revert immediately if tests fail
- Integration tests validate behavior; if they fail, a behavior is broken

**Code Quality:**
- Zero warnings allowed
- Consistent indentation and formatting
- Well-organized code structure

## Course Context

This project is part of the Essential Developer iOS Architect Crash Course. The intentionally legacy design (singletons, tight coupling, no DI) provides practice in:
- Identifying architectural smells
- Safely refactoring with comprehensive test coverage
- Applying clean architecture principles incrementally
- Maintaining working software throughout refactoring
