<p align="center">
    <img src="assets/logo.png" alt="Leviathan app icon" height="100" />
</p>

<h1 align="center">Leviathan Agent Skill</h1>

<p align="center">
    <img src="https://img.shields.io/badge/macOS-14+-2980b9.svg" alt="Requires macOS 14 or later." />
    <img src="https://img.shields.io/badge/iOS%20%2F%20iPadOS-not%20supported-c0392b.svg" alt="Not available on iOS or iPadOS." />
    <img src="https://img.shields.io/badge/Leviathan-2.5+-8e44ad.svg" alt="Requires Leviathan 2.5 or later." />
</p>

An agent skill that lets AI assistants work with your Magic: The Gathering collection in the **Mac version** of [Leviathan](https://apps.apple.com/app/leviathan-magic-organizer/id6460975870). The assistant reads the cards you own, builds decks and cubes from them or improves existing ones, and saves a strategy guide in each list's notes.

It uses the [Agent Skills](https://agentskills.io/home) format, so it works with Claude Code, Codex, Gemini, Cursor and other agents that can run commands on your Mac.

Ask for example:

- "Build me a Modern mill deck from my collection."
- "Improve my Pauper Elves deck with cards I have."
- "Make a 360-card Premodern cube from my cards."
- "Which of my cards are legal in Old School?"


## Requirements

> [!IMPORTANT]
> **Mac only.** The skill controls Leviathan through AppleScript, which exists only on macOS. It does not work with Leviathan on iPhone or iPad, and not with agents that run in the cloud or in a mobile app, such as the Claude or ChatGPT app on iOS. Your collection syncs through iCloud, so changes the agent makes on the Mac show up on your iPhone and iPad as usual.

- A Mac with the Mac version of [Leviathan](https://apps.apple.com/app/leviathan-magic-organizer/id6460975870), 2.5 or later, installed.
- An agent that runs commands locally on that Mac, such as Claude Code, Codex or Gemini CLI in Terminal.
- On iPhone and iPad, the same actions (export the collection, create decks and cubes) are available in the Shortcuts app instead, which you can run yourself.
- The first time the agent talks to Leviathan, macOS asks whether your terminal or agent app may control Leviathan. Allow it.


## Installing

Install into Claude Code, Codex, Gemini, Cursor and more with npx:

```bash
npx skills add https://github.com/zoziapps/leviathan-agent-skill --skill leviathan
```

You can choose which agents to install it for, and whether it is available in one project or in all of them. If you get `npx: command not found`, install Node first, for example with [Homebrew](https://brew.sh): `brew install node`.

Claude Code users can also add this repository as a plugin marketplace and install the plugin from it:

```
/plugin marketplace add zoziapps/leviathan-agent-skill
/plugin install leviathan@leviathan-agent-skill
```

Alternatively, clone this repository and copy the `leviathan` folder wherever your agent looks for skills.


## Using

The skill is called `leviathan`. Most agents pick it up on their own when you mention your collection, a deck or Leviathan. You can also ask for it directly, with `/leviathan` in Claude Code or `$leviathan` in Codex.


## License

The Leviathan skill was created by Stefan Pauwels, [Zozi Apps](https://zoziapps.ch). It is available under the [MIT License](LICENSE).
