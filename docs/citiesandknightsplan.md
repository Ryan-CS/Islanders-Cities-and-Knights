# Cities & Knights Adaptation Plan

This document is the implementation plan for adapting Islanders into a browser-playable
implementation of the Catan Cities & Knights expansion. It supersedes the first draft of the
plan: this revision is grounded in a review of the actual codebase, answers the previous
draft's open questions from the code, and re-orders the work around the base-game gaps that
review uncovered.

## Goal

Add a Cities & Knights game mode that can be hosted once and joined by remote players
through a browser, without each player needing to build or run the application locally.

## Non-goals for the first pass

- Do not redesign the whole application.
- Do not add Seafarers at the same time.
- Do not implement custom art/assets until the rules engine and state model are stable.
- Do not fork the base-game rules into a separate code path. All expansion behavior lives
  behind a variant flag in the shared rules layer.

## Legal and product notes

- This fork is for private experimentation unless licensing and IP questions are resolved.
- Avoid copying official card text, artwork, iconography, or proprietary assets into the repo.
- Use functional placeholders for progress cards and pieces during development.
- Rule details below marked "verify" should be checked against the official rulebook
  (CN3087) before the corresponding phase is implemented.

## How the codebase actually works

Findings from code review that shape everything below.

**Rule pipeline.** All game mutations are pure functions in
`islanders-shared/lib/Rules/`. An `Action` (tagged union in `islanders-shared/lib/Action.ts`)
is sent over socket.io, mapped to a `Rule` in `GameService.mapRules`
(`islanders-server/src/services/GameService.ts`), and applied as
`World -> Result` where `Result` is a success/fail monad (`Rules/Result.ts`). Adding a new
action touches exactly four places: the action class + union in `Action.ts`, the rule module
in `Rules/`, the registry in `Rules.ts`, and the `mapRules` switch. The client dispatches
actions through the Vuex store (`islanders-client/src/store/modules/game.ts`). This pipeline
is a good fit for C&K and should be extended, not bypassed.

**Persistence and undo.** `GameRepository`
(`islanders-server/src/repositories/GameRepository.ts`) stores every world version as a
separate MongoDB document; `getWorld` reads the highest version, `undoMove` deletes it.
State therefore survives server restarts, and reconnect/refresh already replays the latest
world (`GameSocket.checkForReconnect`). Two consequences:

1. Old world documents will not have new C&K fields. All new fields must be optional with
   defaults applied on read, or a normalization step added.
2. Undo restores the previous document wholesale. Any rule that consumes randomness (dice,
   card draws) can be "rerolled" by undoing and repeating. C&K adds many more random,
   hidden events, so undo must be gated (see "Undo policy" below).

**No hidden state.** `GameSocket.setUpSendAction` broadcasts the full `World` — including
every player's resources and development cards — to every socket in the game namespace.
Privacy today is purely client-side rendering. Base-game dev cards already leak this way;
C&K progress cards and commodity hands would too. See "Hidden information policy" below.

**No dice-roll phase.** There is no roll action. `EndTurn`
(`islanders-shared/lib/Rules/EndTurn.ts` → `assignNextPlayerTurn` in `Rules/Helpers.ts`)
rolls the dice for the incoming player and distributes production in the same transition.
C&K requires things to happen *between* roll and production (event die resolution, barbarian
movement, progress-card draws, Aqueduct compensation) and *before* the roll (Alchemist).
A dedicated roll step is a prerequisite, not a nice-to-have.

**Infinite dev-card deck.** `BuyCard` constructs `new DevelopmentCard()`, which picks a
random type with replacement (`Entities/DevelopmentCard.ts`). There is no deck state in the
world. C&K's three progress decks are finite and ordered, so decks must become explicit
world state drawn from server-side.

**Victory points are a mutated counter.** `player.points` is incremented at award sites
(`increasePointsForPlayer` in `Rules/Helpers.ts`). C&K roughly doubles the number of VP
sources (metropolises, defender tokens, VP progress cards, merchant). A single derived
`calculatePoints(player, world)` function is much safer than scattering increments.

**Turn gating via conditions.** Multi-step flows (rolled a 7 → move thief → steal) are
modeled as `Conditions` on the world (`islanders-shared/lib/TurnCondition.ts`), checked by
`EndTurn.verifyTurnConditions`, and mirrored to UI flags in the Vuex `ui` module. This is
the extension point for C&K's multi-step flows (discard, barbarian resolution, knight
actions, targeted progress cards). It works, but each new condition touches several places;
Phase 1 should tidy it enough that adding conditions stays mechanical.

