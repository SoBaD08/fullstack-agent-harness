# Installation & Setup Guide

## 1. File Structure Overview

```
your-project/
├── .claude/
│   ├── CLAUDE.md                          # Project rules (auto-loaded)
│   ├── settings.json                      # Permissions + hooks config
│   ├── skills/
│   │   ├── plan/SKILL.md                  # /plan command
│   │   ├── build/SKILL.md                 # /build command
│   │   ├── evaluate/SKILL.md              # /evaluate command
│   │   └── orchestrate/SKILL.md           # /orchestrate command
│   └── hooks/scripts/
│       ├── protect-files.sh               # Block protected file edits
│       └── auto-format.sh                 # Auto-format on save
├── .harness/                              # Agent communication (created at runtime)
│   ├── spec.md
│   ├── contract.md
│   ├── build-report.md
│   ├── evaluation-report.md
│   └── sprints/
├── ARCHITECTURE.md                        # Architecture design document
└── src/                                   # Actual source code
```

---

## 2. Installation

### Step 1: Copy Files
Copy the `.claude/` directory to your project root:

```bash
# From your project root
cp -r /path/to/fullstack-agent-harness/.claude .
```

### Step 2: Make Hook Scripts Executable
```bash
chmod +x .claude/hooks/scripts/*.sh
```

### Step 3: Verify jq is Installed (used by hooks)
```bash
# macOS
brew install jq

# Ubuntu/Debian
sudo apt-get install jq

# Windows (scoop)
scoop install jq
```

### Step 4: Add to .gitignore
```gitignore
# Add to .gitignore
.harness/
.claude/settings.local.json
```

---

## 3. VS Code Configuration for Claude Code

### 3-1. Install the Claude Code Extension
Install the `Claude Code` extension in VS Code.
(Skip if already installed)

### 3-2. Recommended VS Code Settings

`Ctrl+Shift+P` → `Preferences: Open User Settings (JSON)`:

```jsonc
{
  // Claude Code extension settings
  "claudeCode.terminalUseCtrlEnterToSend": true,   // Send with Ctrl+Enter

  // Terminal profiles (for Claude Code CLI)
  "terminal.integrated.profiles.windows": {
    "Claude Code": {
      "path": "cmd",
      "args": ["/k", "claude"]
    }
  },
  "terminal.integrated.profiles.osx": {
    "Claude Code": {
      "path": "/bin/zsh",
      "args": ["-c", "claude"]
    }
  }
}
```

### 3-3. Open the Claude Code Sidebar
- `Ctrl+Shift+P` → `Claude Code: Open in Side Panel`, or
- Click the Claude icon in the left sidebar

---

## 4. Usage

### Quick Start: Using /orchestrate
Run the entire workflow at once:

```
/orchestrate Implement a user authentication system with JWT.
Need login, signup, and password reset functionality.
```

### Step-by-Step Manual Execution

#### Step 1: Plan
```
/plan Implement user auth system. JWT-based login/signup + password reset
```
→ Generates `.harness/spec.md` → Review before proceeding

#### Step 2: Build
```
/build sprint-1
```
→ Implements Sprint 1 + writes `.harness/build-report.md`

#### Step 3: Evaluate
```
/evaluate sprint-1
```
→ Independent verification + writes `.harness/evaluation-report.md`

#### Step 4: Iterate
If FAIL → incorporate feedback and re-run `/build sprint-1`.
If PASS → proceed with `/build sprint-2`.

### CLI Pipeline Execution (Terminal)
```bash
# Sequential Plan → Build → Evaluate
claude -p "/plan user auth system" && \
claude -p "/build sprint-1" && \
claude -p "/evaluate sprint-1"
```

### Fully Autonomous Mode: /auto-orchestrate
Let the agents run the entire loop without intervention:

```
/auto-orchestrate Implement a user authentication system with JWT.
Need login, signup, and password reset functionality.
```

This will:
1. Spawn a Planner subagent to generate the spec
2. For each Sprint: Build → spawn Evaluator subagent → check PASS/FAIL
3. Auto-retry on FAIL (up to 3 times)
4. Produce a final report when all Sprints pass

Best for well-defined features where you're confident in the requirements.

### Fully Autonomous Mode: harness.sh (Terminal)
For maximum control and logging, run the shell script directly:

```bash
# Make it executable (first time only)
chmod +x harness.sh

# Basic usage
./harness.sh "Build a todo app with user auth and real-time sync"

# With options
./harness.sh "Add payment processing with Stripe" \
  --max-sprints 3 \
  --max-retries 2 \
  --model opus \
  --max-budget 50

# Resume from existing spec after manual fixes
./harness.sh --skip-plan "Continue implementation"
```

**harness.sh options:**

| Flag | Default | Description |
|------|---------|-------------|
| `--max-sprints N` | 5 | Maximum number of Sprints |
| `--max-retries N` | 3 | Max FAIL retries per Sprint |
| `--max-budget N` | 50 | Max USD per agent call |
| `--model MODEL` | opus | Model: opus or sonnet |
| `--skip-plan` | false | Skip planning, use existing spec.md |

**What harness.sh does differently from /auto-orchestrate:**
- Each agent runs as a **separate `claude -p` process** (true context isolation)
- All output is logged to `.harness/logs/`
- Automatically parses evaluation reports for PASS/FAIL
- Manages git tags for Sprint boundaries
- Colored terminal output with timestamps
- Can run overnight / in CI pipelines

---

## 5. Integrating Existing MCP Servers

If you already have MCP servers configured, the Evaluator can leverage them.

### Playwright MCP (for E2E Testing)
To have the Evaluator test UI in a real browser:

```jsonc
// ~/.claude/settings.json (user global)
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@anthropic/mcp-playwright"]
    }
  }
}
```

### GitHub MCP (PR/Issue Integration)
```jsonc
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<your-token>"
      }
    }
  }
}
```

### Check Existing MCP Configuration
```bash
# View currently configured MCP servers
cat ~/.claude/settings.json
```

---

## 6. Customization

### Changing the Tech Stack
Edit the Tech Stack section in `.claude/CLAUDE.md` for your project:
- React → Vue, Svelte, Angular, etc.
- FastAPI → Express, NestJS, Django, etc.
- SQLite → MongoDB, Supabase, etc.

### Adjusting Evaluation Weights
Modify weights in `.claude/skills/evaluate/SKILL.md` to match project priorities:
- Security-critical projects: Security 30%, Functionality 25%
- Design-focused projects: UI/UX 25%, Functionality 25%

### Sprint Sizing
Default: 2-5 Sprints
- Simple features: 1-2 Sprints
- Large-scale features: 5-8 Sprints

### Adding New Commands
Create a `.claude/skills/<name>/SKILL.md` file to auto-register a `/name` command:

```yaml
---
name: my-command
description: Command description
---
Prompt content goes here...
```

---

## 7. Troubleshooting

### "Skill not found"
- Verify the path `.claude/skills/<name>/SKILL.md` is correct
- Ensure the filename is exactly `SKILL.md` (uppercase)

### Evaluator is Too Lenient
- Strengthen the "Maintain Skepticism" section in the evaluate SKILL.md
- Raise the PASS threshold from 70% → 80%

### Context Fills Up Too Quickly
- Reduce Sprint size to increase inter-agent handoff frequency
- Apply `context: fork` to more skills

### Hooks Not Working
- Verify `chmod +x` was applied
- Confirm `jq` is installed
- Check that paths in `.claude/settings.json` are correct
