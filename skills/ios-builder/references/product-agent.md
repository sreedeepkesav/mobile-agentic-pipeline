# Product Agent Guide

Complete reference for the Product Agent in the ios-builder skill. Covers invocation conditions, 5 core capabilities, outputs, and review gates.

## Overview

The Product Agent is a conditional agent that bridges business requirements and technical specifications. It takes informal requirements, designs, and research and produces detailed, executable specs that Code Gen and architects use to build high-quality features.

**Key Principle:** Product Agent is never optional for unclear features. It's either active (always), gated per-task, or skipped for well-specced work.

## When Product Agent Is Invoked vs Skipped

### Invoked (Active)

**Config: `enabled: always`**
- Typical context: Personal/solo development (always need specs)
- Every task triggers Product Agent before Code Gen
- Ensures clarity and design before any code written

**Config: `enabled: per-task` (default) + Spec Assessment**
- One-Liner specs (title only) → Product Agent active
- Partial specs (some context, missing sections) → Product Agent fills gaps
- Incomplete designs → Product Agent extracts components
- Ambiguous requirements → Product Agent clarifies via research
- Design-only (no functional spec) → Product Agent decomposes design to tasks

**Examples triggering Product Agent (per-task mode):**
```
Input: "Add dark mode"
Assessment: One-liner, no AC, no design
Trigger: Product Agent invoked
Output: Full spec.md with AC, design tokens, ViewModel changes, etc.

Input: "Implement the Figma design"
Assessment: Partial (design yes, AC no, API contract no)
Trigger: Product Agent invoked
Output: AC extracted from design, task graph, component map

Input: "Create post feature as per epic #234"
Assessment: One-liner, epic is broad
Trigger: Product Agent invoked
Output: Scoped feature spec, 3 user stories, API contracts, design map
```

### Skipped

**Config: `enabled: never`**
- Typical context: Well-specced enterprise teams with design systems
- Product Agent never runs; Code Gen reads specs directly
- Risk: unclear specs slip through without formalization

**Spec Assessment: Well-Specced**
- Contains: AC (acceptance criteria) + Design (Figma/XD file) + API contracts
- Example:
  ```
  SPEC: User Profile

  AC:
  - User can view their profile info (name, email, avatar)
  - User can edit name, email in-place
  - Save triggers PATCH /users/:id {name, email}
  - Error state shows alert if PATCH fails

  Design: [Figma link]
  API: PATCH /users/:id → {name, email} → {status, user}
  Error Cases:
    - Network timeout: show timeout message
    - 400 validation error: show field errors
    - 401 unauthorized: nav to login
  ```
- Decision: Product Agent skipped; Code Gen starts immediately
- Reasoning: Spec completeness eliminates ambiguity

## 5 Core Capabilities

### Capability 1: Feature Decomposition

Breaks a high-level feature into executable, independent parts.

**Input:** Feature title, epic description, design (optional), rough requirement

**Process:**
1. Identify domain entities (User, Post, Comment, etc.)
2. Map business flows (create → edit → delete → share)
3. Extract user stories (As a [role], I [action] so that [benefit])
4. Identify edge cases and error scenarios
5. Define API endpoints and contracts
6. Map screens and UI flows

**Output: decomposition.md**

Example: "Create social post feature"