**Variant hook exists.** `GameRules` (`islanders-shared/lib/GameRules.ts`) already carries
`gameType`, piece limits, and `pointsToWin`, and is embedded in `World`. The C&K flag
belongs here (e.g. `expansion: 'none' | 'citiesAndKnights'`).

**Naming collision.** `Player.knights` currently counts *played knight development cards*
(for the unimplemented largest-army bonus). C&K knights are board pieces. Rename the
existing field (e.g. `playedKnightCards`) before introducing `Player.knightPieces` to avoid
a silent semantic clash.

**Terminology mapping.** The repo uses `wood/clay/stone/grain/wool` for
lumber/brick/ore/grain/wool, "house" for settlement, and "thief" for robber. The plan and
code should keep repo terminology. Commodities map: paper ← Wood tiles, cloth ← Wool tiles,
coin ← Stone tiles.

## Base-game gaps that block C&K

These are missing from the base game today and are load-bearing for C&K. They are scheduled
explicitly (Phase 1) rather than discovered mid-expansion:

1. **Discard on 7 does not exist.** Rolling a 7 forces thief movement and stealing, but no
   one ever discards half their hand. C&K city walls exist almost entirely to modify the
   discard threshold, so the discard flow (a `SelectCardsToDiscard` action gating every
   over-limit player, not just the current player) must be built first.
2. **No roll phase** (described above).
3. **Largest army / longest road are empty test stubs** (`__tests__/points.spec.ts`).
   Convenient for C&K: largest army does not exist in C&K, so it can stay unimplemented.
   Longest road *is* worth 2 VP in C&K and is genuinely missing; it also interacts with
   knights breaking road continuity. Decide scope early: recommended to implement longest
   road as part of Phase 1 only if wanted for base game too; otherwise defer to Phase 6
   with the knight interaction included.
4. **Per-player action gating.** Today only the current player acts
   (`findPlayer` fails for anyone else). Discards and barbarian resolution require
   simultaneous input from all players. The conditions model needs a per-player variant
   (e.g. `conditions.mustDiscard: { [playerName]: count }`).

## Expansion rule surface

Summary of what C&K adds, with the systems each belongs to. Costs use repo resource names.

1. **Setup** — variant flag; progress decks replace the dev deck (dev cards are *removed*
   entirely in C&K, which conveniently sidesteps the infinite-deck problem for this mode);
   players start with one settlement and one city; barbarian track and event die state;
   per-player improvement tracks; 13 points to win.
2. **Production** — settlements produce 1 base resource as before. Cities produce
   2 base resources on Clay and Grain tiles, and 1 base resource + 1 commodity on Wood
   (paper), Wool (cloth), and Stone (coin) tiles. Thief blocks production as before.
3. **Event die** — rolled with the two production dice; 3 ship faces advance the barbarian
   ship, 3 gate faces (science, trade, politics) trigger progress-card draws. A player draws
   from the matching deck when the red production die is within the range unlocked by their
   improvement level (level 1 draws on red ≤ 2, each level widens by one — verify exact
   ranges against the rulebook). The red die must therefore be stored separately, not just
   the 2-die sum.
4. **Progress cards** — three finite decks (science/trade/politics); hand limit of four
   with immediate discard-down; VP progress cards revealed immediately and permanently;
   cards playable the turn they are drawn; playable before the roll only for Alchemist.
5. **City improvements** — per-player tracks (science/trade/politics), level N costs N
   commodity cards of the matching type; requires at least one city (verify: walls/cities
   pillaged below city count effects). Level 3 abilities: Aqueduct-style compensation
   (science: take any resource when a roll yields you nothing, except on 7), 2:1 commodity
   bank trades (trade), mighty-knight promotion (politics). Levels 4/5 drive metropolises.
6. **Knights** — piece per intersection with `{owner, position, level: 1|2|3, active}`.
   Actions: recruit (1 wool + 1 stone, on own road network), promote (same cost; level 3
   requires politics 3), activate (1 grain), move along own roads, displace weaker opponent
   knights, chase the thief away. A knight acts only if activated on a *previous* turn;
   knights deactivate after acting and after barbarian battles. Knights block opponent
   building/settling on their intersection and break opponent road continuity.
7. **City walls** — cost 2 clay, one per city, max three per player; each raises that
   player's discard threshold from 7 by 2 (7 → 9 → 11 → 13); removed when the city is
   pillaged.
