# Word Hunt — Developer Context

## Deployment
- Hosted on Railway at `https://wordhunt.up.railway.app`
- Entry point: `server.js` (Node + Express + Socket.IO)
- No build step — static files served from `public/`

## File Map
| File | Owns |
|------|------|
| `server.js` | All game logic, state, socket events |
| `public/game.js` | Client rendering, socket handling, DOM updates |
| `public/game.html` | HTML structure (header, board, overlays) |
| `public/game.css` | All styles |
| `public/index.html` | Landing/join page |
| `words.js` | Word list, `getWords(n)` helper |

## Terminology (UI name → code name)
| UI | Code |
|----|------|
| Dawn Guild | `red` team |
| Dusk Guild | `blue` team |
| Pathfinder | `spymaster` role |
| Seeker | `operative` role |
| Abyss | `abyss` card color — instant loss for guessing team |
| Treasure | `treasure` card color — +1 bonus, turn continues |
| Take a Nap | `end-turn` socket event |
| Relic: Peek | `use-peek` power-up |
| Relic: Shield | `activate-shield` power-up |

## Board Setup
19 cards per game: **6 red, 6 blue, 5 neutral, 1 abyss, 1 treasure**. Both teams always start with exactly 6 tiles. Board layout in `game.js` is a 3-4-5-4-3 hexagonal grid (`HEX_ROWS`).

## Game Phases
`lobby` → `captain-clue` → `guessing` → (back to `captain-clue` or) `ended`

- **lobby**: players pick roles, settings are configured
- **captain-clue**: active team's Pathfinder gives a one-word clue + number (0–9; 0 = ∞)
- **guessing**: Seekers click tiles; `guessesLeft` is always set to `Infinity` (no hard cap — turn ends on wrong guess or manual end-turn)
- **ended**: winner determined, overlay shown

## Scoring
- Correct own tile: **+1** to guessing team; `roundCorrect++`
- 3-in-a-row bonus: every 3rd consecutive correct guess in a turn = **+1 extra** (tracked in `game.roundCorrect`, reset on `switchTurn`)
- Wrong guess (neutral, opponent, or abyss): **−0.5** to guessing team, **+0.5** to opponent; turn ends immediately
- Treasure tile: **+1** to guessing team, `game.treasureTeam` set; turn continues (does NOT end the turn)
- Abyss tile: guessing team loses instantly (no score change, just `game.winner` flipped)
- Scores can be fractional (0.5 steps); `formatScore()` omits `.0` suffix

## Turn Flow
```
captain-clue
  └─ Pathfinder emits give-clue
       └─ phase → guessing, guessesLeft = Infinity
            ├─ correct own tile → keep guessing (check tiles-remaining triggers)
            ├─ treasure → +1, keep guessing
            ├─ wrong guess → switchTurn() [or endGame if finalTurnActive]
            ├─ abyss → endGame immediately
            └─ end-turn (only valid in guessing phase) → switchTurn() [or endGame]
```

### `switchTurn(game)`
Flips `currentTeam`, resets shield + `roundCorrect`. **After flipping**, checks `countRemaining(game, game.currentTeam) === 0` — if the incoming team has no tiles left (e.g. opponent accidentally revealed their last tile), calls `endGame()` immediately rather than handing them an empty captain-clue phase.

## Final Turn Mechanic
When a team reveals their **last tile**:
1. If `finalTurnActive` is already true → `endGame()` (final turn just resolved)
2. If the opposing team **also** has 0 tiles → `endGame()` immediately
3. Otherwise → set `finalTurnActive = true`, `finalTurnTeam = opposingTeam`, switch to that team's `captain-clue`

During `finalTurnActive`:
- Wrong guess → `endGame()` (not `switchTurn`)
- Timer expiry → `endGame()`
- end-turn → `endGame()`
- Team finishes their tiles → `endGame()`

## `endGame(game)`
Compares `scores.red` vs `scores.blue`:
- Higher score wins
- Tie → winner = `game.treasureTeam` (whichever team found the treasure) or `'tie'` if neither

## State: Server vs Client
`getPublicState()` strips sensitive data before sending:
- `card.color` is `null` for unrevealed cards **unless** the recipient is a Pathfinder (`spymaster`)
- `tilesRemaining: { red, blue }` is always sent (server-computed) so clients don't need to count — non-Pathfinders can't derive it themselves from the card list
- `guessesLeft` is sent but always `Infinity` during guessing (client uses it for display only)

## Power-ups (per team, one use each)
- **Peek**: Seeker clicks Peek relic, then clicks a tile → server emits `peek-result` **only to that socket** (private). Result shown 3s then cleared.
- **Shield**: Seeker activates shield; next bad guess (neutral/opponent/abyss, NOT treasure) is blocked — turn doesn't end. Shield deducted from `pu.shield`, `shieldActive` reset to false.
- `shieldActive` is reset to `false` in `switchTurn()` at end of every turn.
- Power-ups can be disabled globally via `game.powerupsEnabled`.

## Settings (configurable in lobby, some mid-game)
| Setting | Flag | Lobby-only? |
|---------|------|-------------|
| Rapid Mode (turn timer) | `game.rapidMode` + `game.rapidDuration` (15–300s) | Yes |
| Word Definitions (Pathfinder hover) | `game.definitionLookup` | Yes |
| Power-ups enabled | `game.powerupsEnabled` | No |
| Color Blind Mode | `game.colorBlindMode` | No |

## Roles & Joining
- Roles are **locked on join** — no switching (`if (player.team !== null) return`)
- Only one Pathfinder per team
- Mid-game late joiners can only join as Seekers
- On `new-game`: all players reset to `team: null, role: null`, go back to lobby

## Reconnection
Socket.IO reconnects fire `connect` → client re-emits `join-room`. Server restores `currentRoom`/`currentPlayerId` from the join handler. Disconnect has a **90-second grace period** (`_leaveTimer`) before the player is removed — handles mobile lock-screen and tab-background disconnects.

## Known Edge Cases & Past Bugs
- **Mobile reload loses PID → can't reclaim Pathfinder slot**: If a player reloads via the invite link (`/?join=CODE`) instead of the game URL, they get a new PID. Their old entry sits in `game.players` with `_leaveTimer` running. The `slotTaken` check in `set-team` would block them. Fix: if the blocking player has `_leaveTimer` (disconnected), evict them and allow the rejoining player to take the slot.
- **Wrong guess reveals opponent's last tile**: When Team A guesses a Team B tile as a wrong guess, `card.revealed = true` is set. If that was Team B's last tile, `countRemaining('blue') === 0`. The fix lives in `switchTurn()` — it checks tile count *after* flipping teams and calls `endGame()` instead of handing them an empty `captain-clue`.
- **`end-turn` is gated to `guessing` phase** — captains cannot skip their `captain-clue` turn via end-turn. Keep this in mind if adding "pass" functionality.
- **Treasure doesn't end the turn** and doesn't count toward a team's tile total — `countRemaining` only counts `card.color === team`.
- **`roundCorrect` resets on `switchTurn`** but NOT on shield blocks or peek — a 3-in-a-row streak survives power-up use within the same turn.

## Header Score Bar
- Shows: `Dawn [pts] | [turn indicator pill] | [pts] Dusk`
- Also shows `X tiles left` below each team's score during active gameplay (hidden in lobby/ended)
- `tilesRemaining` comes from server state — do not try to compute on client for non-Pathfinders
