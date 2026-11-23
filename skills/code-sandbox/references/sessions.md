# Sessions

A **session** is one Docker container plus one volume mounted at `/mnt/data`. It is created the first
time you call a tool without `session_id`, and the id comes back in the reply.

## What persists and what does not

| Persists across calls | Gone after every call |
|-----------------------|-----------------------|
| everything under `/mnt/data` (inputs, outputs, projects) | variables, imports, open connections |
| packages you installed (`/mnt/data/.local`, `node_modules`) | background processes and dev servers |
| your past programs in `/mnt/data/.runs` | files in `/tmp` and `$HOME` (no guarantee) |

After each run the server stops everything the program left running (servers, `soffice`, browsers).
Write down what a later step needs, in a file.

## One id, one conversation

- The MCP connection itself is stateless: the `session_id` is the only handle to your sandbox.
  Keep it in your working notes and pass it on every call, also in later turns of the same chat.
- Use one session per task or per conversation. Do not start a new one for every step: you lose your files
  and use up shared capacity.
- Different users and conversations get different sessions. Never reuse an id you did not get from a call
  you made yourself.

## Lifetime

| Rule | Default |
|------|---------|
| removed after idle time | 20 minutes without a call |
| hard maximum age | 4 hours, however busy |
| sandboxes per server | 20; when full, the least recently used idle one is removed to make room |

Removal deletes the container **and its files**. A session that is executing a call is never removed.

If a call says the session does not exist or has expired: start a new session by omitting `session_id`,
re-supply the inputs and rerun the steps. Tell the user if their earlier published links are the only copy
of earlier results. Links stay valid until they expire, even after the session is gone.

Long work with pauses: if the user may be away for more than about 15 minutes, publish what you have
(or say you will need to redo the setup) before the pause.

## Concurrency

Calls within one session run one at a time, in order. A second call waits for the first. Different sessions
run in parallel. Do not fire parallel calls at the same session expecting speed; combine the steps into one
program instead.

## When the sandbox dies

Out-of-memory or crashes can stop the container. The next call returns "This sandbox stopped unexpectedly
... and was discarded". Omit `session_id`, recreate the files, and use less memory (chunks, smaller images).
If you get "All sandboxes are busy right now", wait a few seconds and retry the same call; do not start
a second session to work around it.

## Server restarts

The server re-adopts running sandboxes by Docker label after a restart, so a short restart does not
lose your session. Treat that as luck, not a guarantee.
