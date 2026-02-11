# Mobile Agentic Pipeline

AI-assisted development pipeline for iOS and Android — from idea to committed code, powered by Claude Code skills.

You describe what you want. The pipeline writes a spec, gets your approval, generates Clean Architecture code, tests it, and opens a PR. Two human checkpoints keep you in control.

**[Usage Guide](docs/usage-guide.md)** — all examples, single-agent scenarios, invocation methods, configuration.

## Install

```bash
# Pick your platform + variant:
cp -r skills/ios-builder ~/.claude/skills/ios-builder            # iOS
cp -r skills/ios-builder-lite ~/.claude/skills/ios-builder-lite  # iOS, lightweight
cp -r skills/android-builder ~/.claude/skills/android-builder            # Android
cp -r skills/android-builder-lite ~/.claude/skills/android-builder-lite  # Android, lightweight
```

## Use

Name the pipeline in your prompt:

```
Use ios-builder to add a dark mode toggle in Settings.
```

The pipeline runs end-to-end:

```
Spec → ⏸ You approve → Code (4-phase team) → Test + Lint → Build → PR → ⏸ You merge
```

You don't have to run everything. Ask for just the parts you need:

```
Use ios-builder — just spec out the payment flow, no code yet.
Use android-builder — run tests only.
Use ios-builder — build and ship to TestFlight.
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

Interactive diagrams — click to view:

| Platform | Full | Lite |
|----------|------|------|
| **iOS** (Swift, SwiftUI, MVVM-C) | [View](https://sreedeepkesav.github.io/mobile-agentic-pipeline/ios/full-pipeline.html) | [View](https://sreedeepkesav.github.io/mobile-agentic-pipeline/ios/lite-pipeline.html) |
| **Android** (Kotlin, Compose, MVVM) | [View](https://sreedeepkesav.github.io/mobile-agentic-pipeline/android/full-pipeline.html) | [View](https://sreedeepkesav.github.io/mobile-agentic-pipeline/android/lite-pipeline.html) |

## Roadmap

- [x] Architecture diagrams
- [x] Claude Code skills (iOS Full + Lite, Android Full + Lite)
- [ ] Standalone CLI
- [ ] KMP shared module support

## License

MIT
