# todo — a Claude Code and Codex skill for Clear

Manage your [Clear](https://useclear.com) to-do lists from Claude Code or Codex:
frictionless adds (single or batch), a persistent default list, fuzzy
list-name matching that asks instead of guessing, and confirmation gates on
anything destructive.

```
> add milk, eggs, and bread to my groceries list

✓ Added to Groceries (top): "milk", "eggs", "bread"
  1 · milk
  2 · eggs
  3 · bread
```

*(illustrative — actual output depends on what's already on the list)*

## Prerequisites

- [Claude Code](https://claude.com/claude-code) or [Codex](https://developers.openai.com/codex)
- macOS or Linux (Windows is not supported)
- Node.js + npm (for the Clear CLI)
- A Clear account (Clear Pro required only for creating new lists)

## Install

Claude Code:

```bash
mkdir -p ~/.claude/skills/todo && curl -fsSL https://raw.githubusercontent.com/impending/todo-skill/main/SKILL.md -o ~/.claude/skills/todo/SKILL.md
```

Codex:

```bash
mkdir -p ~/.codex/skills/todo && curl -fsSL https://raw.githubusercontent.com/impending/todo-skill/main/SKILL.md -o ~/.codex/skills/todo/SKILL.md
```

Start a new session afterwards so the skill is picked up. In Codex, the
first `clear-cli` command needs network access, so Codex may ask you to
approve it.

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
/todo default                   # → shows your current default list
```

In Codex, start with `$todo` instead of `/todo` (for example `$todo buy milk`),
or just say it in plain words.

Or just say it: "add eggs to my groceries list", "what's on my shopping
list?", "check off milk", "set my default list to Work", "what's my default
list?". Paste multiple lines to add them all as separate items — one
confirmation for the whole batch, not one per line.

Config lives at `~/.config/clear-skill/config.json` as
`{ "defaultList": { "id", "title" }, "version": 1 }`. Change the default any
time by asking — no need to redo first-run setup.

**New items go to the top by default.** Say "at the bottom", "at position
N", or "after X" to override. Within one position — everything typed with no
override, or everything said to go "after X" — a pasted batch keeps your
typed order. Mixed positions in one paste ("A at the bottom, B at the top")
are treated as separate requests, so B can land above A. "After X" needs X to
already be an open item on the list, uniquely matched, and not itself a line
from the same paste — an ambiguous or missing anchor gets asked about rather
than guessed.

**Pasted text that doesn't read as a task gets a rewrite proposed first.**
This runs on every add, including single items — but plain actions ("buy
milk") and shopping-list nouns ("eggs") go straight through, no prompt.
Notes ("Sarah mentioned the API is slow"), vague topics ("taxes"), bundled
multi-task lines ("call dentist and book flights"), and dictation filler get
a proposed verb-first rewrite instead, shown alongside the original, with one
confirmation for the whole batch — add the rewrites, add as written, or edit.
Prefix a request with `verbatim:` (or just say "as-is") to skip the check
entirely; `--header` adds are always exempt.

## Safety & limitations

- **Adds and reads never wait on you** — with one exception: text that
  doesn't read as a task-shaped item pauses once for a proposed rewrite (see
  Usage & configuration above) before it's added. Plain actions and
  shopping-list nouns still go straight through. Nothing here can lose
  data; viewing and checking off always happen immediately.
- **Anything that removes items or loses access always confirms first**:
  removing items (`rm`), archiving, sweeping completed items, taking down a
  copy link, and signing out. You'll see exactly what's about to happen before it happens.
- **Never guesses between two lists with the same name.** If your wording
  could mean more than one list, it asks — showing each candidate's item
  counts and enough of its id to tell them apart.
- **Whole-list deletion is currently disabled** in this skill — a cross-client
  sync issue in the CLI's `delete` command is pending an upstream fix. Ask to
  archive instead (reversible, and known to work).
- **Sharing a list** mints a fresh access link every time you ask, and the
  skill tells you that before running it — anyone holding the link can join
  the list.
- **Sending a copy** (`send-copy`) gives a read-only link to a frozen copy
  of a list. Anyone holding the link can read it. Short links can be updated
  by sending again, or taken down; `--long` links can never be taken down.

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
- **Asked "which one did you mean?" after saying "after X"** — X didn't
  uniquely match an existing open item (or matched none). The skill asks for
  a valid anchor rather than guessing a position.

## Compatibility & privacy

Built against `@useclear/cli` as of the skill's authoring; the skill defers
to `clear-cli --help` if your installed version disagrees with what's
documented. macOS and Linux only. The only thing this skill stores locally is
your default list's id and title, in the config file above.

## Uninstall

```bash
rm -rf ~/.claude/skills/todo ~/.codex/skills/todo ~/.config/clear-skill
```

Optionally also `clear-cli logout` (signs the CLI out) and
`npm rm -g @useclear/cli`.

## Evals

`evals/evals.json` has the test prompts; its `setup` field explains the
fixture lists to create in your own account before running them.

## License

MIT — see [LICENSE](LICENSE).
