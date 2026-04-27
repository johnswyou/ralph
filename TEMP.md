# Using Ralph with GitHub Copilot CLI

This guide explains how to run Ralph with GitHub Copilot CLI as the coding harness.

Ralph is an autonomous agent loop that repeatedly starts a fresh AI coding tool, asks it to complete one unfinished story from `prd.json`, and continues until all stories pass or the max iteration count is reached.

## Prerequisites

You need:

1. A git repository for your project.
2. `jq` installed.
3. GitHub Copilot CLI installed and authenticated.
4. Ralph files copied into your project.

Install `jq` on macOS:

```bash
brew install jq
```

Install GitHub Copilot CLI with one of:

```bash
npm install -g @github/copilot
```

```bash
brew install copilot-cli
```

Then authenticate:

```bash
copilot login
```

For headless or automated runs, you can authenticate with an environment variable instead:

```bash
export COPILOT_GITHUB_TOKEN="github_pat_..."
```

Copilot CLI checks tokens in this order:

1. `COPILOT_GITHUB_TOKEN`
2. `GH_TOKEN`
3. `GITHUB_TOKEN`

For fine-grained PATs, the token needs the **Copilot Requests** permission.

## Step 1: Copy Ralph into your project

From your project root, create a scripts directory:

```bash
mkdir -p scripts/ralph
```

Copy the Ralph script and Copilot prompt template:

```bash
cp /path/to/ralph/ralph.sh scripts/ralph/
cp /path/to/ralph/COPILOT.md scripts/ralph/
chmod +x scripts/ralph/ralph.sh
```

Your project should now contain:

```text
scripts/ralph/
├── ralph.sh
└── COPILOT.md
```

## Step 2: Create `prd.json`

Ralph needs a `prd.json` file in the same directory as `ralph.sh`.

Create:

```text
scripts/ralph/prd.json
```

Example:

```json
{
  "project": "MyApp",
  "branchName": "ralph/task-priority",
  "description": "Task Priority System - Add priority levels to tasks",
  "userStories": [
    {
      "id": "US-001",
      "title": "Add priority field to database",
      "description": "As a developer, I need to store task priority so it persists across sessions.",
      "acceptanceCriteria": [
        "Add priority column to tasks table: 'high' | 'medium' | 'low' with default 'medium'",
        "Generate and run migration successfully",
        "Typecheck passes"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    }
  ]
}
```

Important fields:

| Field | Purpose |
|---|---|
| `branchName` | Ralph checks out or creates this branch before work begins. |
| `userStories` | Ordered task list for Ralph. |
| `priority` | Lower numbers are completed first. |
| `passes` | Ralph works only on stories where `passes: false`. |

## Step 3: Create `progress.txt`

Create an initial progress log beside `prd.json`:

```bash
cat > scripts/ralph/progress.txt <<'EOF'
# Ralph Progress Log
Started: initial setup
---
EOF
```

Ralph appends learnings here after each iteration.

Your Ralph directory should now look like:

```text
scripts/ralph/
├── COPILOT.md
├── prd.json
├── progress.txt
└── ralph.sh
```

## Step 4: Run Ralph with Copilot CLI

From your project root, run:

```bash
./scripts/ralph/ralph.sh --tool copilot
```

To limit the number of iterations:

```bash
./scripts/ralph/ralph.sh --tool copilot 5
```

By default, Ralph runs up to 10 iterations.

## What Copilot mode does

Each Ralph iteration runs GitHub Copilot CLI in non-interactive autonomous mode.

Internally, Ralph invokes Copilot CLI with flags equivalent to:

```bash
copilot \
  --autopilot \
  --allow-all \
  --no-ask-user \
  --max-autopilot-continues 10 \
  -s \
  -p "$(cat COPILOT.md)"
```

Meaning:

| Flag | Purpose |
|---|---|
| `--autopilot` | Lets Copilot continue working until it believes the task is complete. |
| `--allow-all` | Allows tools, paths, and URLs without prompting. |
| `--no-ask-user` | Prevents Copilot from blocking on clarifying questions. |
| `--max-autopilot-continues 10` | Caps autonomous continuation steps per Ralph iteration. |
| `-s` | Outputs only Copilot's response text. |
| `-p` | Runs a one-shot programmatic prompt and exits. |

## Step 5: Tune Copilot mode if needed

### Change the autopilot continuation cap

Default:

```bash
RALPH_COPILOT_MAX_AUTOPILOT_CONTINUES=10
```

Run with a smaller cap:

```bash
RALPH_COPILOT_MAX_AUTOPILOT_CONTINUES=3 ./scripts/ralph/ralph.sh --tool copilot
```

