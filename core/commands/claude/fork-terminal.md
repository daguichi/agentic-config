---
description: Open new kitty terminal session with optional command and prime prompt
argument-hint: <path> [cmd] [tab|window] [prime_prompt]
project-agnostic: true
allowed-tools:
  - Bash
---

# Fork Terminal

Open a new kitty terminal tab or window, cd to specified path, and optionally run a command with initial context.

## BEHAVIOR

1. PARSE arguments:
   - PATH=$1 (required): directory to cd into
   - CMD=$2 (optional, default: "claude"): command to run after cd
   - MODE=$3 (optional, default: "window"): "tab" or "window"
   - PRIME_PROMPT=$4+ (optional): initial context to prime claude with

2. VALIDATE:
   - Ensure PATH exists
   - If MODE is not "tab" or "window", default to "window"

3. BUILD osascript command:
   - Activate kitty
   - Open new tab (cmd+t) or window (cmd+n) based on MODE
   - Add delay 0.5 for UI sync
   - Type: `cd {PATH} && clear && {CMD}`
   - Press return

4. HANDLE prime_prompt:
   - If PRIME_PROMPT provided and CMD is "claude":
     - Check if cload is available (command -v cload)
     - If available: use `cd {PATH} && clear && cload {PRIME_PROMPT} | claude`
     - Otherwise: just run `cd {PATH} && clear && {CMD}` (user must paste prompt)
   - If PRIME_PROMPT provided but CMD is not "claude":
     - Warn user that prime_prompt only works with claude command
     - Run without prime_prompt

5. EXECUTE osascript command

## EXAMPLE OSASCRIPT STRUCTURE

```bash
osascript -e 'tell application "kitty" to activate' \
  -e 'tell application "System Events" to tell process "kitty" to keystroke "t" using command down' \
  -e 'delay 0.5' \
  -e 'tell application "System Events" to tell process "kitty" to keystroke "cd /path && clear && claude"' \
  -e 'tell application "System Events" to keystroke return'
```

## VARIABLES

PATH=$1
CMD=$2 (default: "claude")
MODE=$3 (default: "window")
PRIME_PROMPT=$4+
