# Nomic Game

A Discord bot that runs a game of [Nomic](https://en.wikipedia.org/wiki/Nomic) — a game where players win by changing the rules of the game itself.

The rules are stored as executable Python in a separate repo. Players submit
rule changes as unified diffs (`.patch` files) via slash commands; passing
proposals are applied to `rules.py` and committed to its git history, so every
rule change leaves an audit trail.

## Repository layout

This is an umbrella repo containing two git submodules:

- **[nomic-engine](nomic-engine/)** — Discord bot (Python + discord.py). Runs the
  game lifecycle, validates patches, hosts polls, applies passed changes.
- **[nomic-rules](nomic-rules/)** — A single `rules.py` file. The *only* file
  players can change via proposals. Tracked in its own git history so the
  audit log isn't polluted by bot code.

```sh
git clone --recurse-submodules <this-repo>
# or, if already cloned:
git submodule update --init
```

## Deployment

### Prerequisites

- Docker & Docker Compose
- A Discord server you have admin rights on
- For production: any host that runs Docker (VPS, Railway, Fly.io, Hetzner, etc.)

### 1. Create the Discord application

1. Go to [discord.com/developers/applications](https://discord.com/developers/applications)
2. Click **New Application** and name it (e.g. "Nomic Bot")
3. Open the **Bot** tab:
   - Click **Reset Token** and save it — this is `DISCORD_TOKEN`
   - Enable **Message Content Intent** under "Privileged Gateway Intents"
4. Note the **Application ID** from the General Information tab for the invite URL below

### 2. Configure environment

```sh
cd nomic-engine
cp .env.example .env
```

Edit `.env`:

- `DISCORD_TOKEN` — the bot token from step 1
- `DISCORD_GUILD_ID` — your server's ID. With this set, slash commands appear
  instantly in that guild. Leave unset for global commands (can take up to an
  hour to propagate to all servers). Right-click your server → "Copy Server
  ID" (requires Developer Mode in Discord's Advanced settings).

### 3. Invite the bot to your server

In the Developer Portal, go to **OAuth2 → URL Generator**:

- **Scopes:** `bot`, `applications.commands`
- **Bot permissions:** Send Messages, Read Message History, View Channels,
  Use Slash Commands, Create Polls

Open the generated URL in your browser and choose your server. The bot does
*not* need Administrator — admin-only commands (`/newgame`, `/startgame`,
`/endgame`) are gated on the *user* having **Manage Server** permission, not
the bot.

### 4. Initialize the rules repo

The engine git-commits each accepted rule change to `nomic-rules`, so it
needs its own git history. If you cloned this repo with
`--recurse-submodules`, you're done. Otherwise:

```sh
git submodule update --init
```

If `nomic-rules/.git` is missing for any reason:

```sh
cd nomic-rules && git init && git add rules.py && git commit -m "Initial rules"
```

### 5. Bring up the bot

From `nomic-engine/`:

```sh
docker compose up --build -d
docker compose logs -f
```

You should see `Nomic bot ready: ...` in the logs. In Discord, run
`/directions` to confirm slash commands work, then `/newgame` to start your
first game.

`docker-compose.yml` bind-mounts `../nomic-rules` into the container at
`/app/rules` so the engine can read and patch `rules.py` directly. Game
state (players, proposals, scores) lives in the named volume `nomic-data`.

### Updating

```sh
git pull --recurse-submodules
cd nomic-engine
docker compose up --build -d
```

The `nomic-data` volume persists across rebuilds, so game state survives
upgrades. **If you change the schema in `state.py`**, you'll need to wipe
the volume: `docker compose down -v` (destroys scores and proposals — start
a fresh game).

## Quick play loop

1. An admin runs `/newgame` to open a join phase.
2. Players run `/join`.
3. An admin runs `/startgame` to close the roster and begin play.
4. On their turn, the active player runs `/propose <description> <file.patch>`
   with a unified diff against `rules.py`.
5. A Discord poll opens for 48 hours (configurable). Players vote with the
   native poll UI.
6. When the poll closes (or when the proposer runs `/tally`), the engine
   counts votes, applies the patch if it passes, awards points, and advances
   the turn to the next player.
7. First player to hit `TARGET_SCORE` (default 100) wins.

Players can also run `/directions` inside Discord for a quick rules summary.

## How rule changes work

`rules.py` is executable Python loaded by the bot at startup. Items can be
tagged with `# #immutable` (locked) or `# #mutable` (freely patchable):

```python
QUORUM_FLOOR = 2  # #immutable

# #mutable
def compute_quorum(players: list) -> int:
    return max(2, (len(players) + 1) // 2)
```

The engine validates every patch through three checks before accepting it:

1. **AST safety** — no file I/O, no networking, no subprocesses, no
   introspection tricks (`__class__`, `getattr`, `__subclasses__`, etc).
2. **Immutability** — `#immutable` items can't be silently modified or have
   their tag removed.
3. **Transmutation detection** — a patch *can* downgrade an `#immutable` tag
   to `#mutable`, but the proposal is then flagged as a transmutation and
   requires unanimous YES from every non-proposer player to pass.

Combined, this means: to change the body of an immutable rule, you must
first transmute it (hard — unanimous), then submit a second amendment (easy —
majority). The same applies to the transmutation threshold itself, so the bar
protects itself.

## Engine-enforced floors

The engine enforces minimum voting requirements *outside* `rules.py`, in
code players can never modify:

- Total participation ≥ 2 votes
- YES fraction ≥ 50%
- For transmutations: full non-proposer participation, unanimous YES

Mutable rules can make voting *stricter* (e.g. require 75% threshold) but
never *looser*. A malicious patch can't, for example, redefine
`compute_quorum` to `return 0` and then pass everything with one vote.

## Making a patch

A proposal is a [unified diff](https://en.wikipedia.org/wiki/Diff#Unified_format)
against `rules.py`. The bot only accepts patches that touch `rules.py` — any
other path in the patch is rejected.

### With git (recommended)

```sh
git clone <nomic-rules-url> && cd nomic-rules
$EDITOR rules.py            # make your change
git diff > my_change.patch  # generate the patch (don't commit)
```

### With plain diff

```sh
# In Discord, run /rules and save the output to rules.py
cp rules.py rules.py.orig
$EDITOR rules.py
diff -u rules.py.orig rules.py > my_change.patch
```

### Example: raise the target score from 100 to 200

```diff
--- a/rules.py
+++ b/rules.py
@@ -86,7 +86,7 @@
 MIN_PLAYERS_TO_START = 2  # #mutable
-TARGET_SCORE = 100        # #mutable
+TARGET_SCORE = 200        # #mutable

 # #mutable
 def can_start_game(players: list) -> bool:
```

Submit with `/propose description:"Raise win threshold to 200" patch:my_change.patch`.

### Tips

- **Test locally first.** Run `patch -p1 < my_change.patch` against a fresh
  copy of `rules.py`. If it doesn't apply cleanly on your machine, it won't
  apply on the bot either.
- **One change per patch.** Smaller patches are easier to read and easier to
  pass. A sweeping rewrite that touches ten things will probably lose the
  vote on the one thing players disagree with.
- **Don't change immutable items in the same patch as their transmutation.**
  Split it: first patch flips `# #immutable` → `# #mutable` (transmutation),
  then a follow-up patch changes the body.
- **Filename must end in `.patch`** and the file must be valid UTF-8, ≤64 KB.

## Commands

**Lifecycle (admin):** `/newgame`, `/startgame`, `/endgame`

**Player:** `/join`, `/turn`, `/players`, `/scores`, `/score`, `/directions`

**Proposals:** `/propose`, `/amend`, `/withdraw`, `/tally`, `/proposals`,
`/proposal`

**Rules:** `/rules` (show current `rules.py`), `/ruleinfo` (key constants)

## Project status

Early development. The schema is not versioned — if you change `state.py`
schema, delete the `nomic-data` volume to recreate.
