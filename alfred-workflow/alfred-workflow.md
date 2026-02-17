# /alfred-workflow -- Create or modify an Alfred workflow

You are creating an Alfred workflow by writing files directly to the Alfred
preferences directory. The user will describe what they want the workflow to do.
You will generate the `info.plist` and any supporting scripts, then write them
to a new workflow directory.

The user's request: $ARGUMENTS

## Workflow directory

Workflows live in:
`~/Library/Application Support/Alfred/Alfred.alfredpreferences/workflows/`

Each workflow gets its own directory. For new workflows, create a directory
named `user.workflow.XXXXXXXX` where `XXXXXXXX` is 8 hex chars (use the
first 8 chars of a generated UUID). The directory must contain at minimum
an `info.plist` file. Add an `icon.png` if the user provides one.

## Step-by-step procedure

1. Parse the user's request into: trigger type, action logic, output type.
2. Generate UUIDs for each node (run `uuidgen` once per node).
3. Write the `info.plist` using the template and reference below.
4. Write any external script files to the same directory.
5. Validate with `plutil -lint <path>/info.plist`.
6. Tell the user the keyword or trigger to use.

## info.plist top-level structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>bundleid</key>
    <string>com.user.my-workflow</string>
    <key>name</key>
    <string>My Workflow</string>
    <key>description</key>
    <string>What it does</string>
    <key>createdby</key>
    <string></string>
    <key>category</key>
    <string>Productivity</string>
    <key>disabled</key>
    <false/>
    <key>version</key>
    <string>1.0.0</string>
    <key>readme</key>
    <string></string>
    <key>webaddress</key>
    <string></string>
    <key>objects</key>
    <array>
        <!-- node dicts here -->
    </array>
    <key>connections</key>
    <dict>
        <!-- source-uid -> array of connection dicts -->
    </dict>
    <key>uidata</key>
    <dict>
        <!-- uid -> {xpos, ypos} -->
    </dict>
    <key>userconfigurationconfig</key>
    <array/>
    <key>variables</key>
    <dict/>
    <key>variablesdontexport</key>
    <array/>
</dict>
</plist>
```

## Node structure

Every node in the `objects` array:
```xml
<dict>
    <key>type</key>
    <string>alfred.workflow.input.keyword</string>
    <key>uid</key>
    <string>XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX</string>
    <key>version</key>
    <integer>2</integer>
    <key>config</key>
    <dict><!-- type-specific --></dict>
</dict>
```

## Connections

Map source UIDs to arrays of connection dicts:
```xml
<key>SOURCE-UID</key>
<array>
    <dict>
        <key>destinationuid</key>
        <string>DEST-UID</string>
        <key>modifiers</key>
        <integer>0</integer>
        <key>modifiersubtext</key>
        <string></string>
        <key>vitoclose</key>
        <false/>
    </dict>
</array>
```

**Modifier bitmask** (for routing by held key):
| Value | Key |
|-------|-----|
| 0 | None (default) |
| 131072 | Shift |
| 262144 | Control |
| 524288 | Option |
| 1048576 | Command |

Combine with bitwise OR for combos. Multiple connections from one source
enable modifier-based branching (e.g., Enter -> clipboard, Cmd+Enter -> open URL).

For conditional nodes, add `sourceoutputuid` to the connection dict matching
the condition's `uid` or `_button1`/`_button2` for dialog buttons.

## uidata

Position each node on the canvas. Flow left-to-right:
```xml
<key>NODE-UID</key>
<dict>
    <key>xpos</key><real>150</real>
    <key>ypos</key><real>50</real>
