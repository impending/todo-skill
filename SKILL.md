---
name: todo
description: Add, view, and manage items on the user's Clear to-do lists via the clear-cli command-line tool (npm package @useclear/cli). Use this whenever the user wants to add something to a to-do or shopping list, check off or remove items, look at what's on a list, or manage their Clear lists (create, rename, archive, share, theme). Triggers include "/todo", "add X to my [name] list", "put X on the groceries list", "todo: X", "check off X", "what's on my [name] list", "sweep my [list]", and "set my default list". Also handles first-time setup: installing clear-cli, signing in, and picking a default list. Do NOT use this for generic conversation about to-dos/lists that isn't about the user's actual Clear account, for discussing Clear-the-app's own development/features, for `todo:` comments found in source code, or for "clear the conversation/screen" requests.
---

# todo — Clear list assistant

Fronts `clear-cli` (npm `@useclear/cli`). The two things this skill protects
hardest are **never writing to the wrong list** (accounts routinely have dozens
of lists, some sharing a title) and **never deleting something without the user
explicitly confirming what's about to be deleted**. Speed everywhere else;
caution only where damage is irreversible or hard to notice.

The flags, exit codes, and command shapes below reflect the CLI version at
authoring time. If a command errors on an unknown flag, or `clear-cli
<command> --help` disagrees with this file, trust `--help` and adapt.

## Setup (first run, or whenever something's broken)

Config lives at `~/.config/clear-skill/config.json`:
```json
{ "defaultList": { "id": "<uuid>", "title": "<title>" }, "version": 1 }
```
(`version` is a schema-migration marker for future changes to this file's
shape — always write `1` today; nothing currently reads or branches on it.)
When writing the config, create the directory first (`mkdir -p
~/.config/clear-skill`) — it won't exist on a fresh machine.

**Changing an existing default**: when the user asks to change their default
list ("set my default list to Work", "/todo default work"), resolve the name
the normal way (see "Resolving a list name") and overwrite `config.json` with
the new `{id, title}` — don't run the full setup flow for this.

