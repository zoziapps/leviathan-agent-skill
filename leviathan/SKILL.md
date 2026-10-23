---
name: leviathan
description: "Read the user's Magic: The Gathering collection from the Leviathan app and create decks or cubes in it, via its AppleScript interface on macOS. Use when the user asks to build, suggest or check a deck or cube from cards they own, asks what is in their collection, or mentions Leviathan."
license: MIT
metadata:
  author: Stefan Pauwels, Zozi Apps
  version: "1.0.0"
---

# Leviathan

Leviathan is the user's Magic: The Gathering collection manager. On macOS it exposes nine AppleScript
commands. Everything below runs through `osascript`; the app launches on demand if it is not running.

```
tell application "Leviathan"
    export collection                 -- JSON text; optional: in folder "Folder Title"
    list card lists                   -- JSON text: every deck and cube with kind, format, card counts
    export card list "Mill"           -- the deck or cube as text in the import format
    create card list "<text>"         -- optional: named "Title", kind deck|cube, format "Modern", notes "..."
    add cards "<text>" to "Mill"      -- adds the lines to an existing deck or cube
    remove cards "<text>" from "Mill" -- removes the lines, board by board
    set notes "<text>" of "Mill"      -- replaces the deck's or cube's notes (plain text, shown under the title)
    open search "Counterspell"        -- shows the search, text optional
    open scan                         -- shows the card scanner
end tell
```

`export card list`, `add cards`, `remove cards` and `set notes` take `kind deck` or `kind cube` when a
deck and a cube share a title; otherwise the title alone is enough. Lists linked to Moxfield or CubeCobra are read only.

`sdef /Applications/Leviathan.app` prints the full dictionary. For a Debug build use
`tell application id "ch.zoziapps.Leviathan.debug"` instead of the name.

## Requirements

- **Local macOS execution.** The agent must be able to run `osascript` locally on the Mac where
  Leviathan is installed. If this capability is unavailable, explain that the skill cannot control
  Leviathan in the current environment and stop before attempting the workflows.
- macOS. The AppleScript interface does not exist on iOS or iPadOS; there, the same actions are available
  as Shortcuts actions in the Shortcuts app, which this skill does not drive.
- **Leviathan 2.5 or later.** Older versions have no scripting dictionary and `osascript` fails with
  "doesn't understand the export collection message". Check before the first command:

  ```bash
  osascript -e 'tell application "Leviathan" to version'
  ```

  If the version is below 2.5, or the app is not installed, tell the user to update or install Leviathan
  from the Mac App Store and stop. Do not fall back to Shortcuts, reading the app's database, or guessing
  the collection.

## Building a deck or cube from owned cards

1. **Export to a file, never into the context.** A collection is several megabytes of JSON.

   ```bash
   osascript -e 'with timeout of 300 seconds
   tell application "Leviathan" to export collection
   end timeout' > collection.json
   ```

2. **Filter with a script** (Python or jq) to a short candidate list, then read only those entries.
   Keep `name`, `set`, `collectorNumber`, `count`, `typeLine`, `manaCost`, `cmc`, `colors`, `oracleText`.
   Sum `count` per `name` across entries before deciding how many copies are available: the export has
   one entry per printing, finish and folder, so the same card can appear several times.

3. **Write the list in the text import format** and save it to a file. Give set and collector number so the
   owned printing is matched. Header lines set title, format and kind:

   ```
   name: Dimir Mill
   format: Modern
   type: deck
   4 Glimpse the Unthinkable (RAV) 212
   2 Mind Funeral (ARB) 29
   20 Island (UNF) 235

   Sideboard
   2 Negate (M20) 69
   ```

   A blank line or a line reading `Sideboard`, `Commander` or `Companion` starts that board. `type: cube`
   makes a cube. Lines the parser cannot read are reported back, not silently dropped.

