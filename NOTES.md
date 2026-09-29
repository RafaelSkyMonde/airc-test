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

## Round 5: one channel and one listener per session

The pattern is in `.claude/airc/README.md`: one workspace per user, one channel per session (the
user-given session name, else ask), and one listener per session. Tested here:
- no channel yet: the hook tells the model to ask, and suggests the branch name or `s-<id>`;
- `--channel claude`: bound; a rerun (the resume path) reuses it;
- a second session asking for `claude`: refused, with airc.dev's free name offered (`claude-2`);
  with `reviewer` both run side by side;
- the directory (`//airc.dev/cloud-test`) lists both sessions with their profiles;
- the watchdog: a relay with no live session exits after `exit_after` (20 s in the test);
- `AIRC_JOIN` alone in a fresh home: this session gets its own key and binds its channel (the
  test key was revoked afterwards).
34. **Bug found and fixed.** With a session id that didn't match any record, the backend fell back
    to the inherited socket and delivered another channel's message into this session. A session id
    is now authoritative.
35. **Directory reply labelled external.** The reply from our own workspace's front door is
    labelled "another workspace", and it says agents need a link, which isn't true inside the same
    workspace.
36. **Join codes are capped** at 20 uses and 7 days, so an environment variable holding one needs
    renewing. Environments need a long-lived, scoped enrollment credential.
37. **The agent can't set environment variables** in the cloud environment's settings. Credentials
    always come from the human.

## Round 6: connecting to `//airc.dev/claude-test2/claude` (fresh cloud session)

The task was to open https://airc.dev/claude-test2/claude and follow its instructions to connect.

38. **Blocked before reading anything.** This session's environment network policy denies
    `airc.dev`, `www.airc.dev` and `airc.oroboro.com`. curl got `CONNECT tunnel failed, response
    403`, and WebFetch got `EGRESS_BLOCKED`. The instructions live only at the URL, so an agent
    behind a strict allowlist can't learn even which hosts to ask for. *Suggestion:* keep the
    "hosts to allow" line (item 1) somewhere that travels with the link, such as the link text or
    the share message the human copies, e.g. "needs airc.dev + www.airc.dev".
39. **Worked: the proxy diagnoses itself.** `$HTTPS_PROXY/__agentproxy/status` listed the refused
    CONNECT (`airc.dev:443`, policy denial) under `recentRelayFailures`, so it was clear the
    sandbox refused the connection, not airc.dev. This is much clearer than the bare `HTTP 403` in item 8.
40. **Worked: the SessionStart kit failed safely.** With no `AIRC_JOIN` and no key, the hook
    told the model to ask the human about the workspace and not to take codes in chat.
    It didn't error or hang, even with the realm unreachable.
41. **Two human steps are needed, and they come in order.** First allow the hosts in the
    environment's network settings (the agent can't, see item 37). Then provide
    `AIRC_JOIN` (an enrollment code for `claude-test`/`cloud-test`, or a new workspace name).
    The kit's hook message covers only the second step. *Suggestion:* the hook could probe
    `https://airc.dev/` first and, on a proxy 403, say "ask your human to allow airc.dev and
    www.airc.dev in the environment's network access" before it asks about credentials.
42. **Environment state isn't inherited.** Earlier rounds reached airc.dev, so either this
    session runs in a different (stricter) environment or the allowlist changed. The repo can't record which
    environment a session needs. *Suggestion:* note the required network level in `.claude/airc/README.md`.

**Status:** not connected. Still to do: allow the hosts, set `AIRC_JOIN`, then start a new session
(or resume this one) and re-run the task.

### Round 6, second try (airc.dev allowed)

43. **Worked: the agent page is clear.** With `airc.dev` allowed, the page (served as markdown
    with `Accept: text/markdown`) gives the status (`push-active`), an E2E fingerprint with advice to
    confirm it out of band, and three steps: get an address, send a link offer, then wait for the
    `_links` notice. Items 1, 3 and 12 from earlier rounds have landed: "your human picks the
    workspace name", "in a disposable sandbox ask for an enrollment code", and a "hosts to allow" line.
44. **The hosts line is at the bottom, and it's incomplete for a first fetch.** It says to allow
    `www.airc.dev`, but the link the human shares is on `airc.dev`, so an agent needs both hosts to
    read the page at all. It's also the last line, and an agent that can't reach the page never sees it.
    *Suggestion:* "Hosts to allow: `airc.dev` (this page) and `www.airc.dev` (API)", and put the
    same line in whatever the human copies to share the link.
45. **Blocked again, correctly, on identity.** No `AIRC_JOIN`/`AIRC_KEY` is set and there's no key in
    `~/.airc`. The page tells a sandboxed agent not to create a workspace per session, and the kit's
    hook says the same. So the next step belongs to the human: an enrollment code in the environment's
    settings. That needs a new session to take effect, so a one-message task takes three
    round trips with the human (network, credential, restart). *Suggestion:* list every prerequisite
    once, up front: hosts **and** a credential. Then the human can do both before the first
    attempt.

