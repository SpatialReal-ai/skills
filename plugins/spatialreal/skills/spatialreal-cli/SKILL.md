---
name: spatialreal-cli
description: Manage a SpatialReal account with the spatialreal command-line tool. Use when the user wants to log in, link a project and get an API key, create, generate or edit a SpatialReal agent, list avatars, publish an agent with an embed link, set a webhook, check usage or billing, or fix an error the spatialreal command printed.
---

<!-- spatialreal-cli skill -->

# SpatialReal CLI

`spatialreal` manages a SpatialReal account from the terminal: apps and API keys, agents and
their definition files, avatars, embed links, knowledge documents, webhooks, usage and billing.
It is built for coding assistants: JSON output when piped, an error `code` with a `hint`, and
never a prompt. The web console is https://app.spatialreal.ai.

Check that it is installed with `spatialreal --version`. If that fails, tell the person and,
when they agree, install it: `npm install -g @spatialreal/cli` (Node.js 20 or newer).

## Rules

1. **Discover, don't guess.** `spatialreal --help --json` lists every command with its flags,
   positional arguments and whether it writes or deletes (`needs_yes_when` for the ones that
   need `--yes` only sometimes). `spatialreal <command> --help` shows one. Use only commands and
   flags it lists.
2. **Read JSON.** Output is JSON when piped; add `--json` to be sure. On failure stdout holds
   `{"error": {"code", "message", "hint"}}`: act on `code`, and try the `hint` first.
3. **Logging in needs the person.** Run `spatialreal login` and wait for it. It opens the
   approval page in the person's browser and prints the address and a code on stderr; tell the
   person to approve the login there. The command ends with `logged_in: true` as soon as they
   do.
4. **Confirm before deleting or spending.** A command that deletes, rotates a secret, cancels,
   resumes a subscription or starts a payment fails without `--yes` (exit code 2,
   `confirmation_required`) and changes nothing; its
   `error.details.would` says what it would do. Run it with `--dry-run` (or read `would`), tell
   the person, and add `--yes` only after they confirm. Any command that writes accepts
   `--dry-run`.
5. **Never handle secrets.** The CLI never prints a token, an API key or a webhook secret, and
   neither do you. Do not open, print, copy or hand-edit `.env`, `spatialreal.json` or the
   saved credentials; `spatialreal status` tells you which SpatialReal variables the env file
   sets, never their values (add `--env-path <file>` when the project uses another env file
   than `.env`). The API key
   (`SPATIALREAL_API_KEY`) is for server code only: never put it in browser or mobile code, a
   bundler variable exposed to the browser, a log line or a URL.
6. **Know what the CLI cannot do.** Creating an avatar, adding knowledge documents and
   anything else not listed in `--help --json` happen in the web console: tell the person. The
   CLI does not run realtime sessions either; sessions run in the customer's app or on an embed
   link. Every agent has an avatar; never build an agent integration without one.

## Start

```bash
spatialreal whoami                   # not_logged_in? follow rule 3
spatialreal apps list
spatialreal init --app <app_id>      # or: --create-app "My app"
spatialreal status
```

The code that shows an avatar or an agent in the customer's app (token endpoint, SDK, page) is
in the `spatialreal` skill, installed next to this one.
This skill covers the account: what the CLI creates, lists and changes.

`init` links the directory: `spatialreal.json` (app, profile, tracked agent files), `.env` with
`SPATIALREAL_APP_ID` and a new `SPATIALREAL_API_KEY` (`--env-path` for another file), `.env` in
`.gitignore`. Its output and `status` give `open_api_base`, the host the
customer's server calls with the key. If the output reports `env_file_tracked_by_git`, tell the
person to run the `git rm --cached` it names.

## Agents

1. Pick or make the agent. An existing one: `spatialreal agents list`. A new one: **generate it
   from a description** (the best result; `spatialreal docs show agent-generate`):

   ```bash
   spatialreal agents generate start "A receptionist for a dental clinic that books appointments"
   # the reply asks a question: ask the person, then pass their answer
   spatialreal agents generate say "<answer>"
   # repeat until the output has generation.phase "succeeded" and an agent_id
   spatialreal agents generate end
   # to change the agent afterwards, open a session on it:
   spatialreal agents generate start "<what to change>" --agent <agent_id>
   ```

   Output with `turn: in_progress` means it is still working: run
   `spatialreal agents generate status --wait-seconds 60` until `turn` is `complete`, and send
   nothing meanwhile. Never answer the engine's questions yourself; they are for the person. Or start from a
   template: `spatialreal agents templates list`, then
   `spatialreal agents create --from-template <template_id> --name "Support"`.