</dict>
```
Triggers at x=150, actions at x=400, outputs at x=650. Vertical spacing ~150.

## Script type integers

Used in the `type` config key of script and script filter nodes:

| Value | Interpreter |
|-------|-------------|
| 0 | /bin/bash |
| 5 | /usr/bin/osascript (AppleScript) |
| 7 | /usr/bin/osascript -l JavaScript (JXA) |
| 8 | External script (uses `scriptfile` key) |
| 11 | /bin/zsh |

**Use type 0 (bash) for shell scripts** unless you specifically need zsh
features, in which case use type 11. **Do NOT use type 7 -- that is JXA
(JavaScript for Automation), not zsh.** This is a common mistake.

### scriptargtype

| Value | Behaviour |
|-------|-----------|
| 0 | `{query}` is substituted into the script text. Escaping bitmask applies. |
| 1 | Query passed as argv (`$1` / `"$@"`). Safer. **Preferred.** |

When using scriptargtype=0, set `escaping` to 102 (sensible default bitmask).
When using scriptargtype=1, escaping is irrelevant.

### The zsh --no-rcs gotcha

Alfred runs scripts with `/bin/zsh --no-rcs`. This means:
- `.zshrc` is NOT loaded. PATH is minimal: `/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin`
- `.zshenv` IS loaded (the one exception).
- Use full paths to executables, or set PATH in workflow `variables`.

## Common node types -- quick reference

### Keyword input
Type: `alfred.workflow.input.keyword`, version: 2
```xml
<key>keyword</key><string>kw</string>
<key>text</key><string>Title in results</string>
<key>subtext</key><string>Subtitle</string>
<key>withspace</key><true/>
<key>argumenttype</key><integer>0</integer>
```
argumenttype: 0=required, 1=optional, 2=none.

### Script filter
Type: `alfred.workflow.input.scriptfilter`, version: 3
```xml
<key>keyword</key><string>sf</string>
<key>title</key><string>Title</string>
<key>subtext</key><string>Subtitle</string>
<key>runningsubtext</key><string>Loading...</string>
<key>withspace</key><true/>
<key>argumenttype</key><integer>1</integer>
<key>script</key><string>./filter.sh "$1"</string>
<key>scriptargtype</key><integer>1</integer>
<key>type</key><integer>0</integer>
<key>escaping</key><integer>102</integer>
<key>queuemode</key><integer>1</integer>
<key>queuedelaymode</key><integer>0</integer>
<key>queuedelayimmediatelyinitially</key><true/>
<key>queuedelaycustom</key><integer>3</integer>
<key>alfredfiltersresults</key><false/>
<key>alfredfiltersresultsmatchmode</key><integer>0</integer>
```
Script must output JSON:
```json
{"items": [{"uid":"id","title":"Title","subtitle":"Sub","arg":"value","valid":true}]}
```
Items can also have: `icon`, `mods` (cmd/alt/ctrl/shift/fn with subtitle+arg),
`variables`, `match`, `autocomplete`, `quicklookurl`, `text` (copy/largetype).

### Run script action
Type: `alfred.workflow.action.script`, version: 2
```xml
<key>script</key><string>echo "$1"</string>
<key>scriptargtype</key><integer>1</integer>
<key>type</key><integer>0</integer>
<key>escaping</key><integer>102</integer>
<key>concurrently</key><false/>
```

### Open URL
Type: `alfred.workflow.action.openurl`, version: 1
```xml
<key>url</key><string>https://example.com/{query}</string>
<key>browser</key><string></string>
<key>skipqueryencode</key><false/>
<key>spaces</key><string></string>
<key>utf8</key><true/>
```

### Clipboard output
Type: `alfred.workflow.output.clipboard`, version: 3
```xml
<key>clipboardtext</key><string>{query}</string>
<key>autopaste</key><false/>
<key>transient</key><false/>
```

### Notification output
Type: `alfred.workflow.output.notification`, version: 1
```xml
<key>title</key><string>Done</string>
<key>text</key><string>{query}</string>
<key>lastpathcomponent</key><false/>
<key>onlyshowifquerypopulated</key><true/>
<key>removeextension</key><false/>
<key>soundname</key><string></string>
```

### Universal action (text/URL)
Type: `alfred.workflow.trigger.universalaction`, version: 1
```xml
<key>acceptsfiles</key><false/>
<key>acceptsmulti</key><integer>0</integer>
<key>acceptstext</key><true/>
<key>acceptsurls</key><false/>
<key>name</key><string>Action Name</string>
```

**Required**: Universal Actions must connect to an **Argument Utility** node
before reaching a Run Script. Direct UA → Run Script connections silently
fail (the script never fires). The correct chain is:

```
Universal Action → Arg & Vars Utility → Run Script
```

The Argument Utility should pass `{query}` through:
```xml
<key>argument</key><string>{query}</string>
<key>passthroughargument</key><false/>
<key>variables</key><dict/>
```

Set `vitoclose` to `false` on the UA → Argument Utility connection.

### Conditional
Type: `alfred.workflow.utility.conditional`, version: 1
```xml
<key>conditions</key>
<array>
    <dict>
        <key>inputstring</key><string>{query}</string>
        <key>matchmode</key><integer>0</integer>
        <key>matchstring</key><string>value</string>
        <key>outputlabel</key><string>Matched</string>
        <key>uid</key><string>COND-UUID</string>
    </dict>