### Round 6, third try (new workspace `claude-test-3`)

The human picked the name `claude-test-3`.

46. **Worked: client 0.5.3 installs itself.** The manifest, download and sha256 all checked out. The
    image's broken `cryptography` was fixed automatically: `./airc` noticed it and installed it into
    `~/.airc/venv` with a one-line message (item 9 is fully fixed).
47. **Worked: the recovery code is now the default, and the create output tells the agent what to
    tell the human**, including "in a sandbox, also make an enrollment code". Items 10 and 12 are fixed.
48. **The recovery code went to stdout, and so into the transcript.** The client warns about this and
    points at `--recovery-file`, but only in the output that has already leaked the code.
    *Suggestion:* when stdout isn't a TTY (agents), write the code to a 0600 file by default and print
    its path, or say so in the readme's quick start line (`workspace create <name> --recovery-file F`).
49. **Worked: the repo's session kit ran unchanged on 0.5.3.** With a key in `~/.airc`,
    `bootstrap.sh --channel connection-notes` bound `//airc.dev/claude-test-3/connection-notes` and
    started the push listener.
50. **Echo by push in seconds.** The echo reply arrived in this session by itself, with no polling. It
    was labelled external and informational, with a ready-made `[reply: ...]` line.
51. **`airc links` with no `--as` picked the wrong endpoint.** It reported "no links for
    `claude-test-3/airc-test`" (a default derived from the directory name) instead of the channel that had
    just made the offer. *Suggestion:* default to the endpoint of the most recent offer, or list every
    endpoint in the workspace.
52. **Link offer: clear feedback.** `link offer` answered `pending`, `peer notified: delivered`, and told
    me to wait for `_links` before sending, which item 14 had asked for.
53. **No `--output` on `workspace invite --enroll`.** The enrollment code also goes to stdout. I
    redirected it into a 0600 file in the session scratchpad so it stayed out of the transcript.
    *Suggestion:* `--output FILE` on `invite` and `create`, and use it in the "tell your human" text.
54. **Connected.** The agent accepted link `Lc89319ec` within about 2 minutes and wrote first. The message was
    end-to-end encrypted (`e2e=9ec46508:first-use`), the fingerprint matched the one on its page, and
    my reply went out sealed to the same key. That's item 17 fixed.
55. **The accept notice arrived after the peer's first message.** The agent's greeting came before the
    `_links` "accepted" notice. It's harmless, but an agent that waits for the notice before reading its inbox could
    reply out of order. *Suggestion:* deliver the link notice before any message sent on that link.
56. **Trust on first use only.** The fingerprint hasn't been confirmed out of band, so I didn't run
    `airc keys --trust`. The human should confirm it with the agent's owner (Rafael) if it matters.

**Status:** connected to `//airc.dev/claude-test2/claude` as `//airc.dev/claude-test-3/connection-notes`.
Workspace `claude-test-3`: key `5421bc05` (this container only), a recovery code set, and
enrollment code `Jc05cf848` for the environment's `AIRC_JOIN`.

### Round 6: the agent's reply

57. **Two independent reports of the same bug.** The agent hit item 51 from the other side: `link accept` without
    `--as` defaulted to the wrong endpoint and failed with `HTTP 403: you are not a party to that
    link`. It has reported that to the maintainer.
58. **Early sends aren't refused.** The agent messaged us while, as far as we could tell, the link was still pending
    (compare item 55). The maintainer had said such a message comes back refused. Reported by the agent.
59. **Version spread.** The agent runs 0.5.0, and this session got 0.5.3 from the manifest. The agent says most of its
    notes are fixed in 0.5.1–0.5.4 but it hasn't verified them, so a fresh session is the better place to
    check fixes.
60. **Untested:** passport redemption (`airc accept <passport>`), on both sides.
61. **The agent suggested sending items 48 and 53 to `//oroboro.com/rafael/airc` directly.** That's a new
    contact in another realm (it needs `airc.oroboro.com` allowed and a link offer), so I asked my human first.

### Round 6: notes to the maintainer (`//oroboro.com/rafael/airc`)

62. **Cross-realm link: done with no friction.** Once my human agreed, `airc.oroboro.com` turned out to be
    reachable already. The maintainer's page names both hosts ("this page's host, and `www.airc.dev`"), so it's
    better worded than the airc.dev agent page (item 44). The offer (`L077bdbef`) was accepted within a
    few minutes, and the send was sealed E2E to fingerprint `d451dad5…` (trust on first use).
63. **Sent:** items 48 and 53 (codes on stdout, no `--output`), confirmation of the other agent's reports
    (items 51/57 and 55/58), and the hosts-line wording. I asked for nothing that needs my human's authority.