2. Shape it as a file (details: `spatialreal docs show agent-files`):

   ```bash
   spatialreal agents pull <agent_id>          # writes agents/<name>.json
   # edit agents/<name>.json; agents schema / agents options say what is allowed
   spatialreal agents validate agents/<name>.json
   spatialreal agents diff <name>
   spatialreal agents push <name> --dry-run
   spatialreal agents push <name>
   ```

   The avatar is set in the web console or through `agents create --from-template`.
3. Choose how users reach it:
   - **No code**: an embed link (`spatialreal docs show agent/reach/page-and-embed`):
     `spatialreal embed-links create --agent <agent_id> --mode standalone --published true`
     prints a `url` to share; `--mode iframe` with `--allowed-frame-origins` gives a URL for an
     `<iframe>` on the customer's site. A published link is public and every session through it
     is billed to the account: tell the person before publishing, and suggest
     `--max-session-minutes` and `--max-concurrent-sessions`.
   - **Own interface**: the SDK with the `agent_id` as `agentId`; the `spatialreal` skill has
     the code.
4. Optional: knowledge documents (added in the web console). `agents documents set` replaces
   the whole list, so read it first with `spatialreal agents documents get <agent_id>`, then
   `spatialreal agents documents set <agent_id> --documents <existing_id>,<new_id>`. A
   background: `spatialreal agents background set <agent_id> --background-id <id>`. Session
   events: `spatialreal webhook set --url <https_url> --events session.ended` (also
   `session.summarized`; the signing secret goes to `.env` as `SPATIALREAL_WEBHOOK_SECRET`).
   Payloads and signature verification: `spatialreal docs show agent/reach/webhooks`.

## Avatars

`spatialreal avatars list --public` (ready-made avatars) or `spatialreal avatars list` (the
person's own). The SDK's `avatarId` is the row's `character_id`. New avatars are created in the
web console; the CLI shows their progress (`spatialreal avatars applies list`,
`spatialreal avatars tasks list`) and can rename, copy or delete one the person owns.

## What the integration code needs from here

- `SPATIALREAL_APP_ID` and `SPATIALREAL_API_KEY` are in `.env` after `init`; read them from the
  environment in server code, never from the file yourself.
- `open_api_base` (printed by `spatialreal status`) is the host the customer's server calls for
  session tokens. It is not the CLI's own host, and a key works only on the host of its own
  environment (`401 invalid api key` otherwise).
- `agentId` is an `agent_id` from `spatialreal agents list`; `avatarId` is a `character_id`.

## Agent files: protect the person's work

- `local_changes` on `pull`: the file has edits. Run `spatialreal agents diff <name>`; push
  them, or (only if the person wants to discard them) copy the file aside and
  `spatialreal agents pull <name> --force`.
- `agent_changed_on_server` on `push`: the agent changed after the pull. Copy the edited file
  aside, `spatialreal agents pull <name> --force`, re-apply the edits, validate, diff and push.
  `spatialreal agents push <name> --force` overwrites the server's changes: only after the
  person has seen the diff and agreed.

## Error recovery by code

| `code` | Do this |
| --- | --- |
| `not_logged_in` | Rule 3: `spatialreal login`, tell the person to approve it in the browser, wait for the command to end |
| `confirmation_required` | Rule 4: show `error.details.would` or the `--dry-run` output; `--yes` only after the person agrees |
| `local_changes`, `agent_changed_on_server` | See "Agent files" above |
| `invalid_runtime_config` | Fix every entry of `errors` (the first is in `field`), then `agents validate` |
| `profile_mismatch` | A `--profile` or `SPATIALREAL_PROFILE` differs from the project's: drop it, or use the profile the message names |
| `credits_exhausted` | The account is out of credits: `spatialreal billing credits`. Adding credits is the person's decision: `spatialreal billing packages`, then `spatialreal billing buy --package-id <id> --success-url <https_url>` returns a payment URL for the person to open (it never charges by itself, and needs `--yes` after they confirm) |
| `generation_busy`, `generation_session_closed`, other `generation_…` | See the error table of `spatialreal docs show agent-generate` |
| `no_project` | `spatialreal init --app <app_id>` |
| `skill_exists` | The person has their own skill file; replace it only if they agree (`spatialreal skill install --force`) |
| anything else | Follow `hint`; `spatialreal docs show errors` explains every code the CLI produces |

## Documentation

`spatialreal docs show <page>` prints a page of https://docs.spatialreal.ai as Markdown,
`spatialreal docs search "<words>"` finds a page, `spatialreal docs list` lists them all. The
`spatialreal` skill says which page to start with for each part of the product.

Three guides are about the CLI itself and ship with it (no network needed):
`spatialreal docs show agent-files`, `spatialreal docs show agent-generate`, `spatialreal docs show errors`.
