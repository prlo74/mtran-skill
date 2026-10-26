# mtran

A Claude skill that transfers a Spotify playlist into Apple Music on a Mac.

Give Claude a Spotify playlist link and MTran reads the track list, creates a playlist with the same name in Music.app, then searches Apple Music for each song and adds it.

## What it does

1. **Reads the Spotify playlist** in a browser (no Spotify login needed for public playlists). It scrolls the virtualized track list and extracts exact title and artist text, not screenshots, until every track is captured.
2. **Opens Music.app** on your Mac through computer use, after you approve access.
3. **Creates the Apple Music playlist** with the Spotify playlist's exact name, using AppleScript through a real-Mac shell.
4. **Adds each track** through UI automation: it searches title plus artist, matches on both, and clicks Add. AppleScript cannot add streaming catalog tracks, so this step cannot be scripted.
5. **Reports back** with the playlist name, how many tracks were added, and a list of anything it could not match confidently.

## Behavior worth knowing

- Runs end to end without pausing for confirmation. You can set a stopping point ("just the first 10", or "stop at this song").
- Never guesses: ambiguous or missing tracks are skipped and listed rather than adding the wrong song.
- Prefers the plain studio version over remixes, slowed or karaoke versions unless the Spotify track is one.
- Re-locates UI elements each run instead of replaying fixed coordinates.
- Writes to your real Apple Music library.

## Requirements

- macOS with Music.app.
- A Claude session linked to your Mac (the desktop app bridge, `mcp__remote-devices__*` tools).
- A browser tool (Claude in Chrome or the built-in browser).
- A shell that runs on the Mac itself, such as a Desktop Commander style MCP. A sandboxed Linux shell has no `osascript`; without a real-Mac shell the skill falls back to creating the playlist through the UI.

## Usage

"Move this Spotify playlist to Apple Music: `<link>`" or "run MTran on this playlist."

## Install

This is a standard skill folder containing one `SKILL.md`.

- **Claude Code:** copy this folder to `~/.claude/skills/mtran/`.
- **Claude apps:** zip the folder and upload it as a custom skill in your Claude settings.

## Files

- `SKILL.md`: the skill definition and instructions.