8. **Barbarians** — ship advances one space per ship face; on reaching the last track
   space, battle: barbarian strength = total cities + metropolises of all players, defender
   strength = total active knight levels. Defenders win → single highest contributor gets a
   defender VP token, tied top contributors instead each draw a progress card of their
   choice. Barbarians win → player(s) with the lowest active-knight contribution lose one
   city (downgraded to settlement, wall destroyed); metropolises are immune; players with no
   cities are exempt. Then reset ship, deactivate all knights. The thief cannot move (verify:
   discard/robber behavior on 7 pre-first-attack) until the first battle resolves.
9. **Metropolises** — one per discipline. First player to improvement level 4 places it on
   one of their non-metropolis cities (+2 VP, city cannot be pillaged); a player reaching a
   *higher* level in that discipline takes it; level 5 makes it permanent.
10. **Merchant** — placed via the Merchant progress card; worth 1 VP to its owner and grants
    2:1 trade in the resource of its tile; moves when the card is played again.
11. **Victory** — 13 VP target; sources: settlements 1, cities 2, metropolis +2, longest
    road 2, defender tokens 1 each, VP progress cards 1 each, merchant 1.

## Proposed state model

Sketch, not contract. All new fields optional-with-defaults so historical world documents
still load.

```ts
// GameRules
expansion: 'none' | 'citiesAndKnights';

// Shared
interface Commodities { paper: number; cloth: number; coin: number; }
type Discipline = 'science' | 'trade' | 'politics';

// Player additions (rename existing `knights` -> `playedKnightCards` first)
commodities: Commodities;
knightPieces: Knight[];          // { position, level: 1|2|3, active }
cityWalls: number;               // or MatrixCoordinate[] if walls render per-city
improvements: Record<Discipline, 0|1|2|3|4|5>;
progressCards: ProgressCard[];   // hidden; see hidden-information policy
defenderTokens: number;

// World additions
barbarian: { position: number; attacks: number };   // thief locked until attacks > 0
eventDie: 'Ship' | Discipline | 'None';
redDie: 1|2|3|4|5|6 | 'None';
progressDecks: Record<Discipline, ProgressCardType[]>;  // server-authoritative, shuffled
metropolises: Record<Discipline, string | undefined>;   // owner player name
merchant?: { owner: string; position: HexCoordinate };
```

Keep `Commodities` separate from `Resources` rather than widening the `Resources`
interface: every helper in `Resources.ts` and every trade action enumerates the five base
fields, and base-mode behavior must not change. Add parallel commodity helpers and a
combined `handSize(player)` for discard counting.

## New actions inventory

Planned additions to `Action.ts` / `Rules.ts` / `GameService.mapRules` (final names may vary):

- `RollDice` (replaces the implicit roll in EndTurn; base mode can keep auto-roll behavior
  by auto-dispatching it, or adopt the explicit phase too — decide in Phase 1)
- `SelectCardsToDiscard` (base 7s and C&K walls)
- `BuildKnight`, `PromoteKnight`, `ActivateKnight`, `MoveKnight` (move covers displacement;
  chasing the thief is a move onto the thief's adjacent intersection triggering a
  `MoveThief`-style condition)
- `BuildCityWall`
- `BuyCityImprovement`
- `PlayProgressCard` (parameterized like `PlayCard`; per-card parameters for targeted cards)
- `DiscardProgressCard` (hand-limit enforcement)
- `PlaceMetropolis` (choosing which city when a level-4/5 threshold is crossed)
- Extensions to `BankTrade`/`HarborTrade` parameters for commodities (4:1 bank and 3:1
  generic harbor apply to commodities; 2:1 resource harbors do not; trade level 3 grants
  2:1 commodity bank trades)

## Hidden information policy

Progress cards are the most consequential hidden information in C&K. Options:

- **A. Accept the leak for the MVP.** Matches current dev-card behavior; zero server work.
  Fine among friends, unacceptable long-term.
- **B. Per-player world filtering before emit.** The server already knows
  player-name → socket-id (`GamePlayerSockets` in `App.ts`/`GameSocket.ts`). Replace the
  single `namespace.emit(newWorld, …)` with per-socket emits of a world whose *other*
  players' `progressCards`/`devCards` are reduced to counts. Also applies to `getWorld`,
  reconnect, and spectators.

Recommendation: ship Phase 4 with A, implement B as its own slice (Phase 9) before calling
the mode complete. Design rule modules so they never *depend* on the client knowing hidden
state, so B is purely a transport-layer change.

## Undo policy

Undo currently deletes the latest world version, which allows rerolling randomness. With
C&K's extra randomness (event die, three decks, barbarian battles):

