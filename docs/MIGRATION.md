# Stack migration spec

The plan for moving Budmintan from plain JS on GitHub Pages to TypeScript + Vite + Preact on Cloudflare Pages, one phase at a time, without a broken club night. Written for Claude, who does all development. Owner-facing summary: the Claude Docs doc "แผน Tech Stack ระยะยาว Budmintan" (25 Oct 2026).

## Status

Update this table at every gate. Work only on the first phase that is not Done.

| Phase | Goal | Status |
|---|---|---|
| 0 | Real nights on the current app | In progress (owner) |
| 1 | Cloudflare Pages, own domain, private repo, staging Supabase | Not started |
| 2 | TypeScript + Vite, UI unchanged | Not started |
| 3 | Preact, one tab at a time | Not started |
| 4 | Sell-ready (outline only) | Not started |

## Owner decisions (25 Oct 2026)

- Claude is the only developer. Pick for Claude's speed and safety, not for hiring.
- No App Store or Google Play for now. Stay a PWA.
- Selling will be a monthly subscription per club. Plans are not decided: build nothing plan-specific until the owner decides.
- Target stack: TypeScript (strict), Vite, Preact, Supabase (unchanged), Cloudflare Pages, GitHub Actions.
- **Free plans only until the first paying club** (27 Oct 2026): GitHub Free (private repo, Actions minutes), Cloudflare Pages Free, Supabase Free for production and staging (the two free projects), Gmail SMTP. No domain purchase: Phase 1 uses `<project>.pages.dev`. Buying a domain later means one more URL change and reinstall, so do it once, at Phase 4 at the latest.
- **Free-plan guards.** Supabase Free pauses an inactive project after about a week and has no daily backup: Claude reminds the owner before a long break, and at the end of every month asks the owner to export the database (dashboard backup or SQL dump) and keeps `supabase/migrations/` current. Upgrade to Supabase Pro at the first paying club. Check each service's pricing page before relying on a limit; figures here come from memory.

## Invariants

These hold in every phase. A phase that breaks one is not done.

1. **Green.** Every suite passes: `npm test`, `npm run e2e`, `npm run explore`, `npm run test:db`, `npm run uat`. Record the counts at the start of a phase (the baseline) and end with the same or more passing checks. Last known baseline (22 Oct 2026): unit 26, flows 144, smoke 35, a11y 45, UAT 53 pass / 0 fail / 5 manual, explore 20/20.
2. **Unsynced data survives the upgrade.** A phone may hold a local cache with `dirty: true` written by the old build. The new build reads it and syncs it. Keep these localStorage keys and their meaning: `bmm-cache-<userId>-<clubId>` (`{ S, dirty, … }`, `S.v === 4`), `bmm-club-<userId>`, `bmm-theme`, `bmm-wake`, `bmm-logs`, `bmm-upd`, `bmm-dev`.
3. **Database contract frozen.** Phases 1–3 add no migrations to production. RPC names, arguments and return shapes (`sync_session`, `close_session`, `live_night`, `create_club`, `leave_club`, `delete_club`) and `manage-admins` stay as they are.
4. **supabase-js stays external.** Load it as the pinned UMD script from jsDelivr (`window.supabase`), never bundled. The e2e harness swaps it for `tests/fake-supabase.js` by routing `**/supabase.js`; bundling it would silently test against the real client.
5. **Same behaviour, same look.** Product rules in `CLAUDE.md` still apply (Thai UI, no emoji, 44 px targets, colour tokens, never auto-draw, matching and rating rules). A visible change during a migration phase is a bug unless the owner asked for it.
6. **Safe deploys.** Work on a branch `stack/phase-N[-step]`. Merge to `main` only when green and after the owner says OK. Never merge between 17:00 and 24:00 Asia/Bangkok, when nights run.

## Phase 0: real nights (owner)

Gate to leave: the owner has run 3 real nights on the current app with no serious bug. Claude fixes bugs on the current code during this phase; no stack work.

## Phase 1: hosting

Goal: the same files served from Cloudflare Pages on a stable URL, repo private, a staging database.

Owner-only steps (ask, then wait): accept `<project>.pages.dev` (no domain purchase yet); create the Cloudflare account; connect the GitHub repo in Cloudflare (build command empty, output directory `/`); add the new URL to Supabase → Authentication → URL Configuration (Site URL and Redirect URLs); switch the repo to private; tell admins to reinstall.

Steps:

1. Add `_headers` at the repo root: `/` and `/index.html` and `/live.html` get `Cache-Control: no-cache`; `/js/*`, `/css/*`, `/img/*` get `Cache-Control: public, max-age=31536000, immutable` (safe because every link carries `?v=`).
2. Confirm every path is relative (`manifest.webmanifest` uses `./`; links have no leading `/`), so the app works at `/budmintan/` and at `/`.
3. Staging database: create a second Supabase project (free tier) and apply every file in `supabase/migrations/` in name order, plus `manage-admins`. `js/config.js` picks the project by hostname: production domain → production project, anything else (Cloudflare preview URLs, localhost) → staging.
4. Old address: point GitHub Pages at a branch `gh-pages` holding one `index.html` that says the app moved, links to the new URL and redirects after 3 s, carrying `live.html?t=…` links across. Keep it live for 2 weeks after admins move, then the owner makes the repo private (Pages then stops on the free plan).
5. Update `CLAUDE.md` (Hosting line, staging) and `README.md`.

Done when: the new URL serves the app; `index.html` comes back `no-cache`; sign-in, forgot-password email link and the live QR link work on the new URL; a preview URL talks to staging; the old URL forwards; Status says Done.

Rollback: DNS and Pages settings only; GitHub Pages still holds the old app until step 4.

## Phase 2: TypeScript + Vite, UI unchanged

