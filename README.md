# poke-mon

```text
 .================================================================.
 |  ____   ___  _  _______      __  __  ___  _   _               |
 | |  _ \ / _ \| |/ / ____|    |  \/  |/ _ \| \ | |              |
 | | |_) | | | | ' /|  _| _____| |\/| | | | |  \| |              |
 | |  __/| |_| | . \| |__|_____| |  | | |_| | |\  |              |
 | |_|    \___/|_|\_\_____|    |_|  |_|\___/|_| \_|              |
 '================================================================'
        [::] terminal reminder unit online [::]
```

`poke-mon` exists because coding sessions are easy to drift away from. Codex
finishes a turn, asks for approval, or waits for the next prompt, and ten
minutes later you realize the session has been sitting idle the whole time.

This hook uses native macOS system sounds, terminal notifications, or both to
gently nudge you back when Codex needs your attention. The first nudge can be
subtle, and later reminders can taper into more noticeable sounds if you are
actually away from the keyboard.

It is intentionally small: a single shell script you can drop into
`~/.codex/hooks`.

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
cp ./poke-mon ~/.codex/hooks/poke-mon
chmod +x ~/.codex/hooks/poke-mon
```

Add the hook to `~/.codex/config.toml`.

Linux:

```toml
notify = ["/home/YOU/.codex/hooks/poke-mon"]
```

macOS:

```toml
notify = ["/Users/YOU/.codex/hooks/poke-mon"]
```

Replace `YOU` with the username on that machine, then restart Codex.

## One-Shot Install Helper

From the repo directory:

```bash
mkdir -p ~/.codex/hooks
cp ./poke-mon ~/.codex/hooks/poke-mon
chmod +x ~/.codex/hooks/poke-mon

HOOK_PATH="$HOME/.codex/hooks/poke-mon"

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
            "command": "env REMINDER_MESSAGE='Waiting for input' REMINDER_ONCE=1 REMINDER_STDOUT_FALLBACK=0 /home/YOU/.codex/hooks/poke-mon"
          }
        ]
      }
    ]
  }
}
```

macOS command path:

```json
"command": "env REMINDER_MESSAGE='Waiting for input' REMINDER_ONCE=1 REMINDER_STDOUT_FALLBACK=0 /Users/YOU/.codex/hooks/poke-mon"
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
| `REMINDER_STATE_DIR` | `$XDG_STATE_HOME/poke-mon` or `~/.local/state/poke-mon` | State directory for token, PID, and log files |
| `REMINDER_PLAY_LOCAL_SOUND` | `auto` | Set to `0`, `false`, `no`, or `off` to disable local sound |
| `REMINDER_SOUND_FILE` | `/System/Library/Sounds/Ping.aiff` | macOS sound used with `afplay` |
| `REMINDER_SOUND_FILES` | unset | Space-separated sound sequence for immediate + reminder notifications |
| `REMINDER_STDOUT_FALLBACK` | `1` | Set to `0` when stdout has protocol meaning |
| `REMINDER_PROTOCOLS` | `osc777 osc9` | Space-separated terminal protocols: `osc777`, `osc9`, `bell` |
| `REMINDER_CMUX_NOTIFY` | `1` | Set to `0` to skip direct `cmux notify` |

Example custom `notify` config:

```toml
notify = ["env", "REMINDER_MESSAGE=Waiting for input", "REMINDER_DELAYS=60 180 600", "/home/YOU/.codex/hooks/poke-mon"]
```

To use only cmux notifications:

```toml
notify = ["env", "REMINDER_PROTOCOLS=osc777", "/home/YOU/.codex/hooks/poke-mon"]
```

To add a plain terminal bell fallback:

```toml
notify = ["env", "REMINDER_PROTOCOLS=osc777 osc9 bell", "/home/YOU/.codex/hooks/poke-mon"]
```

To use only a local macOS sound and skip cmux's notification sound:

```toml
notify = ["env", "REMINDER_CMUX_NOTIFY=0", "REMINDER_PROTOCOLS=", "REMINDER_SOUND_FILE=/System/Library/Sounds/Morse.aiff", "/Users/YOU/.codex/hooks/poke-mon"]
```

To make later reminders progressively more noticeable:

```toml
notify = ["env", "REMINDER_CMUX_NOTIFY=0", "REMINDER_PROTOCOLS=", "REMINDER_SOUND_FILES=/System/Library/Sounds/Tink.aiff /System/Library/Sounds/Tink.aiff /System/Library/Sounds/Pop.aiff /System/Library/Sounds/Funk.aiff /System/Library/Sounds/Sosumi.aiff", "/Users/YOU/.codex/hooks/poke-mon"]
```

When there are more reminders than listed sounds, the final listed sound is
reused.

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
~/.codex/hooks/poke-mon
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
~/.codex/hooks/poke-mon --cancel
```

Show script help:

```bash
~/.codex/hooks/poke-mon --help
```

## Notes

- `cmux` understands `OSC 777` notifications from terminal output.
- `OSC 9` support varies by terminal emulator. It is harmless in unsupported
  terminals, but it may simply do nothing.
- Over SSH, the sound/notification is produced by the local terminal stack, not
  by opening a connection from the remote host back to your laptop.
- On local macOS sessions, the hook also plays `REMINDER_SOUND_FILE` with
  `afplay` unless `REMINDER_PLAY_LOCAL_SOUND=0`.
