# Agent Discoverable Config

[한국어](README.md) · [English](README.en.md)

Users increasingly ask AI agents to install apps and change their preferences.
A request such as “turn off the recording start sound” or “transcribe in Japanese”
should let an agent find the installed app and make the right change.

With installed binary apps, this process often stops at discovery. The configuration
file may be hard to find, or its fields may lack descriptions and allowed values.
Documentation elsewhere does little if an agent cannot find it from the installed app.

This guide defines **conventions for discovering and changing preferences using the
installed app alone**. It is intended for developers of macOS, Windows, and Linux apps.
The goal is to lead an agent to the active configuration file and provide the information
needed to edit it there.

The conventions do not depend on a particular agent's automatic discovery behavior.
They support several routes: installation notes, executable help, and conventional
user configuration locations. They are practical design guidance, not an established
standard that every agent follows.

This guide covers design principles, format choices, and discovery, editing, and
application workflows. Implementation examples and test results are in the
[separate validation report](docs/benchmarks.en.md).

## From a user's request to the active configuration

A natural-language request should connect to this workflow:

```text
“Turn off the app's recording start sound”
  → Find the installed app / executable
  → Read readme.txt, --help, or a conventional user configuration location
  → Find the config.* file that the app actually loads
  → Read field descriptions, allowed values, and application instructions
  → Change only the requested values
  → Reload / restart
  → Verify that the app applied the changes
```

The goal is not to make an agent read a particular instruction file. Whichever route
it takes should lead to the same active configuration file, and **that file should
provide enough information to make the change**. Discovery and configuration should
be possible starting from the installed app's location.

## Best practices and designs to avoid

A good design lets the agent obtain answers from the installed app. The comparison
below focuses on reaching and applying the correct configuration, beyond simply
having an instruction file.

| Situation | Best practice | Worst practice |
| --- | --- | --- |
| Discovering the location | Installed `readme.txt` and `--help` lead to the active user file | Location is documented only somewhere inaccessible from the installation |
| File naming | A clear name such as `config.toml` | An obscure name or settings stored only in an internal database |
| Understanding fields | Explain the function, type, allowed values, and units next to each key | Undocumented abbreviations and numeric codes |
| Active configuration | Distinguish defaults and examples from the active user file | Imply that editing an example in the install folder applies changes |
| Saving files | GUI saves retain standard field descriptions | The first save removes all explanatory comments |
| New installations | Provide a way to obtain an annotated default file | No file, example, or documented creation method |
| Applying external edits | Explain reload/restart steps and when changes take effect | Recommend pressing Save in a window that overwrites edits with old values |
| Invalid values | Identify the field, allowed values, and whether previous values are retained | Silently ignore values or replace them with defaults |
| Verification | Check the values the app loaded and the resulting behavior | Report success just because text changed in a file |
| Upgrades | Define precedence and migration between old and new files | Rename files and lose existing user preferences |

Suppose the user asks to turn off the recording start sound. This file does not tell
an agent whether `sound` controls an effect or system output, or whether `0` means
disabled or default:

```ini
[settings]
sound=100
mode=1
limit=30
```

The same settings can provide enough context in the file itself:

```ini
[settings]
# Recording start sound volume. Integer 0..200 (%), default 100.
# 0 disables the sound. Separate from muting system output during recording.
sound_volume=100

# true: record while held / false: press again to stop.
# Allowed values: true/false. Default: true.
hold=true

# Recording limit in minutes. Allowed values: 10/20/30/60. Default: 30.
limit_minutes=30
```

This example assumes an INI parser that supports `#` comments. Examples must match
the syntax the app actually supports.

## Make instructions discoverable in the installed app

Ship a UTF-8 `readme.txt` with the installation. An agent should be able to find it
from the installed app's location and follow it to the active user configuration.

| Platform | Suggested instruction location | Typical user configuration location |
| --- | --- | --- |
| macOS | `App.app/Contents/Resources/readme.txt` | `~/Library/Application Support/<app-id>/config.toml` |
| Windows | `readme.txt` beside the installed executable | `%APPDATA%\<app-id>\config.toml` |
| Linux | `<prefix>/share/doc/<app-id>/readme.txt` | `${XDG_CONFIG_HOME:-~/.config}/<app-id>/config.toml` |

Use the platform's package structure: bundle resources on macOS, the executable
directory on Windows, and an app-specific documentation directory on Linux.
A generically named instruction file in a shared executable directory is ambiguous.
Packaging may change user data locations. If the app provides a path command,
prefer its active path over a generic path listed in documentation.

Start the instructions with:

1. The app name and the purpose of these installation instructions.
2. User configuration locations and the command that prints the active path.
3. The distinction between installed examples/defaults and the active user file.
4. Editable formats, comment support, and reload/restart steps.
5. How to create the configuration when it does not exist yet.
6. Where credentials are stored and how they relate to general preferences.

The instructions should be complete using only files in the installed app.
Update them alongside the configuration schema. Check every distribution format;
including instructions in an MSIX does not cover a separately distributed EXE.

## Make the configuration itself understandable and editable

Use a recognizable name such as `config.toml`, `config.ini`, or `config.<app-name>`.
Where practical, use the same format and key names across platforms. Paths can vary
while a setting's meaning and value representation remain consistent.

### Choosing TOML, INI, or JSON

For a new app with human-editable preferences, **consider TOML first**. Comments,
typed values, and arrays let descriptions sit beside values. This recommendation
follows the format's properties, not a comparison of agent success rates by format.