- Mark worlds produced by random events (roll, draw) as undo barriers (e.g. a
  `randomEvent: true` flag checked by `GameRepository.undoMove`, alongside the existing
  same-player check).
- Deterministic actions (builds, trades, knight moves before combat) remain undoable.

## Implementation phases

Each phase lands behind `expansion: 'citiesAndKnights'` unless marked base-game. Base mode
must pass the existing test suite unchanged after every phase.

### Phase 0: Baseline verification

- Build and run the Docker setup; verify two browsers can join one hosted game.
- Note the shared-package build coupling: server and client import
  `islanders-shared/dist`, so shared changes require rebuilding shared before either
  consumer picks them up. Document the dev loop in `docs/dev-setup-notes.md`.
- Confirm reconnect/refresh restores state (it should, via Mongo world versions).

Exit: a documented, repeatable dev setup; two-browser game verified.

### Phase 1: Variant scaffolding + base prerequisites (base-game changes)

- Add `expansion` to `GameRules`; expose it in game creation (`CustomizeGame.vue`,
  `LockMapAction` currently carries only `pointsToWin`).
- Rename `Player.knights` → `playedKnightCards`.
- Extract the dice roll from `assignNextPlayerTurn` into an explicit `RollDice`
  action/phase; store `redDie` separately from the sum.
- Implement discard-on-7 with a per-player `mustDiscard` condition and
  `SelectCardsToDiscard` action; block `EndTurn` (and other actions) until resolved.
- Introduce `calculatePoints(player, world)` and replace scattered point increments.
- Add regression tests pinning base-mode behavior.

Exit: base game unchanged in behavior except the (now real) roll phase and discard rule;
a game can be created in either mode; C&K mode starts without errors.

### Phase 2: Commodities and production

- Add `Commodities` to shared model and `Player`; add helpers and `handSize`.
- In C&K mode, change city production per the table in "Expansion rule surface"
  (hook: `assignRessourcesToPlayers` / `numberOfResourcesForPlayer` in `Rules/Helpers.ts`,
  which currently uses only the piece's `value` multiplier and cannot distinguish
  city-vs-settlement output types — this is the main refactor of the phase).
- Count commodities in discard-on-7 hands.
- Extend bank/harbor trade rules for commodities.
- C&K setup: initial placements are one settlement and one city (Pregame flow in
  `EndTurn.stateChanger` and the `BuildHouseInitial`/`BuildRoadInitial` rules).

Exit: cities produce commodities correctly in C&K mode; base-mode production byte-identical.

### Phase 3: Event die and barbarian track

- Roll the event die inside `RollDice` in C&K mode; store `eventDie` and advance
  `barbarian.position` on ship faces (battle resolution itself stubbed until Phase 6).
- Thief locked until `barbarian.attacks > 0`; verify pre-first-attack behavior of 7s
  (discard yes/no) against the rulebook and encode it in a test.
- Disable dev cards (`BuyCard`/`PlayCard`) in C&K mode.

Exit: C&K turns resolve event die before production; ship advances; thief locked.

### Phase 4: City improvements and progress-card draws

- Add improvement tracks, `BuyCityImprovement` with commodity costs, and the
  at-least-one-city requirement.
- Add server-side shuffled finite `progressDecks` to world state; draws triggered by gate
  faces + red-die eligibility; hand limit 4 with `DiscardProgressCard`.
- VP progress cards apply immediately via `calculatePoints`.
- Level 3 abilities: Aqueduct compensation (needs "produced nothing this roll" detection in
  the production step), 2:1 commodity trades, mighty-promotion gate (consumed in Phase 5).

Exit: correct players draw from correct decks; hand limit enforced; improvements purchasable.

### Phase 5: Knights and city walls

- `Knight` entity and the four knight actions; placement/movement constrained to the
  player's road network (note: no road-graph utility exists today — `placeRoad` does only
  local adjacency checks — so this phase builds the first real connectivity/pathing helper,
  which longest-road can later reuse).
- Blocking: opponents cannot build through or settle on a knight's intersection.
- Activation economics and the acted-this-turn / activated-this-turn distinctions.
- `BuildCityWall`; walls raise the owner's discard threshold; enforce per-city and
  per-player limits.

Exit: knight lifecycle legal and tested; walls affect discard thresholds.

### Phase 6: Barbarian battle resolution

- Strength calculation, defender reward (single winner token vs. tie draws), pillage
  selection among lowest contributors (city → settlement, wall destroyed, metropolis and
  city-less players immune), ship reset, mass knight deactivation, thief unlock after
  first battle.
