# Prototype → Figma

A Claude skill that converts working Claude Code prototypes into structured Figma design files — exploding each interaction flow into separate frames, using your design system components, and annotating interaction details natively in Figma.

## Why this exists

When you build a prototype in Claude Code, getting async feedback from cross-functional partners (PMs, designers, engineers) is hard. They can't easily walk through a live prototype on their own. This skill bridges that gap by translating your prototype into a Figma file that reviewers can browse, comment on, and understand without running anything.

## What it produces

- **One frame per interaction state** — every meaningful step in a user flow gets its own frame
- **Design system components** — uses real DS components from your Figma file's linked libraries; elements with no DS match are built from primitives and flagged with a "No DS match" badge
- **Native Figma annotations** — Dev Mode annotations with filterable categories (Interaction, Navigation, Validation, Error Handling, etc.) explaining triggers, transitions, conditions, and edge cases
- **Flow arrows** — visual connectors showing the path a user takes through states
- **Overview frame** — table of contents with a legend and open questions for reviewers
- **Code Connect mappings** — optionally links Figma components back to your codebase

## Requirements

- **Figma MCP tools** — this skill uses `use_figma`, `search_design_system`, `get_design_context`, `get_metadata`, `get_screenshot`, `get_code_connect_map`, `get_context_for_code_connect`, `get_code_connect_suggestions`, `send_code_connect_mappings`, `add_code_connect_map`, `whoami`, and `create_new_file`
- **A working prototype** — built in Claude Code, typically using your team's actual component library
- **A target Figma file** — optional; if you don't provide one, the skill creates a new file automatically

## Installation

### Claude.ai (Team / Enterprise)
1. Download the latest zip from the [Releases](../../releases) page (or clone this repo and zip the root folder)
2. Go to **Settings → Customize → Skills → "+" → "+ Create skill"**
3. Upload the zip
4. Share with your org via the skill sharing feature

### Claude Code
```bash
git clone https://github.com/alima-max/prototype-to-figma-skill.git ~/.claude/skills/prototype-to-figma
```

### Other clients (Cursor, VS Code, Copilot CLI, etc.)
Add `SKILL.md` and `figma-patterns.md` to your client's skills or context directory. Refer to your client's documentation for the exact location.

### Claude API
Upload via the `/v1/skills` endpoint. See [Skills API docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) for details.

## Usage

Once installed, just ask Claude naturally — no Figma URL required:

- *"Take this prototype and put it in Figma so the team can review it"*
- *"Explode this prototype into Figma frames for async feedback"*
- *"Create Figma specs from my prototype for design review"*
- *"Make this prototype reviewable by the design team"*

If you want the output in a specific existing Figma file, include the URL:
- *"Put this in Figma: figma.com/design/abc123/..."*

Otherwise, Claude will create a new Figma file automatically and share the link when done.

## How it works

1. **Resolves the target file** — uses the file URL you provide, or creates a new Figma file automatically via `whoami` + `create_new_file`
2. **Analyzes the prototype** — reads the source code to inventory components, map interaction flows, and identify all UI states
3. **Maps to the design system** — searches the target Figma file's linked libraries for matching DS components; unmatched elements are built from primitives and flagged with a "No DS match" badge
4. **Plans the page structure** — organizes frames by flow, with branching paths for success/error states
5. **Builds in Figma** — creates frames, imports and configures DS component instances; never calls `createComponent()` or skips elements
6. **Annotates interactions** — adds native Figma annotations with filterable categories and a legend
7. **Verifies and presents** — screenshots the output, optionally creates Code Connect mappings, and shares the file URL with a summary

## Updating

When a new version is released, you'll need to update your installation manually. No re-authentication required — it's just replacing files.

### Claude.ai (Team / Enterprise)
1. Download the latest zip from the [Releases](../../releases) page
2. Go to **Settings → Customize → Skills**, find "Prototype → Figma", and re-upload the new zip

### Claude Code
```bash
cd ~/.claude/skills/prototype-to-figma
git pull
```

### Other clients
Replace `SKILL.md` and `figma-patterns.md` with the latest versions from this repo.

To see what changed, check the [commit history](../../commits/main) or [releases](../../releases).

## Compatibility

This skill works across all Figma MCP clients, but the output varies based on what tools your client supports:

| Client | Builds in Figma | Code Connect |
|---|---|---|
| Claude Code, Claude Desktop, Cursor, VS Code, Copilot CLI, Augment Code, Factory, Firebender, Codex | ✅ | ✅ |
| Android Studio, Gemini CLI, Kiro, Amazon Q, Openhands | ❌ (produces Prototype Spec Document instead) | ❌ |
| Replit | ✅ | ❌ |

See `SKILL.md` for full details on what each tier produces.

## Customization

This skill was designed for teams whose prototypes use their actual shared component library. If your setup is different, you may want to fork and adjust:

- **Component matching logic** — if your code and Figma use very different naming conventions
- **Frame sizes** — defaults to 1440×900 (desktop) or 390×844 (mobile)
- **Primitive fallback patterns** — `figma-patterns.md` contains helpers for Button, Input, Card, and Banner; add your own for other component types
- **Flow granularity** — the skill groups micro-interactions into single annotated frames by default; you might want more or fewer frames per flow

## License

MIT