```markdown
# Feature Decomposition: Create Social Post

## Domain Entities
- Post: {id, userId, title, body, imageUrls[], createdAt}
- Image: {id, url, uploadedAt}
- PostDraft: {id, userId, title, body, imageUrls[], savedAt}

## User Stories

### Story 1: Create Post with Text
As a user, I want to write and publish a post so that others see my thoughts.

Acceptance Criteria:
- Text input accepts up to 280 characters
- Post button is disabled if text is empty
- Tapping Post calls POST /posts {title, body}
- Success: nav to post detail; show confirmation
- Failure: show error, keep text, enable retry

### Story 2: Add Images to Post
As a user, I want to attach images to my post so that I can share photos.

Acceptance Criteria:
- "Add image" button opens photo library
- Selected image is shown in preview
- Up to 5 images allowed
- Each image is uploaded before POST /posts
- If any image fails: show error, keep other images
- Success: all images uploaded, post published

### Story 3: Draft Auto-Save
As a user, I want my post draft saved automatically so I don't lose my work.

Acceptance Criteria:
- Every 3 seconds while typing: save draft locally
- Tapping back: ask "Save draft?" or auto-save
- Reopening composer: load last saved draft
- Clearing draft: clear old data

## API Contracts

POST /posts
Request: {title: string, body: string}
Response: {id, status, user: {...}}
Errors: 400 (validation), 401 (unauthorized), 500 (server)

POST /uploads
Request: multipart {image: File}
Response: {id, url}
Errors: 400 (invalid file), 413 (too large)

GET /posts/:id/drafts
Response: {id, title, body, imageUrls}

## Screens
1. PostComposerView: text input, image preview, post button
2. ImagePickerView: library picker, preview grid
3. PostDetailView: displayed post, interactions

## Edge Cases
- Network interruption during post creation
- User cancels during image upload
- User rapid-clicks post button (debounce)
- User logs out with draft in progress
- Image file deleted between selection and upload
```

### Capability 2: Spec Writing (AC + API Contracts)

Converts decomposition and research into formal specification with acceptance criteria and API contracts.

**Output: spec.md**

```markdown
# Specification: Create Social Post

## Overview
Feature to enable users to create, edit, and save text and image posts.

## Acceptance Criteria

### AC1: Post Creation (Text Only)
Given: User is on Composer screen
When: User enters text and taps "Post"
Then:
  - Text is sent to POST /posts {title: "", body: <text>}
  - On success (200): Navigate to PostDetailView, show "Posted!"
  - On failure (400): Show validation error, keep text
  - On failure (401): Clear text, navigate to LoginView
  - On failure (5xx): Show "Server error", enable retry

### AC2: Image Attachment
Given: User is on Composer with text entered
When: User taps "Add Image" and selects from library
Then:
  - Image preview shown immediately (local)
  - Image uploaded via POST /uploads before post creation
  - Progress indicator shown during upload
  - On success: Image shown in grid
  - On failure: Show "Image failed to upload", don't create post

### AC3: Multiple Images
Given: User has attached image #1
When: User taps "Add Image" again
Then:
  - Can select and upload image #2, #3, up to 5 total
  - Images display in grid
  - Removing image: tap X, image removed from preview and upload queue

### AC4: Draft Auto-Save
Given: User is typing in Composer
When: User pauses (3+ seconds without typing)
Then:
  - Current text and images saved to local POST /posts/:id/drafts
  - On next app launch: draft available in Composer
  - User can clear draft (trash icon)

### AC5: Error Handling
Given: Any error occurs (network, API, file)
When: Error response received
Then:
  - Error message shown (not technical, user-friendly)
  - User can retry (button) or cancel (discard and back)
  - Draft saved before showing error (user doesn't lose work)

## API Contracts

### POST /posts
**Purpose:** Create a new post

**Request:**
```json
{
  "title": "string (optional, default '')",
  "body": "string (required, max 280 chars)",
  "imageIds": ["string (optional, array of upload IDs)"]
}
```

**Response (200 OK):**
```json
{
  "id": "uuid",
  "userId": "uuid",
  "title": "string",
  "body": "string",
  "imageUrls": ["https://..."],
  "createdAt": "2024-01-15T10:30:00Z"
}
```

**Errors:**
- 400 Bad Request: {error: "body required"} or {error: "body exceeds 280 chars"}
- 401 Unauthorized: {error: "user not authenticated"}
- 500 Internal Server Error: {error: "database error"}

### POST /uploads
**Purpose:** Upload image file and return URL

**Request:**
```
multipart/form-data
- image: File (PNG, JPG, max 5MB)
```

**Response (200 OK):**
```json
{
  "id": "uuid",
  "url": "https://cdn.example.com/uploads/uuid.jpg"
}
```

**Errors:**
- 400 Bad Request: {error: "invalid file type"}
- 413 Payload Too Large: {error: "file exceeds 5MB"}
- 500 Internal Server Error: {error: "upload service error"}

### GET /posts/:id/drafts
**Purpose:** Fetch user's last saved draft for post creation

**Response (200 OK):**
```json
{
  "id": "uuid",
  "title": "string",
  "body": "string",
  "imageIds": ["uuid"],
  "savedAt": "2024-01-15T10:30:00Z"
}
```

**Response (404 Not Found):** No draft exists

## Edge Cases & Error Handling

| Scenario | Handling |
|----------|----------|
| Network timeout during POST /posts | Retry dialog, keep text/images |
| Image upload fails on image #2 of 3 | Show error, don't create post, keep all data |
| User logs out with draft | Clear draft on logout, notify user |
| Image deleted from device before upload | Handle gracefully, remove from preview |
| User rapid-clicks Post button | Debounce 500ms, disable button during upload |
| API returns 401 (token expired) | Clear text, navigate to login, prompt re-auth |

## Design Tokens (from Figma)
- Primary button color: #007AFF
- Error text color: #FF3B30
- Border radius: 8pt
- Spacing unit: 8pt (multiples of 8)

## Screens & Navigation
1. **Composer Screen**: Form with text input, image grid, post button
2. **Image Picker**: Native photo library picker
3. **Post Detail**: Display created post with images
```

