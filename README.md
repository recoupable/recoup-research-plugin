# Recoup Research Plugin

Agent plugin for research workflows on the
[Recoup](https://recoupable.com) platform. A home for skills, commands,
and helpers that let AI agents run artist research, audience analysis,
trend scans, and market intelligence — turning raw signals into briefs
that feed downstream content and platform work.

This plugin is a **starter skeleton**: manifests are in place so the
marketplace can register the repo, but skills, agents, and commands
will arrive in follow-up releases.

## Install

### Claude Code (CLI)

```bash
claude plugin install https://github.com/recoupable/recoup-research-plugin
```

### Claude Cowork

1. Open the plugin marketplace (puzzle-piece icon in the sidebar).
2. Click **Add custom plugin** and paste:
   `https://github.com/recoupable/recoup-research-plugin`
3. Approve the requested tool permissions.
4. Restart the Cowork session so manifests load.

### Cursor

1. Cursor → Settings → Plugins → **Add custom plugin**.
2. Paste the GitHub URL above.
3. Restart Cursor so `.cursor-plugin/plugin.json` loads.

## Layout

```
.claude-plugin/plugin.json   # Claude Code manifest
.codex-plugin/plugin.json    # Codex manifest
.cursor-plugin/plugin.json   # Cursor manifest
skills/                      # (to be added) research workflow skills
agents/                      # (to be added) research personas
commands/                    # (to be added) slash commands
```

## Roadmap

- Artist research skills (identity, brand, voice, audience).
- Audience and listener analysis (Chartmetric, Spotify, web socials).
- Trend scans across charts, playlists, and social platforms.
- Competitive and label-level market intelligence briefs.

## Support

- Email: `support@recoupable.com`
- Website: <https://recoupable.com>

## License

[Apache-2.0](./LICENSE)
