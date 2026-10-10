# Execution Backends

An execution backend is the client that runs the stages. This directory holds one file for each
backend, and this file tells you how to set them up and which one to use. Do it at the start of a
project, before the first stage.

| File | Backend | Use it |
|---|---|---|
| `cli.md` | the `distil` command | by default |
| `backend-api.md` | the REST API, from Python | when the user prefers it, or the work is already scripted in Python |

Both backends need the CLI installed and signed in. The API backend authenticates with the
access token that `distil access-token` prints.

## Set up the CLI

1. Run `distil --version`. If the command answers, go to step 4.
2. Say that you will install the CLI, then run the install script:

   ```bash
   curl -fsSL https://cli-assets.distillabs.ai/install.sh | sh
   ```

   The script writes one binary to `~/.local/bin/distil`. It changes nothing else.
3. Run `distil --version` again. If the shell does not find the binary, add `~/.local/bin` to
   `PATH` and run the command again. If the install failed, see § When the install is
   impossible.
4. Run `distil whoami`. If it prints a user, the CLI is ready. If it prints no user, ask whether
   the user already has a distil labs account, then run the matching command:

   ```bash
   distil signup                                           # no account yet; opens a browser
   distil auth                                             # has an account; opens a browser
   ```

   Both commands print a one-time code and open the distil labs sign-in page. The user signs in
   or creates an account there, checks that the code matches, and confirms it. The command then
   prints `Logged in as <email>`. If the browser does not open, the command prints an address to
   visit. It works from any device, so on a machine with no browser the user opens it on their laptop
   or phone. The code expires after a few minutes; if it does, run the command again.

   With a session already in place, both print `You are already logged in as <email>`. Run
   `distil logout` first to switch accounts.

The CLI keeps its session in `~/.config/distillabs/session.json` (under `$XDG_CONFIG_HOME` when
that is set) and refreshes it, so a long project needs no second sign-in.
`distil update` replaces the binary in place.

## Choose the backend

**The CLI is the default.** The API backend is a first-class alternative, not only a fallback:
take it whenever the user prefers it, or whenever the work is already scripted in Python. Ask the
user, or take the choice your operator already made. A stated preference settles it.

Record which backend and why in `run.md` at the start of the project. Each later stage uses that
backend.

## When the install is impossible

The install script supports Linux x86_64, Linux arm64, macOS on Intel, and macOS on Apple
Silicon only. The skill cannot run stages in these conditions, with either backend, because the
API backend needs the CLI for its access tokens:

| Condition | What to do |
|---|---|
| Windows | Run the CLI inside WSL. |
| No network route to `cli-assets.distillabs.ai` | Install from a machine that has one. |
| The environment permits no new binary | Run the project from a machine that does. |
| The user declines the install | The choice belongs to the user. Stop and say what the CLI is for. |

Without the CLI you can still prepare the data files and the config with the user, so they are
ready to run once the CLI is available.

## What the choice does not change

Both backends create the same entities on the same platform. They accept the same config
overrides, and they identify each entity by the same UUID. Stage files cite operations by § name,
and both backend files use the same § names.

So the choice of backend changes the commands only. It changes no result, and it changes no
stage protocol. A project can also move from one backend to the other, because an id from one
works in the other.

Both files also use the same entity names, so a citation such as § The Dataset resolves in
either.
