# Execution Backends

An execution backend is the client that runs the stages. `cli.md` runs them through the
`distil` command, and this file tells you how to set it up. Do it at the start of a project,
before the first stage.

## Set up the CLI

1. Run `distil --version`. If the command answers, use `cli.md` and go to step 4.
2. Say that you will install the CLI, then run the install script:

   ```bash
   curl -fsSL https://cli-assets.distillabs.ai/install.sh | sh
   ```

   The script writes one binary to `~/.local/bin/distil`. It changes nothing else.
3. Run `distil --version` again. If the command answers, use `cli.md`. If the shell does not
   find the binary, add `~/.local/bin` to `PATH` and run the command again. If the install
   failed, see § When the install is impossible.
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

## When the install is impossible

The install script supports Linux x86_64, Linux arm64, macOS on Intel, and macOS on Apple
Silicon only. The skill cannot run stages in these conditions:

| Condition | What to do |
|---|---|
| Windows | Run the CLI inside WSL. |
| No network route to `cli-assets.distillabs.ai` | Install from a machine that has one. |
| The environment permits no new binary | Run the project from a machine that does. |
| The user declines the install | The choice belongs to the user. Stop and say what the CLI is for. |

Without the CLI you can still prepare the data files and the config with the user, so they are
ready to run once the CLI is available.

## Entity names

`cli.md` uses the platform's entity names, so a citation such as § The SeedDataset resolves
there. `cli.md` § The entity model lists the command aliases the CLI accepts on top of those
names.
