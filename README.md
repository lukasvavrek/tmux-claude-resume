# tmux-claude-resume

Reboot your machine, start tmux, and every pane that was running [Claude Code](https://claude.com/claude-code) comes back in its folder with **its own** conversation resumed - even when several sessions ran in parallel in the same repo.

[tmux-resurrect](https://github.com/tmux-plugins/tmux-resurrect) already restores sessions, windows, panes and working directories. It can't know *which* Claude conversation lived in which pane, and `claude --continue` only picks the most recent one per directory. This script fills that gap.

## How it works

1. A Claude Code `SessionStart` hook tags the pane with its session id (`@claude_sid` pane option). Fires again on `/clear` and `/resume`, so the tag stays current.
2. A tmux-resurrect post-save hook snapshots `session:window.pane -> session id` to `~/.claude/tmux-panes.tsv`, in sync with resurrect's own save.
3. On restore, resurrect runs this script instead of `claude`. It looks up its pane and runs `claude --resume <id>`, or `claude --continue` if the pane has no id.

One POSIX shell script, no dependencies beyond tmux and Claude Code.

## Install

```sh
git clone https://github.com/lukasvavrek/tmux-claude-resume ~/.tmux-claude-resume
ln -s ~/.tmux-claude-resume/tmux-claude-resume /usr/local/bin/   # or anywhere on PATH
```

**`~/.tmux.conf`** (needs [TPM](https://github.com/tmux-plugins/tpm)):

```tmux
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-resurrect'
set -g @plugin 'tmux-plugins/tmux-continuum'
set -g @continuum-restore 'on'
set -g @resurrect-processes '"claude->tmux-claude-resume"'
set -g @resurrect-hook-post-save-all 'tmux-claude-resume save'
run '~/.tmux/plugins/tpm/tpm'
```

**`~/.claude/settings.json`** (merge into existing `hooks`):

```json
{
  "hooks": {
    "SessionStart": [
      { "hooks": [{ "type": "command", "command": "tmux-claude-resume hook" }] }
    ]
  }
}
```

Reload tmux (`tmux source ~/.tmux.conf`, then `prefix + I` to install plugins). Claude sessions started from now on get tagged automatically.

Continuum saves every 15 minutes. Save manually with `prefix + Ctrl-s`, restore with `prefix + Ctrl-r`.

## Limitations

- Mapping is by pane address (`session:window.pane`) as of the last save. Rearranging panes after a save is fine; it's re-snapshotted next save.
- Flags you launched `claude` with (e.g. `--model`) aren't carried over; the session itself is.
- Sessions started before installing the hook have no tag and fall back to `--continue` until restarted.
- Override the map location with `TMUX_CLAUDE_RESUME_MAP`.

## License

MIT
