# Claude Code Plugin Template

This is a template repo. When a new plugin is created from it (`gh repo create --template fnrhombus/claude-code-plugin-template`), this file tells Claude (and future-you) what to do.

## First-time setup for a new plugin

After cloning the new repo:

1. **Search-and-replace `TODO-plugin-name`** in `.claude-plugin/plugin.json`, `README.md`, and `package.json` with the actual plugin name (kebab-case, e.g. `claude-code-thingfix`).
2. **Search-and-replace `TODO-repo-name`** with the actual GitHub repo name (usually the same as the plugin name).
3. **Update `.claude-plugin/plugin.json`** with a real `description`.
4. **Write the actual plugin code** in `src/index.ts`. Use [`@fnrhombus/claude-code-hooks`](https://github.com/fnrhombus/claude-code-hooks) for the typed hook wrapper — it handles all the stdin/stdout/envelope plumbing.
5. **Update `hooks/hooks.json`** if the hook event or matcher is different from the default `PreToolUse` / `Bash`.
6. **Rewrite `README.md`** as *advertising* — lead with the problem, show the before/after, keep install minimal. See `fnrhombus/claude-code-pathfix` for a reference.
7. **Add the `claude-code-plugin` topic** to the repo: `gh repo edit --add-topic claude-code-plugin`. This is how `fnrhombus/claude-plugins` (the central marketplace) discovers the plugin — without this topic, the plugin will never show up in `/plugin install`.
8. **Add the `AUTOMERGE_PAT` repository secret.** Required for the release-please + auto-merge + marketplace-dispatch chain. The PAT needs `repo` + `workflow` scope, and `workflow` scope on `fnrhombus/claude-plugins` so the dispatch step can fire the marketplace rebuild. `gh secret set AUTOMERGE_PAT --body "<token>"`.
9. **Commit + push** to `main`.

## Publishing a new version

Use [conventional commits](https://www.conventionalcommits.org/). `feat:` bumps minor, `fix:` bumps patch, `feat!:` (or `BREAKING CHANGE:` in the body) bumps major. `docs:`, `refactor:`, `chore:`, `ci:` don't bump.

The flow is fully automatic:

1. Merge a `feat:` or `fix:` PR to `main`.
2. `release-please.yml` opens (or updates) a `chore(main): release vX.Y.Z` PR with the proposed bump + auto-generated `CHANGELOG.md` entry.
3. `auto-merge.yml` enables auto-merge on that PR; once required checks pass it merges.
4. The merge triggers release-please's second run, which creates the tag and a GitHub Release.
5. The same workflow then dispatches `update-marketplace.yml` on `fnrhombus/claude-plugins`, so the new version lands in the marketplace within seconds.

No manual version bumps, no manual tagging, no manual marketplace pokes.

## Why a PAT?

Two cross-trigger requirements force the use of `AUTOMERGE_PAT`:

1. **In-repo workflow chaining.** GitHub suppresses workflow runs for `GITHUB_TOKEN`-actored events to prevent loops. The release PR has to be opened by a PAT-actored event so its merge can trigger downstream workflows (the tag-creating second run of release-please, plus this repo's `test`/`build` jobs if any).
2. **Cross-repo dispatch.** `GITHUB_TOKEN` is scoped to its own repo; it can't dispatch a workflow on `fnrhombus/claude-plugins`. The PAT bridges that boundary so the marketplace rebuild fires immediately on release instead of waiting for its daily cron.

## What NOT to do

- **Don't hand-edit `fnrhombus/claude-plugins/.claude-plugin/marketplace.json`.** Both the daily cron and the per-release dispatch overwrite it. Update the source (`.claude-plugin/plugin.json` here, bumped automatically by release-please) and let the dispatch propagate.
- **Don't put `"hooks": "./hooks/hooks.json"` in `.claude-plugin/plugin.json`.** Claude Code auto-loads `hooks/hooks.json`; listing it again causes a duplicate-load error. The `hooks` field is only for *additional* hook files beyond the standard one.
- **Don't forget the `claude-code-plugin` topic.** Without it, the marketplace has no way to discover the repo, and users can't install the plugin.
- **Don't skip the `dist/` commit.** Plugins distributed via `/plugin install` are served directly from the GitHub repo contents — there's no build step on the user side. If this plugin has a build step (tsup, etc.), commit `dist/` alongside `src/` so the plugin hook can actually run what it advertises.
