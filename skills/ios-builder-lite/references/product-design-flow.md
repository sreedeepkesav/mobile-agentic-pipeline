# Product + Design Flow

## Overview

The Product + Design phase is the entry point to the iOS Lite Pipeline. It takes your idea (a text description, a feature request, or a task) and produces a complete design specification ready for the Coder team.

**Duration**: ~10-15 minutes of AI work
**Output**: Spec, screens, design tokens, edge cases
**Next step**: Review Gate 1 (you approve or request edits)

---

## Phase: Research

The Product + Design agent researches best practices before planning anything.

### iOS Human Interface Guidelines

- Studies Apple's HIG for the feature type
- Understands platform conventions (gestures, navigation patterns, accessibility)
- Identifies standard components (buttons, lists, navigation bar, tab bar)
- Notes spacing, typography, and color usage standards

### Similar App Patterns

- Research how existing apps solve the same problem
- Understand user expectations (e.g., how do timer apps handle paused state?)
- Identify common edge cases (empty state, error states, loading)
- Note any iOS-specific patterns (pull-to-refresh, swipe-to-delete, etc.)

### SwiftUI Best Practices

- How to structure view hierarchy
- State management patterns (ViewState, @Published, ObservableObject)
- Async/await loading patterns
- Accessibility (labels, identifiers)
- Performance considerations (list rendering, image loading)

### API Design Conventions

- RESTful vs GraphQL patterns
- Request/response shapes
- Error response formats
- Pagination and filtering
- Authentication (token-based, OAuth)

---

## Phase: Planning

Once research is complete, the agent plans the feature in detail.

### Feature Specification

A clear written spec with:
- **Feature name**: One-line title (e.g., "Pull-to-Refresh on Timeline Screen")
- **Problem**: What problem does this solve?
- **Solution**: How the feature works from the user's perspective
- **Acceptance Criteria**: Testable conditions (e.g., "When user pulls down, loading indicator appears. When data arrives, list updates.")
- **Scope**: What's in scope, what's out of scope
- **Success metrics**: How to measure if the feature works (if applicable)

### Screen List with Navigation Flows

- **Screen inventory**: All screens involved in the feature
- **Purpose**: What each screen does
- **Navigation paths**: How users get from one screen to another
- **Entry point**: Which screen starts the flow
- **Exit point**: Where the flow ends

**Example**:
```
Screen: Timer List
├─ Display: List of timers with start/pause/delete buttons
├─ Navigation out: Tap timer → Timer Detail screen
└─ Navigation out: Tap + button → Create Timer screen

Screen: Timer Detail
├─ Display: Full timer state (remaining time, duration, name)
├─ Navigation in: From Timer List (tap timer)
└─ Navigation back: Back button → Timer List
```

### API Contracts

For each API call needed:
- **Endpoint**: GET /timers, POST /timers, etc.
- **Request shape**: What params do you send?
- **Response shape**: What data comes back?
- **Error responses**: What errors are possible?

**Example**:
```
GET /timers
Request: (no body)
Response: {
  "timers": [
    { "id": "uuid", "name": "Workout", "duration": 300, "remainingTime": 150, "isRunning": true }
  ]
}
Error 401: User not authenticated
Error 500: Server error
```

### Data Models

Entities and their relationships:
- **Timer**: id, name, duration, remainingTime, isRunning, createdAt
- **User**: id, email, theme (if relevant)
- Relationships: User has many Timers

### Edge Cases

Identified scenarios that need special handling:

- **Loading**: Initial data load (show spinner, disable buttons)
- **Empty**: No timers yet (show "Create your first timer" message)
- **Error**: Network failure (show error message, retry button)
- **Offline**: No network (show cached data, disable create)
- **Timeout**: API call takes too long (show error after 10 seconds)
- **Concurrent**: User creates timer while list is loading (queue the action)

---

## Phase: Design

UI and visual design based on the plan.

### UI Wireframes

HTML/SVG mockups for each screen showing:
- Layout (positions of elements)
- Component types (buttons, lists, form fields)
- Labels and placeholders
- Empty states and error messages

