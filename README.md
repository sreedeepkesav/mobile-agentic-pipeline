# Mobile Agentic Pipeline

AI-assisted development pipeline for iOS and Android — from idea to committed code, powered by Claude Code skills.

You describe what you want. The pipeline writes a spec, gets your approval, generates Clean Architecture code, tests it, and opens a PR. Two human checkpoints keep you in control.

## Install

```bash
# Pick your platform + variant:
cp -r skills/ios-full ~/.claude/skills/ios-full-pipeline      # iOS, full pipeline
cp -r skills/ios-lite ~/.claude/skills/ios-lite-pipeline      # iOS, lightweight
cp -r skills/android-full ~/.claude/skills/android-full-pipeline  # Android, full pipeline
cp -r skills/android-lite ~/.claude/skills/android-lite-pipeline  # Android, lightweight
```

## Use

Name the pipeline in your prompt:

```
Use ios-full-pipeline to add a dark mode toggle in Settings.
```

The pipeline runs end-to-end:

```
Spec → ⏸ You approve → Code (4-phase team) → Test + Lint → Build → PR → ⏸ You merge
```

You don't have to run everything. Ask for just the parts you need:

```
Use ios-full-pipeline — just spec out the payment flow, no code yet.
Use android-full-pipeline — run tests only.
Use ios-full-pipeline — build and ship to TestFlight.
```

See the [full usage guide](docs/usage-guide.md) for all scenarios, single-agent usage, invocation methods, and configuration options.

## Full vs Lite

| Full | Lite |
|------|------|
| Bootstrap auto-configures your environment | Zero config, just start |
| Coordinator routes bugs/features/refactors differently | Single flow for everything |
| Pipeline Memory learns across runs | No memory |
| Configurable stages (skip test, skip deploy) | All stages always run |
| Sprint batch (parallel task execution) | One task at a time |
| Single-agent runs (just test, just lint, just deploy) | Always end-to-end |

Both use the same 4-phase Code Gen team and produce the same quality code.

## Architecture

Interactive diagrams — open in a browser:

| Platform | Full | Lite |
|----------|------|------|
| **iOS** (Swift, SwiftUI, MVVM-C) | [View](architecture/ios/full-pipeline.html) | [View](architecture/ios/lite-pipeline.html) |
| **Android** (Kotlin, Compose, MVVM) | [View](architecture/android/full-pipeline.html) | [View](architecture/android/lite-pipeline.html) |

## Roadmap

- [x] Architecture diagrams
- [x] Claude Code skills (iOS Full + Lite, Android Full + Lite)
- [ ] Standalone CLI
- [ ] KMP shared module support

## License

MIT