Run through the steps below whenever the config is missing, unreadable, or its
`defaultList.id` no longer resolves against the account (see "Resolving a list
name" below):

1. **CLI present?** `command -v clear-cli`. If missing, check for `node`/`npm` first
   (`command -v npm`) — if those are missing too, stop and tell the user to install
   Node.js (https://nodejs.org) themselves; don't attempt anything. Otherwise run
   `npm install -g @useclear/cli`. If that fails with a permissions error (EACCES),
   don't retry with sudo — explain it's an npm permissions issue and give the user
   the exact command to run themselves (their own shell may have a different
   npm/node setup, e.g. nvm, that resolves it cleanly).

2. **Signed in?** `clear-cli whoami`. Exit code 3 means not signed in. Run
   `clear-cli login` — this prints a `localhost` URL and waits for the user to
   finish sign-in (Apple or Google) in their browser. Since this can take a
   while, run it in the background, surface the printed URL immediately so the
   user can click it, and check `whoami` every 5-10 seconds rather than
   tightly looping — after a few checks, just wait for the user to say they're
   done instead of continuing to poll.

3. **Default list set?** If `config.json` doesn't exist yet, or its list id
   doesn't show up in a fresh `clear-cli lists --json`, this account needs a
   default picked. Fetch the lists and:
   - Zero lists (brand-new account): try `clear-cli create` for a starter
     list (suggest a name like "To-dos"). If that fails because the account
     lacks Clear Pro, explain plainly that a first list must be created in the
     Clear app itself, wait for the user to do that, then re-fetch and
     continue — don't leave them stuck with no default.
   - Few lists (roughly ≤10): show them and ask which should be the default.
   - Many lists: ask the user to just say a name or two and resolve it the
     normal way (see "Resolving a list name"), rather than dumping the whole
     account on them.
   - Offer "create a new list" as an option too (needs Clear Pro — see below).
   Write the result to `config.json` as `{id, title}`.

4. Confirm with one line ("You're set — default list is **Groceries**. Try
   `/todo buy milk`.") and move on to whatever the user actually asked for.

## The list cache

Fetch `clear-cli lists --json` once per conversation and reuse it for name
resolution — no need to refetch on every single add. **Do** refetch:
- immediately before showing a destructive-op confirmation (the counts in that
  confirmation must reflect the current state, not a stale cache)
- after any `create`/`rename`/`archive`/`unarchive`/`move-list` this
  skill just performed (`delete` is currently disabled — see below)
- if a resolution attempt comes back empty (the cache may be stale)

Entries with a non-null `error` field aren't real lists — exclude them from
matching candidates entirely, *unless* the user explicitly asked for that
exact list (by name or id) or it's the configured default, in which case
surface the error and stop rather than silently falling through to something
else.

`clear-cli lists --archived` shows archived lists separately — only fetch
this when relevant (e.g. resolving a target for `unarchive`, or the user asks
about an archived list by name).

## Resolving a list name

Work through these in order and stop at the first one that produces a single
confident answer:

1. **No list mentioned** → use the configured default (after confirming its id
   still resolves — see Setup step 3).
2. **Exact title match** (case-insensitive, ignoring stray leading/trailing
   emoji or whitespace), and it's the *only* list with that title → use it.
   If the exact title matches more than one list (duplicate titles are
   common — think several lists all named after the same person), that's not
   confident — fall through to step 4.
3. **Unique prefix or substring match** → use it.
4. **Semantic match** — the user's wording clearly points at one list even
   though it's not a literal substring (e.g. "swim gear" → "Swimming",
   "warehouse run" → "Costco"). Use it only when one candidate is clearly
   the best fit; name the matched list explicitly in your response so a wrong
   guess is obvious to the user rather than silently absorbed.
5. **Still ambiguous** (duplicate titles, no clear semantic winner, or several
   plausible candidates) → ask the user, showing each candidate's title, open/
   done counts, and enough of its id to tell duplicates apart. Never guess
   between two lists with the same name.

One exception: destructive operations (below) fold any non-exact-unique
resolution into their confirmation step rather than auto-proceeding.

## Adding items (the hero path)

```
clear-cli add <list-id> [--top | --at <n>] [--header] -- "<item text>"
```

- `/todo buy milk` → resolves to the default list.
- `/todo buy milk @groceries` or "add buy milk to groceries" → resolves "groceries".
- **Always pass item text after `--`** (e.g. `add <id> -- "-1 discount item"`).
  Without it, text starting with `-` gets parsed as an unknown CLI flag and the
  add fails. This isn't a style preference — it's the difference between the
  add working and erroring on ordinary user text like "-5% off coupon".
- **Multi-line input is a batch, not one request**: if the user pastes several
  lines, add each one as its own item, in order. If a line fails partway
  through, keep going with the rest and report which lines succeeded and which
  didn't — don't abandon the whole batch over one bad line. If different lines
  are clearly headed to different lists, group by list and give each its own
  confirmation receipt.
- After the batch is fully added (not after each individual item), confirm
  with the resolved list name (this is what makes a wrong fuzzy match
  visible) and show the tail of the list from one *fresh* `clear-cli show
  <id>` call — a handful of the most recent open items, not the whole list:
  ```
  ✓ Added to Groceries: "buy milk"
    … (last few open items)
  ```

## Everything else

| Intent | Command |
|---|---|
| what's on a list | `clear-cli show <id>` |
| overview of lists | `clear-cli lists` |
| check off / uncheck | `clear-cli check <id> <items…>` / `clear-cli uncheck <id> <items…>` (items by number from `show`, or by text) |
| edit item text | `clear-cli edit <id> <item> "<new text>"` |
| reorder an item | `clear-cli move <id> <item> --to <n>` (1-based target position among open items) |
| reorder a list | `clear-cli move-list <id>` |
| new list | `clear-cli create "<title>"` — needs Clear Pro; if it errors on that, say so plainly and suggest an existing list instead of retrying |
| rename a list | `clear-cli rename <id> "<new title>"` |
| bring back from archive | `clear-cli unarchive <id>` |
| share a list | `clear-cli share <id>` — mints a **fresh** link each time that grants access to whoever holds it; mention that before running it |
| themes / fonts | `clear-cli themes`, `clear-cli fonts` to browse; `clear-cli theme <id> [themeId]`, `clear-cli font <id> [fontId]` to get/set |
| account | `clear-cli whoami`; `clear-cli logout` |

### Known CLI quirks

When a user reports something that sounds like a bug, check here before
concluding the command actually failed:

- **Share link thumbnail**: the social-preview thumbnail on a shared link is
  always the app default, not the list's actual look — not fixable from here.
- **Theme visibility**: theme changes may only be visible to the account that
  set them on a shared list — if a user says a theme "didn't show up" for
  someone else, that's likely why, not a broken `theme` command.
- **`delete` and cross-client sync**: see below — currently disabled.

## Destructive operations — always confirm

`clear-cli rm`, `clear-cli archive`, `clear-cli sweep`, and `clear-cli logout`
can lose data or access with no undo. Handle all of them the same way:

1. Resolve the target fresh — this is the same refetch required by "The list
   cache" above, so it's one `lists --json` call, not two. For `rm`, also
   re-fetch `show <id>` so the item numbers you're about to act on are
   current, not from an earlier turn in the conversation.
2. State exactly what will happen, concretely — not "clear the list" but
   "Sweep **Groceries** (id …4F2A, 3 open · 12 done) — remove all 12
   completed items? Repeating items get re-added at the top." For `rm`, list
   every item text that will be removed, not just how many.
3. Ask for one explicit confirmation (AskUserQuestion works well here). If the
   list itself was resolved ambiguously (matching steps 3–5 above), this is
   also where that gets confirmed — one question, not two — since the user is
   already reading a concrete restatement of the target.
4. Only then run the command.

`rm`, `archive`, `sweep`, and `logout` have no CLI-level confirmation of
their own — this skill's confirmation step is the only safety net, so don't
skip it even though the command itself won't stop you. (`unarchive` is the
reversal of `archive` and needs no confirmation.)

### `clear-cli delete <list>` — currently disabled, do not run

**Do not run `clear-cli delete` for any reason, even with explicit user
confirmation.** Deletes issued through it appear to succeed (the CLI reports
success and the list disappears from `lists --json`), but the Clear mobile
app has been observed still showing "deleted" lists after a full refresh —
so the delete may not be reliable end-to-end. Until this is confirmed fixed
CLI-side:

- If the user asks to delete a whole list, explain that list deletion is
  temporarily turned off in this skill pending a CLI fix, and offer `archive`
  instead (reversible via `unarchive`, and known to work) or `sweep` if what
  they actually want is just to clear completed items.
- If the user insists on deleting anyway, tell them to run
  `clear-cli delete <list> --yes` themselves directly in a terminal — don't
  run it on their behalf.
- Re-evaluate this restriction once someone has verified a delete shows up
  correctly removed on all clients, not just via the CLI's own `lists --json`.

## When things go sideways

- **Exit code 3 mid-task** ("not signed in") — the session expired. Re-run the
  sign-in flow from Setup step 2, then retry the original command once.
- **`create` fails without Pro** — report it plainly ("this account doesn't
  have Clear Pro, so I can't create new lists") and suggest adding to an
  existing list instead.
- **A list in the cache has a non-null `error`** — see "The list cache" above;
  treat it as unusable unless it's the explicit target.