### Capability 3: Task Graph + Prioritization

Builds a directed acyclic graph (DAG) of dependencies for Sprint Batches or complex features.

**Output: task-graph.json**

```json
{
  "feature": "Create Social Post",
  "tasks": [
    {
      "id": "T1",
      "title": "Domain: Post entity and protocols",
      "type": "domain",
      "dependencies": [],
      "estimated_effort": 30,
      "owner": "domain-lead"
    },
    {
      "id": "T2",
      "title": "Data: Post repository and API client",
      "type": "data",
      "dependencies": ["T1"],
      "estimated_effort": 90,
      "owner": "data-lead"
    },
    {
      "id": "T3",
      "title": "Data: Image upload and storage",
      "type": "data",
      "dependencies": ["T1"],
      "estimated_effort": 60,
      "owner": "data-lead"
    },
    {
      "id": "T4",
      "title": "Presentation: Composer view",
      "type": "presentation",
      "dependencies": ["T1"],
      "estimated_effort": 80,
      "owner": "presentation-lead"
    },
    {
      "id": "T5",
      "title": "Presentation: Image picker integration",
      "type": "presentation",
      "dependencies": ["T3", "T4"],
      "estimated_effort": 50,
      "owner": "presentation-lead"
    },
    {
      "id": "T6",
      "title": "Integration: DI wiring and module",
      "type": "integration",
      "dependencies": ["T2", "T3", "T4", "T5"],
      "estimated_effort": 40,
      "owner": "integration"
    }
  ],
  "execution_plan": [
    {
      "phase": 1,
      "parallel_tasks": ["T1"],
      "critical_path": true
    },
    {
      "phase": 2,
      "parallel_tasks": ["T2", "T3", "T4"],
      "critical_path": true
    },
    {
      "phase": 3,
      "parallel_tasks": ["T5"],
      "critical_path": true,
      "blocks_on": ["T3", "T4"]
    },
    {
      "phase": 4,
      "parallel_tasks": ["T6"],
      "critical_path": true,
      "blocks_on": ["T2", "T3", "T4", "T5"]
    }
  ],
  "critical_path_tasks": ["T1", "T2 or T4", "T5", "T6"],
  "total_effort_estimate": "hours: 350"
}
```

### Capability 4: Design Integration

Reads design files (Figma, XD, etc.) and maps design components to code components.

**Requires:** Design file link in task input

**Process:**
1. Fetch design file from Design MCP (Figma API)
2. Extract components (Button, TextField, Card, etc.)
3. Extract design tokens (colors, spacing, typography)
4. Identify reusable UI patterns
5. Map design components to SwiftUI View structure

**Output: design-map.md**

