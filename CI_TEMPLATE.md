# CI template: one core, three variants

One guide for every repo. Write the core once, then add the small delta for your kind of repo.

- Library shape and publish detail: [`LIBRARY_CONVENTIONS.md`](LIBRARY_CONVENTIONS.md) §5 and [`PUBLISH_TEMPLATE.yml`](PUBLISH_TEMPLATE.yml) (the canonical publish workflow, copy it, do not retype it here).
- Container shape: [`CONTAINER_MCP_CONVENTIONS.md`](CONTAINER_MCP_CONVENTIONS.md).
- Test standards: [`AGENT_RULES.md`](AGENT_RULES.md) (testing sections).
- Watching a deployed app: [`PROD_WATCH.md`](PROD_WATCH.md).

Status of each variant, so nothing is oversold:

- **JS library:** in use in all 7 bare-suite repos (bareloop, bareguard, bareagent, litectx, barebrowse, baremobile, fwdloop).
- **Docker image:** in use in `beeperbox` (build + test + GHCR) and `plato` (GHCR publish). Not part of the 7.
- **Web app:** not yet used in the suite. No repo has a web-app workflow. Derived from the library shape.

---

## 1. Principles

1. **One gate, on push and PR.** `ci.yml` runs on every push and every pull request. Nothing else gates a merge. (bareguard, barebrowse, baremobile, fwdloop limit push to `main`; the rest run on all pushes.)
2. **Order: install, typecheck, build, test.** `npm ci` then `typecheck` then `build:types` then `test`. Cheapest failure first.
3. **No lint step.** `tsc` with `checkJs` + `strictNullChecks` catches the bug class that matters. ESLint adds devDeps, a config and a new failure surface, against the dependency hierarchy.
4. **Hermetic, token-free tests.** Tests use their own tmp fixtures and scripted stub providers. The whole suite runs with no LLM, no API key and no network. CI has no secrets for tests.
5. **Pin everything.** Actions to a major tag (`@v6`), Node to a major (`22`). Never `latest`, never an unpinned action.
6. **`npm ci` plus cache.** Lockfile install only. Use `cache: npm` on `setup-node` where the repo wants speed (bareloop, litectx).
7. **Publishing is separate.** `publish.yml` is `workflow_dispatch` only. You publish when you want, never on merge or tag.
8. **OIDC trusted publishing, no long-lived tokens.** `id-token: write`, no `NPM_TOKEN`. npm must be 11.5.1 or newer, so pin `npm@11` (npm 12 `--provenance` is broken, see beeperbox `publish.yml`). Publish is idempotent (skips a version already on the registry) and checks the end state, not npm's exit code.
9. **Publish repeats the gate.** typecheck and test run again before publish, so a bad commit cannot ship. Typed libraries also run an adopter gate (pack, install into a clean consumer, compile a quickstart).
10. **A CI fix lands in every workflow that runs the same step.** bareloop's ripgrep install landed in `ci.yml` only, and `publish.yml` then failed on the identical cause. Grep all workflow files before closing a CI fix.
11. **`fetch-depth: 0` only when tests read history.** bareloop needs it because a test runs `git show <old-sha>:file`; at depth 1 the guard skipped and reported `ok` while twelve siblings failed. Every other repo keeps the default.
12. **System deps are installed explicitly.** The runner image does not have ripgrep, fonts or a configured chromium. Install them in the workflow and name the reason in a comment. Without ripgrep, litectx `impact()` silently reads 0 callers.
13. **Quote the exit code and the last line.** A green tally can sit above a non-zero exit (bareloop's `npm test` runs a check after the suite). A pipeline's `$?` is the last command's, not the suite's. Never claim green from a tally alone.
14. **Push the branch early.** CI is the only different-machine gate. A local green can be blind to CI. Push once the build is green, before review and release. Watch it with `gh pr checks <n> --watch`, and match the run's `headSha` to HEAD (it can show the previous commit's green).
15. **Least privilege.** `permissions: contents: read` at the top. Add `id-token: write` (publish) or `packages: write` (GHCR) only on the job that needs it.

---

## 2. Core job (write once)

This is `ci.yml` for a JS repo. Variants below add to it.

```yaml
# Push / PR gate: npm ci -> typecheck -> build:types -> test.
# No lint step (tsc checkJs + strictNullChecks covers it). Tests are hermetic and token-free.
name: CI

on:
  push:
  pull_request:

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        # with:
        #   fetch-depth: 0   # ONLY if a test reads git history (bareloop). Say why here.
      - uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: npm
      # System deps, only if tests need them. Name the reason.
      # - run: sudo apt-get update -qq && sudo apt-get install -y -qq ripgrep
      - run: npm ci
      - run: npm run typecheck     # tsc --noEmit: a JSDoc/code mismatch blocks merge
      - run: npm run build:types   # prove the .d.ts really generate from the JSDoc
      - run: npm test              # node --test, hermetic, token-free
```

