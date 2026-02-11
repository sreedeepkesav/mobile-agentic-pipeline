# Bootstrap and Configuration Guide

Complete reference for the Bootstrap stage of the ios-full-pipeline skill. Covers context detection, MCP discovery, spec assessment, project state detection, and stage configuration.

## Overview

Bootstrap runs once on first execution (or when --reconfigure is passed). It accomplishes 5 sequential steps that configure the entire pipeline for your project, establishing context, detecting available tools, assessing requirements, and determining the project's current state.

## Step 1: Context Detection

Determines the work context via free-form string input (not enum). The Coordinator uses context to adjust default stage activation and routing behavior.

**Question Asked:**
```
Quick setup: what's the context?
(e.g., "We're a startup with 3 developers building a product",
"I'm working on this solo", "I'm a contractor building client features")
```

### Context Patterns and Implications

#### Work Context
- **Indicators:** "We're a team", "Product team", "5 developers", "Product + Design + Backend", etc.
- **Interpretation:** Multiple people, established team structure, designs typically provided, specs vary
- **Product Agent:** Per-task. Activated for features/design work. Skipped if well-specced ticket.
- **Default Stage Config:** All stages active (Product, Scaffold, Code Gen, Test, Lint, Build, Deploy)
- **Routing Implication:** Team members are layer leads; Coordinator manages parallel execution and code ownership

#### Personal/Solo Context
- **Indicators:** "Working solo", "Just me", "Personal project", "Hobby", etc.
- **Interpretation:** Single developer, rough ideas, specs are minimal or non-existent, full ownership
- **Product Agent:** Always active. Solo devs need structured specs and designs before coding.
- **Default Stage Config:** Product always active; Scaffold, Test, Lint always active; Build/Deploy per-task
- **Routing Implication:** All layers owned by one developer; focus on clarity and quality gates

#### Freelance/Contract Context
- **Indicators:** "Client contract", "Freelance", "Building for client", "Contractor", etc.
- **Interpretation:** External specs/requirements vary; may be incomplete; delivery-focused
- **Product Agent:** Per-task. Activated when specs are partial or missing. Skipped if client provides detailed specs.
- **Default Stage Config:** All stages active; may skip Deploy if client self-deploys
- **Routing Implication:** Specs may come from client; clarification rounds are expected

#### Open Source Context
- **Indicators:** "Community project", "Open source", "Public repo", "PRs from contributors", etc.
- **Interpretation:** Specs come via issues/PRs; community involvement; decentralized
- **Product Agent:** Per-task. Activated to formalize issues and PRs into executable specs.
- **Default Stage Config:** Product, Code Gen, Test, Lint, Build active; Deploy may not apply; Scaffold per-task
- **Routing Implication:** Task source is issues/PR tracker; contribution quality gates apply

## Step 2: MCP Discovery

Scans the environment for active MCPs (Model Context Protocols) that provide task, design, and code sources.

**Discovery Process:**
1. Searches environment for task/project management MCPs (Asana, Jira, Linear, GitHub Issues, etc.)
2. Searches environment for design tool MCPs (Figma, Adobe XD, Sketch, etc.)
3. Searches environment for code platform MCPs (GitHub, GitLab, Bitbucket, etc.)
4. If no MCPs found, falls back to local `backlog.md`

**Output Example:**
```yaml
active_mcps:
  - name: github
    type: code_source
    capabilities: [pull_requests, issues, code_review, commits]
  - name: figma
    type: design_source
    capabilities: [components, prototypes, design_tokens, export]
  - name: linear
    type: task_source
    capabilities: [issues, epics, sprints, backlogs]
fallback:
  source: local
  file: backlog.md
```

**Usage:** Coordinator uses MCPs to fetch tasks, designs, and code context. Code Gen pulls design tokens from Figma. Product Agent reads issues/specs from task tracker.

## Step 3: Spec Quality Assessment

Evaluates how well-specified the feature/task is. Determines whether Product Agent is bypassed or activated.

### Assessment Levels

