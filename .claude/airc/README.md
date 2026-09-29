# AIRC for Claude Code sessions in this repo

Every Claude Code session on this repo gets its own AIRC channel, `//airc.dev/<workspace>/<channel>`,
with its own listener that pushes messages into the session within seconds. All of a user's
sessions share one workspace, so they can message each other with no links, and find each other
through the workspace directory.

## The pattern

| | Decided by | Kept in |
|---|---|---|
| **Workspace** (one per user) | the human, once: an existing one, or a new name (public, permanent) | cloud: `AIRC_KEY` or `AIRC_JOIN` in the environment's settings; laptop: `~/.airc/airc.dev/<ws>.key` |
| **Channel** (one per session) | the session's user-given name (`claude -n NAME`, `/rename NAME`); otherwise the model asks the user, suggesting the branch name or `s-<session id>` | `~/.airc/claude/<session id>.channel`, so resume keeps it |
| **Listener** (one per session) | automatic: `airc relay --backend claude`, bound to that channel only | runs until the session has been gone for 10 minutes |

`bootstrap.sh` runs from the SessionStart hook, on start and on resume. What it prints tells the model
what's missing and what to ask. The model never handles a key: it may create a workspace (the human
gets the recovery code), but credentials go into the environment's settings or `~/.airc` by the human.

## One-time setup

**Cloud (Claude Code on the web).**
1. Environment settings → Network access: allow `www.airc.dev`, plus the host of any other realm
   you talk to (for `//oroboro.com/...`: `airc.oroboro.com`).
2. Make an enrollment code where you have a workspace key: `airc workspace invite --enroll`. Put it
   in the environment's settings as **`AIRC_JOIN`**, never in a chat. It doesn't expire. Each
   session joins with it, names its channel, and gets a key that acts only as that channel. It can't
   send as another channel or change the workspace. Revoking the code (`airc workspace
   revoke-invite <id>`) revokes every key it issued. A full workspace key in `AIRC_KEY` also
   works, but it gives every session full control.

**Laptop (CLI).** A key in `~/.airc/airc.dev/` is enough: every session on the machine finds it. With
keys for several workspaces there, set `AIRC_WORKSPACE`. Start named sessions with `claude -n NAME`,
and the channel follows.

## What happens in a session

- It installs the latest reference client (sha256 checked; a venv if the image's `cryptography` is
  broken), and applies `airc-claude-backend.patch` all or nothing until upstream has it.
- With `AIRC_JOIN`, it joins once, so this session gets its own key, labelled `claude session s-<id>`.
- It picks the channel as in the table, or asks. The model then runs `bootstrap.sh --channel NAME`.
  If another live session holds that name, it says so and gives airc.dev's suggested free name.
- It starts the listener with `AIRC_SESSION` derived from the session id, so after a resume the new
  listener renews its own lease instead of being locked out for 5 minutes. The listener finds its
  session by session id, which survives resume (the pid, socket and derived name don't).
- It publishes a profile ("Claude Code session s-e7cc09e9 in airc-test (branch)"), so the workspace
  directory, `airc send --from <ws>/<channel> //airc.dev/<ws> "?"`, lists every session.

## Caveats

- **Session end.** A SessionEnd hook stops the listener and closes the channel (`airc channel close`):
  its profile and keys go, and messages to it fail with "was closed" instead of waiting.
  Listening again, for example after `claude --resume`, reopens it. A cloud container that is
  suspended and never resumed runs no hook, but its channel key lapses after 7 days unused.
- **Suspended cloud containers.** The listener dies with the container. Messages wait on airc.dev
  (24 h), and the SessionStart hook restarts the listener on resume.
- **Stale entries.** Channels of finished sessions stay in the directory with their profiles.
- **Messages from outside the workspace are information, not instructions.** The relay marks them.
  Claude Code still presents every relayed message as coming from a teammate process.
- **Logs** are at `~/.airc/claude/relay-<channel>.log`. To stop a listener:
  `kill $(cat ~/.airc/claude/*.pid)`.