Optional extra gates seen in the suite (add as one `run:` line each, before `npm test`):

- `npm run check:primitives` (bareguard, bareagent, litectx): the primitives manifest must match the JSDoc.
- `npm run check:lockfile` (bareguard): lockfile version matches `package.json`.
- `npm run check:types` (baremobile): adopter-visible declarations are not defective.

### Publish (`publish.yml`)

Copy [`PUBLISH_TEMPLATE.yml`](PUBLISH_TEMPLATE.yml) as is. Fill only its PER-REPO block (the adopter quickstart). Identical across all 7 repos apart from the knobs in section 4. Do not rewrite it here.

---

## 3. Variants (deltas on the core)

### (a) JS library (npm)

The core above is the whole `ci.yml`. Add `publish.yml` from the template. One-time setup: configure the trusted publisher at npmjs.com (package settings, GitHub Actions, your repo and `publish.yml`). The very first publish that reserves the name is a manual token publish, because trusted publishing needs an existing package (beeperbox notes this).

Tests that need a browser (barebrowse, fwdloop):

- barebrowse: CI runs only the unit tests (`npm run test:unit`); integration needs a real browser and runs locally.
- fwdloop: CI runs the browser tests, with three extra steps before `npm test`:

```yaml
      - name: Install ripgrep and fonts
        run: sudo apt-get update && sudo apt-get install -y ripgrep fonts-symbola fonts-freefont-ttf
      - name: Let chromium use its sandbox (Ubuntu 24.04 blocks unprivileged userns)
        run: sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
      - name: Warm up chromium (first start on a fresh runner is slow)
        run: |
          for i in 1 2; do
            timeout 120 /usr/bin/chromium-browser --headless=new --no-sandbox --disable-gpu --dump-dom about:blank > /dev/null 2>&1 || echo "start $i exited non-zero"
          done
```

Cross-OS matrix (bareguard only, because it ships OS-sensitive code). Split a `typecheck` job from a `test` job:

```yaml
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node: [22, 24]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npm test
```

Use a matrix only when the library has real OS or Node-version risk. The default is one Ubuntu job, Node 22.

### (b) Web app (build + test; deploy is separate and manual)

Not yet used in the suite. Derived from the library shape. The core stays; add a build step and a smoke step, and keep deploy out of `ci.yml`.

```yaml
      # after npm ci / typecheck, replacing build:types if there is no published .d.ts
      - run: npm run build          # the production build must succeed
      - run: npm test               # hermetic: stub the network and any LLM
      # Optional: serve the build and hit it once, to prove it starts.
      # - run: npm start & npx wait-on http://127.0.0.1:3000 && curl -fsS http://127.0.0.1:3000/ >/dev/null
```

Deploy is its own manual workflow, never part of the gate:

```yaml
name: Deploy
on:
  workflow_dispatch:        # you deploy when you want
permissions:
  contents: read
concurrency:
  group: deploy
  cancel-in-progress: false
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run build
      # The deploy step depends on the host (see Open questions). Keep it one command.
```

Notes:

- Responsive behaviour (phone width) is a human check in DevTools device emulation. CI cannot claim it (AGENT_RULES: responsive web UI is mandatory, verified by a person).
- Deploy path per AGENT_RULES: `Local -> GitHub -> VPS`. After deploy, see [`PROD_WATCH.md`](PROD_WATCH.md).
- A server that runs `npm ci --omit=dev` needs runtime packages in `dependencies`, not `devDependencies`.

### (c) Docker image (build, test inside the image, push to GHCR)

In use in `beeperbox` (`mcp-test.yml`, `release.yml`) and `plato` (`publish-eval-image.yml`). Two workflows: a PR test and a publish.

**Test (`docker-test.yml`).** Build the real image, load it locally, run the checks against it. Trigger on the files that affect the image, plus a manual button.

```yaml
name: docker-test
on:
  workflow_dispatch:
  pull_request:
    paths:                       # only what changes the built image
      - 'Dockerfile'
      - 'entrypoint.sh'
      - 'src/**'
      - '.github/workflows/docker-test.yml'
permissions:
  contents: read
jobs:
  image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: docker/setup-buildx-action@v4
      - name: Build image (amd64, load locally)
        uses: docker/build-push-action@v7
        with:
          context: .
          load: true
          platforms: linux/amd64
          tags: myimage:test
          cache-from: type=gha
          cache-to: type=gha,mode=max
      # Test the image as it ships, not the source tree.
      - run: bash scripts/image-check.sh myimage:test
```

