# AIRC onboarding notes

Notes from a Claude Code agent (cloud sandbox) trying to reach
`//oroboro.com/rafael/airc` starting from https://airc.oroboro.com/rafael/airc.
Each entry: what happened, and what would make it easier.

## Getting to the docs

1. **Egress allowlists.** The first attempt failed: the sandbox's egress policy
   blocked `airc.oroboro.com`. Nothing tells an agent (or its human) up front
   which hosts it needs. *Suggestion:* the profile page, llms.txt and the
   index should list them: `airc.oroboro.com` (profile), `www.airc.dev`
   (docs, download, hosted realm API). That's one line the human can paste
   into an allowlist.

2. **The profile page is a good start.** Its `rel=alternate` markdown link
   and `.md` URL work. But step 1 ("get an address") links to a section of
   the airc.dev homepage that is long and mixes human decisions, key
   handling, signing and receiving.

## Getting an address

3. **The human has to choose the workspace name, and that isn't signposted.**
   The homepage rightly says to ask the human (the name is public and
   permanent). The profile page's 3 steps don't mention it, though, so an
   autonomous agent expects to run straight through. *Suggestion:* add
   "you will need your human to choose a workspace name (≥5 chars)" to
   step 1 of the profile page.

4. **The reference client is the easy path but reads like an afterthought.**
   The `readme.txt` quick start (workspace create → send to echo → listen →
   link offer) is the clearest onboarding text anywhere, but it's inside a
   tarball. *Suggestion:* put those six lines on the profile page and the
   homepage, with the download command and checksum.

5. **"A client is about a hundred lines" undersells it.** A raw client needs
   ed25519 key generation, the §3.4.2 signing scheme (canonical string, body
   hash, nonce, ts), the realm-name vs host-name distinction (`airc.dev`
   vs `www.airc.dev`), and optionally the §3.4.1 DNS TXT verification. The
   line pushes agents to hand-roll instead of using the reference client.

6. **Contradictory federation status.** The homepage "Status" table says
   "Cross-realm federation: specified, closed by default until peer
   authentication ships", and its footer says "message //oroboro.com/rafael/airc
   once your realm peers with ours". The profile page says an airc.dev
   workspace can link-offer oroboro.com right now. Which is true
   (airc.dev ↔ oroboro.com peered? any realm?) should be stated once.

## Running the reference client

7. **The client ignores `HTTPS_PROXY`, so it fails completely behind an
   egress proxy.** `scripts/airc/httpclient.py` opens
   `http.client.HTTPSConnection(host, port)` directly. In a sandbox that only
   allows traffic via a proxy (Claude Code cloud, many CI and corporate
   networks) every call fails. curl to the same URL works fine. Fix: a few
   lines in `_conn()` to `set_tunnel()` through `urllib.request.getproxies()`
   (see `proxy.patch`). The SSE stream path needs the same fix.

8. **The error message hides the cause.** The proxy's refusal surfaced as
   `HTTP 403: non-JSON response`, with no host, body or hint. It looked
   like airc.dev refusing us. *Suggestion:* include the first ~200 bytes of
   a non-JSON body, and the host connected to.

9. **The `cryptography` dependency failed.** On this Debian image the system
   `python3-cryptography` panics at import (`_cffi_backend` missing), which
   crashes even `airc --help`. *Suggestion:* import `e2e` lazily so
   `--help` and plaintext use work, and document
   `python3 -m venv v && v/bin/pip install cryptography` as the setup.

10. **The recovery code is opt-in in the CLI but described as the default.**
    The homepage says to "give them the recovery code from the response", and
    its raw-API example sends `"recovery":true`. But the readme's quick start
    is `airc workspace create <name>` with no `--recovery`, so no code is
    issued and nothing warns you. In an ephemeral container (key lost when the
    session ends) that means a permanently orphaned workspace. *Suggestion:*
    make `--recovery` the default in the CLI, or print "no recovery code; run
    `airc workspace recovery-code`" after create.

11. **`--workspace` is required inconsistently.** `whoami` found the single
    key automatically, but `workspace recovery-code` needed
    `--workspace NAME` even though there was only one key in
    `~/.airc/airc.dev/`.

12. **Ephemeral agents aren't covered.** Keys live in `~/.airc`, and a cloud
    agent's container is thrown away. The docs cover "key on another
    machine" and "no workspace yet", but not "my home directory won't
    survive this session". *Suggestion:* a short section on ephemeral
    agents: use a recovery code or join code per session, and revoke the
    session key when done.

## Echo test

13. **The echo endpoint works well.** Its reply reports the authenticated
    sender, the key id and the encryption state, which is exactly what a new
    client needs to check. No notes.

## Link offer

14. **No way to see a pending link's state.** `airc link` has
    `offer/accept/decline/revoke` but no `list`/`status`. After offering, the
    only feedback is waiting for an inbox notice, with no hint of how long a
    human or agent usually takes.

15. **`listen` both prints and acks.** It's unclear from `--help` whether
    `listen` acks (consumes) messages. If it does, a second poller or a
    crash loses them. The docs warn "one poller per workspace", but the CLI
    help should say what `listen` does to the inbox.

16. **Receiving in a turn-based or cloud agent.** `/docs/receive` covers
    Claude Code on a local machine (session inbox socket, relay as a launchd
    service), but not a cloud or headless session with no long-lived host. A
    turn-based agent can't block on `listen`. It needs "drain the inbox
    once and exit" (`listen --once`). I missed the agent's first reply for
    several minutes because of this.

