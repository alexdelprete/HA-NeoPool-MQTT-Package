# Claude instructions — HA-NeoPool-MQTT-Package

Project-level guidance for Claude sessions working in this repo.

## "tag and release" command

When the user says **"tag and release"**, run the **entire** release flow end-to-end without further confirmation prompts. The flow assumes the `[Unreleased]` section of `CHANGELOG.md` already contains the changes for the new version (typically populated by prior commits in this session).

### 1. Pick the new version

- Read the current version from line 1 of `ha_neopool_mqtt_package.yaml` (format: `# HA NeoPool MQTT vN.M`).
- Inspect the `[Unreleased]` section in `CHANGELOG.md`:
  - **Breaking entity removals/renames** → major bump (`vN.0` where `N = old_major + 1`).
  - **Additive entities or behavior changes** → minor bump (`vN.(M+1)`).
  - **Bug fixes / docs only** → minor bump too. This project does not use patch-level versions; the scheme is two-component `major.minor` only (precedent: v3.5 → v4.0 → v5.0 → v5.1).
- If the `[Unreleased]` section is empty or missing, abort and tell the user there is nothing to release.

### 2. Update files

- `ha_neopool_mqtt_package.yaml` line 1: `# HA NeoPool MQTT v<old>` → `# HA NeoPool MQTT v<new>`.
- `CHANGELOG.md`: `## [Unreleased]` → `## [v<new>] — <YYYY-MM-DD>` (today's date).

### 3. Commit

Commit message style: **plain imperative, no conventional-commit prefix** (matches existing precedent like commit [`286702e`](https://github.com/alexdelprete/HA-NeoPool-MQTT-Package/commit/286702e) "Update version from v4.0 to v5.0 in YAML file"):

```text
Update version from v<old> to v<new> in YAML file

<one-paragraph summary derived from the CHANGELOG section's bullets —
not a verbatim copy>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

Stage **only** the two changed files (never `git add -A`):

```bash
git add ha_neopool_mqtt_package.yaml CHANGELOG.md
git commit -m "$(cat <<'EOF'
...message via HEREDOC...
EOF
)"
```

### 4. Push the commit

```bash
git push origin main
```

The default branch is `main`. This repo doesn't use PRs — push directly.

### 5. Create an **annotated** tag

```bash
git tag -a v<new> <commit-sha> -m "$(cat <<'EOF'
v<new> — <short title from CHANGELOG, e.g. "Hydrolysis percent display fixes">

<2-3 line summary referencing the major changes>

See CHANGELOG.md for the full list.
EOF
)"
```

Always **annotated** (`-a`), never lightweight. Always reference the specific commit SHA (the version-bump commit just created).

### 6. Push the tag

```bash
git push origin v<new>
```

### 7. Create the GitHub Release

Extract the CHANGELOG section for the new version and use it as release notes:

```bash
python -c "
import re, pathlib
text = pathlib.Path('CHANGELOG.md').read_text(encoding='utf-8')
m = re.search(r'## \[v<new>\] — .*?\n(.*?)(?=\n## \[|\Z)', text, re.S)
print(m.group(1).strip() if m else 'See CHANGELOG.md')
" > .git/release-notes-v<new>.md

gh release create v<new> \
  --title "v<new>" \
  --notes-file .git/release-notes-v<new>.md

rm .git/release-notes-v<new>.md
```

Notes:

- `.git/` is git-ignored, so the temp file doesn't accidentally end up tracked.
- `gh` CLI is already authenticated for this repo (scope `repo`).
- Do **not** mark prereleases unless the user explicitly says so.

### 8. Report back

After the workflow completes, summarize:

- The new version
- The version-bump commit SHA (short form)
- The tag (annotated, points to the commit SHA)
- The GitHub Release URL (returned by `gh release create`)

## Other project conventions

- **No pre-commit hooks** in this repo (no `.pre-commit-config.yaml`, no `.git/hooks/pre-commit`). The companion integration repo (`ha-sugar-valley-neopool`) has them; this one doesn't.
- **No linter configs**: validate YAML with `python -c "import yaml; yaml.safe_load(open(...))"` after edits to the package file.
- **Commit style**: plain imperative ("Add X", "Fix Y", "Update Z"). No `feat:` / `fix:` prefixes. The companion integration does use conventional commits — don't carry that over.
- **Remote URL**: `https://github.com/alexdelprete/HA-NeoPool-MQTT-Package.git` (the repo was renamed from `HA-NeoPool-MQTT` in March 2026; local remote was updated to the new URL).
- **Docs**: `docs/TASMOTA_NEOPOOL_DRIVER_REFERENCE.md` (driver internals) and `docs/PARITY_WITH_INTEGRATION.md` (parity matrix with the Python integration) are the two reference docs. Keep them in sync when adding/removing entities.

## Companion integration

The Python integration at `d:\OSILifeDrive\Dev\HASS\- My Integrations -\ha-sugar-valley-neopool` is the architectural leader. When designing new entities for this package, mirror the integration's `EntityDescription` definitions. See `docs/PARITY_WITH_INTEGRATION.md` for the current parity matrix.