4. **Create it**, reading the file so quoting is never a problem:

   ```bash
   osascript -e "tell application \"Leviathan\" to create card list (read POSIX file \"$PWD/deck.txt\" as «class utf8»)"
   ```

   The result is one line such as `Created deck "Dimir Mill" with 60 cards in 14 slots.` followed by any
   unrecognized lines. Relay that to the user. Then `open search` or `activate` brings the app forward.
   Printings the app has not seen before are completed from Scryfall in the background; give that a few
   seconds before you `export card list` to check the result, or a line still shows without its printing.

## Completing or improving an existing deck

1. **Find the deck.** `list card lists` gives exact titles, kinds and formats. Match the user's words to a
   title; if two lists fit, ask.

2. **Read the deck and the collection.** `export card list "<title>"` returns the deck in the import
   format, so each line already carries set and collector number. Export the collection to a file as
   above. Compare per card name: what the deck wants, what is owned, and, if the user cares about
   other decks, what other lists already use (export those too). Leviathan tracks decks and cubes marked
   as built, whose cards are set aside. Ignore those allocations by default unless the user explicitly
   asks to respect them; do not warn about sharing cards between lists unless that matters to the request.

3. **Decide the changes as a delta**, not a rewrite: cards to cut, cards to add, from owned cards unless the
   user asked for a wish list. Keep the deck's format legal and its size and copy limits intact.

4. **Apply the delta** with two calls, each read from a file:

   ```bash
   osascript -e "tell application \"Leviathan\" to remove cards (read POSIX file \"$PWD/cuts.txt\" as «class utf8») from \"Mill\""
   osascript -e "tell application \"Leviathan\" to add cards (read POSIX file \"$PWD/adds.txt\" as «class utf8») to \"Mill\""
   ```

   Both texts use the card line format with board words; a remove line without set and collector number
   removes any printing of that card. The results report what was changed and what was not found.
   Relay both, then `export card list` once more if the user wants to see the final list.

Never recreate an existing deck with `create card list` to change it: that makes a second list and loses
the slot history of the first.

## Saving the strategy guide in the deck or cube

Whatever you tell the user about the list, also save it in its notes, so it stays with the list in the
app. For a deck: game plan, key cards and why they are in, mulligan and sideboard advice, and what is
missing or assumed (basics, cards not owned). For a cube: what the cube is about, the era or card pool,
the supported archetypes per color pair and their key cards, how the fixing and the mana work, and what
is missing or assumed. Write it as plain text with short paragraphs and descriptive section titles on their
own lines, separated by blank lines. No Markdown markup, since the app shows notes verbatim. Start with
a short overview that works as a preview. The app has a dedicated reader/editor for long descriptions,
so include useful detail without imposing a one-screen limit; avoid repetition.

- **New deck or cube:** pass the text with the `notes` parameter of `create card list`, read from a file:

  ```bash
  osascript -e "tell application \"Leviathan\" to create card list (read POSIX file \"$PWD/deck.txt\" as «class utf8») notes (read POSIX file \"$PWD/guide.txt\" as «class utf8»)"
  ```

- **Existing deck or cube:** after the delta is applied, write the text with `set notes`:

  ```bash
  osascript -e "tell application \"Leviathan\" to set notes (read POSIX file \"$PWD/guide.txt\" as «class utf8») of \"Mill\""
  ```

  `set notes` replaces the whole text. `list card lists` returns the current notes; keep what the user
  wrote themselves above your text, and replace only an earlier text of yours. Start it with a line
  such as `Strategy guide (AI-generated, 2026-10-20)` or `Cube overview (AI-generated, 2026-10-20)` so it is
  recognizable next time.

## Export fields

One JSON object per collection entry. Empty fields are omitted.

