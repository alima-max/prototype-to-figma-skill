# Prototype → Figma

A Claude skill that converts working Claude Code prototypes into structured Figma design files — exploding each interaction flow into separate frames, using your design system components, and annotating interaction details natively in Figma.

## Why this exists

When you build a prototype in Claude Code, getting async feedback from cross-functional partners (PMs, designers, engineers) is hard. They can't easily walk through a live prototype on their own. This skill bridges that gap by translating your prototype into a Figma file that reviewers can browse, comment on, and understand without running anything.

## What it produces

- **One frame per interaction state** — every meaningful step in a user flow gets its own frame
- **Design system components** — searches your Figma file's linked libraries and uses real DS components instead of drawing rectangles that look like buttons
- **Interaction annotations** — color-coded callouts explaining triggers, transitions, conditions, and edge cases
- **Flow arrows** — visual connectors showing the path a user takes through states
- **Overview frame** — table of contents with a legend and open questions for reviewers
- **Code Connect mappings** — optionally links Figma components back to your codebase

## Requirements

- **Figma MCP tools** — this skill uses `use_figma`, `search_design_system`, `get_design_context`, `get_metadata`, `get_screenshot`, `get_code_connect_map`, `get_context_for_code_connect`, `get_code_connect_suggestions`, `send_code_connect_mappings`, `add_code_connect_map`, `whoami`, and `create_new_file`
- **A working prototype** — built in Claude Code, typically using your team's actual component library
- **A target Figma file** — either an existing file or the skill will create a new one

## Installation

### Claude.ai (Team / Enterprise)
1. Download or clone this repo
2. Zip the `prototype-to-figma/` folder (containing `SKILL.md` and `references/`)
3. Go to **Settings → Customize → Skills → "+" → "+ Create skill"**
4. Upload the zip
5. Share with your org via the skill sharing feature

### Claude Code
Drop the `prototype-to-figma/` folder into `~/.claude/skills/`, or install via:
```bash
# If published as a plugin
/plugin install prototype-to-figma@your-org/your-repo
```

### Claude API
Upload via the `/v1/skills` endpoint. See [Skills API docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) for details.

## Usage

Once installed, just ask Claude naturally:

- *"Take this prototype and put it in Figma so the team can review it"*
- *"Explode this prototype into Figma frames for async feedback"*
- *"Create Figma specs from my prototype for design review"*
- *"Make this prototype reviewable by the design team"*

Claude will ask for the Figma file URL (or create a new file), analyze the prototype code, and build the frames.

## How it works

1. **Analyzes the prototype** — reads the source code to inventory components, map interaction flows, and identify states
2. **Maps to the design system** — searches the target Figma file's linked libraries for matching components via `search_design_system`, and checks Code Connect mappings for precise code↔design links
3. **Plans the page structure** — organizes frames by flow, with branching paths for success/error states
4. **Builds in Figma** — creates frames, imports and configures DS component instances, builds scratch elements for unmatched components
5. **Annotates interactions** — adds color-coded callouts (yellow = triggers, green = success, red = errors, blue = flow arrows) and a legend
6. **Verifies and presents** — screenshots the output, optionally creates Code Connect mappings, and summarizes what was built

## Customization

This skill was designed for teams whose prototypes use their actual shared component library. If your setup is different, you may want to fork and adjust:

- **Component matching logic** — if your code and Figma use very different naming conventions
- **Frame sizes** — currently defaults to 1440×900 (desktop) or 390×844 (mobile)
- **Annotation style** — colors, fonts, and callout format are defined in `references/figma-patterns.md`
- **Flow granularity** — the skill groups micro-interactions into single annotated frames by default; you might want more or fewer frames per flow

## License

MIT
