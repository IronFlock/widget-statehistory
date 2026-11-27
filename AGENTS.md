# widget-statehistory development instructions

## Working agreements

The organization maintains these agreements in the IronFlock `.github` repository at `.ai/working-agreements.md`.

- Check the branch and working tree before editing. On main/master, create a
  topic branch; otherwise keep the current branch. If another session owns
  the branch or has uncommitted edits, ask where this work belongs before editing.
  Do not create a worktree without explicit consent.
- Preserve other sessions' changes. Re-read a file immediately before changing
  it; do not overwrite a file that appeared after inspection.
- Commit or push only when Marko explicitly requests it for this piece of work.
  Do not amend, merge, rebase, tag, stash, or change remote settings implicitly.
  Never add attribution trailers or generated-by notices to Git messages or PRs.
- Production changes, releases, and registry pushes require explicit authorization
  for the exact action. Do not hide production mutations in validation recipes.
- Never print secret-file contents or copy credentials into project knowledge.
  Do not describe private signing-key whereabouts in public documentation.
- For code changes, add user-visible release notes for the version that will
  actually ship them, preserving existing entries. Documentation-only setup,
  tests, and invisible refactoring need no invented release-note entry.

## Shared knowledge

Store durable, verified project facts in the repository documentation; store task status and
remaining work in `docs/handoff.md`. Include evidence paths and distinguish
observations from assumptions. Shared files are reviewed repository content,
not an automatic mirror of either agent's private memory or chat history.

Skills live in `.agents/skills/`; `.claude/skills/` contains relative links to
those same folders. Edit the canonical skill, not a duplicate. Tool permissions,
credentials, hooks, and agent-specific runtime settings remain separate.

Read the relevant repository documentation when starting work. When resuming work, read the relevant
handoff and verify it against the current branch and files. Record checks
and remaining work there when handing a task to another developer or agent.

## Commands

- `npm run start` — Vite dev server on port 8000 serving `demo/index.html`, with `vite build --watch` running in parallel so `dist/` stays current.
- `npm run build` — Production build via Vite (library mode, ES output, `echarts` externalized).
- `npm run watch` — `vite build --watch` only, no dev server.
- `npm run types` — Regenerate `src/definition-schema.d.ts` from `src/definition-schema.json` using `json-schema-to-typescript`. Run this whenever the JSON schema changes.
- `just lint` — what the CI gate checks besides the build: regenerating the committed generated files (`npm run types`) changes nothing. `just build` = `npm run build`.
- `just merge` / `just release-patch` (`release-minor`, `release-major`) — pull requests into `main` from a feature branch with everything committed: a no-release one, or a release one whose merge publishes to npm in CI (`.github/workflows/build-publish.yml`), then the release GitHub App tags the bare version and creates the GitHub release. Never `npm publish`, tag or edit the version by hand.
- `npm run link` / `npm run unlink` — Build and `npm link` this package into the studio checkout at `../../studio` (the host app, `~/git/ironflock/studio`), or put the published package back there. Only studio's appliance build (`build:appliance`) reads the widget from `node_modules`: its dev server (`npm run dev`) already aliases `widget-statehistory` to this checkout's `dist/` and serves its `src/*.json`, so `npm run watch` here is enough for dev. Studio tracks both `package-lock.json` and `bun.lock` (its Dockerfile runs `bun install --frozen-lockfile`), and `unlink` reinstalls with npm only: run `bun install` in studio as well if that moved the version.

No test runner, linter, or formatter scripts are wired up. `.prettierrc` exists but is not invoked via npm.

Node `>=24.9.0`, npm `>=10.0.2` (enforced via `engines`). The CI workflow uses Node 24.

## Architecture

This repo is one entry in a family of IronFlock dashboard widgets (`../widget-*` siblings). Each widget is a standalone Lit web component published to npm and consumed by the IronFlock platform's dashboard frontend.

### Entry point and versioned tag

- Single source: `src/widget-statehistory.ts` exporting `WidgetStateHistory extends LitElement`.
- The custom element tag is registered as `widget-statehistory-versionplaceholder`. At build time, `@rollup/plugin-replace` (configured in `vite.config.ts`) substitutes `versionplaceholder` with the current `package.json` version, producing tags like `widget-statehistory-1.0.13`.
- Consumers must keep the version-suffixed tag in sync with the installed package version. `demo/index.html` resolves this dynamically via `unsafeStatic` from `package.json`.

### Inputs and platform contract

The widget receives two reactive Lit `@property` objects from the host dashboard:

- `inputData: StateHistoryConfiguration` — typed via `src/definition-schema.d.ts`, generated from `src/definition-schema.json`. The JSON schema is the authoritative contract used by the IronFlock dashboard editor to render the widget's configuration UI; descriptions in it are intentionally written for AI agents and end users (see commit `f2e58ea`). Edit the JSON, then run `npm run types`.
- `theme: { theme_name, theme_object }` — light/dark theme objects (see `demo/themes/`).

Theming: `registerTheme()` resolves the text colour as a `var(--re-text-color, <theme value>)` chain rather than reading the host's custom properties through `getComputedStyle`. The host property still wins over `theme_object`, but nothing is snapshotted, so a board style edit repaints the tile live. The widget paints no background of its own: studio's `widget-item` already paints `--re-tile-background-color` on the tile, and painting it again here composited a translucent tile colour twice.


`src/default-data.json` provides the demo payload.

### Rendering

Uses ECharts via tree-shaken `echarts/core` plus `CustomChart`, `CanvasRenderer`, and the Title/Tooltip/Grid/DataZoom/Legend/Toolbox components. State bars are drawn with a custom `renderItem`. `echarts` is a `peerDependency` and externalized in the Rollup output (`external: [/^echarts/]`) — the host page provides it.

Two things to keep in mind when touching `applyData()`:

- **Every component used in the option template must be in `echarts.use([...])`.** Importing it is not enough. An unregistered component makes ECharts log `Component X is used but not imported` and silently drop it — which is how the zoom toolbox's `restore` button went missing despite being fully configured.
- **Write single-instance components through `setComponent()`, never as `option.<key>.<prop> = …`.** Only the first draw uses the plain-object template; every later draw re-reads the option via `getOption()`, which normalises `title`, `tooltip`, `xAxis`, `legend` and `toolbox` to one-element **arrays**. A direct assignment then sets a property on the Array and is silently ignored, so config changes appear to do nothing until the chart is recreated. (`widget-linechart` avoids this differently — it keeps a config fingerprint and does a `notMerge` rebuild from the template whenever any of those values change.)

### Build pipeline

Vite library mode (`build.lib`, ES format, fileName `widget-statehistory`) emits `dist/widget-statehistory.js` plus type declarations. The output banner adds the Record Evolution copyright. `process.env.NODE_ENV` is hard-coded to `production` in `vite.config.ts`.

### Release flow (from README)

1. `just release-patch` opens a release pull request; its merge runs `.github/workflows/build-publish.yml`: `npm ci`, build, publish via npm trusted publishing (OIDC — no `NPM_TOKEN`), then the bare-version tag and the GitHub Release.
2. After publish, register the new version with the platform via SQL:
   ```sql
   select swarm.f_update_widget_master('{"package_name": "widget-statehistory", "version": "X.Y.Z"}'::jsonb);
   ```
3. Run `npm run build` locally so the version-stamped tag matches before restarting the local node web server container.
4. If the widget appears in `dashboard-template.yml`, bump its version there too.

## `aiSelection` in `src/definition-schema.json`

The schema root carries an `aiSelection` block next to `title` and `description`. It is **not** JSON Schema and describes no config field — it exists so the IronFlock AI's Widget Builder can pick the right widget for a given shape of data, using knowledge only the widget author has:

```jsonc
"aiSelection": {
  "dataShape": "…what columns this widget consumes and what each one means…",
  "useWhen":   ["…a situation, naming the properties that express it…"],
  "notFor":    ["…a situation this widget is wrong for, naming the widget to use instead…"]
}
```

It is inert everywhere else, and must stay that way: `json2ts` ignores it (the generated `.d.ts` is byte-identical with and without it), the dashboard config editor renders only `schema.properties`, and the AI service's `validate_widget` validates *configs* against the schema, skipping unknown Draft-7 keywords.

When maintaining it:

- `notFor` is the high-value half and the part plain descriptions always omit. Every entry must name the widget that *should* be used, or it rejects without routing.
- Write for an LLM with no other documentation: describe the visible result and the user's intent, not the implementation.
- Prefer entries that discriminate against a *neighbouring* widget. Generic rejections are cheap; the ones that pay are those an author could plausibly get wrong.
- The `notFor` lists are a set across all `widget-*` repos and are meant to be reciprocal — if this widget routes to another for some case, that widget should usually route back for the converse. Changing one side is a cue to check the other.
- Update it whenever a property changes what this widget can *do*, not just how it looks.