| Format | Strengths | Considerations | Suitable use |
| --- | --- | --- | --- |
| TOML | Comments, explicit value types, arrays, and groups | Specify the supported syntax/version and parser | General preferences that people read and agents edit selectively |
| INI | Compact, easy to edit for flat settings | Comments, types, arrays, and escaping vary by parser | Simple settings in apps already using an INI library |
| JSON | Broad tooling and consistent nesting, arrays, and types | Standard JSON has no comments | Existing JSON designs with bundled schemas or description commands |
| JSONC | JSON-like structure with comments | A standard JSON parser cannot be assumed to accept it | Apps and tools explicitly supporting the same JSONC syntax |

The [TOML specification](https://toml.io/en/v1.0.0) defines `#` comments, value types,
and arrays. [Standard JSON (RFC 8259)](https://www.rfc-editor.org/rfc/rfc8259) has no
comment syntax. Document INI comments, escaping, and lists according to the chosen parser.

JSON is suitable for agent editing too. The question is where descriptions come from.
Before migrating an existing JSON app solely for this purpose, consider bundling a
schema or providing a command that prints descriptions.

For example, `config.json` holds values while `config.schema.json` describes the
same keys. Installation notes or help should explain the relationship:

```json
{
  "auto_send": false,
  "recording_start_sound_volume": 0
}
```

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "auto_send": {
      "type": "boolean",
      "description": "Presses the actual Enter key after pasting.",
      "default": true
    },
    "recording_start_sound_volume": {
      "type": "integer",
      "description": "Recording start sound volume (%). 0 disables it.",
      "minimum": 0,
      "maximum": 200,
      "default": 100
    }
  }
}
```

[JSON Schema annotations](https://json-schema.org/understanding-json-schema/reference/annotations)
can convey meanings and defaults. Declaring `default` does not make an app apply it
automatically; the loader must implement defaults and validation.

The decisive factor is **whether an agent can determine meaning, valid values, and
application steps**, rather than the extension. Well-documented JSON is better than
undocumented TOML. A comment-capable format is more direct when the goal is to get
all necessary information by opening one configuration file.

### Should an existing app migrate to TOML?

Do not migrate an existing INI or JSON app just because this guide recommends TOML.
First check whether the current format can support descriptions, discovery, and
application instructions. Plan a migration when arrays, nesting, or cross-platform
consistency justify the cost.

Changing the extension alone is insufficient. Verify the parser, value representation,
and compatibility with existing installations. Define how to retain original files
and migrate both preferences and credentials.

### What each field description should include

Write descriptions into the active user file that the GUI saves. Avoid designs where
only the distributed template has comments and the first GUI save removes them.

| Information | Example |
| --- | --- |
| Function in user terms | “Recording start sound volume” |
| Value type | Integer, boolean, string, string array |
| Default | `100` |
| Allowed values, range, and unit | `0`–`200`, `%` |
| Special values | `0` disables the sound |
| Dependencies | Service, model, or OS restrictions |
| When it takes effect | Immediately, on reload, on restart, or on the next recording |
| Side effects | Automatic sending presses the actual Enter key after pasting |

Describe separate settings clearly enough to distinguish “mute the recording sound”
from “mute system output during recording.”

```toml
# Presses the actual Enter key after pasting.
# Type: boolean. Allowed values: true / false. Default: true.
auto_send = true

# Recording start sound volume. Integer 0..200 (%), default 100.
# 0 disables the sound. Separate from muting system output during recording.
recording_start_sound_volume = 100

# Recording control. "hold": record while held / "toggle": press again to stop.
# Default: "hold". Restart the app after editing this file.
recording_control = "hold"
```

The file header should identify the app, format, editing/application steps, and
credential separation. State whether custom comments are preserved. Where practical,
generate validation rules, defaults, descriptions, and documentation from one schema.

## Provide configuration discovery through the executable

Provide at least the following capabilities; command names may differ:

```text
app --help
  Explain the path-discovery command and local instruction location

app --config-path
  Print the absolute path of the active user configuration to stdout
```

Handle discovery commands before creating a GUI, recording, making network requests,
or requesting accessibility/microphone permissions, then exit. They should work on a
new installation. A path-only command should not change preferences or credentials.
If the file does not exist yet, explain that the printed path is its creation location.

Additional capabilities can simplify initial setup and verification:

- Print default configuration with field-description comments.
- Create an initial configuration without overwriting an existing file.
- Validate edited configuration and report error locations and allowed values.
- Show loaded non-secret preferences.
- Reload settings or support reliable file watching.

For Windows GUI executables, check console output handling as well. Help and paths
must be available when an agent captures stdout through a pipe or redirects it to a file.

## From a file edit to an applied change

Document a workflow an agent can follow:

1. Identify the installed app version and active configuration file.
2. Wait for ongoing work to finish and close the app if required.
3. Read relevant fields and change requested values while preserving other values.
4. Validate syntax, types, and ranges, then replace the file atomically.
5. Reload or restart using the documented procedure.
6. Verify the saved file, loaded values, and actual behavior separately.

If a GUI can overwrite external edits with old in-memory values, explicitly instruct
users to close the app before editing and restart afterward. Opening Settings should
not be assumed to reload the file.

Prefer separating credentials from general preferences. If they share a file, say so
in the header and local instructions, and advise against logging the entire contents.
Protect credential files and backups using platform-appropriate access controls,
such as `0600` on macOS/Linux or a user-restricted ACL on Windows.
Editing a configuration file does not grant OS microphone, accessibility, or global
shortcut permissions.

When changing names or schemas, define legacy loading, precedence, and subsequent
save behavior. Upgrades must preserve existing languages, shortcuts, and credentials.
