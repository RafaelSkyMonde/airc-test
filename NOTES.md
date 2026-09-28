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
