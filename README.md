# Board Game Simulation — Clean Architecture in Java

> **Note on repository access:** This is the public showcase repository for my Software Design & Architecture coursework at Manchester Metropolitan University. The original repository contains provided starter code and remains private for academic integrity reasons. This README documents my design and implementation. Representative code excerpts are in [`showcase/`](./showcase).

## Demo


**[Watch: all nine rule variations + a saved game replay (mp4)](#)**

## What it is

A Java console application simulating a two- to four-player board game loosely based on Snakes and Ladders. It implements all five specified rule variations — single die, exact-end bounce back, hit-forfeit, wormhole teleportation, and a large 4-player board — plus two advanced features: a **state machine** for game lifecycle and **save/replay** of completed games.

The point of the project wasn't the game; it was the architecture. Every game rule is a swappable strategy injected at runtime — the engine contains no `if (variation)` logic anywhere.

## What I did (and why)

### Clean Architecture
The application follows Clean Architecture (Ports and Adapters): dependencies point inward only — infrastructure depends on use cases, use cases depend on the domain model, and the domain has zero outward dependencies. I structured it this way because the module's real question was "how do you build software that can change safely?" — and this project forced me to answer it at every class boundary.

### All five rule variations as interchangeable strategies
`EndRule`, `HitRule`, and `TeleportRule` are interfaces with two implementations each (`ExactEndRule`/`OvershootEndRule`, `ForfeitHitRule`/`IgnoreHitRule`, `WormholeTeleportRule`/`NoTeleportRule`). Adding a new end rule means implementing the interface and injecting it — no existing code changes (Open/Closed Principle in practice, not just in a flashcard).

### Game states (State pattern)
A three-state machine — `Ready → InPlay → GameOver` — replaces procedural `if/switch` lifecycle logic. Each state owns its behaviour; `Game` just delegates each roll to `currentState.handleRoll()`.

### Save & replay
Every completed game is saved as a `GameRecord` — board size, players, the full rule configuration, and the complete sequence of die values. A replay isn't stored output printed back: the application reconstructs the exact rule objects, feeds the saved dice sequence into a `FixedDiceShaker`, and **re-runs the whole game engine** to produce identical output. I built it this way because a genuine replay exercises the entire engine, proving the rules are deterministic and correctly reconstructable from data.

### Four design patterns, deliberately
- **Strategy** — rule families (`EndRule`, `HitRule`, `TeleportRule`)
- **State** — game lifecycle without switch statements
- **Observer** — `Game` fires typed events (`MoveEvent`, `WinEvent`, `ForfeitEvent`) to a `ConsoleObserver`; the domain never calls `System.out`
- **Decorator** — `DoubleDiceDecorator` wraps any shaker and calls `shake()` twice; `RecordingDiceShaker` wraps the innermost shaker to capture die values for replay

### Spring DI as the composition root
Spring Boot's DI container wires the whole object graph in `AppConfig`. The domain knows nothing about Spring. Swapping the in-memory store for a file-based one means changing one `@Bean` declaration.

## How it's structured

| Package | Responsibility |
| --- | --- |
| `uk.ac.mmu.game` | Spring Boot entry point, driving adapters (`Play`, `Replay`), composition root (`AppConfig`) |
| `uk.ac.mmu.game.applicationcode.domainmodel` | All game mechanics — `Game`, `Player`, rules, states, dice, events, ports |
| `uk.ac.mmu.game.applicationcode.usecase` | Use case interactors and the `Required` port |
| `uk.ac.mmu.game.infrastructure` | `Technology` — the persistence adapter |

## Running it

```bash
./mvnw spring-boot:run
```

Nine games play in sequence (demonstrating each variation and combinations), then:

```
Enter a Game Id to replay.
```

Enter any ID from 1–9 to watch the engine replay that exact game from its saved dice sequence.

## Honest evaluation

### What works well
All five variations and both advanced features work; the Clean Architecture separation means the game logic could be tested with no dependency on Spring or the filesystem. Strategy injection means any rule combination assembles at runtime — as the nine-game sequence demonstrates. Observer decoupling means a second observer (e.g. a file logger) could be added with zero domain changes.

### Limitations I'd fix next
- **In-memory persistence** — games are lost on exit. A JSON file adapter (Jackson) would slot into the existing port with only an `AppConfig` change — the architecture already paid for that flexibility.
- **Fragile replay reconstruction** — `GameReplayer` pattern-matches rule display-name strings to rebuild rules. Storing an enum code in `GameRecord` would remove that fragility.
- **Procedural game setup** — `UseCase.play()` constructs all nine games by hand; a `GameConfiguration` record would let games be driven from a menu or config file.
- **ANSI colours** — applied unconditionally; a runtime terminal-support check would be more portable.

## What I'd do differently

Add **unit tests** per rule strategy (they're pure, mockable, and would have caught replay reconstruction edge cases), and a file-based persistence adapter as described above.

## Tech stack

Java 21 · Spring Boot (DI container) · Apache Maven · Clean Architecture · Strategy / State / Observer / Decorator patterns · SOLID principles

## Acknowledgements

Coursework for the Software Design & Architecture module at Manchester Metropolitan University. The private original repository began from starter code provided by the module team; the architecture, implementation, and documentation above are my own work.
