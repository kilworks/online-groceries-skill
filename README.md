# online-groceries-skill

A Claude Code skill that builds an online grocery basket on French retail sites (Carrefour, Courses U / Super U, with Leclerc, Auchan and Picard for comparison), starting from the store's promotions instead of searching products one by one.

It works through your own logged-in browser (Claude in Chrome or a similar tool). It never pays, never types a password and never creates an account: you log in, you check the basket, you pay.

## Install

```sh
git clone https://github.com/kilworks/online-groceries-skill
cp -r online-groceries-skill/online-groceries ~/.claude/skills/
```

Then ask Claude "do my groceries" (or "fais mes courses"). On first run it asks for your postcode, household, budget, diet and equipment, and saves them to `~/.claude/skills/online-groceries/preferences.md`, which never leaves your machine.

Prices, fees and site endpoints were observed in 2026 and change over time; the skill reads live values before relying on them.
