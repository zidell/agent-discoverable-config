# Agent Discoverable Config: Validation Report

[한국어](benchmarks.md) · [English](benchmarks.en.md) · [Design guide](../README.en.md)

This report records the implementation and installation-based tests performed with
KeyScribe as of 2026-10-02. Design recommendations and observed validation coverage
are separate.

## Implementation example: KeyScribe

| Feature | macOS | Windows | Linux |
| --- | --- | --- | --- |
| Installed instructions | `.app/Contents/Resources/readme.txt` | Beside the executable; included in MSIX | `<prefix>/share/doc/keyscribe/readme.txt` |
| Active preferences | `~/Library/Application Support/keyscribe/config.toml` | `%APPDATA%\keyscribe\config.toml` | `${XDG_CONFIG_HOME:-~/.config}/keyscribe/config.toml` |
| Credentials | `user_config.json` in the same directory | Same | Same |
| Path discovery | Executable `--help` and `--config-path` | Same | Same |
| Field descriptions | Template comments reused on save | Template comments reused on save | Descriptions generated for each key on save |
| First launch | File created from bundled template | Annotated default file created | Annotated default file created |
| Applying external edits | Restart | Restart | Restart |

A standalone Windows EXE attempts to create an embedded `readme.txt` beside itself
on normal startup if it is absent. Creation is not guaranteed in a read-only directory;
failure does not prevent startup. MSIX and build artifacts include the file directly.

On Linux, normal startup migrates `config.ini`, or otherwise `settings.ini`, when
`config.toml` is absent. It maps legacy fields to the common TOML keys and arrays,
moves the API key into `user_config.json`, preserves additional JSON properties,
and retains the original INI unchanged. TOML takes precedence when present.
`--config-path` reports the current file without migrating it.

For example, `hold=false` becomes `recording_control="toggle"` and `sound_volume=50`
becomes `recording_start_sound_volume=50`. Terms become individual array entries;
replacement order is retained. `"ACME, Inc."` remains one term. Files are atomically
saved with `0600` permissions, with credentials written before TOML. Invalid TOML or
JSON stops startup rather than saving defaults or silently reverting to legacy INI.
The retained INI may still contain the previous API key.

Initial file creation does not persist environment-provided API keys. Discovery
commands start no GUI and create no default files. GUI saves regenerate standard
comments but do not preserve arbitrary custom comments. Detailed user instructions
are shipped as `readme.txt` with the app.

## Agent test results

The 16-request, 44-field test below was performed during the discovery and documentation
improvements, when Linux still used INI. Later TOML migration validation is recorded
separately; the earlier results do not validate the new schema.

### Metrics

“Reasoning success” here means observed outcomes, not measurements of internal thought:

```text
Setting mapping success = correctly changed requested fields / requested field changes
Request success = requests passing all field, syntax, and preservation checks / requests
```

Adding multiple words to one array counts as one field change. Changing that field
again in a later request counts separately. Runtime success is reported separately.

### Conditions

Three independent agents performed edits, and one agent validated the actual app.
One tested Linux installation instructions, one tested macOS and Windows file-based
installation fixtures, and one tested Linux help discovery without reading `readme.txt`.

- Editing agents started without conversation history.
- Source repository and online documentation access were prohibited.
- Only installation and isolated user-account locations were provided.
- Linux used the built and installed native binary and defaults from the actual loader.
- macOS and Windows used file-based fixtures containing installation notes and templates.
- Requests were sequential; a configuration snapshot was retained after each step.
- The parent agent independently checked expected values, syntax, unrelated values, and comments.
- Credentials were empty or fake; real accounts and personal preferences were not changed.

### Scenarios

| Platform | Sequential requests | Requests / changed fields |
| --- | --- | --- |
| Linux | Disable auto-send and sound, hide widget → toggle recording, 60-minute limit, 24-hour retention → Japanese, two terms, two replacement rules → top-right widget, 50% sound | 4 / 11 |
| macOS | Disable auto-send, enable toggle, disable sound → top-right widget, Japanese, 60-minute limit, 24-hour retention → add terms and replacements → right Ctrl, disable muting during recording | 4 / 11 |
| Windows | The same user requests as macOS, in a Windows installation fixture | 4 / 11 |
| Linux, instructions ignored | The Linux requests using only executable help and configuration comments, without reading `readme.txt` | 4 / 11 |

