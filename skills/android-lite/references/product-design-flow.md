# Product + Design Flow for Android Lite

## Overview

The Product + Design Agent (30 minutes) researches, plans, and designs your feature or app before any code is written. It produces design artifacts that guide the 4-phase Coder.

---

## Research Phase

The Product + Design Agent conducts focused research on:

### Material Design 3 & Compose
- Jetpack Compose API (Column, Row, Box, LazyColumn, Button, TextField, etc.)
- Material Design 3 color system (primary, secondary, tertiary, error, surface)
- Typography and spacing scales
- Component guidelines (buttons, cards, dialogs, sheets, chips)
- Theme customization with Material3 theming

### Similar App Patterns
- Competitor apps on Google Play Store
- Common navigation patterns (bottom nav, drawer, nested navigation)
- List and detail screen layouts
- Form patterns (input validation, error messages)
- Empty state, loading state, error state UX

### Android Architecture
- Clean Architecture principles (domain, data, presentation layers)
- MVVM and StateFlow/UiState patterns
- Coroutines and Flow for async operations
- Hilt dependency injection
- Navigation Compose type-safe routing

### API & Data Patterns
- RESTful API design (GET, POST, PUT, DELETE)
- Pagination and caching strategies
- Error handling and retry logic
- Request/response serialization (Kotlinx.serialization)
- Rate limiting and throttling

---

## Planning Phase

### 1. Spec with Acceptance Criteria

```markdown
# Feature: Recipe Search

## Description
Allow users to search the recipe database by name, cuisine, or ingredients.

## User Stories

### US1: Basic Search
As a user, I want to enter a search term and see matching recipes.
- AC1: Search field accepts text input
- AC2: Submit button triggers search
- AC3: Results display within 2 seconds
- AC4: Empty results show "No recipes found"

### US2: Search History
As a user, I want to see recent searches for quick access.
- AC1: Last 5 searches stored locally
- AC2: Tap a recent search to rerun it
- AC3: Clear search history button available

## Edge Cases
- Search with 0-2 characters: reject, show "Minimum 3 characters"
- Search with special characters: sanitize or reject
- Network timeout: show "Connection failed, try again"
- Empty database: show helpful message

## Constraints
- Search API has 100 req/min rate limit
- Response payload < 5 MB
- Support offline mode (cached results)
```

### 2. Screens & Navigation Graph

```markdown
# Screen List

## RecipeListScreen
- **Description**: Shows all recipes or filtered results
- **Components**:
  - SearchBar (text input + filter button)
  - LazyColumn of RecipeCards
  - FAB for new recipe
  - LoadingIndicator, ErrorMessage, EmptyState

## RecipeDetailScreen
- **Description**: Shows recipe details, ingredients, instructions
- **Components**:
  - Hero image
  - Recipe title, rating, servings
  - TabRow (Ingredients, Instructions, Nutrition)
  - Share button, Favorite button
  - Related recipes carousel

## Navigation Graph
```
RecipeListScreen (start)
  ├─ navigate to RecipeDetailScreen(recipeId: String)
  └─ navigate to SearchHistoryScreen

RecipeDetailScreen
  └─ navigateUp to RecipeListScreen

SearchHistoryScreen
  └─ navigate to RecipeListScreen with query
```

### 3. API Contracts

```markdown
# API Specifications

## GET /recipes/search
Query: q (string, min 3 chars)
Response: 200 OK
{
  "recipes": [
    {
      "id": "string",
      "name": "string",
      "imageUrl": "string",
      "cuisine": "string",
      "rating": float,
      "servings": int
    }
  ],
  "total": int
}

Error Responses:
- 400: Invalid query
- 429: Rate limit exceeded
- 500: Server error

## GET /recipes/{id}
Response: 200 OK
{
  "id": "string",
  "name": "string",
  "imageUrl": "string",
  "ingredients": [
    {
      "name": "string",
      "amount": float,
      "unit": "string"
    }
  ],
  "instructions": [
    {
      "step": int,
      "description": "string"
    }
  ]
}
```

### 4. Data Models

```markdown
# Domain Entities

## Recipe
- id: String (unique identifier)
- name: String (non-empty)
- ingredients: List<Ingredient>
- instructions: List<Instruction>
- cuisine: String
- servings: Int (>= 1)
- prepTime: Int (minutes)
- cookTime: Int (minutes)
- difficulty: Difficulty (enum: Easy, Medium, Hard)