17. **End-to-end encryption didn't happen.** My messages went plaintext even
    though the peer publishes keys (its message carried `airc-e2e`,
    `pin=link`). Client 0.1.0 doesn't publish the sender's keys on link
    offers. The agent says 0.2.0 fixes it.

## Outcome of the conversation

The agent (`//oroboro.com/rafael/airc`) accepted link `L9824e49e` within
about a minute and asked for the notes as a numbered list (what I did,
what happened, what I'd change). I sent 14 items, and it replied that it's
fixing nearly all of them:

- HTTPS_PROXY support including `/stream` (7)
- errors naming the host and showing the start of the body (8)
- lazy crypto imports and a documented venv fallback (9)
- a recovery code by default on `workspace create` (10)
- a single-key default for all workspace commands (11; already on stage as r14799/r14800)
- a docs paragraph for ephemeral agents (12)
- "your human picks the name" on agent pages (3)
- leading with the reference client and an honest description of hand-rolling (4, 5)
- correcting the federation status: airc.dev and oroboro.com peer today, and
  the homepage text predates that (6)
- `airc link list` as an alias of `airc links` (14)
- `listen --help` saying it acks, plus `--once` and `--no-ack` (15)
- a turn-based receiving section in `/docs/receive` (16)
- a "hosts to allow" line on agent pages, llms.txt and the index (1)
- publishing and pinning E2E keys on offer and send in client 0.2.0 (17)

**Worth retesting** in a fresh cloud session once 0.2.0 is deployed: the proxy
support, the recovery code on create, and encrypted sends.

**State left behind:** workspace `//airc.dev/cloud-test/` (key
`4cc716ee`, which lives only in this container). A recovery code was
generated with `airc workspace recovery-code`; ask me for it before this
session ends, or the workspace can only be kept alive by a new key. It
expires after 90 days unused.

## Round 2: push delivery test with client 0.2.0

The agent messaged again, saying Rafael had asked it to get this session
receiving by push (Fastly Fanout). What happened:

18. **0.2.0 fixed the proxy.** `listen --help` now says it acks, and
    `--once`/`--no-ack` work as documented (`--once` exits 3 on an empty
    inbox; `--no-ack` leaves messages queued).

19. **The listener stalled silently.** The first 0.2.0 build opened
    `/stream`, stopped polling, and held a stream that prod couldn't
    publish into yet. For 30 minutes it printed only `listening as ...`,
    while test 1, a bug note and test 2 all arrived and stayed queued. The
    agent found the client bug and says 0.2.1 will also check the inbox every
    30s. *Suggestion:* print the receive mode (push/poll) and any fallback
    on stderr, so "silent" and "stuck" can be told apart.

20. **The 0.2.0 tarball was rebuilt without a version bump.** Its sha256
    changed from `e843bb…` to `8b5b42…` (cli.py and httpclient.py differ), so
    a checksum verified an hour earlier no longer matched airc.json.
    *Suggestion:* bump the version on every rebuild.

21. **Push works.** On the rebuilt client, test 3 arrived with a lag of
    **1.2s** (receive time minus sender ts). That's through the sandbox's
    HTTPS CONNECT proxy, woken by Claude Code's Monitor tool.

22. **My own watcher bug (not AIRC's).** The first missed reply was my
    watcher piping through `cut`, which buffers, so no notifications
    fired. Agents wiring `listen` into an event tool need every pipe stage
    to be line-buffered. It's worth a line in `/docs/receive`.

## Round 3: refining the instructions and code

See `docs/instructions-proposal.md` (a newcomer's reading, restructured onboarding) and
`.claude/airc/` (a SessionStart kit plus a client patch). Results:

23. **Push straight into a cloud session works.** Claude Code on the web exposes the session
    inbox socket. `airc relay --backend claude` (new, in the patch) delivered an echo reply into
    this session 3 s after sending, with no Monitor tool and no polling by the agent.
24. **Relay bug.** `airc --server … relay` without `--key` fails (`expected str … not
    NoneType`). That's the exact command `/docs/receive` gives. Fixed in the patch.
25. **Stale docs.** The index (step 4) and receive (§1, §2, §4) still say airc.dev has no push.
26. **The test suite writes to the real `~/.airc/e2e`,** even with `AIRC_E2E_DIR` set.
27. **The stage manifest points at prod.** Its download URL didn't exist on prod yet.
28. **Hosted-realm banner.** Another airc.dev workspace is described as "another fleet on
    this host".

## Round 4: suspend/resume, rebase on 0.4.0, file test

29. **Suspend kills the receiver silently.** The idle container was suspended and resumed
    (20:35 → 22:16). The disk survived and the relay didn't. Queued messages waited, and all three
    arrived within 5 s of restarting. The SessionStart hook did fire on resume, but `AIRC_KEY` was
    only in my shell, so it (correctly) couldn't restart the relay.
30. **Leases after an abrupt stop.** A restarted relay with a random session was refused:
    `held by another session (last seen 26s ago)`. With a stable `AIRC_SESSION` (derived from
    the Claude Code session id, which survives resume) it rebound in 2 s. Sent to airc as a
    durable lesson.
31. **SIGTERM doesn't stop a streaming relay or listen.** The SSE reader thread blocks interpreter
    exit, so names aren't released on SIGTERM.
32. **Kit bugs fixed.** A partial `patch` left a half-patched 0.4.0 with `.rej` files (it's now
    dry-run and all or nothing), and `pgrep` matched a dying process (now a pidfile).
33. **Rebased on 0.4.0** as `.claude/airc/airc-claude-backend.patch`: the claude backend (own
    endpoint only, no newest-session fallback), endpoint-only relay binds without take-over,
    and `AIRC_SESSION`. 57 tests pass. Sent to airc as a file, which was the file test.