### Observations

| Test | Discovery and editing result | Validation coverage |
| --- | --- | --- |
| Installed Linux app | 4/4 requests, 11/11 fields | Actual GLib parser, app loader, and GTK controls in new processes |
| macOS installation fixture | 4/4 requests, 11/11 fields | TOML syntax, macOS-supported syntax subset, values, comments, credential-file preservation |
| Windows installation fixture | 4/4 requests, 11/11 fields | TOML syntax, values, comments, credential-file preservation |
| Linux, instructions ignored | 4/4 requests, 11/11 fields | Actual GLib parser, app loader, and GTK controls in new processes |
| Total | 16/16 requests and 44/44 fields: 100% in this sample | Coverage differs by platform as shown above |

The Linux installation route used local instructions, `--help`, and `--config-path`.
The macOS/Windows agent found settings through instructions near the executable.
That single agent tested both platforms: after looking at the macOS bundle root,
it used the Windows guide's common location descriptions to find macOS
`Contents/Resources`. These are not two independent agent discovery trials.

The help-only agent discovered `--config-path` through `--help` and completed four
requests using configuration comments. Semantically equivalent whitespace was accepted.
The initial file was prepared before the test; a separate check confirmed that printing
the path does not create a new file.

For each Linux discovery route, the initial state and four edited states were loaded
in fresh processes: ten snapshots in total, with actual Settings control values checked.
Xvfb and a separate D-Bus isolated the display/session. No actual recording, external
STT API, key injection, or clipboard transfer was performed in these agent-edit trials.

Linux native binary and DEB builds, installed instructions, GUI-free discovery,
legacy migration, initial file creation, environment-key non-persistence, and existing
native tests also passed. Windows settings source passed eight unit tests in an isolated
Rust project on Linux. An additional integration test checked initial file creation,
environment-key non-persistence, and loading external edits. This validates Windows
settings code, not a full Windows app build or UI run. Native macOS build/run and
native Windows app execution were not performed in this Linux environment.

### Limits of these results

This is a small controlled sample. It does not establish 100% success across agents,
user expressions, or installations, nor end-to-end installation, OS permission approval,
transcription, and automatic pasting. Discovery on a fresh installation with no initial
configuration still needs a separate trial.

One Linux agent printed the complete configuration before reading the instruction
warnings. Its API key was empty, so no real secret leaked. This supports credential
separation and secret-free diagnostic output, independently of editing success.

### Follow-up TOML migration validation

Linux native tests and actual binary/DEB builds passed again. Fifteen core tests covered:

- Migration from both legacy INI names, precedence, and unchanged originals.
- Language, models, shortcuts, recording mode, retention, terms, replacements, permission state, and API keys.
- JSON metadata preservation, `0600` permissions, and environment-key non-persistence.
- TOML comments, multiline arrays, Unicode, quotes, backslashes, and comma-containing terms.
- Boundary values, 19 invalid TOML fixtures, malformed JSON/INI errors, and existing-data preservation.

The actual loader and GTK Settings were checked across six states, including defaults,
sequential changes, and automatic INI migration. A fresh blind agent correctly changed
four requests and eleven fields in TOML, retaining `ACME, Inc.` as one array entry.

The first installation fixture still contained old INI instructions. The agent used
the binary's reported path and actual configuration comments to complete the edits.
This exposed a documentation/binary version mismatch. Instructions were updated for
TOML, and the final installation and DEB instructions were verified byte-for-byte
against the current document.

## Remaining validation and improvements

KeyScribe implements installed instructions, field comments, path discovery, and Linux
compatibility. Native behavior across all platforms, preservation of custom comments,
a configuration-validation command, and automatic reload remain outside the completed
implementation and validation described here.