**Example for Timer List screen**:
```
┌─────────────────────────┐
│ Timers                  │  ← Navigation title
├─────────────────────────┤
│ [+]                     │  ← Floating action button (add timer)
│ Workout        [▶][⏹]   │  ← Timer row
│ 5:30 remaining          │
├─────────────────────────┤
│ Meditation     [⏸][⏹]   │  ← Another timer
│ 2:15 remaining          │
├─────────────────────────┤
│                         │
└─────────────────────────┘
```

### Component Layouts

Detailed breakdown of each component:
- **TimerRow**: Icon + Name + Duration + Buttons
- **CreateButton**: Floating action button, position, size
- **LoadingIndicator**: Spinner style, position
- **ErrorBanner**: Error message box, dismiss button

### Color Palette

Define colors used:
- **Primary**: Brand color (e.g., #007AFF for blue)
- **Background**: App background (e.g., #FFFFFF for light)
- **Text**: Text color (e.g., #000000 for dark text)
- **Error**: Error messages (e.g., #FF3B30 for red)
- **Success**: Success states (e.g., #34C759 for green)
- **Disabled**: Disabled button state (e.g., #C7C7CC for gray)

### Typography

Define text styles:
- **Title 1**: 34pt, semibold (screen titles)
- **Title 2**: 28pt, semibold (section headers)
- **Body**: 17pt, regular (main content)
- **Callout**: 16pt, regular (secondary content)
- **Caption**: 12pt, regular (helper text)

### Spacing Tokens

Define margin and padding values:
- **xs**: 4pt (tight spacing)
- **sm**: 8pt (small spacing)
- **md**: 16pt (standard spacing)
- **lg**: 24pt (large spacing)
- **xl**: 32pt (extra large spacing)

### Screen-by-Screen Description

For each screen, write a detailed description that the Coder will read:

**Timer List Screen**:
- Primary content: Vertical list of Timer rows
- Top navigation: Title "Timers" with no back button (root screen)
- Floating action button (bottom right): Creates new timer
- Pull-to-refresh: Refreshes timer list
- Each Timer row shows: name, remaining time, start/pause/stop buttons
- Empty state: "No timers yet" with "Create one" button
- Error state: "Failed to load timers" with "Retry" button
- Loading state: Spinner in center of screen

**Timer Detail Screen**:
- Primary content: Large timer display (remaining time in big text)
- Buttons: Start, Pause, Reset, Delete
- Edit section: Name field, duration picker
- Back button (top left): Returns to Timer List
- Save button (top right): Saves changes and returns to list
- State: If timer is running, show countdown updating every second
- State: If paused, show "Paused" badge and remaining time
- State: If stopped, show "Stopped" badge

---

## Output: Specification Files

### 1. spec.md

Feature specification with acceptance criteria.

**Structure**:
```markdown
# Feature: Pull-to-Refresh on Timeline Screen

## Problem
Users want to manually refresh the timeline without leaving the screen.

## Solution
Add pull-to-refresh gesture to timeline list. Pull down → loading spinner → when data arrives, list updates.

## Acceptance Criteria
- [ ] Pull-to-refresh gesture recognized on timeline list
- [ ] Loading spinner shows while fetching new data
- [ ] Timeline list updates with fresh data after pull
- [ ] Pull-to-refresh disabled when already loading
- [ ] Pull-to-refresh succeeds even when offline (shows cached data)
- [ ] Pull-to-refresh fails gracefully with error message

## Scope
- Add pull-to-refresh to timeline screen only
- Update existing list, do not add pagination
- Use native iOS pull-to-refresh component

## Out of Scope
- Auto-refresh on timer
- Refresh other screens (profile, settings)
- Analytics tracking

## Success Metrics
- Refresh completes in <2 seconds on 4G
- No flashing or layout shift during update
```

### 2. screens.md

Screen inventory and navigation flows.

**Structure**:
```markdown
# Screens

## Screen: Timer List (Root)
- **Purpose**: Display all user timers
- **Components**: TimerRow (repeating), FloatingActionButton, PullToRefresh
- **Navigation out**: Tap TimerRow → Timer Detail
- **Navigation out**: Tap FAB → Create Timer
- **States**: Loading, Success, Empty, Error

## Screen: Timer Detail
- **Purpose**: Show timer details, edit name/duration, start/pause/reset
- **Components**: Timer display, Edit form, Action buttons
- **Navigation in**: From Timer List (TimerRow tap, ID passed)
- **Navigation back**: Back button or save button → Timer List
- **States**: Loading, Success, Error

## Navigation Flow
```
Timer List (with pull-to-refresh)
    ├─ User pulls down → Load timers → Update list
    ├─ User taps timer → Timer Detail screen
    │   └─ User taps edit, saves → Back to Timer List
    └─ User taps + → Create Timer screen
        └─ User taps save → Back to Timer List
```

### 3. design-tokens.json

Structured design tokens for theming.

**Structure**:
```json
{
  "colors": {
    "primary": "#007AFF",
    "background": "#FFFFFF",
    "text": "#000000",
    "textSecondary": "#666666",
    "error": "#FF3B30",
    "success": "#34C759",
    "disabled": "#C7C7CC"
  },
  "typography": {
    "title1": { "size": 34, "weight": "semibold" },
    "title2": { "size": 28, "weight": "semibold" },
    "body": { "size": 17, "weight": "regular" },
    "callout": { "size": 16, "weight": "regular" },
    "caption": { "size": 12, "weight": "regular" }
  },
  "spacing": {
    "xs": 4,
    "sm": 8,
    "md": 16,
    "lg": 24,
    "xl": 32
  },
  "cornerRadius": {
    "small": 8,
    "medium": 12,
    "large": 16
  }
}
```

### 4. edge-cases.md

Edge case scenarios and handling.

**Structure**:
```markdown
# Edge Cases

## Loading State
- **Trigger**: Initial app launch or pull-to-refresh
- **Display**: Centered spinner, no data visible
- **Duration**: <2 seconds typical
- **User action**: Cannot interact with list (buttons disabled)
- **Recovery**: Data arrives → spinner disappears, list shows

## Empty State
- **Trigger**: User has no timers created
- **Display**: Centered "No timers yet" message with "Create one" button
- **User action**: Tap button → Create Timer screen

## Error State
- **Trigger**: Network request fails
- **Display**: Error message, "Retry" button, no data visible
- **Message**: "Failed to load timers. Check your connection."
- **User action**: Tap Retry → attempt again

## Offline State
- **Trigger**: Device offline (no network)
- **Display**: Cached timers shown if available, with "Offline mode" badge
- **User action**: Can view, but cannot create/edit
- **Recovery**: When online → badge disappears, can edit again

## Timeout State
- **Trigger**: Network request takes >10 seconds
- **Display**: Error message, "Retry" button
- **Message**: "Request took too long. Please try again."
- **User action**: Tap Retry

## Concurrent Actions
- **Trigger**: User creates timer while list is loading
- **Behavior**: Queue the create request
- **Behavior**: After list loads, process create
- **Result**: New timer appears in list
```

---

## Review Gate 1

Once all outputs are complete, the pipeline pauses.

**You see**:
- Feature spec with scope and acceptance criteria
- Screen wireframes and navigation flows
- Design tokens (colors, typography, spacing)
- Edge case handling plan
- API contracts (if applicable)

**You decide**:
- **Approve** → Start Coder phase (all 4 phases, Build, Git)
- **Edit** → Request specific changes (scope, screens, colors, etc.)
  - Product+Design revises and re-pauses at gate
  - Repeat until approved
- **Reject** → Abandon spec, start fresh from your idea
  - Reset to idea entry
  - Provide new description if needed

---

## Tips for Better Specs

### Be Specific About Scope

**Bad**: "Build a notes app"
**Good**: "Build a notes app with create, list, detail, and delete. Dark mode not required."

### Describe Key Flows

**Bad**: (no navigation description)
**Good**: "From home screen, user taps [+], enters note text, taps save, returns to home with new note in list."

### Call Out Edge Cases

**Bad**: (no mention of errors)
**Good**: "If network fails, show error message and retry button. If offline, show cached notes."

### Provide Design Guidance

**Bad**: (no design hints)
**Good**: "Use iOS standard blue for primary button. Use light gray background. Follow iOS spacing guidelines (16pt margins)."

---

## See Also

- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-builder-lite/SKILL.md` — Full pipeline overview
- `/sessions/cool-happy-noether/mnt/outputs/mobile-agentic-pipeline/skills/ios-builder-lite/references/code-gen-phases.md` — What Coder does with your spec
