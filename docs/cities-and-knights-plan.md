# Cities & Knights Adaptation Plan

This document tracks the plan for adapting Islanders into a browser-playable implementation of the Cities & Knights expansion. It is intentionally implementation-oriented: the goal is to preserve the existing real-time browser/server architecture while adding expansion rules in small, testable slices.

## Goal

Add a Cities & Knights game mode that can be hosted once and joined by remote players through a browser, without each player needing to build or run the application locally.

## Non-goals for the first pass

- Do not redesign the whole application.
- Do not add Seafarers at the same time.
- Do not implement custom art/assets until the rules engine and state model are stable.
- Do not fork the base-game rules into a separate code path unless necessary. Prefer a variant/configuration layer.

## Legal and product notes

- This fork is for private experimentation unless licensing and IP questions are resolved.
- Avoid copying official card text, artwork, iconography, or proprietary assets into the repo.
- Use functional placeholders for progress cards and pieces during development.

## Why this repo is a reasonable base

Islanders already separates shared game logic, server, and client. That is useful for Cities & Knights because most expansion rules must be authoritative on the server and reflected to all browsers. The existing immutable/world-update style should also make it easier to add deterministic rule transitions and test them.

## Expansion rule surface

Cities & Knights changes or adds the following major systems:

1. Game setup
   - Replace the base development deck with progress-card decks.
   - Add commodity resources: paper, cloth, coin.
   - Add event die and barbarian track state.
   - Add city improvement board state per player.
   - Add knight pieces, city walls, metropolises, merchant, and barbarian ship.
   - Start with one settlement and one city per player instead of two settlements.

2. Turn structure
   - Roll Dice phase.
   - Production phase.
   - Action phase.
   - Resolve event die before production dice.
   - Allow Alchemy-like pre-roll effect later, after progress cards exist.

3. Production changes
   - Settlements produce as before.
   - Cities on forest, pasture, and mountain produce one base resource plus one commodity.
   - Cities on fields and hills produce two base resources.
   - Desert produces nothing.
   - Robber behavior is inactive until after the first barbarian attack.

4. Progress cards
   - Three decks: science, trade, politics.
   - Progress-card eligibility based on event die discipline plus red die range from city improvement level.
   - Immediate play of VP progress cards.
   - Four-card progress-card hand limit.
   - Cards can be played the same turn they are drawn.

5. City improvements
   - Per-player tracks: science, trade, politics.
   - Commodity costs increase by level.
   - Level 3 permanent abilities:
     - Science: no-production compensation, except on a 7.
     - Trade: 2:1 commodity trading.
     - Politics: promote strong knights to mighty knights.
   - Level 4 and 5 interact with metropolis control.

6. Knights
   - Piece types: basic, strong, mighty.
   - State: owner, location, strength, active/inactive.
   - Actions: recruit, promote, activate, move, displace, chase robber.
   - Knights block opponent routes and settlement sites.
   - Knights contribute only when active during barbarian attacks.
   - Knights become inactive after knight actions and after barbarian attack resolution.

7. City walls
   - One wall per city.
   - Up to three walls per player.
   - Each wall increases the discard threshold on a 7 by two cards.
   - A wall is removed if its city is pillaged.

8. Barbarians
   - Event die ship result advances the barbarian ship.
   - When the ship reaches the attack space, resolve attack.
   - Barbarian strength equals number of cities, including metropolises.
   - Defender strength equals total strength of active knights.
   - If defenders win, award defender VP token or tied progress-card draw.
   - If barbarians win, pillage eligible cities from the lowest-contributing player or players.
   - Metropolises cannot be pillaged.
   - Reset barbarian ship and deactivate all knights after the attack.

9. Metropolises
   - One each for science, trade, politics.
   - First player to level 4 temporarily controls the matching metropolis.
   - First player to level 5 permanently controls the matching metropolis.
   - A metropolis is worth 2 additional VP and makes the city worth 4 VP total.
   - A player must have an available city to place a metropolis.

10. Victory points
   - Target is 13 VP.
   - Existing sources remain where applicable.
   - Add defender VP tokens.
   - Add metropolis VP.
   - Add merchant VP.
   - Add progress-card VP.

## Implementation phases

### Phase 0: Baseline verification

- Build and run the current Docker setup.
- Verify multiple browser clients can join the same hosted game.
- Identify the authoritative server-side action pipeline.
- Identify how hidden player state is modeled and transmitted.
- Add a short `docs/dev-setup-notes.md` after verification.

Exit criteria:

- A developer can run the app locally.
- Two browser sessions can connect to the same game.
- The main game-state transition points are documented.

### Phase 1: Variant scaffolding

- Add a game variant enum or configuration: `base` and `citiesAndKnights`.
- Keep base-game behavior unchanged.
- Add placeholder state containers for C&K-only systems.
- Add tests confirming base mode remains stable.

Exit criteria:

- A game can be created in either base mode or C&K mode.
- C&K mode can start without breaking base gameplay.

