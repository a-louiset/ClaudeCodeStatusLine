# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

This repo ships exactly two scripts — `statusline.sh` (Bash, macOS/Linux) and `statusline.ps1` (PowerShell 5.1+, Windows) — that Claude Code invokes as its `statusLine.command`. Both scripts are functionally equivalent ports of each other and **must stay in sync**: any behavioral change to one needs the same change in the other, and the `VERSION` constant at the top of both files must match (`1.4.2` at the time of writing).

There is no build step, no package manager, no test suite, and no linter. Releases are cut by bumping `VERSION` in both scripts on a `chore/bump-version-X.Y.Z` branch and merging via PR — see recent `git log` for the cadence. The install path users follow lives in `INSTALL.md`, which is written as a runbook for Claude Code to execute when a user asks to install/update; treat it as the source of truth for install instructions and keep `README.md` aligned with it.

## Running and testing locally

The script reads a JSON payload on stdin (the same shape Claude Code passes at runtime) and writes ANSI-colored output to stdout. To exercise it manually:

```bash
echo '{"model":{"display_name":"Opus 4.7 (1M context)"},"cwd":"'"$PWD"'","context_window":{"context_window_size":1000000,"current_usage":{"input_tokens":50000,"cache_read_input_tokens":100000}}}' | ./statusline.sh
```

Empty stdin must print the literal `Claude` and exit 0 — that is the fallback Claude Code shows before a session has data.

To force-refresh during testing, delete the caches: `rm -rf /tmp/claude/statusline-*` (or `$env:TEMP\claude\statusline-*` on Windows). Cache TTLs are 60s for usage data and 24h for the GitHub release check.

## Architecture: how a single render works

Each render is one process invocation. The flow is:

1. **Parse stdin JSON** — model name (strip `(1M context)` suffix into a trailing token), context window size, and token counts (`input_tokens + cache_creation_input_tokens + cache_read_input_tokens`).
2. **Pick a rate-limit data source.** Claude Code embeds `rate_limits.{five_hour,seven_day}` directly in stdin JSON on recent versions — that is the preferred source ("builtin" path). When builtin values are all zero **and** reset timestamps are missing, this is treated as an upstream API failure and the script falls back to the cached API response rather than displaying misleading 0%. Genuine post-reset zeros include valid `resets_at` timestamps and are trusted.
3. **Always refresh the OAuth `/api/oauth/usage` cache when stale**, even if builtin data is used — because `extra_usage` (paid-overage credits) is only exposed through that endpoint, never via stdin. The fetch is throttled by `cache_max_age` (60s) and stampede-locked across concurrent panes by `touch`ing the cache file before the curl call. If the fetch produces no valid JSON, the empty sentinel is removed so the next render retries.
4. **Render** model | cwd@branch (+/-) | tokens | effort | 5h | 7d | extra, then optionally append a second line if a newer GitHub release exists.

The cache file is namespaced by a SHA-256 prefix of `CLAUDE_CONFIG_DIR` so multiple Claude installs (different config dirs) don't share a single cache.

## OAuth token resolution

`get_oauth_token` (sh) / its PS counterpart try sources in a specific order, and this order matters:

1. `$CLAUDE_CODE_OAUTH_TOKEN` env var (explicit override).
2. **macOS Keychain** via `security find-generic-password`. Service name is `Claude Code-credentials`, but when `$CLAUDE_CONFIG_DIR` is set, Claude Code appends an 8-char SHA-256 prefix of that path: `Claude Code-credentials-<hash>`. Match this exactly.
3. `$CLAUDE_CONFIG_DIR/.credentials.json` (Linux plaintext fallback).
4. GNOME Keyring via `secret-tool lookup service "Claude Code-credentials"` (with a 2s `timeout` to avoid hanging on a locked keyring).

Any change to how Claude Code stores credentials needs all four paths revisited.

## Cross-platform pitfalls to respect

- **`date` command differs between GNU (Linux) and BSD (macOS).** The script tries GNU first (`date -d "@$epoch"`), falls back to BSD (`date -j -r "$epoch"`), and accepts whichever succeeds. Do not pipe `date` output through `sed`/`tr` for fallback chaining — the last pipe stage masks the exit code and the GNU fallback never runs. See `format_reset_time` for the correct pattern.
- **`stat`** also differs: `stat -c %Y` (GNU) vs `stat -f %m` (BSD). Always try both.
- **`shasum` vs `sha256sum`** — same dual-fallback pattern.
- **PowerShell 5.1 compatibility** is required (Windows 10/11 default). Do not use PS7-only features: no `??` null-coalescing (use the `Coalesce` helper), no `` `e `` ANSI escape (use `[char]0x1b`).
- **ISO 8601 parsing on BSD** must strip fractional seconds, trailing `Z`, and `±HH:MM` offsets before feeding to `date -j -f`. UTC timestamps must be parsed with `env TZ=UTC` so local-time conversion works.

## Color thresholds

`usage_color` / `Get-UsageColor` is the single source of truth for percentage coloring: green <50, yellow ≥50, orange ≥70, red ≥90. Apply this to every percentage segment (5h, 7d, extra) — do not hardcode a different palette per segment.