- If longest road is in scope, implement it here with knight-interruption included.

Exit: battle outcomes match the rulebook for win/loss/tie/no-city edge cases.

### Phase 7: Metropolises, merchant, and victory

- Metropolis award/steal logic on improvement thresholds, `PlaceMetropolis` flow,
  pillage immunity (already respected in Phase 6).
- Merchant piece state (placed via progress card in Phase 8; state and VP land here).
- `pointsToWin` defaults to 13 in C&K; `calculatePoints` covers all C&K sources; win check
  stays in `EndTurn` (verify: C&K win-on-your-turn-only nuance for progress-card VPs).

Exit: VP totals correct across all sources; game ends at 13.

### Phase 8: Progress-card effects

Implement in dependency order, reusing the conditions system for targeted flows:

1. Simple grants/mirrors of existing mechanics: Engineering (free wall), Road Building,
   Smithing (free promotions), Medicine (cheap city), Crane (improvement discount),
   Mining/Irrigation (terrain-count grants), Alchemist (choose next roll — requires the
   Phase 1 roll phase).
2. Targeted single-step: Deserter, Bishop, Spy-style card theft, Commercial Harbor,
   Resource/Trade Monopoly, Master Merchant, Wedding.
3. Multi-step or board-altering: Diplomat, Intrigue, Inventor (swap number tokens),
   Merchant, Merchant Fleet, Saboteur, Constitution/Printer (VP, from Phase 4).

Cards not yet implemented are excluded from deck composition (deck contents are already
world state, so this is data, not code).

Exit: each implemented card has unit tests and a client flow; decks contain only
implemented cards.

### Phase 9: Client/UI and hidden information

- Game creation option; commodity display; improvement tracks; barbarian track + event die;
  knight states and action affordances; progress-card hand; walls, metropolises, merchant,
  defender tokens. (`Overview.vue`, `Player.vue`, `Map.vue`, `Trade.vue`, and the `ui`
  store module are the main touch points.)
- Implement per-player world filtering (option B above).
- Generic prompt pattern for multi-step condition flows so Phase 8 group 2–3 cards don't
  each need bespoke UI plumbing.

Exit: a remote player can complete a C&K game from the browser alone; other players'
progress cards and exact hands are not present in their client's world payload.

### Phase 10: Hardening

- Undo-barrier implementation per the undo policy.
- Reconnect/refresh testing mid-flow (mid-discard, mid-battle, mid-card).
- Deployment notes for remote play; known-limitations doc.

Exit: hosted instance survives real multi-player sessions; gaps documented.

## Testing strategy

- Every rule module gets deterministic unit tests in `islanders-shared/__tests__/`,
  following the existing spec style (construct world → apply rule → assert on `Result`).
- Randomness (dice, event die, shuffles) must be injectable or applied in a separate step
  from its consequences, so consequences are testable deterministically — the Phase 1 roll
  extraction and Phase 4 pre-shuffled decks both serve this.
- A base-mode regression suite is the guardrail for every phase; C&K assertions never run
  in base mode and vice versa.
- One scripted two-client smoke test (manual is acceptable initially) per phase that
  touches the socket layer.

## Suggested first issues

1. Document dev setup and the shared-package rebuild loop (Phase 0).
2. Add `expansion` flag to `GameRules` + game creation UI, no behavior change.
3. Rename `Player.knights` → `playedKnightCards`.
4. Extract `RollDice` from `EndTurn`; store `redDie`.
5. Implement discard-on-7 with per-player conditions.
6. Add `Commodities` model + helpers with tests.
7. C&K city production in `assignRessourcesToPlayers`.
8. Event die + barbarian track state.

## Open questions (genuinely open — product decisions, not code archaeology)

- Is longest road in scope (it is missing from the base game entirely)? Recommended: yes,
  in Phase 6, since the road-graph work happens in Phase 5 anyway.
- Is hidden-information filtering (option B) required before friends-and-family play, or
  is the MVP leak acceptable until Phase 9?
- Should base mode also adopt the explicit roll phase and discard-on-7 (rules-correct but
  changes existing behavior), or stay frozen? Recommended: adopt both — discard-on-7 is a
  base-game rule the current implementation simply lacks.
- Undo in C&K: barrier-based policy above, or disable undo entirely in C&K mode for
  simplicity?

## Working principle

Keep every change behind the `citiesAndKnights` variant flag until the expansion is
playable. Protect base-game behavior with regression tests. Prefer extending the existing
action → rule → world pipeline over inventing parallel mechanisms.