Run with a larger cap:

```bash
RALPH_COPILOT_MAX_AUTOPILOT_CONTINUES=20 ./scripts/ralph/ralph.sh --tool copilot
```

### Pin a model

```bash
RALPH_COPILOT_MODEL=gpt-5.3-codex ./scripts/ralph/ralph.sh --tool copilot
```

Or:

```bash
RALPH_COPILOT_MODEL=claude-sonnet-4.5 ./scripts/ralph/ralph.sh --tool copilot
```

Available model names depend on your Copilot CLI account and organization settings.

### Set reasoning effort

```bash
RALPH_COPILOT_EFFORT=high ./scripts/ralph/ralph.sh --tool copilot
```

Supported effort values:

```text
low
medium
high
xhigh
```

You can combine model and effort:

```bash
RALPH_COPILOT_MODEL=gpt-5.3-codex \
RALPH_COPILOT_EFFORT=xhigh \
./scripts/ralph/ralph.sh --tool copilot
```

## Step 6: Monitor progress

Watch terminal output during each iteration.

Ralph exits successfully when Copilot outputs:

```text
<promise>COMPLETE</promise>
```

Check story status:

```bash
cat scripts/ralph/prd.json | jq '.userStories[] | {id, title, passes}'
```

Check progress notes:

```bash
cat scripts/ralph/progress.txt
```

Check commits:

```bash
git log --oneline -10
```

## Step 7: Understand the per-iteration workflow

For each iteration, Copilot receives the instructions in `COPILOT.md`.

It should:

1. Read `prd.json`.
2. Read `progress.txt`.
3. Check out or create the branch from `branchName`.
4. Pick the highest-priority story where `passes: false`.
5. Implement only that one story.
6. Run the project's quality checks.
7. Commit the changes if checks pass.
8. Mark that story as `passes: true`.
9. Append learnings to `progress.txt`.
10. Output `<promise>COMPLETE</promise>` only when every story passes.

## Optional: Install Ralph skills for Copilot CLI

Ralph includes skills for generating PRDs and converting PRDs to `prd.json`.

Install them for Copilot CLI:

```bash
mkdir -p ~/.copilot/skills
cp -r skills/prd ~/.copilot/skills/
cp -r skills/ralph ~/.copilot/skills/
```

Then start Copilot CLI:

```bash
copilot
```

You can ask Copilot to use the skills:

```text
Use the /prd skill to create a PRD for adding task priorities.
```

```text
Use the /ralph skill to convert tasks/prd-task-priorities.md to prd.json.
```

## Security note

Copilot mode uses `--allow-all` so Ralph can run unattended.

This means Copilot CLI can read files, write files, run shell commands, and access URLs without asking for approval during the iteration.

Only run Ralph with Copilot CLI in repositories you trust.

Avoid running it from:

- Your home directory
- Directories containing secrets
- Repositories with untrusted scripts
- Shared machines where broad file access is risky

## Troubleshooting

### `Error: 'copilot' command not found`

Install Copilot CLI:

```bash
npm install -g @github/copilot
```

Or:

```bash
brew install copilot-cli
```

Then verify:

```bash
copilot --version
```

### Copilot asks you to log in

Run:

```bash
copilot login
```

Or export a token:

```bash
export COPILOT_GITHUB_TOKEN="github_pat_..."
```

### Ralph stops before all stories are complete

Check the progress log:

```bash
cat scripts/ralph/progress.txt
```

Then inspect unfinished stories:

```bash
cat scripts/ralph/prd.json | jq '.userStories[] | select(.passes == false)'
```

Run Ralph again:

```bash
./scripts/ralph/ralph.sh --tool copilot
```

### Copilot does not finish a story in one iteration

Increase the continuation cap:

```bash
RALPH_COPILOT_MAX_AUTOPILOT_CONTINUES=20 ./scripts/ralph/ralph.sh --tool copilot
```

If the story is still too large, split it into smaller stories in `prd.json`.

### Copilot made changes but did not commit

Check status:

```bash
git status
```

Review the output and `progress.txt`. If checks failed, fix the issue or rerun Ralph after adjusting the story.

## Recommended workflow

1. Write or generate a PRD.
2. Convert it to `prd.json`.
3. Make sure each story is small enough for one autonomous iteration.
4. Run:

   ```bash
   ./scripts/ralph/ralph.sh --tool copilot
   ```

5. Review commits and progress after each iteration.
6. Stop when Ralph prints:

   ```text
   <promise>COMPLETE</promise>
   ```
