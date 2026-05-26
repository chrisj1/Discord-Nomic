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

### 4. Set up the rules repo (don't use the submodule for deployment)

The engine git-commits to `nomic-rules` on every accepted proposal, and the
`rules-http` sidecar serves it via smart HTTP on port 8418. Both of these
require `nomic-rules/.git` to be a real directory, but in a submodule it's
a *file* pointing to `../.git/modules/nomic-rules` — a path that doesn't
exist inside the containers.

**Clone the rules repo as a standalone checkout** outside the umbrella,
then point the engine at it via `.env`:

```sh
git clone https://github.com/chrisj1/Discord-Nomic-rules.git ~/nomic-rules-live
echo "RULES_REPO_PATH=$HOME/nomic-rules-live" >> ~/Discord-Nomic/nomic-engine/.env
```

(The submodule directory inside the umbrella is fine for local development
of the rules — it's only a problem when bind-mounted into a container.)

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

A second container (`rules-http`) runs nginx + `git-http-backend` on
**port 8418** and exposes the same `nomic-rules` directory read-only over
smart HTTP. Players `git clone` it to pull the latest rules to diff
against. Because it's plain HTTP, it sits cleanly behind any HTTP reverse
proxy — this deployment uses Cloudflare Tunnel to expose it as
`https://nomic.chrisjerrett.com/nomic-rules`. If you don't want external
access, leave the `ports:` mapping LAN-only and players will need to fall
back to the `/rules` Discord download.

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
TRANSMUTATION_THRESHOLD = 1.0  # #immutable

# #mutable
def compute_quorum(players: list) -> int:
    return max(2, (len(players) + 1) // 2)
```

The engine validates every patch through four checks before accepting it:

1. **AST safety** — no file I/O, no networking, no subprocesses, no
   introspection tricks (`__class__`, `getattr`, `__subclasses__`, etc).
   `hashlib` and `random` *are* allowed since they're pure computation.
2. **Immutability** — `#immutable` items can't be silently modified or have
   their tag removed.
3. **Transmutation detection** — a patch *can* downgrade an `#immutable` tag
   to `#mutable`, but the proposal is then flagged as a transmutation and
   requires unanimous YES from every non-proposer player to pass.
4. **`is_valid_patch`** — a mutable callback in `rules.py`. Players can vote
   in any gameplay-level rule (size limit, MD5 proof-of-work, description
   format, per-player cooldown, etc.) and the engine enforces it on every
   future proposal.

Combined, this means: to change the body of an immutable rule, you must
first transmute it (hard — unanimous), then submit a second amendment (easy —
majority). The same applies to the transmutation threshold itself, so the bar
protects itself.

## Other mutable callbacks the engine consults

These all live in `rules.py` and can be changed via proposals:

- **`compute_quorum(players)`** / **`compute_passing_threshold(players)`** — voting math.
- **`next_player(current_id, players)`** — turn order (round-robin by default; can be random, score-weighted, etc.).
- **`can_propose(proposer_id, turn_id, players)`** — gates `/propose`.
- **`can_vote(voter_id, proposer_id, players) -> int`** — returns the *weight* of
  the voter's vote. `True`/`False` (legacy bool) and `1`/`0` mean one vote /
  excluded. Bigger ints = weighted votes. A patch could make this
  `random.randint(1, 5)` for chaotic weighted voting, or scale by player
  points for plutocracy.
- **`check_winner(players)`** — return a discord_id to end the game.
- **`award_points(passed, proposer_id, players)`** — returns `{id: delta}` for
  whatever scoring scheme you want.

## Engine-enforced floors

The engine enforces minimum voting requirements *outside* `rules.py`, in
code players can never modify:

- Total participation ≥ 1 vote
- YES fraction ≥ 50%
- For transmutations: full non-proposer participation, unanimous YES

Mutable rules can make voting *stricter* (e.g. require 75% threshold) but
never *looser*. A malicious patch can't, for example, redefine
`compute_quorum` to `return 0` and pass with zero votes.

## What the bot announces

After every tally (whether the poll expired or `/tally` was called early):

- The verdict (`✅ Passed (3✅ 1❌)` or `❌ Failed`)
- A `patch applied` note when applicable
- A `points: <@alice> +10 → 35` line for every non-zero award, including
  the player's new total
- A separate `🎯 Next turn: <@bob>` message for the next proposer (unless
  someone just won)

## Making a patch

A proposal is a [unified diff](https://en.wikipedia.org/wiki/Diff#Unified_format)
against `rules.py`. The bot only accepts patches that touch `rules.py` — any
other path in the patch is rejected.

### With git from the running bot (recommended — always current)

The engine ships a read-only smart-HTTP git server that serves the live
`rules.py` repo. Clone it once, then `git pull` to refresh before each new
proposal:

```sh
git clone https://nomic.chrisjerrett.com/nomic-rules
cd nomic-rules
git pull                     # before each new patch
$EDITOR rules.py             # make your change
git diff > my_change.patch   # generate the patch (don't commit)
git checkout rules.py        # reset for the next patch
```

### With plain diff (no git access)

```sh
# In Discord, run /rules and save the attached file as rules.py
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