| Field | Meaning |
|---|---|
| `name`, `set`, `setName`, `collectorNumber`, `lang`, `scryfallId`, `oracleId` | The owned printing. |
| `count`, `finish` (`nonfoil`, `foil`, `etched`), `proxy`, `folder` | What the user has and where. |
| `used`, `usedIn`, `unused` | Copies of this card name, any printing, set aside in built, non-frozen decks and cubes, excluding maybeboards (matching the UI): `used` is the allocated total, `usedIn` lists `list`, `kind`, `board` and `count`, `unused` is the owned total of the name minus `used`, clamped to zero. Per card name, so every entry of the name shows the same numbers; do not sum them across entries. `used` and `usedIn` are absent when no copies are allocated; `unused` is always there. Ignore these allocations unless the user asks to respect them. |
| `typeLine`, `oracleText`, `manaCost`, `cmc`, `colors`, `colorIdentity`, `keywords`, `producedMana`, `power`, `toughness`, `loyalty`, `rarity`, `layout` | Gameplay data. |
| `releasedAt` (`YYYY-MM-DD`) | Release date of this printing. Use it for era filters such as "1995 to 2002". |
| `legal`, `restricted`, `banned` | Format names as Scryfall spells them: `Modern`, `Commander`, `Pauper`, `Legacy`, `Vintage`, `Pioneer`, `Standard`, `Oldschool`, `Premodern`. |
| `faces` | Present on multi-face cards, each with `name`, `typeLine`, `oracleText`, `manaCost`, `colors`. The top-level `oracleText` is empty for them. |
| `priceUSD`, `priceEUR` | Last known prices. |
| `ignorePrinting` | True when the user did not record which printing they own. |

Entries whose card has no `oracleText` and no `faces` are placeholders the app has not completed yet;
treat them as unknown cards.

## Rules of thumb

- Only use cards the export contains, and never more copies than the summed `count`, unless the user
  asks for a wish list. Say which cards are missing if a classic build needs them.
- **Ignore built-list allocations by default.** Leviathan lets the user mark decks and cubes as built,
  setting their cards aside. Unless the user explicitly asks otherwise, use the full owned `count`,
  including cards allocated to built lists. Do not add caveats about cards being in other lists or claim
  that Leviathan cannot set copies aside.
- **Respect allocations only when requested.** If the user asks for unused cards or to leave other built
  decks/cubes intact, use `unused` and `usedIn`: they account only for built, non-frozen lists, excluding
  maybeboards, just like the UI. When editing a built list, its existing cards are already included in
  `used`; allow it to keep or reuse its own copies rather than subtracting them twice. When restricting
  availability, explain which lists hold the copies excluded.
- **Proxies count as available by default.** A `proxy` entry is a card the user plays with. Skip proxy
  entries only when the user asks for no proxies; this is independent of respecting built-list allocations.
- **Basic lands are almost never tracked.** Most people do not enter their Plains, Islands, Swamps,
  Mountains and Forests, so the export shows few or none, and the few it shows are often odd printings
  like Secret Lair or foreign copies. Treat basics as available in any number: never shrink or reshape a
  deck because of recorded basic counts, never ask whether the user has enough. Write one line per basic
  and **pick a printing from a set that is legal in the deck's format**: a basic from the export if one
  fits, otherwise the set code alone, as in `6 Island (3ED)`, and the app chooses a printing from that
  set. Old School decks take Alpha, Beta, Unlimited or Revised basics (`LEA`, `LEB`, `2ED`, `3ED`);
  Premodern takes Fourth through Seventh Edition or any expansion up to Scourge (`4ED`, `5ED`, `6ED`,
  `7ED`, `INV`, `ONS`); Standard takes a set currently in Standard; Modern, Pioneer, Legacy, Vintage,
  Commander and Pauper accept any printing, so use what the export has or a recent core set. Plain
  `6 Island` without a set is the last resort, since it leaves the printing to the app. In the report, say
  how many basics the deck assumes. Only count-limit basics when the user says the collection is complete
  including lands.
- Respect the format's deck size and copy limits: 60 cards and 4 copies for constructed, 100 singleton
  for Commander, 360 or 540 singleton for a cube.
- For a cube, `kind cube` or `type: cube`; boards are ignored.
- Before creating or changing a list, check the obvious: deck size, copy limits, land count against the
  curve, every card legal in the stated format, every card actually owned in the count used.
- The free tier allows one deck and one cube. The command then fails with a script error naming the
  limit. Report it, do not retry.
- `open search` and `open scan` can arrive before the window exists when the script launches the app
  cold. Call `activate` first and retry once if nothing happened.