Goal: a typed, bundled build of the same app. No rendering changes.

Entry: Phase 1 Done; baseline recorded.

Target layout:

```
src/
  core/      algo.ts (from js/algo.js), types.ts (the S v4 shape)
  data/      db.ts (from js/db.js), config.ts
  app/       main.ts (from js/app.js)
  live/      live.ts (from js/live.js)
css/app.css  unchanged
index.html, live.html   Vite entries
```

Steps:

1. Vet and pin exact versions of `vite` and `typescript` with the dependency-vetting skill. Add no other runtime dependency in this phase.
2. `vite.config.ts`: `base: './'`, two inputs (`index.html`, `live.html`), output `dist/`, `define: { __APP_VERSION__ }` = short git sha + build time. Keep `img/`, icons and `manifest.webmanifest` as public files.
3. Port `js/algo.js` to `src/core/algo.ts` as an ES module with named exports and the same names. Write `src/core/types.ts` from the State shape in `CLAUDE.md`. Strict mode, no `any`.
4. Golden test before deleting `js/algo.js`: for the explore seeds and a recorded real night, run old and new algo side by side and assert identical draws (same random source) and ratings equal to 1e-9. Keep the test as `tests/algo.golden.test.ts` with fixtures, not the old file.
5. Port `js/db.js` to `src/data/db.ts` (strict) and `js/config.js` to `src/data/config.ts`. Generate database types with Supabase's type generator and commit them.
6. Move `js/app.js` to `src/app/main.ts` with `// @ts-nocheck` on line 1, imports instead of globals (except `window.supabase`). Same for `live.ts`.
7. Versioning: remove `tools/stamp.js`, `npm run stamp`, `?v=` links and the stamp unit test. `myStamp()` returns `__APP_VERSION__`. `checkUpdate()` keeps its behaviour (fetch `index.html?check=<now>` with `no-store`, reload once per new version, never while an input has focus) but compares the hashed entry script name instead of `?v=`. Error logs keep sending the version.
8. Tests: `tests/algo.test.js` imports from `src/core` (Node 22 runs `.ts` by type stripping; add `--experimental-strip-types` if it does not). The e2e, explore, a11y and UAT servers serve `dist/`; their npm scripts run `npm run build` first. The `**/supabase.js` route stays.
9. Upgrade test (Playwright): build the last pre-phase commit in a git worktree, load it, make a change while offline so the cache is `dirty`, then load the new build on the same origin and assert the change reaches the fake database.
10. Scripts: `typecheck` (`tsc --noEmit`), `build`, `preview`. CI in `.github/workflows/ci.yml` on every push and PR: install, typecheck, unit, golden, test:db, build, e2e, a11y (Playwright 1.56.1 with `npx playwright install --with-deps chromium` in CI; the local sandbox uses `/opt/pw-browsers`).
11. Cloudflare: build command `npm run build`, output `dist`. `_headers` moves to `public/` and drops the `?v=` reasoning (Vite file names carry a hash).
12. Update `CLAUDE.md`: Hosting paragraph (no stamp, hashed assets), Commands, the `algo.js`/`app.js` paths everywhere they are named.

Done when: Invariants hold against `dist/`; golden and upgrade tests pass; `tsc` is clean with strict on `src/core` and `src/data`; one real night has run on the new build; Status says Done.

Rollback: Cloudflare "Rollback to this deployment" on the last Phase 1 build; revert the merge.

## Phase 3: Preact, one tab at a time

Goal: views as Preact components, strict types everywhere, same DOM and look.

Entry: Phase 2 Done and one real night on it.

Pattern (strangler):

- Vet and pin `preact` and `@preact/preset-vite`. Add nothing else without a reason written in this file.
- `src/app/store.ts` holds `S` and `ui` and a `subscribe()`; `changed()` and `render()` notify it. Components read through a `useStore()` hook and change state only through the existing action functions, so undo, sync and the cache keep one path.
- `render()` mounts the Preact view into `#view` for migrated tabs and keeps the `innerHTML` path for the rest. Delegated `data-act` click handlers keep working for unmigrated markup.
- Clock ticks stay in place: a 30 s timer updates only the text of `courtAge`/`waitText`/`hmSince` nodes; nothing remounts, so a tap during the tick still hits its button.
- Keep every id, class and data attribute the tests and CSS use. Changing a selector means changing the test in the same commit, on purpose.
- Move each tab's Thai strings into `src/i18n/th.ts` while migrating it.

Order, one PR each: Settings → People → Rank → Summary → Courts (with banner, dock, toast and header last). Courts goes last because it is used during play.

Each tab is done when: e2e, a11y and UAT are green; before/after screenshots (phone and iPad, light and dark) show no change that was not asked for, checked by looking at them; the tab file has no `@ts-nocheck`.

Phase done when: no `innerHTML` views remain, `@ts-nocheck` is gone, strict everywhere, `CLAUDE.md` describes the component layout, Status says Done.

Rollback: revert the tab's PR; the other tabs are independent.

## Phase 4: sell-ready (outline)

Write a spec section here for each item, and get the owner's OK, before building it.

- Local storage in IndexedDB, reading the old localStorage cache once.
- Sync as an append-only event log replayed into state (ratings are already replayed), replacing last-save-wins between two admins.
- Monthly billing per club once the owner picks plans: `clubs.plan`, `clubs.paid_until`, payment webhook in an Edge Function.
- Self sign-up with email verification and rate limits; privacy page; club data export and deletion (PDPA).
- A service worker for offline start, with an explicit update prompt.
- Supabase Pro backups and an uptime check.

## Owner-only actions

Anything involving money, accounts, DNS, dashboard settings, repo visibility or deploy timing. Ask in one short Thai message listing exactly what to click, then wait.