#### Well-Specced
- **Criteria:** Contains acceptance criteria (AC) + design files + API contracts
- **Example:**
  ```
  AC:
  - User can tap "Create Post" to open composer
  - Composer has text field, photo button, post button
  - POST /posts accepts {title, body, images}

  Design: [Figma link]
  API: [OpenAPI spec or contract]
  ```
- **Product Agent:** Skipped. Code Gen starts immediately.
- **Reasoning:** Clear requirements eliminate ambiguity; spec clarity drives execution speed

#### Partial Spec
- **Criteria:** Contains some detail (e.g., AC or design or API) but not all
- **Example:**
  ```
  "Add dark mode toggle to Settings"
  (has design link, but no AC or API contract)
  ```
- **Product Agent:** Activated. Fills gaps: formalizes AC, extracts API needs, details screen flows.
- **Output:** Augmented spec.md with missing sections

#### One-Liner
- **Criteria:** Task title only; no context, design, or spec
- **Example:** "Fix login crash"
- **Product Agent:** Activated (even for bugs). Clarifies issue, determines scope, proposes solution.
- **Output:** spec.md with reproduction steps, root cause hypothesis, fix scope

### Per-Task Override

At the Coordinator level, any task can override default spec assessment:

```yaml
# In task tracker or via --spec-level flag
spec_quality: well_specced  # override: product agent skips
spec_quality: partial       # override: product agent fills gaps
spec_quality: one_liner     # override: product agent fully active
```

## Step 4: Project State Detection

Scans the project directory to understand existing architecture and Swift/Xcode setup.

**Detection Process:**

1. **Source Presence:** Checks for `*.swift` files
2. **Package Management:** Detects `Package.swift` (SPM) or `Podfile` (CocoaPods)
3. **Build Setup:** Looks for `.xcodeproj` or `.xcworkspace`
4. **Folder Structure:** Identifies established patterns:
   - `Domain/` → Domain layer exists
   - `Data/` → Data layer exists
   - `Presentation/` → Presentation layer exists
   - `Network/` → Separate networking module
   - `Coordinator/` or `Navigation/` → Navigation layer
5. **Architecture Inference:** Detects MVVM, MVC, or layered architecture

**Output Example (Existing Project):**
```yaml
project_state:
  swift_present: true
  package_manager: spm
  build_tool: xcodebuild
  folder_structure:
    - Domain
    - Data
    - Presentation
    - Network
  detected_architecture: layered_mvvm_c
  existing_patterns:
    - async_await_preferred
    - dependency_injection_enabled
    - generic_error_handling
```

**Output Example (No Source):**
```yaml
project_state:
  swift_present: false
  package_manager: none
  build_tool: none
  detected_architecture: none
  scaffold_needed: true
```

**Impact:**
- **Existing project:** Code Gen inherits architecture, patterns, and conventions
- **No source:** Scaffold stage creates initial structure and folders

## Step 5: Stage Configuration

Determines which pipeline stages are active, per-task, or never.

### Always Active
- **Coordinator:** Routes tasks, manages feedback
- **Code Gen:** Generates or modifies code
- **Git:** Commits changes, creates branches

### Configurable
- **Product Agent:** `always`, `never`, `per-task` (default: per-task based on spec quality)
- **Scaffold:** `always`, `never`, `per-task` (default: per-task if no source detected)
- **Test:** `always`, `never`, `per-task` (default: per-task)
- **Lint:** `always`, `never`, `per-task` (default: per-task)
- **Build:** `always`, `never`, `per-task` (default: per-task)
- **Deploy:** `always`, `never`, `per-task` (default: never)

### Full .pipeline.yml Schema

