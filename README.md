# codex-turn-ended-reminder

A small `notify` hook for the OpenAI Codex CLI. It sends a terminal
notification when Codex is waiting for you, then gently re-notifies on a
tapering schedule.

The hook was designed for cmux and uses cmux's `OSC 777` notification sequence
by default. It also emits a best-effort `OSC 9` terminal notification sequence,
so it can work in other terminal emulators that support terminal-triggered
notifications.

Remote machines do not need SSH access back into your laptop. The script writes
notification escape sequences to the active terminal, SSH TTY, or tmux client
TTY, so the local terminal stack can surface the alert.

## Features

- Includes the machine name by default: `Codex @ <host>`
- Sends an immediate notification when a Codex turn ends
- Repeats at `2, 5, 10, 20, 40` minutes by default
- Cancels stale reminder loops when a new turn ends
- Supports remote SSH sessions through terminal escape sequences
- Supports tmux client TTYs
- Uses `cmux notify` directly when available
- Emits configurable terminal notification protocols: `osc777`, `osc9`, `bell`
- Optionally plays a local macOS sound with `afplay`
- Can also be wired to Codex permission/input prompts

## Install

Clone this repo on the machine where Codex runs, then install the hook:

```bash
mkdir -p ~/.codex/hooks
cp ./codex-turn-ended-reminder ~/.codex/hooks/codex-turn-ended-reminder
chmod +x ~/.codex/hooks/codex-turn-ended-reminder
```

Add the hook to `~/.codex/config.toml`.

Linux:

```toml
notify = ["/home/YOU/.codex/hooks/codex-turn-ended-reminder"]
```

macOS:

```toml
notify = ["/Users/YOU/.codex/hooks/codex-turn-ended-reminder"]
```

Replace `YOU` with the username on that machine, then restart Codex.

## One-Shot Install Helper

From the repo directory:

```bash
mkdir -p ~/.codex/hooks
cp ./codex-turn-ended-reminder ~/.codex/hooks/codex-turn-ended-reminder
chmod +x ~/.codex/hooks/codex-turn-ended-reminder

HOOK_PATH="$HOME/.codex/hooks/codex-turn-ended-reminder"

printf '\nAdd this to ~/.codex/config.toml:\n\n'
printf 'notify = ["%s"]\n' "$HOOK_PATH"

printf '\nOptional PermissionRequest command for ~/.codex/hooks.json:\n\n'
printf 'env REMINDER_MESSAGE='\''Waiting for input'\'' REMINDER_ONCE=1 REMINDER_STDOUT_FALLBACK=0 %s\n' "$HOOK_PATH"
```

## Permission/Input Prompts

The `notify` entry in `config.toml` fires when a Codex assistant turn ends. To
also get notified when Codex is waiting on a permission/input prompt, merge this
into `~/.codex/hooks.json`.

Linux:

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "env REMINDER_MESSAGE='Waiting for input' REMINDER_ONCE=1 REMINDER_STDOUT_FALLBACK=0 /home/YOU/.codex/hooks/codex-turn-ended-reminder"
          }
        ]
      }
    ]
  }
}
```

macOS command path:

```json
"command": "env REMINDER_MESSAGE='Waiting for input' REMINDER_ONCE=1 REMINDER_STDOUT_FALLBACK=0 /Users/YOU/.codex/hooks/codex-turn-ended-reminder"
```

If `hooks.json` already exists, keep its existing entries and add only the
`PermissionRequest` array under the top-level `hooks` object. Restart Codex
after editing `hooks.json`.

`REMINDER_ONCE=1` is recommended for permission hooks because not every Codex
version exposes a matching "prompt answered" cancellation event.

## Configuration

Configure behavior with environment variables.

| Variable | Default | Description |
| --- | --- | --- |
| `REMINDER_TITLE` | `Codex @ <host>` | Notification title |
| `REMINDER_MESSAGE` | `Turn ended` | Notification body |
| `REMINDER_DELAYS` | `120 180 300 600 1200` | Reminder delays, in seconds |
| `REMINDER_ONCE` | `0` | Set to `1` to send only the immediate notification |
| `REMINDER_STATE_DIR` | `$XDG_STATE_HOME/codex-turn-ended-reminder` or `~/.local/state/codex-turn-ended-reminder` | State directory for token, PID, and log files |
| `REMINDER_PLAY_LOCAL_SOUND` | `auto` | Set to `0`, `false`, `no`, or `off` to disable local sound |
| `REMINDER_SOUND_FILE` | `/System/Library/Sounds/Ping.aiff` | macOS sound used with `afplay` |
| `REMINDER_STDOUT_FALLBACK` | `1` | Set to `0` when stdout has protocol meaning |
| `REMINDER_PROTOCOLS` | `osc777 osc9` | Space-separated terminal protocols: `osc777`, `osc9`, `bell` |

Example custom `notify` config:

```toml
notify = ["env", "REMINDER_MESSAGE=Waiting for input", "REMINDER_DELAYS=60 180 600", "/home/YOU/.codex/hooks/codex-turn-ended-reminder"]
```

To use only cmux notifications:

```toml
notify = ["env", "REMINDER_PROTOCOLS=osc777", "/home/YOU/.codex/hooks/codex-turn-ended-reminder"]
```

To add a plain terminal bell fallback:

```toml
notify = ["env", "REMINDER_PROTOCOLS=osc777 osc9 bell", "/home/YOU/.codex/hooks/codex-turn-ended-reminder"]
```

## tmux

The script writes directly to tmux client TTYs when possible. If notifications
still do not pass through tmux, enable passthrough:

```tmux
set -g allow-passthrough on
```

Reload tmux:

```bash
tmux source-file ~/.tmux.conf
```

## Test

Run the installed hook:

```bash
~/.codex/hooks/codex-turn-ended-reminder
```

Test only the terminal notification sequence:

```bash
printf '\e]777;notify;Codex;Remote test\a'
```

Test the best-effort `OSC 9` fallback:

```bash
printf '\e]9;Codex: Remote test\a'
```

Cancel any active reminder loop:

```bash
~/.codex/hooks/codex-turn-ended-reminder --cancel
```

Show script help:

```bash
~/.codex/hooks/codex-turn-ended-reminder --help
```

## Notes

- `cmux` understands `OSC 777` notifications from terminal output.
- `OSC 9` support varies by terminal emulator. It is harmless in unsupported
  terminals, but it may simply do nothing.
- Over SSH, the sound/notification is produced by the local terminal stack, not
  by opening a connection from the remote host back to your laptop.
- On local macOS sessions, the hook also plays `REMINDER_SOUND_FILE` with
  `afplay` unless `REMINDER_PLAY_LOCAL_SOUND=0`.
