# todo — a Claude Code skill for Clear

Manage your [Clear](https://useclear.com) to-do lists from Claude Code:
frictionless adds (single or batch), a persistent default list, fuzzy
list-name matching that asks instead of guessing, and confirmation gates on
anything destructive.

```
> add milk, eggs, and bread to my groceries list

✓ Added to Groceries: "milk", "eggs", "bread"
  … milk, eggs, bread
```

*(illustrative — actual output depends on what's already on the list)*

## Prerequisites

- [Claude Code](https://claude.com/claude-code)
- macOS or Linux (Windows is not supported)
- Node.js + npm (for the Clear CLI)
- A Clear account (Clear Pro required only for creating new lists)

## Install

From the root of this repository:

```bash
mkdir -p ~/.claude/skills/todo
cp SKILL.md ~/.claude/skills/todo/SKILL.md
```

## First run — what to expect

The first time you use the skill it checks a few things and fixes what's
missing:

1. **CLI missing?** Installs it with `npm install -g @useclear/cli`.
2. **Not signed in?** Runs `clear-cli login`, which prints a `localhost` URL
   for you to open and complete sign-in (Apple or Google) in your browser.
3. **No default list yet?** Asks which list should be the default (or helps
   you create one, if your account supports it) and saves the choice to
   `~/.config/clear-skill/config.json`.

None of these steps run again once they've succeeded — later invocations go
straight to your request.

## Usage & configuration

```
/todo buy milk                  # → your default list
/todo buy milk @groceries       # → the Groceries list
```

Or just say it: "add eggs to my groceries list", "what's on my shopping
list?", "check off milk", "set my default list to Work". Paste multiple
lines to add them all as separate items — one confirmation for the whole
batch, not one per line.

Config lives at `~/.config/clear-skill/config.json` as
`{ "defaultList": { "id", "title" }, "version": 1 }`. Change the default any
time by asking — no need to redo first-run setup.

## Safety & limitations

- **Adds and reads never wait on you.** Anything that can't lose data —
  adding, viewing, checking off — happens immediately.
- **Anything that removes items or loses access always confirms first**:
  removing items (`rm`), archiving, sweeping completed items, and signing
  out. You'll see exactly what's about to happen before it happens.
- **Never guesses between two lists with the same name.** If your wording
  could mean more than one list, it asks — showing each candidate's item
  counts and enough of its id to tell them apart.
- **Whole-list deletion is currently disabled** in this skill — a cross-client
  sync issue in the CLI's `delete` command is pending an upstream fix. Ask to
  archive instead (reversible, and known to work).
- **Sharing a list** mints a fresh access link every time you ask, and the
  skill tells you that before running it — anyone holding the link can join
  the list.

## Troubleshooting

- **Session expired mid-task** — the skill notices, re-runs sign-in, and
  retries automatically.
- **`npm install` fails with a permissions error (EACCES)** — don't run it
  with `sudo`. Run the install command yourself in your own shell, which may
  have a different Node/npm setup (e.g. nvm) that resolves it cleanly.
- **"Needs Clear Pro" when creating a list** — your account doesn't have Pro;
  add to an existing list instead.
- **Something about themes or sharing looks wrong** — check "Known CLI
  quirks" in `SKILL.md` before assuming it's broken.

## Compatibility & privacy

Built against `@useclear/cli` as of the skill's authoring; the skill defers
to `clear-cli --help` if your installed version disagrees with what's
documented. macOS and Linux only. The only thing this skill stores locally is
your default list's id and title, in the config file above.

## Uninstall

```bash
rm -rf ~/.claude/skills/todo ~/.config/clear-skill
```

Optionally also `clear-cli logout` (signs the CLI out) and
`npm rm -g @useclear/cli`.

## Evals

`evals/evals.json` has the test prompts; its `setup` field explains the
fixture lists to create in your own account before running them.

## License

MIT — see [LICENSE](LICENSE).