</array>
<key>elselabel</key><string>else</string>
```
matchmode: 0=is, 1=is not, 2=contains, 3=not contains, 4=begins with,
5=ends with, 6=regex, 14=is empty, 15=is not empty.
Route connections using `sourceoutputuid` matching the condition's `uid`.

### Arg and Vars utility
Type: `alfred.workflow.utility.argument`, version: 1
```xml
<key>argument</key><string>{query}</string>
<key>passthroughargument</key><false/>
<key>variables</key>
<dict>
    <key>myvar</key><string>value</string>
</dict>
```

### Junction (merge point)
Type: `alfred.workflow.utility.junction`, version: 1
No config needed. Merges multiple paths into one output.

## User configuration fields

Add to `userconfigurationconfig` array for user-facing settings:
```xml
<dict>
    <key>type</key><string>textfield</string>
    <key>variable</key><string>api_key</string>
    <key>label</key><string>API Key</string>
    <key>description</key><string>Your API key</string>
    <key>config</key>
    <dict>
        <key>default</key><string></string>
        <key>placeholder</key><string>Enter key</string>
        <key>required</key><true/>
        <key>trim</key><true/>
    </dict>
</dict>
```
Types: `textfield`, `textarea`, `checkbox` (default bool, returns "0"/"1"),
`filepicker` (filtermode: 0=files, 1=folders, 2=both),
`popupbutton` (pairs: array of [value, label] arrays).

Access in scripts as env vars (`$api_key`) or in fields as `{var:api_key}`.
List secret variable names in `variablesdontexport`.

## Placeholders

| Syntax | Meaning |
|--------|---------|
| `{query}` | Current argument passed to node |
| `{var:name}` | Workflow variable |
| `{const:alfred_workflow_cache}` | Cache dir path |
| `{const:alfred_workflow_data}` | Data dir path |

## XML encoding reminder

info.plist is XML. Escape in all string values:
`&` -> `&amp;`, `<` -> `&lt;`, `>` -> `&gt;`, `"` -> `&quot;`

## Node connectivity rules

Input nodes (`input.keyword`, `input.scriptfilter`) are flow entry points — they
cannot receive connections from other nodes. Action nodes (`action.script`,
`action.openurl`), output nodes (`output.clipboard`, `output.notification`),
utility nodes (`utility.conditional`, `utility.argument`, `utility.junction`),
and trigger nodes (`trigger.universalaction`) can receive connections. Universal
Actions feed into action/utility/output nodes, not into input nodes.

## Working directory

Alfred sets the working directory to the workflow directory when running scripts.
This is why `./script.sh` and `./filter.py` work as script paths. External files
should use absolute paths or `$HOME`.

## Multi-step workflow patterns

### Pattern: Single-script with native dialogs (recommended)

For workflows needing user choices (file pickers, confirmations, optional
inputs), **prefer a single Run Script that uses osascript dialogs** over
multi-node chaining. This is more reliable and avoids escaping issues:

```
Universal Action → Arg & Vars Utility → Run Script (does everything)
```

The Run Script uses `choose from list` for pickers, `display dialog` for
text input, writes results to temp files, and launches terminal commands.
All user interaction happens via native macOS dialogs inside one script.

