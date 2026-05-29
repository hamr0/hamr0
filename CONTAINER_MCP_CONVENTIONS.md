# Shipping a containerized app / MCP server — standard advisory

The fixed reference for how I build something that ships as a **Docker image**
and/or exposes a **Model Context Protocol** server. Its spine is the set of
container-correctness foot-guns that bite every project once and never again
once you know them; around that it fixes the MCP tool-surface discipline.

**Start with [`AGENT_RULES.md`](AGENT_RULES.md).** That is the parent standard
(POC-first, dependency hierarchy, simple-over-clever, open-source-only, the
security & robustness invariants, the testing trophy). This advisory does **not**
repeat it — it adds only what is specific to shipping an *image* (a different
boundary than the npm tarball in [`LIBRARY_CONVENTIONS.md`](LIBRARY_CONVENTIONS.md):
an image + a wire protocol, not a package + an import). When the two ever seem to
disagree, AGENT_RULES wins.

The worked example throughout is **beeperbox** (a Beeper Desktop container that
exposes an MCP server) — it earned each of these the hard way.

---

## 1. Container correctness (the core)

These are cheap, universal, and a real foot-gun if skipped. Satisfy them as you
write the Dockerfile / entrypoint, not as a cleanup pass.

- **Forward signals to the real workload.** A shell entrypoint as PID 1 does
  **not** propagate `SIGTERM` — `docker stop` waits its grace period then
  `SIGKILL`s your process mid-write (corrupt DB, lost flush). Trap `TERM`/`INT`,
  forward to the child, and **re-wait until it's actually gone** — bash's `wait`
  returns early on trap delivery, so a single `wait` returns before the child has
  finished shutting down:
  ```bash
  trap 'kill -TERM "$CHILD_PID" 2>/dev/null' TERM INT
  while kill -0 "$CHILD_PID" 2>/dev/null; do
    wait "$CHILD_PID" 2>/dev/null || true
  done
  ```
  This is the single most-skipped container correctness bug.

- **Healthcheck the externally-reachable path, not an internal shortcut.** Probe
  *through* your forwarder/proxy so a crash in **either** layer fails the check —
  not a localhost shortcut that stays green while the public path is down. Set
  `--start-period` to real warmup time so a slow boot isn't flagged unhealthy.

- **Bind loopback inside the container; expose only what's deliberately public.**
  Internal services on `127.0.0.1`; one forwarder on `0.0.0.0` for the intended
  port; explicit `EXPOSE`. (The AGENT_RULES least-privilege-binding invariant, in
  its container form.) Prefer IPv4 loopback for the forwarder target — some hosts
  and IPv6-disabled containers can't reach `[::1]`, and the forwarder then drops
  traffic silently.

- **Insecure-but-convenient default is OK only when the safe boundary is
  documented and the open state is loud.** An unauthenticated default is
  acceptable *only* if (a) it's safe within a stated boundary (e.g. loopback-only)
  and (b) the open state prints a visible warning at startup. A silent open
  default is the anti-pattern. Mirror the same opt-in across every exposed
  surface (VNC password, MCP token) so the mental model is one rule, not three.

- **Pin-optional builds with an integrity hook.** A rolling/latest default is
  fine — it's what a weekly-rebuild cron needs — but always provide a
  `*_VERSION` + `*_SHA256` build-arg path for reproducible, verified builds, and
  **write down what the rolling default actually trusts** (e.g. "TLS only, no
  published signature"). The caveat being explicit is the point; most projects
  leave it implicit.

- **Slim base, no recommends, clean the cache in the same layer, `.dockerignore`.**
  `*-slim` base, `--no-install-recommends`, `rm -rf /var/lib/apt/lists/*` in the
  same `RUN`, and a `.dockerignore` that keeps the build context small and
  secrets out. Boring, universal, keeps the image and context lean.

- **Every non-obvious workaround pays for itself with a "why" comment.** This is
  how you enforce simple-over-clever *when forced to be clever*: when a build step
  is surprising (a binary-offset extraction, an omitted-flag quirk, an
  arch-specific branch), the cleverness is paid for with a comment explaining the
  failure it avoids. Cleverness without a rationale comment is the thing to reject
  in review.

---

## 2. MCP server discipline

An MCP server is JSON-RPC over stdio or HTTP — and the temptation is always to
add surface. Resist it.

- **Stdlib transport.** The transport is small enough to hand-roll on the stdlib
  (`http` / stdin-stdout). Reach for an MCP SDK only when the protocol surface
  genuinely outgrows it — a single backing API behind a handful of tools does not.
  Textbook dependency hierarchy: vanilla → stdlib → external.

- **Opinionated, curated tool surface — mechanism vs policy.** Expose a
  *normalized, minimal* tool set, **not** a 1:1 passthrough of the upstream API.
  Default *out* on new tools; each one earns its place by being something an agent
  actually needs, shaped the way an agent can use it. The curation is the value —
  a thin proxy adds none.

- **Opt-in token on HTTP transport, loud when off, loopback-only when open.** The
  stdio transport is implicitly local; the HTTP transport is the one that needs a
  bearer/token gate. Same rule as §1's loud-open-default: off is allowed only on
  loopback and only with a startup warning.

- **Validate tool args at the boundary; return structured errors, don't crash.**
  Agent input is untrusted input (AGENT_RULES invariant: validate at every trust
  boundary). Validate every tool's params before hitting the backing API, and turn
  a backing-API failure into a structured MCP error result — never a server crash,
  never a leaked stack trace.

---

## 3. What stays out of this doc

Project-specific build gotchas live where they bite — as comments in the
Dockerfile/entrypoint and in the project's adopter-facing `context.md` — **not**
lifted into this advisory (surgical-changes / no-duplication). This doc carries
the *generalizable rule* ("comment your workarounds"); the specific
binary-offset trick, the IPv6-loopback lesson, the GPU-flag quirk stay in-place
next to the code they explain.

The publish/distribution mechanics differ from a library: there's no npm tarball,
no `exports`/`files`/`types`, no `prepublishOnly`. The image is published to a
container registry (GHCR) on tag/release; the deploy path is
`Local → GitHub → GHCR → host`, per the AGENT_RULES deployment strategy.
