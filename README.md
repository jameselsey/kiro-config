# kiro-config

Shared Kiro AI assistant configuration — steering files that encode project standards, conventions, and SOPs so every session starts with the right context.

## What's in here

```
steering/
├── android-standards.md     # Android project conventions (auto-injected in Android projects)
├── android-testing.md       # Testing conventions and known gotchas (auto-injected in Android projects)
└── android-new-app.md       # Step-by-step SOP for starting a new Android app (manual)
```

## How steering files work

Kiro reads steering files and injects their content into the AI's context automatically. There are three inclusion modes:

| Mode | Behaviour | Used for |
|------|-----------|----------|
| `always` | Injected in every session | Universal rules |
| `fileMatch` | Injected when a matching file exists in the workspace | Tech-stack-specific rules |
| `manual` | Only injected when you reference it with `#filename` in chat | SOPs, checklists |

The Android files use `fileMatchPattern: "**/*.gradle.kts"` — they only activate when Kiro detects a Gradle Kotlin DSL file, so they won't appear in Python, TypeScript, or other projects.

## Setup (one-time per machine)

Clone this repo and symlink the steering directory into Kiro's user config:

```bash
git clone https://github.com/jameselsey/kiro-config ~/dev/projects/kiro-config
ln -s ~/dev/projects/kiro-config/steering ~/.kiro/steering
```

Verify it worked:
```bash
ls ~/.kiro/steering
# should show: android-standards.md  android-testing.md  android-new-app.md
```

Kiro picks up changes immediately — no restart needed.

## Using the manual SOP

When starting a new Android app, pull in the setup checklist by typing in Kiro chat:

```
#android-new-app
```

This injects the full step-by-step SOP into the session context.

## Keeping it up to date

These files are living documentation. When you discover a new gotcha, fix a recurring mistake, or establish a new convention, update the relevant steering file and commit. Everyone who has cloned this repo gets the update on next `git pull`.

```bash
cd ~/dev/projects/kiro-config
git pull
```

## Contributing

If you're a developer on this team:
1. Clone and symlink as above
2. When you find something missing or wrong, open a PR
3. Keep entries specific and actionable — "always do X" or "never do Y", not vague principles