Advantages over multi-node chaining:
- No flow-chaining bugs (Script Filters are input-only nodes)
- No state-passing between nodes (everything is in one script's scope)
- No shell escaping across node boundaries
- Easier to debug (single script, single log entry)

### Pattern: Keyword-triggered Script Filter chaining (fragile)

Script Filters are input nodes, so you can't connect an action directly into
one. The workaround is to trigger Alfred's search programmatically:

```bash
osascript -e 'tell application id "com.runningwithcrayons.Alfred" to search "mykeyword "'
```

**Warning:** This pattern is fragile. It can race with Alfred's UI state,
and the Script Filter receives no argument from the previous node. State
must be passed via temp files. Prefer the single-script pattern above.

### Pattern: External trigger chaining

Invoke one workflow from another, or from a script:
```bash
osascript -e 'tell application id "com.runningwithcrayons.Alfred" to run trigger "trigger-name" in workflow "com.user.my-workflow" with argument "some-arg"'
```
Requires an External Trigger node in the target workflow. External Trigger node
type: `alfred.workflow.trigger.external`, version: 1, config key `triggerid`
(string).

## Safe text passing

When user input contains shell metacharacters (backslashes, quotes, `$`,
braces — common in code, LaTeX, URLs), passing it as a shell argument through
multiple layers will break. The robust pattern:

1. **Node A** writes the text to a temp file: `printf '%s' "$1" > /tmp/workflow-state`.
   `printf '%s'` is safe — no interpretation of backslashes or special chars.
2. **Node B** reads it back: `text=$(/bin/cat /tmp/workflow-state)`
3. For structured data, write JSON using Python (`json.dump`) rather than
   constructing it in shell.

Avoid passing user text through: shell argument interpolation, AppleScript
string interpolation, or `{query}` substitution in URL/clipboard nodes when the
text may contain special characters.

## osascript dialogs

For simple user prompts mid-flow (optional inputs, confirmations).

**`set -e` warning:** osascript returns non-zero when the user cancels a
dialog. If your script uses `set -e`, the script will die silently on
cancellation. Always use `|| true` on command substitutions that capture
dialog output, then check the result with `-z`:

```bash
choice=$(osascript -e '...' 2>&1) || true
if [ -z "$choice" ] || [ "$choice" = "false" ]; then
    exit 0  # user cancelled, exit cleanly
fi
```

**Text input dialog:**
```bash
result=$(osascript -e 'display dialog "Enter value:" default answer "" buttons {"Cancel","OK"} default button "OK"' 2>&1) || true
if [ -n "$result" ]; then
    value=$(echo "$result" | sed 's/button returned:OK, text returned://')
fi
```

**Choose from list:**
```bash
choice=$(osascript -e 'choose from list {"Option A", "Option B", "Option C"} with prompt "Pick one:"' 2>&1) || true
# Returns "false" if cancelled, empty if error
```

**Simple confirmation:**
```bash
osascript -e 'display dialog "Are you sure?" buttons {"No","Yes"} default button "Yes"'
# Exit code 1 if "No" or cancelled
```

## Terminal launch recipes

Opening a terminal window is a common workflow endpoint.

**Terminal.app:**
```bash
osascript <<'APPLESCRIPT'
tell application "Terminal"
    activate
    do script "cd '/path/to/dir' && some_command"
end tell
APPLESCRIPT
```

**iTerm2:**
```bash
osascript <<'APPLESCRIPT'
tell application "iTerm2"
    activate
    set newWindow to (create window with default profile)
    tell current session of newWindow
        write text "cd '/path/to/dir' && some_command"
    end tell
end tell
APPLESCRIPT
```

**Important:** If the command or path is constructed from variables, use an
unquoted heredoc (without the single quotes around the delimiter) so shell
variables expand. Only safe when the variable content has no AppleScript
metacharacters (single quotes, backslashes). For untrusted content, write a
launcher script to a temp file and run that instead.

## Debugging

Alfred has a built-in debugger. Open it via Alfred Preferences → Workflows →
select the workflow → click the bug icon (top-right of the canvas). The
debugger log shows:
- Which nodes fire and in what order
- The argument passed to each node (or `(null)` if none)
- Script output and errors with exit codes
- Error codes like `-2700` (AppleScript/JXA error) that reveal type mismatches

When a workflow misbehaves, ask the user to run it with the debugger open and
share the log. Key things to look for:
- `(null)` arguments → the previous node didn't pass data (connection or
  script issue)
- Error `-2700` with "SyntaxError" → the script is being run as the wrong
  interpreter (check the `type` integer — common mistake is using 7/JXA
  instead of 0/bash)
- Nodes not appearing in the log → the connection to them is missing or broken
- `Code 1` or other non-zero exit → the script itself failed (check paths,
  permissions, missing executables)

The agent cannot access the debugger directly, but the user can relay the log
output for diagnosis.

## Rules

- Always validate with `plutil -lint` after writing.
- Generate real UUIDs with `uuidgen`, never use placeholder strings.
- Set a `bundleid` on every workflow (required for external triggers, good practice always).
- Prefer `scriptargtype=1` (argv) over `scriptargtype=0` ({query} substitution).
- For shell scripts, use `type=0` (bash). Use `type=11` only if zsh features are needed. **Never use `type=7`** -- it is JXA, not zsh.
- Leave hotkey triggers unconfigured (hotkey=0, hotmod=0) -- users set their own.
- When the workflow needs executables not on the minimal PATH, either use full paths in scripts or add a PATH variable to the workflow's `variables` dict.
- After writing the workflow, tell the user the keyword/trigger and any setup steps needed.