```markdown
# Design Map: Create Social Post

## Design Tokens

### Colors
- Primary: #007AFF (system blue)
- Background: #FFFFFF
- Border: #E5E5EA
- Error: #FF3B30
- Text Primary: #000000
- Text Secondary: #8E8E93

### Typography
- Heading 1: SF Pro Display 24pt bold (line height 28)
- Heading 2: SF Pro Display 20pt semibold (line height 24)
- Body: SF Pro Display 16pt regular (line height 20)
- Caption: SF Pro Display 12pt regular (line height 16)

### Spacing
- xs: 4pt
- sm: 8pt
- md: 16pt
- lg: 24pt
- xl: 32pt

### Corners & Shadows
- Border radius: 8pt (standard), 4pt (small)
- Shadow: 0 1px 3px rgba(0,0,0,0.12)

## Design Components

### PostComposerView Layout
```
├─ NavigationBar
│  ├─ "Cancel" button (left)
│  ├─ "Post" button (right, primary)
│  └─ Title: "New Post" (center)
├─ ScrollView
│  ├─ TextField (multi-line, placeholder "What's on your mind?")
│  │  └─ Character count (current/280)
│  ├─ Divider
│  ├─ ImageGrid (max 5 thumbnails)
│  │  └─ "Add Image" button (if < 5 images)
│  └─ Spacing: padding md (16pt)
└─ BottomActionBar (sticky)
   └─ "Post" button (full width, disabled if empty)
```

### Components to Implement in Code

| Design Component | Code Component | Notes |
|------------------|---|---|
| Button (Primary) | SwiftUI Button + .buttonStyle(.filledPrimary) | Reuse app button system |
| Button (Secondary) | SwiftUI Button + .buttonStyle(.plain) | |
| TextField | SwiftUI TextField + Custom modifier | Max 280 chars, live count |
| ImageGrid | SwiftUI LazyVGrid (2 columns) | Thumbnail size 80x80 |
| Divider | SwiftUI Divider | Standard spacing |
| NavigationBar | Custom NavigationBar component | Supports left/right actions |
| Card (Image) | SwiftUI ZStack with overlay | Show remove button on hover |

## Screen Layouts

### Composer Screen
- Safe area top: 16pt
- Text field: 100pt height (expands with text up to 200pt)
- Image grid: Dynamic height (grid of thumbnails)
- Bottom button: Sticky, 56pt height, md spacing

### Image Picker (Native)
- Standard iOS PHPickerViewController
- Multi-select enabled (max 5)
- Filtering: photos only

## Accessibility
- Text size: Supports Dynamic Type (min: small, max: xxxLarge)
- High contrast: Button text is white on blue (7:1 ratio)
- VoiceOver: Images have alt text, button labels clear
```

**Output: design-tokens.json**

```json
{
  "colors": {
    "primary": "#007AFF",
    "background": "#FFFFFF",
    "border": "#E5E5EA",
    "error": "#FF3B30"
  },
  "spacing": {
    "xs": 4,
    "sm": 8,
    "md": 16,
    "lg": 24,
    "xl": 32
  },
  "typography": {
    "heading1": {
      "font": "SF Pro Display",
      "size": 24,
      "weight": "bold",
      "lineHeight": 28
    }
  },
  "components": {
    "button_primary": {
      "backgroundColor": "#007AFF",
      "textColor": "#FFFFFF",
      "borderRadius": 8,
      "padding": [8, 16]
    }
  }
}
```

### Capability 5: Research

Gathers context and best practices to inform specifications.

**Triggered when:**
- Feature is unfamiliar (e.g., "Build a photo editor")
- API is new (e.g., "Use Vision framework for image recognition")
- HIG compliance matters (iOS design patterns)
- Competitor analysis helps (e.g., "How do other social apps handle posts?")

**Research sources:**
- Apple HIG (Human Interface Guidelines) — iOS design patterns
- Swift/iOS documentation (async/await, SwiftUI, framework APIs)
- API documentation (if integrating external API)
- Web search for best practices and examples
- Codebase history (how similar features were built before)

**Output: research-notes.md**