```yaml
version: "1.0"
project:
  name: MyApp
  context: work  # or: personal, freelance, open_source

bootstrap:
  context_auto_detect: true
  mcp_discovery: true
  spec_assessment: true
  project_state_detection: true

stages:
  coordinator:
    enabled: true

  product_agent:
    enabled: per-task  # or: always, never
    config:
      review_gate: manual  # or: always, auto
      research_enabled: true
      design_integration: true

  scaffold:
    enabled: per-task  # or: always, never
    config:
      folder_structure: layered
      swift_version: "5.9"
      minimum_deployment_target: "14.0"

  code_gen:
    enabled: true
    config:
      architecture: layered_mvvm_c
      di_strategy: assembly_pattern
      error_handling: custom_errors
      async_strategy: async_await

  test:
    enabled: per-task  # or: always, never
    config:
      frameworks: [xctest]
      coverage_minimum: 0.75

  lint:
    enabled: per-task  # or: always, never
    config:
      rules: swiftlint

  build:
    enabled: per-task  # or: always, never
    config:
      scheme: MyApp

  deploy:
    enabled: never  # or: always, per-task
    config:
      targets: [testflight]

  git:
    enabled: true
    config:
      auto_commit: true
      branch_strategy: feature-branches

memory:
  enabled: true
  persistence: local  # or: git_committed
```

### Example: Work Context Config

```yaml
project:
  name: ProductTeamApp
  context: work

stages:
  product_agent:
    enabled: per-task
    config:
      review_gate: manual

  scaffold:
    enabled: never  # existing project

  test:
    enabled: per-task

  lint:
    enabled: per-task

  build:
    enabled: per-task

  deploy:
    enabled: per-task  # TestFlight for team
```

### Example: Personal/Solo Context Config

```yaml
project:
  name: SideProject
  context: personal

stages:
  product_agent:
    enabled: always  # always need specs
    config:
      review_gate: auto

  scaffold:
    enabled: per-task

  test:
    enabled: always  # solo dev → strict quality

  lint:
    enabled: always

  build:
    enabled: per-task

  deploy:
    enabled: per-task  # maybe AppStore
```

### Example: Freelance/Contract Config

```yaml
project:
  name: ClientApp
  context: freelance

stages:
  product_agent:
    enabled: per-task  # client specs vary
    config:
      review_gate: manual

  test:
    enabled: per-task

  lint:
    enabled: per-task

  build:
    enabled: always  # testable builds for client

  deploy:
    enabled: never  # client handles deployment
```

### Example: Open Source Context Config

```yaml
project:
  name: OpenSourceLib
  context: open_source

stages:
  product_agent:
    enabled: per-task  # issues/PRs as specs
    config:
      review_gate: auto

  test:
    enabled: always  # community quality standards

  lint:
    enabled: always

  build:
    enabled: always

  deploy:
    enabled: never  # GitHub Releases separate
```

## Initial Interactive Questions

Bootstrap asks these questions in order. Auto-detected values are prefilled; undetected values require input.

```
1. Quick setup: what's the context?
   [Text input] → Classify as work/personal/freelance/open_source

2. MCPs detected: [List found MCPs]
   Continue? (y/n)

3. Project structure detected: [Shows detected architecture]
   Does this match your setup? (y/n)
   If no: Describe your structure briefly

4. Which stages should always run?
   - Product Agent: [auto | manual | per-task] (default: per-task)
   - Scaffold: [always | never | per-task] (default: per-task)
   - Test: [always | never | per-task] (default: per-task)
   - Lint: [always | never | per-task] (default: per-task)
   - Build: [always | never | per-task] (default: per-task)
   - Deploy: [always | never | per-task] (default: never)

5. Review gates for Product Agent?
   - Auto-approve specs: (y/n) (default: n for work, y for solo)

6. Save configuration?
   - Save to .pipeline.yml: (y/n) (default: y)
```

## Reconfigure Behavior

**Full Reconfigure:**
```bash
ios-full --reconfigure
```
Re-runs all 5 bootstrap steps. Updates `.pipeline.yml`.

**Single Step Reconfigure:**
```bash
ios-full --reconfigure context
# Re-run only context detection
```

**Other Single Steps:**
```bash
ios-full --reconfigure mcp_discovery
ios-full --reconfigure spec_assessment
ios-full --reconfigure project_state
ios-full --reconfigure stages
```

## Output Artifacts

After Bootstrap completes:

1. `.pipeline.yml` — Configuration file (committed to git for team; `.gitignored` for solo/freelance)
2. `.pipeline/bootstrap/` — Log of bootstrap execution (metadata, detected values)
3. `docs/PROJECT_SETUP.md` — Human-readable project setup summary (for onboarding)