### Phase 2: Resources and commodities

- Extend card/resource model with paper, cloth, and coin.
- Update city production rules for C&K mode.
- Update bank/port trade logic to handle commodities where allowed.
- Update discard-on-7 logic to count resource plus commodity cards.

Exit criteria:

- Cities produce commodities correctly.
- Commodity cards are hidden from other players.
- Existing base-game resource logic is unchanged in base mode.

### Phase 3: Event die and turn sequence

- Add event die result to roll resolution.
- Add red-die tracking where needed for progress-card eligibility.
- Add barbarian-track state and movement.
- Keep progress-card draw stubbed until Phase 4.

Exit criteria:

- C&K turns resolve event die before production.
- Barbarian ship advances on ship results.
- Robber remains inactive until first barbarian attack.

### Phase 4: City improvements and progress-card draw

- Add city improvement tracks per player.
- Implement improvement purchase costs.
- Implement eligibility ranges for progress-card draws.
- Add placeholder progress-card deck objects.
- Implement progress-card hand limit.

Exit criteria:

- Players can buy improvements while they have at least one city.
- Correct players draw from the correct deck after eligible event rolls.
- Players discard to progress-card hand limit at the correct time.

### Phase 5: Knights and walls

- Add knight pieces and locations.
- Implement recruit, promote, activate, move, displace, and chase robber.
- Enforce route connectivity and blocking rules.
- Add city wall build/remove logic and discard-threshold changes.

Exit criteria:

- Knights can be placed and moved legally.
- Active/inactive rules are enforced.
- Route and settlement blocking account for knights.
- City walls affect discard thresholds.

### Phase 6: Barbarian attack resolution

- Calculate barbarian and defender strengths.
- Resolve defender victory reward.
- Resolve barbarian victory pillage targets.
- Reset barbarian track and deactivate knights.
- Activate robber after first attack.

Exit criteria:

- Attack outcomes match the rules for single loser, tied losers, metropolis protection, and no-city edge cases.
- All active knights deactivate after each attack.

### Phase 7: Metropolises, merchant, and VP integration

- Implement metropolis control for tracks.
- Add VP accounting for metropolises, merchant, defender tokens, and progress-card VPs.
- Set C&K win target to 13 VP.

Exit criteria:

- VP totals update correctly.
- A player wins at 13 VP on their turn in C&K mode.

### Phase 8: Progress-card effects

Implement cards in priority order:

1. Cards that map to existing mechanics or simple grants.
2. Cards that require targeted selection.
3. Cards that alter dice or board numbers.
4. Cards with multi-step UI flows.

Suggested early cards:

- Engineering: build one city wall at no cost.
- Road Building: build two roads at no cost.
- Smithing: promote up to two knights at no cost.
- Medicine: upgrade a settlement to a city at reduced cost.
- Crane: discount a city improvement.
- Mining and Irrigation: resource grants by adjacent terrain.

Defer until UI targeting is clean:

- Intrigue.
- Espionage.
- Taxation.
- Invention.
- Merchant and Merchant Fleet.
- Monopoly-style cards.

Exit criteria:

- Each implemented progress card has unit tests and at least one client flow.
- Unimplemented cards cannot be drawn or are clearly marked as disabled.

### Phase 9: Client/UI work

- Add C&K game creation option.
- Display commodities separately from resources.
- Display city improvement boards.
- Display barbarian track and event die results.
- Display knight state and available knight actions.
- Display progress-card hand and deck categories.
- Display metropolis, city wall, merchant, and defender-token state.

Exit criteria:

- A remote player can play a C&K test game without reading server logs.
- Hidden cards remain hidden.
- Action prompts prevent illegal choices where possible.

### Phase 10: Test and deployment hardening

- Add tests for deterministic rule transitions.
- Add server/client synchronization tests where feasible.
- Add a Docker deployment note for remote browser play.
- Add a known-limitations document.

Exit criteria:

- A hosted instance supports multiple remote browser players.
- Reconnect and refresh behavior does not corrupt game state.
- Known missing progress cards or UI gaps are documented.

## Suggested first issues

1. Verify local Docker setup and document exact commands.
2. Trace current game-state action pipeline.
3. Add C&K variant flag without changing base-game behavior.
4. Add commodity types to shared model.
5. Add C&K city production tests.
6. Add event die and barbarian track state.
7. Add city improvement track model.
8. Add knight entity model and placement rules.

## Open questions

- Does the current server persist game state across process restarts?
- How are private player hands transmitted to the client today?
- Can the current UI support multi-step targeted actions cleanly, or does it need a generic action-prompt system?
- Are game variants already present anywhere in the model?
- Does the current route calculation have an extension point for knight blocking?
- Should unimplemented progress cards be excluded from decks or included as disabled placeholders?

## Working principle

Keep every change behind a Cities & Knights variant flag until the expansion is playable. Protect base-game behavior with regression tests.