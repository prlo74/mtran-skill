---
name: mtran
description: "Use when the user wants to copy, transfer, import, or recreate a Spotify playlist as an Apple Music playlist on their Mac."
---

# MTran — transfer a Spotify playlist to Apple Music

## When to use
The user gives a Spotify playlist link (or names a playlist) and wants the same songs available in Apple Music, e.g. "move this Spotify playlist to Apple Music", "recreate my Spotify playlist in Apple Music", "run MTran on this playlist".

## Prerequisites
- This session must be linked to the user's Mac (the `mcp__remote-devices__*` tools present) with Music.app installed.
- A browser tool (Claude in Chrome, or the built-in browser) must be available to read the Spotify playlist page. No Spotify login is required for a public playlist.
- A shell that actually runs on the Mac itself (e.g. a Desktop Commander-style MCP with `start_process`) is needed for step 3. `device_bash` runs inside an isolated Linux VM on the machine, not macOS itself — it has no `osascript` and cannot control Music.app. If no real-Mac shell is available, do step 3 via UI instead (see step 3 fallback).

## Autonomy
Run the whole workflow end-to-end without pausing for confirmation, unless the user says otherwise for that request. The user may give an explicit stopping point (e.g. "run it until this song '________'" or "just do the first 10") — if so, stop adding tracks once that point is reached and report as usual; otherwise process every track in the source playlist.

## Steps

1. **Get the playlist name and full track list from Spotify — read it, don't eyeball it.**
   - Open the Spotify playlist URL in a browser tab (Claude in Chrome: `tabs_create_mcp` + `navigate`, or the built-in browser's `preview_start`).
   - Read the playlist name and owner from the page header — this exact name becomes the new Apple Music playlist's name (see step 3). Never ask the user to rename it; only deviate if they explicitly asked for a different name.
   - Spotify's web player virtualizes the track list (only visible rows are in the DOM), so scroll down through the list and call `get_page_text` / `read_page` repeatedly, collecting each row's Title + Artist (ignore Album/Date/Duration columns), until every track reported under the playlist title (e.g. "63 songs") has been captured. De-duplicate by comparing accumulated rows, not by assuming one scroll = one page of new rows.
   - Use structured text extraction (`get_page_text`/`read_page`), never screenshots, to build this list — it must be exact strings for the search step later.
   - If the user gave a stopping-point track, truncate the extracted list right after that track.

2. **Get computer access and bring Music.app to the front.**
   - Call `mcp__remote-devices__computer_resolve_access` with app name "Music", then `mcp__remote-devices__computer_request_access` with the identity it returns, and wait for the user's approval.
   - Use `mcp__remote-devices__computer_open_application` (or `computer_app_bring_to_current_space`) to open/focus Music.app.

3. **Create the destination playlist without clicking through the UI.**
   - Use the Spotify playlist's exact name (from step 1) as the new playlist name — always, no confirmation needed.
   - Verified working: run, via a real-Mac shell (e.g. Desktop Commander's `start_process`, NOT `device_bash`),
     `osascript -e 'tell application "Music" to make new user playlist with properties {name:"<PLAYLIST NAME>"}'`
   - This was tested directly against Music.app: playlist creation and deletion via AppleScript both work cleanly. It replaces the New Playlist button, typing the name, and pressing Enter that a human has to do.
   - Fallback if no real-Mac shell is available: create the playlist via the UI instead, using `computer_app_ax_find` to locate the New Playlist control rather than fixed coordinates.

4. **For each track, search Apple Music's catalog and add it to the new playlist via the UI — this step cannot be scripted.**
   Confirmed by direct test: Music.app's AppleScript `add` command only accepts local file/alias references. Passing an Apple Music catalog track (e.g. by its music.apple.com URL) fails with error “-43: File...wasn’t found” — AppleScript has no path to add a streaming catalog track to a playlist. So this step must use UI automation:
   - Use `computer_app_ax_find` to locate the Music app's search field, `computer_app_click` it, and `computer_app_type` "<title> <artist>".
   - Identify the correct result by matching BOTH title and artist text (common one-word song titles return many unrelated results) — use `computer_app_ax_find`/`computer_app_screenshot` to read result rows rather than clicking blind. Prefer an exact or near-exact title+artist match.
   - If the track is already in the user's library (from another playlist or a purchase), that's fine — just add it to the new playlist like any other match, no special-casing needed, though still verify it's the right version (see pitfalls) before adding.
   - Click that result's "+"/"Add" control (or its "..." menu → "Add to Playlist" → the playlist created in step 3).
   - If no confident match exists (ambiguous or missing), skip it and record it as "not found" — don't guess and add the wrong song. Keep going with the remaining tracks; don't stop the run for this.

5. **Report results back to the user.**
   - State the destination playlist name, how many of the N Spotify tracks were successfully added, and list any tracks that couldn't be matched so the user can add them manually.

## Notes / pitfalls
- Never replay fixed mouse coordinates from a prior run — window position and scroll state differ each time. Always re-locate UI elements via `computer_app_ax_find` or a fresh screenshot on every run.
- Explicit-content tags, "(Slowed + Reverb)", remix, and karaoke versions commonly clutter search results — prefer the plain studio version unless the Spotify source track itself is a remix/alternate version.
- When an artist has multiple same-titled tracks (common with short releases), disambiguate using the album name shown on the Spotify side.
- This writes to the user's real Apple Music library — proceed without pausing for approval (per Autonomy above), but never invent or force a match just to avoid reporting a miss.
- If testing this skill again, clean up any throwaway/test playlists created via AppleScript afterward (`delete user playlist "<name>"`) so nothing extra is left in the user's library.