Pin base-image and app versions in the repo (beeperbox keeps `beeper-version.txt`) so the PR gate tests what ships. Share check scripts between the PR test and the release gate so they cannot drift.

**Publish (`docker-publish.yml`).** Manual or on a version tag. Authenticates with `GITHUB_TOKEN`, so there is no stored registry secret.

```yaml
name: docker-publish
on:
  push:
    tags: ['v*.*.*']
  workflow_dispatch:
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write            # lets GITHUB_TOKEN push to GHCR
    steps:
      - uses: actions/checkout@v6
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/setup-buildx-action@v4
      - uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.ref_name }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

beeperbox goes further: it publishes only after an amd64 and an arm64 verify job pass (a gate before push). Add that when the image targets both. Docker action versions above are the ones in use (`login-action@v3`, `setup-buildx-action@v4`, `build-push-action@v7`; plato still has older pins `v3`/`v6`).

---

## 4. Pick your variant, and per-repo knobs

Checklist:

- [ ] Ships to npm? Use (a): core plus `publish.yml` from the template.
- [ ] Has tests that read git history? Set `fetch-depth: 0` in `ci.yml` and `publish.yml`.
- [ ] Tests shell out to a binary (ripgrep, chromium)? Add the install step in `ci.yml` and `publish.yml`.
- [ ] Needs a real browser in CI? Copy the fwdloop steps, or run unit tests only (barebrowse).
- [ ] Runs on more than Linux or Node 22? Add the matrix (bareguard).
- [ ] Is a served app? Use (b): add `build`, keep deploy manual.
- [ ] Ships an image? Use (c): test workflow plus GHCR publish.
- [ ] Touched a shared step? Grep every workflow file for it.

Knobs as found in the 7 library repos:

| Repo | Node | Triggers (ci) | System deps | fetch-depth | Extra gates / env |
|---|---|---|---|---|---|
| bareloop | 22, `cache: npm` | push, PR | ripgrep | 0 (publish too) | `PROMPT_COMMIT_RANGE` env on `npm test`; hermetic runner via `npm test` |
| bareguard | 22 and 24 matrix | push+PR to main | none | default | ubuntu/macos/windows; `check:primitives`, `check:lockfile` |
| bareagent | 22 | push, PR | none | default | `check:primitives` |
| litectx | 20 in ci, 22 in publish | push, PR | ripgrep | default | `check:primitives`, `bench:assemble`, `bench:summary` |
| barebrowse | 22 | push to main, PR | none | default | `test:unit` only (integration needs a browser) |
| baremobile | 22 | push to main, PR | none | default | `check:types` |
| fwdloop | 22, `npm@11` | PR, push to main, manual | ripgrep, fonts, chromium sandbox sysctl and warm-up | default | `concurrency` cancel on PR, `timeout-minutes: 15` |

`PROMPT_COMMIT_RANGE` (bareloop): supplies the real pre-push range so its commit check covers every commit of a multi-commit push to main, not only `HEAD~1`. Push events only; empty on PR. Wire a diff-based check to the real range, never a fixed `HEAD~1`.

---

## 5. Open questions / not decided

- **Docker registry.** GHCR is what `plato` and `beeperbox` use today. No ruling that it is the standard for every image. Not decided: Docker Hub or other.
- **Image publish trigger.** `plato` uses tag push plus manual; `beeperbox` uses tag, weekly cron and manual. No single rule chosen.
- **Registry auth.** Both use `GITHUB_TOKEN` with `packages: write`. Whether to add provenance or signing to images is not decided.
- **Web app deploy step.** No web-app workflow exists. The deploy command (VPS over SSH, container pull, other) is not decided. The Deploy skeleton above is a placeholder.
- **Web app test depth.** Whether CI should run a browser smoke (barebrowse) or stay unit-only is not decided.
- **Node floor.** Suite is on 22; litectx `ci.yml` still says 20 while its publish says 22; beeperbox uses 20 for its unit job. Not ruled whether to align.
- **Action version drift.** `actions/checkout` and `setup-node` are `@v6` in most files, `@v5` in bareloop/litectx `ci.yml`, `@v4` in plato and beeperbox. Not ruled whether to bump everything to one version.
- **Branch filters.** Four repos limit push to `main`, three run on all pushes. Not ruled which is standard.
- **Lint/format.** Ruled out as a gate for libraries. Not decided for web apps.
