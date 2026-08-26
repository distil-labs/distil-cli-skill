# Execution Backends

An execution backend is the client that runs the stages. This directory holds one file for each
backend, and this file tells you which one to use.

| File | Backend | Use it |
|---|---|---|
| `cli.md` | the `distil` command | always, when the CLI installs |
| `backend-api.md` | the REST API, from Python | only when the CLI cannot install |

**The CLI is the default.** Install it at the start of a project, before the first stage. The
API backend is a first-class alternative, not only a fallback: take it whenever the user
prefers it, whenever the work is already scripted in Python, or when the install is impossible.

## Choose the backend

1. Ask the user, or take the choice your operator already made. A stated preference settles it.
   Record which backend and why in `run.md`, whether the reason is a failed install or a plain
   preference, and skip to step 5 for the API backend.
2. Run `distil --version`. If the command answers, use `cli.md` and go to step 5.
3. Say that you will install the CLI, then run the install script:

   ```bash
   curl -fsSL https://cli-assets.distillabs.ai/install.sh | sh
   ```

   The script writes one binary to `~/.local/bin/distil`. It changes nothing else.
4. Run `distil --version` again. If the command answers, use `cli.md`. If the shell does not
   find the binary, add `~/.local/bin` to `PATH` and run the command again. If the install
   failed, use `backend-api.md` and write the reason in `run.md`.
5. Run `distil whoami`. If it prints a user, the CLI is ready. If it prints no user, ask whether
   the user already has a distil labs account, then run the matching command:

   ```bash
   distil signup                                           # no account yet; opens a browser
   distil auth                                             # has an account; opens a browser
   distil auth --email <email> --password <password>       # has an account; no browser
   ```

   `distil signup` finishes signed in, so it needs no `distil auth` after it, but the user has
   to sign in on the page after submitting the form — and confirm their email address first, if
   the page asks for it. Both browser commands wait up to 20 minutes. If the browser does not
   open, they print a URL to paste. There is no headless `signup`, so on a machine without a
   browser the account has to exist already.

Record the backend in `run.md` at the start of the project. Each later stage uses that backend.

## When the install is impossible

The install script supports Linux x86_64, macOS on Intel, and macOS on Apple Silicon only. Use
`backend-api.md` in these conditions:

| Condition | Reason |
|---|---|
| Windows without WSL | The script has no Windows binary. |
| No network route to `cli-assets.distillabs.ai` | The script cannot download the binary. |
| The environment permits no new binary | The install writes an executable file. |
| The user declines the install | The choice belongs to the user. |

The API backend needs Python, `pip install requests pyyaml`, and the `DL_USERNAME` and
`DL_PASSWORD` environment variables. `backend-api.md` § Prerequisites gives the detail.

## What the choice does not change

Both backends create the same entities on the same platform. They accept the same config
overrides, and they identify each entity by the same UUID. Stage files cite operations by § name,
and both backend files use the same § names.

So the choice of backend changes the commands only. It changes no result, and it changes no
stage protocol. A project can also move from one backend to the other, because an id from one
works in the other.

Both files also use the same entity names, so a citation such as § The SeedDataset resolves in
either. `cli.md` § The entity model lists the command aliases the CLI accepts on top of those
names.