```markdown
# Research Notes: Create Social Post

## Apple HIG for Text Input
- Text input should allow up to 280 characters for social posts (Twitter standard)
- Multi-line text field should expand as user types
- Character count should be shown (e.g., "42/280")
- Primary action button (Post) should be prominent and enabled/disabled based on input

Reference: https://developer.apple.com/design/human-interface-guidelines/

## Image Handling Best Practices
- Use PHPickerViewController for modern photo library access (iOS 14+)
- Compress images before upload (max 5MB recommended)
- Show upload progress to user (ProgressView)
- Handle failures gracefully: retry without re-asking for image

Reference: https://developer.apple.com/documentation/photokit/phpickerviewcontroller

## Async/Await for Network
- Use URLSession with async/await (iOS 15+)
- Implement proper error handling (Result type or throws)
- Debounce rapid POST requests (500ms minimum)
- Implement exponential backoff for retries

Reference: https://developer.apple.com/documentation/foundation/urlsession

## Draft Auto-Save Pattern
- Save draft every 3 seconds (debounced)
- Use @State publisher to trigger saves
- Store drafts in UserDefaults or CoreData
- Load drafts on app launch

Reference: Combine framework documentation

## Competitor Analysis: Twitter, Instagram, Threads
- Twitter: 280 char limit, single media or carousel
- Instagram: Multi-step (photo → filter → caption → post)
- Threads: Similar to Twitter (300 char, images)
- Common pattern: Draft auto-save, image preview, error retry

Recommendation: Follow Twitter/Threads pattern (simpler UX)
```

## Product Spec Record

Each completed spec is stored as:

**Location:** `docs/specs/SPEC-NNN-feature-name.md`

**Example:** `docs/specs/SPEC-001-create-social-post.md`

**Format:**
```markdown
---
feature_id: SPEC-001
feature_name: Create Social Post
date_created: 2024-01-15
date_updated: 2024-01-15
status: approved
product_agent: claude
---

[Full spec content as produced above]

## Sign-Off

Approved by: [user or team lead]
Approved on: [date]
Feedback incorporated: [list of changes from review]
```

## Review Gate Detail

Product Agent outputs are gated by a review process. Configuration determines gate behavior.

**Config options:**

### review_gate: "manual" (default for teams)
- Product Agent submits spec to user/reviewer
- Reviewer can: Approve, Edit, Reject
- **Approve:** Spec sent to Code Gen immediately
- **Edit:** Feedback sent back to Product Agent; PA incorporates changes; resubmitted
- **Reject:** Spec rejected; PA re-researches or clarifies with user; new spec submitted

**Example flow:**
```
PA: Spec ready for review
User: Edit - "Can we make images optional?"
PA: Incorporates feedback, modifies AC4
PA: Resubmitted spec
User: Approve
→ Spec sent to Code Gen
```

### review_gate: "auto" (default for solo developers)
- Product Agent generates spec; auto-approved after sanity checks
- Checks: AC completeness, API contracts valid, no contradictions
- Spec immediately sent to Code Gen
- User can later request changes (triggers Path 7 PR Review)

### review_gate: "always" (strictest)
- Spec is reviewed multiple times: after decomposition, after spec, after design map
- Gate after each phase
- Typically for complex or high-risk features

## When Product Agent Is Invoked

**Coordinator routes to Product Agent if:**

1. Config: `stages.product_agent.enabled: always`
   - Every task triggers PA

2. Config: `stages.product_agent.enabled: per-task` AND
   - Spec assessment is one-liner OR
   - Spec assessment is partial OR
   - No AC provided OR
   - Design file present but no design-to-code map

3. Per-task override: Task specifies `--product-agent-on` or spec_quality flag

**Coordinator skips Product Agent if:**

1. Config: `stages.product_agent.enabled: never`
   - PA never runs

2. Spec assessment is well-specced (AC + design + API)
   - Complete specs skip PA

## Product Agent Outputs Summary

| Output | File | Purpose |
|--------|------|---------|
| Decomposition | `decomposition.md` | User stories, entities, flows, edge cases |
| Specification | `spec.md` | AC, API contracts, error handling |
| Task Graph | `task-graph.json` | Dependencies, execution order, effort estimates |
| Design Map | `design-map.md` | Components, tokens, layout, accessibility |
| Design Tokens | `design-tokens.json` | Colors, spacing, typography (code-ready) |
| Research Notes | `research-notes.md` | Best practices, competitor analysis, HIG guidance |
| Spec Record | `docs/specs/SPEC-NNN-*.md` | Permanent spec file (Git tracked) |

All outputs are plain text (markdown + JSON) and committed to Git in the project.
