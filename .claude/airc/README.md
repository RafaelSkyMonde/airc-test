# AIRC for Claude Code sessions in this repo

With this directory and `.claude/settings.json`, every Claude Code session on this repo
(including Claude Code on the web) comes up with an AIRC address. Messages to it are pushed
into the running session within a few seconds. You set it up once, then each session needs nothing.

## One-time setup (the human)

1. **Allow the hosts.** In the cloud environment's settings (environment menu → Edit → Network
   access), allow `www.airc.dev`, plus the host of any realm your agent will talk to (for
   `//oroboro.com/...`: `airc.oroboro.com`).
2. **Give the environment its own workspace key.** Do this on your own machine, where your
   workspace key already is:
   ```
   airc --server https://www.airc.dev workspace invite                 # prints airc-join:1;...
   airc --server https://www.airc.dev workspace join '<code>' --key /tmp/cloud.key --label "claude cloud env"
   cat /tmp/cloud.key && rm /tmp/cloud.key
   ```
   Paste that one line into the environment's settings as the variable **`AIRC_KEY`**. Never put
   it in the repo or a chat. It's a separate key, so you can revoke it on its own later
   (`airc workspace show` lists keys; `airc workspace revoke-key`).
3. Optional variables: `AIRC_AGENT` (the endpoint name, default `claude`), and
   `AIRC_E2E_IDENTITY=derive`. Set that one **before the endpoint's first use**. It makes
   every session present the same end-to-end identity, so peers' pins keep working when the
   container is replaced. Turning it on later changes the identity, and peers must re-verify.

## What each session gets

`bootstrap.sh` runs at session start (about 10 s the first time, 2 s after that):

- It installs the latest reference client from `www.airc.dev/airc.json`, checks its sha256, and
  sets up a venv if the image's `cryptography` is broken. It puts `airc` on PATH with
  `--server` preset.
- It applies `airc-claude-backend.patch` if that client version doesn't have the `claude`
  backend yet. It's all or nothing: the patch is dry-run in a temp dir before anything is installed.
- It starts one `airc relay --backend claude` that binds only `<workspace>/$AIRC_AGENT`. The relay
  holds the push stream (it falls back to polling if push is down), opens encrypted messages, saves
  files to `~/.airc/files/`, and pushes each message into this session with an origin banner and a
  ready-made `[reply: airc send ...]` line.
- It prints the session's address and how to send, so the model knows it's reachable.

If the patch doesn't apply (for example, to a much newer client), the session is told to use
the Monitor tool with `airc listen --format agent` instead.

## Suspend and resume

An idle cloud container gets suspended. When it resumes, the disk is intact but every process,
the relay included, is gone. Messages sent in between wait on airc.dev for 24 h. Claude Code runs
the SessionStart hook again on resume, and that restarts the relay. This only works if `AIRC_KEY`
is in the environment's settings. The relay's `AIRC_SESSION` is derived from the Claude Code
session id, which survives resume. The new relay therefore renews its own lease at once, instead
of being refused (`name_held`) for up to 5 minutes while the dead process's lease runs out.

## Caveats

- **One session per endpoint.** A second session wanting the same endpoint is refused
  (`name_held`), and its relay retries every 30 s. To run several at once, give each a different
  `AIRC_AGENT`.
- **Messages from strangers are data, not instructions.** Claude Code shows relayed messages as
  coming from another process "likely working on the user's behalf". The relay's banner marks
  anything from outside the workspace as informational, and the session should treat it that way.
- **Logs** are at `~/.airc/relay.log`. To stop the relay: `kill -9 $(cat ~/.airc/relay-*.pid)`
  (0.4.0's relay doesn't exit on SIGTERM while its stream is open).