## Ingredient
- name: String
- amount: Float
- unit: String (cup, tsp, g, etc.)

## Instruction
- step: Int (1-based)
- description: String

## SearchQuery
- id: String (timestamp or UUID)
- text: String
- timestamp: Long
```

### 5. Design Tokens

```json
{
  "colors": {
    "primary": "#6750A4",
    "onPrimary": "#FFFFFF",
    "secondary": "#625B71",
    "tertiary": "#7D5260",
    "error": "#B3261E",
    "surface": "#FFFBFE",
    "surfaceVariant": "#E7E0EC"
  },
  "spacing": {
    "xs": 4,
    "sm": 8,
    "md": 16,
    "lg": 24,
    "xl": 32
  },
  "typography": {
    "displayLarge": { "size": 57, "weight": 400 },
    "titleLarge": { "size": 22, "weight": 500 },
    "bodyMedium": { "size": 14, "weight": 500 },
    "labelSmall": { "size": 12, "weight": 500 }
  }
}
```

### 6. Edge Cases & Error Handling

```markdown
# Error Scenarios

## Network Errors
- Timeout (5 sec): Show "Connection timeout. Try again?"
- No internet: Show "No network. Using cached results."
- 429 Rate Limited: Show "Too many requests. Wait and try again."
- 500 Server Error: Show "Server error. Our team is investigating."

## Input Validation
- Empty search: Disable submit button
- < 3 chars: Show inline "Min 3 characters"
- Special chars only: Show "Invalid search"
- SQL injection attempt: Sanitize and search normally

## Empty States
- No recipes in database: Show "No recipes yet"
- No search results: Show "No recipes match 'xyz'. Try another search"
- No search history: Show "No recent searches"

## Data Consistency
- Recipe deleted after loading detail: Show "Recipe no longer available"
- Image broken link: Show placeholder image
- Missing ingredient data: Show "Data incomplete"
```

---

## Design Phase

### Material Design 3 Wireframes

```
RecipeListScreen
┌─────────────────────────────────────┐
│ 🔍 Search recipes...          ⋯    │  ← SearchBar + filter button
├─────────────────────────────────────┤
│ 🥗 Margherita Pasta    4.5★ 2 servings
├─────────────────────────────────────┤
│ 🍝 Carbonara            4.8★ 4 servings
├─────────────────────────────────────┤
│ 🥘 Ratatouille         4.2★ 6 servings
├─────────────────────────────────────┤
│ [✥ New Recipe]                      │  ← FAB
└─────────────────────────────────────┘
```

### Component Styles

- **SearchBar**: Outlined text field with leading icon, max width 100%
- **RecipeCard**: Card with image, title, rating, servings; clickable
- **FAB**: Floating action button (Material Design 3 style)
- **LoadingIndicator**: Center-screen circular progress
- **ErrorMessage**: Snackbar with dismiss and retry buttons
- **EmptyState**: Centered icon + message + optional action

### Color Application
- Primary (`#6750A4`): Buttons, FAB, selected states
- Secondary (`#625B71`): Chip filters, minor actions
- Tertiary (`#7D5260`): Accents, rating stars
- Error (`#B3261E`): Error messages, validation
- Surface (`#FFFBFE`): Backgrounds, cards

### Spacing & Padding
- Screen edges: 16dp (md)
- Card padding: 16dp (md)
- Button height: 48dp
- FAB size: 56dp
- List item padding: 16dp vertical, 8dp horizontal

---

## Output Artifacts

The Product + Design Agent produces:

1. **spec.md** — Feature spec with user stories and AC
2. **screens.md** — Screen list, wireframes, navigation graph
3. **api-contracts.md** — API endpoints, payloads, errors
4. **data-models.md** — Domain entities with relationships
5. **design-tokens.json** — Colors, spacing, typography
6. **edge-cases.md** — Error scenarios, validation rules

---

## Review Gate #1

After Product + Design completes, all artifacts are presented for approval:

- **Approve**: Proceed to Coder (Phase 1: Architect)
- **Edit**: Request changes (clarify AC, adjust scope, add screens)
- **Reject**: Return to research phase with feedback

No code is written until approved.
