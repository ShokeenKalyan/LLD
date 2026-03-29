# LLD #04 — Snake & Ladder / Board Game Framework

> **Language:** TypeScript
> **Difficulty:** ⭐⭐⭐ (Intermediate)
> **Patterns Used:** Template Method, Strategy, Observer, Factory, State
> **Core Challenge:** Designing a *generic* board game framework, not just Snake & Ladder

---

## 1. Requirements

### Functional Requirements
- Support a configurable N×N board (default 10×10 = 100 cells)
- 2–6 players take turns rolling dice and moving their piece
- Special cells: snakes (move backward), ladders (move forward), portals (teleport)
- A player wins by landing exactly on the last cell (cell 100)
- If a player's roll would overshoot the last cell, they stay put (bounce-back rule)
- Support multiple dice (e.g., two dice rolled together)
- Players can be human or AI (bot)

### Non-Functional / Design Requirements
- **Generic framework**: the game loop must not hardcode Snake & Ladder logic
- Easy to add new cell types (MysteryCell, BombCell) without changing the loop
- Easy to swap dice rules (weighted dice, one die, two dice)
- Support game observers for UI rendering and event logging

### What interviewers watch for
- Do you use **Template Method** for the game loop, or hardcode everything?
- Do you model the **Cell as a polymorphic entity** with a `land()` hook?
- Do you identify the **three axes of variation**: dice rolling, cell effects, player turns?
- Do you handle the overshoot / exact-win rule?
- Do you think about **extensibility** — what if we add Chess next month?

---

## 2. The Core Design Insight

Most candidates build a `SnakesAndLaddersGame` class with all the logic inline. That gets you a 6/10.

The senior answer: **separate the invariant game loop from the variant rules**.

### What never changes (the template):
1. Pick the next player
2. Roll the dice
3. Move the player
4. Apply the cell's effect
5. Check win condition
6. Repeat

### What varies (the strategy / hook):
- How dice are rolled (1 die? 2 dice? weighted?)
- What a cell does when a player lands on it (snake? ladder? nothing? bomb?)
- What the win condition is (reach cell 100? capture the king?)
- How the next player is picked (round-robin? skip on snake?)

This maps directly to **Template Method** (fixed skeleton) + **Strategy** (pluggable steps).

---

## 3. Entities & Relationships

```
Game (Context + Template)
  ├── Board
  │     └── Cell[100]       ← polymorphic: Normal | Snake | Ladder | Portal
  ├── Player[]
  ├── Dice (interface)      ← StandardDice | WeightedDice | TwoDice
  ├── TurnManager           ← round-robin, skip logic
  ├── GameState             ← WAITING | IN_PROGRESS | PAUSED | FINISHED
  └── GameObserver[]        ← UI, logger, analytics

Cell (abstract)
  ├── cellId: number
  └── land(player, game): void   ← THE extensibility hook

Player
  ├── position: number
  ├── PlayerType: HUMAN | BOT
  └── status: ACTIVE | SKIPPED | WON

Dice (interface)
  └── roll(): number
```

---

## 4. Full TypeScript Implementation

```typescript
// ============================================================
// ENUMS
// ============================================================

enum GameState {
  WAITING     = "WAITING",      // Not enough players yet
  IN_PROGRESS = "IN_PROGRESS",  // Game is running
  PAUSED      = "PAUSED",       // Temporarily halted
  FINISHED    = "FINISHED",     // A winner has been declared
}

enum PlayerType {
  HUMAN = "HUMAN",
  BOT   = "BOT",
}

enum PlayerStatus {
  ACTIVE  = "ACTIVE",   // Eligible to take turns
  SKIPPED = "SKIPPED",  // Skip next turn (some house rules)
  WON     = "WON",      // Has reached the winning cell
}

// ============================================================
// DICE — Strategy Pattern
// ============================================================

// Every dice variant implements this interface.
// The game loop only calls roll() — it doesn't care how many dice
// are used or whether they are weighted.
interface Dice {
  roll(): number;
  getDescription(): string;
}

// Standard single die — fair random 1–6
class StandardDice implements Dice {
  constructor(private readonly faces: number = 6) {}

  roll(): number {
    // Math.random() → [0, 1), multiply by faces, floor, +1 → [1, faces]
    return Math.floor(Math.random() * this.faces) + 1;
  }

  getDescription(): string {
    return `Standard ${this.faces}-faced die`;
  }
}

// Two dice rolled together — sum of both
// Real Snake & Ladder uses this variant
class TwoDice implements Dice {
  private die: StandardDice;

  constructor(faces: number = 6) {
    this.die = new StandardDice(faces);
  }

  roll(): number {
    return this.die.roll() + this.die.roll();
  }

  getDescription(): string {
    return "Two 6-faced dice";
  }
}

// Weighted dice — rigged for testing or "easy mode"
// Accepts a weights array where weights[i] = probability of rolling (i+1)
class WeightedDice implements Dice {
  constructor(private readonly weights: number[]) {
    const total = weights.reduce((a, b) => a + b, 0);
    if (Math.abs(total - 1.0) > 0.001) {
      throw new Error("Weights must sum to 1.0");
    }
  }

  roll(): number {
    let random = Math.random();
    for (let i = 0; i < this.weights.length; i++) {
      random -= this.weights[i];
      if (random <= 0) return i + 1;
    }
    return this.weights.length; // fallback
  }

  getDescription(): string {
    return "Weighted die (biased)";
  }
}

// Deterministic dice — for unit testing the game logic
// Feed in a sequence of values; rolls them in order
class DeterministicDice implements Dice {
  private index = 0;

  constructor(private readonly sequence: number[]) {}

  roll(): number {
    const value = this.sequence[this.index % this.sequence.length];
    this.index++;
    return value;
  }

  getDescription(): string {
    return `Deterministic dice: [${this.sequence.join(", ")}]`;
  }
}

// ============================================================
// PLAYER
// ============================================================

class Player {
  public position: number = 0;      // 0 = start (off-board), 1–100 = on board
  public status: PlayerStatus = PlayerStatus.ACTIVE;
  public turnsSkipped: number = 0;  // for house rules that skip turns
  public rollHistory: number[] = []; // audit trail

  constructor(
    public readonly playerId: string,
    public readonly name: string,
    public readonly type: PlayerType = PlayerType.HUMAN
  ) {}

  // Move the player forward by `steps` cells.
  // The caller (game loop) handles overshooting — this is just raw position update.
  moveTo(targetCell: number): void {
    this.position = targetCell;
  }

  markWon(): void {
    this.status = PlayerStatus.WON;
  }

  skipNextTurn(): void {
    this.turnsSkipped++;
    this.status = PlayerStatus.SKIPPED;
  }

  // Called at start of turn — check if this player's skip penalty is over
  clearSkipIfReady(): void {
    if (this.status === PlayerStatus.SKIPPED && this.turnsSkipped > 0) {
      this.turnsSkipped--;
      if (this.turnsSkipped === 0) this.status = PlayerStatus.ACTIVE;
    }
  }

  isBot(): boolean {
    return this.type === PlayerType.BOT;
  }
}

// ============================================================
// CELL — Template Method Hook (the extensibility core)
// ============================================================

// Cell is an abstract base class. The `land()` method is the hook —
// subclasses override it to implement their effect.
//
// This is the key design: the game loop calls cell.land(player, game)
// and does NOT know or care what kind of cell it is.
// Adding a new cell type (BombCell, FreezCell) = new subclass only.
abstract class Cell {
  constructor(public readonly cellId: number) {}

  // Called when a player lands on this cell.
  // The `game` reference lets cells affect game state (e.g., skip turns).
  abstract land(player: Player, game: BoardGame): void;

  abstract getSymbol(): string; // for display: "·", "S↓", "L↑", etc.
}

// A plain cell with no special effect
class NormalCell extends Cell {
  land(_player: Player, _game: BoardGame): void {
    // No effect — player simply stays here
  }

  getSymbol(): string {
    return "·";
  }
}

// Snake cell — sends the player to a lower cell (the snake's tail)
class SnakeCell extends Cell {
  constructor(
    cellId: number,
    public readonly tailCell: number // where the snake's tail is
  ) {
    super(cellId);
    if (tailCell >= cellId) {
      throw new Error(`Snake tail (${tailCell}) must be below head (${cellId})`);
    }
  }

  land(player: Player, game: BoardGame): void {
    const from = player.position;
    player.moveTo(this.tailCell);
    // Notify the game (Observer pattern) about this event
    game.emit({
      type: "SNAKE_BITE",
      player,
      from,
      to: this.tailCell,
      message: `${player.name} bitten by snake at ${from} → slid to ${this.tailCell}`,
    });
  }

  getSymbol(): string {
    return `S↓${this.tailCell}`;
  }
}

// Ladder cell — sends the player to a higher cell (the top of the ladder)
class LadderCell extends Cell {
  constructor(
    cellId: number,
    public readonly topCell: number // where the ladder reaches
  ) {
    super(cellId);
    if (topCell <= cellId) {
      throw new Error(`Ladder top (${topCell}) must be above base (${cellId})`);
    }
  }

  land(player: Player, game: BoardGame): void {
    const from = player.position;
    player.moveTo(this.topCell);
    game.emit({
      type: "LADDER_CLIMB",
      player,
      from,
      to: this.topCell,
      message: `${player.name} climbed ladder at ${from} → reached ${this.topCell}`,
    });
  }

  getSymbol(): string {
    return `L↑${this.topCell}`;
  }
}

// Mystery cell — effect is random each time (extensibility demo)
// Could be: +10, -10, skip turn, double next roll, etc.
class MysteryCell extends Cell {
  private effects: Array<(player: Player, game: BoardGame) => void>;

  constructor(cellId: number) {
    super(cellId);
    // Define a pool of possible effects
    this.effects = [
      (p, g) => {
        const bonus = 5;
        p.moveTo(Math.min(p.position + bonus, g.getBoard().getSize()));
        g.emit({ type: "MYSTERY", player: p, from: p.position - bonus, to: p.position, message: `${p.name} got mystery bonus: +${bonus}` });
      },
      (p, g) => {
        p.skipNextTurn();
        g.emit({ type: "MYSTERY", player: p, from: p.position, to: p.position, message: `${p.name} gets mystery penalty: skip next turn` });
      },
    ];
  }

  land(player: Player, game: BoardGame): void {
    // Pick a random effect
    const effect = this.effects[Math.floor(Math.random() * this.effects.length)];
    effect(player, game);
  }

  getSymbol(): string {
    return "?";
  }
}

// ============================================================
// BOARD
// ============================================================

// The board is a flat array of Cells indexed 1..size.
// It is configured once at game start and never mutated during play.
class Board {
  private cells: Map<number, Cell> = new Map();
  private readonly size: number;

  constructor(size: number = 100) {
    this.size = size;
    // Initialize all cells as NormalCell
    for (let i = 1; i <= size; i++) {
      this.cells.set(i, new NormalCell(i));
    }
  }

  // Override a cell with a special tile (snake, ladder, mystery)
  setCell(cell: Cell): void {
    if (cell.cellId < 1 || cell.cellId > this.size) {
      throw new Error(`Cell ${cell.cellId} is out of board bounds (1–${this.size})`);
    }
    if (cell.cellId === this.size) {
      throw new Error(`Cannot place a special cell on the winning cell (${this.size})`);
    }
    this.cells.set(cell.cellId, cell);
  }

  getCell(position: number): Cell {
    const cell = this.cells.get(position);
    if (!cell) throw new Error(`Cell ${position} does not exist`);
    return cell;
  }

  getSize(): number {
    return this.size;
  }

  // Pretty-print the board for debugging
  print(): void {
    console.log("\n=== Board Layout ===");
    for (let i = this.size; i >= 1; i -= 10) {
      const row: string[] = [];
      for (let j = i; j > i - 10 && j >= 1; j--) {
        const sym = this.cells.get(j)?.getSymbol() ?? "·";
        row.push(`${j}:${sym}`.padEnd(10));
      }
      console.log(row.join(" "));
    }
    console.log("===================\n");
  }
}

// ============================================================
// GAME EVENTS — Observer Pattern
// ============================================================

// A GameEvent is emitted by the game loop and cells.
// Observers (UI, logger, analytics) receive these without coupling to internals.
interface GameEvent {
  type: string;
  player: Player;
  from?: number;
  to?: number;
  roll?: number;
  message: string;
}

interface GameObserver {
  onEvent(event: GameEvent): void;
}

// Simple console logger — a default observer
class ConsoleLogger implements GameObserver {
  onEvent(event: GameEvent): void {
    const prefix = {
      ROLL:          "🎲",
      MOVE:          "➡️ ",
      SNAKE_BITE:    "🐍",
      LADDER_CLIMB:  "🪜",
      WIN:           "🏆",
      MYSTERY:       "❓",
      SKIP:          "⏭️ ",
    }[event.type] ?? "📌";
    console.log(`  ${prefix} ${event.message}`);
  }
}

// ============================================================
// TURN MANAGER
// ============================================================

// Manages player turn order. Default: round-robin.
// Extracted so alternate turn rules (skip-on-snake, reverse-on-ladder)
// can be swapped in without changing the game loop.
class TurnManager {
  private currentIndex: number = 0;

  constructor(private readonly players: Player[]) {}

  // Get the player whose turn it currently is
  getCurrentPlayer(): Player {
    return this.players[this.currentIndex];
  }

  // Advance to the next eligible (ACTIVE) player
  advance(): void {
    let attempts = 0;
    do {
      this.currentIndex = (this.currentIndex + 1) % this.players.length;
      attempts++;
      // Avoid infinite loop if all players are SKIPPED/WON
      if (attempts > this.players.length) break;
    } while (
      this.players[this.currentIndex].status === PlayerStatus.WON
    );
  }

  // Are all non-won players done? (game over condition when no active players)
  allPlayersFinished(): boolean {
    return this.players.every(p => p.status === PlayerStatus.WON);
  }
}

// ============================================================
// BOARD GAME — Template Method
// ============================================================

// BoardGame is the abstract template class.
// The game loop is fixed (playTurn is final in spirit).
// Subclasses override hooks: initialiseBoard(), isWinCondition(), etc.
//
// This is the Template Method pattern: the skeleton is here,
// the specific behaviour is in concrete subclasses or injected strategies.
abstract class BoardGame {
  protected state: GameState = GameState.WAITING;
  protected board!: Board;
  protected turnManager!: TurnManager;
  private observers: GameObserver[] = [];

  constructor(
    protected readonly players: Player[],
    protected readonly dice: Dice
  ) {}

  // ── TEMPLATE METHODS (overridden by subclasses) ─────────────

  // Subclass configures the board — places snakes, ladders, mysteries
  protected abstract initialiseBoard(): Board;

  // Subclass defines the win condition
  // Default: reaching or passing the last cell
  protected isWinCondition(player: Player): boolean {
    return player.position >= this.board.getSize();
  }

  // Subclass can add extra logic after a move (e.g., bonus turns on doubles)
  protected onAfterMove(_player: Player, _roll: number): void {}

  // ── PUBLIC API ───────────────────────────────────────────────

  addObserver(observer: GameObserver): void {
    this.observers.push(observer);
  }

  // Emit an event to all observers
  emit(event: GameEvent): void {
    for (const obs of this.observers) {
      obs.onEvent(event);
    }
  }

  getBoard(): Board {
    return this.board;
  }

  getState(): GameState {
    return this.state;
  }

  // ── GAME LIFECYCLE ───────────────────────────────────────────

  start(): void {
    if (this.players.length < 2) {
      throw new Error("Need at least 2 players to start");
    }
    this.board = this.initialiseBoard();
    this.turnManager = new TurnManager(this.players);
    this.state = GameState.IN_PROGRESS;
    console.log(`\n=== Game started with ${this.players.length} players ===`);
    console.log(`Dice: ${this.dice.getDescription()}\n`);
  }

  // Play a single turn for the current player.
  // Returns true if the game is over after this turn.
  playTurn(): boolean {
    if (this.state !== GameState.IN_PROGRESS) {
      throw new Error(`Game is not in progress (state: ${this.state})`);
    }

    const player = this.turnManager.getCurrentPlayer();

    // Handle SKIPPED player — clear skip and pass the turn
    if (player.status === PlayerStatus.SKIPPED) {
      player.clearSkipIfReady();
      this.emit({ type: "SKIP", player, message: `${player.name} skips this turn` });
      this.turnManager.advance();
      return false;
    }

    // ── ROLL ────────────────────────────────────────────────────
    const roll = this.dice.roll();
    player.rollHistory.push(roll);
    this.emit({ type: "ROLL", player, roll, message: `${player.name} rolled ${roll}` });

    // ── MOVE ────────────────────────────────────────────────────
    const rawTarget = player.position + roll;
    const boardSize = this.board.getSize();

    // Overshoot rule: if the roll would take the player past the last cell,
    // they stay put (some variants use bounce-back — configurable via override)
    if (rawTarget > boardSize) {
      this.emit({
        type: "MOVE",
        player,
        from: player.position,
        to: player.position,
        message: `${player.name} stays at ${player.position} (overshoot — needs exact ${boardSize - player.position})`,
      });
      this.turnManager.advance();
      return false;
    }

    // Move the player to the raw target first
    player.moveTo(rawTarget);
    this.emit({
      type: "MOVE",
      player,
      from: player.position - roll, // approximate pre-move position for display
      to: rawTarget,
      message: `${player.name} moves to cell ${rawTarget}`,
    });

    // ── CELL EFFECT ─────────────────────────────────────────────
    // This is the polymorphic dispatch: cell.land() does the right thing
    // for SnakeCell, LadderCell, MysteryCell, or NormalCell — all the same call.
    const cell = this.board.getCell(player.position);
    cell.land(player, this);

    // ── WIN CHECK ────────────────────────────────────────────────
    if (this.isWinCondition(player)) {
      player.markWon();
      this.state = GameState.FINISHED;
      this.emit({
        type: "WIN",
        player,
        message: `${player.name} WON the game!`,
      });
      return true;
    }

    // ── POST-MOVE HOOK ───────────────────────────────────────────
    this.onAfterMove(player, roll);

    this.turnManager.advance();
    return false;
  }

  // Run the entire game to completion (for simulation / demo)
  run(maxTurns: number = 1000): Player | null {
    this.start();
    let turns = 0;

    while (this.state === GameState.IN_PROGRESS && turns < maxTurns) {
      const finished = this.playTurn();
      turns++;
      if (finished) break;
    }

    if (this.state !== GameState.FINISHED) {
      console.warn(`[GAME] Max turns (${maxTurns}) reached without a winner`);
      return null;
    }

    const winner = this.players.find(p => p.status === PlayerStatus.WON) ?? null;
    console.log(`\n=== Game over in ${turns} turns. Winner: ${winner?.name ?? "None"} ===\n`);
    return winner;
  }
}

// ============================================================
// SNAKES AND LADDERS — Concrete Game
// ============================================================

// SnakesAndLaddersGame is the concrete subclass.
// It only implements initialiseBoard() — all game loop logic lives in BoardGame.
// Replacing this with ChessGame or LudoGame = just a new subclass.
class SnakesAndLaddersGame extends BoardGame {
  protected initialiseBoard(): Board {
    const board = new Board(100);

    // Classic snake positions: head → tail
    const snakes: [number, number][] = [
      [99, 78], [95, 75], [92, 88], [89, 68],
      [74, 53], [64, 60], [62, 19], [49, 11],
      [46, 25], [16, 6],
    ];

    // Classic ladder positions: base → top
    const ladders: [number, number][] = [
      [2, 38], [7, 14], [8, 31], [15, 26],
      [21, 42], [28, 84], [36, 44], [51, 67],
      [71, 91], [80, 100],
    ];

    for (const [head, tail] of snakes) {
      board.setCell(new SnakeCell(head, tail));
    }

    for (const [base, top] of ladders) {
      // Ladder's "top" can be cell 100 (win instantly) — skip setCell for 100
      if (top < 100) {
        board.setCell(new LadderCell(base, top));
      }
    }

    // A couple of mystery cells for fun
    board.setCell(new MysteryCell(33));
    board.setCell(new MysteryCell(55));

    return board;
  }

  // Override: exact landing on 100 required to win (stricter rule)
  protected isWinCondition(player: Player): boolean {
    return player.position === this.board.getSize();
  }
}

// ============================================================
// GAME FACTORY — Factory Pattern
// ============================================================

// Factory encapsulates game construction — client code doesn't
// need to know which concrete game class or dice to use.
class BoardGameFactory {
  static createSnakesAndLadders(playerNames: string[], useTwoDice: boolean = false): SnakesAndLaddersGame {
    const players = playerNames.map(
      (name, i) => new Player(`P${i + 1}`, name, PlayerType.HUMAN)
    );
    const dice: Dice = useTwoDice ? new TwoDice() : new StandardDice();
    const game = new SnakesAndLaddersGame(players, dice);
    game.addObserver(new ConsoleLogger());
    return game;
  }

  // For automated testing — uses deterministic dice
  static createTestGame(playerNames: string[], diceSequence: number[]): SnakesAndLaddersGame {
    const players = playerNames.map(
      (name, i) => new Player(`P${i + 1}`, name, PlayerType.BOT)
    );
    const dice = new DeterministicDice(diceSequence);
    return new SnakesAndLaddersGame(players, dice);
  }
}

// ============================================================
// DEMO
// ============================================================

function main() {
  console.log("=== Demo: Standard Snake & Ladder ===\n");
  const game = BoardGameFactory.createSnakesAndLadders(["Alice", "Bob", "Charlie"]);
  const winner = game.run(500);
  console.log(`Winner: ${winner?.name ?? "No winner"}`);

  console.log("\n=== Demo: Deterministic test (verify ladder at cell 2→38) ===");
  // Player 1 rolls 2 → lands on cell 2 (ladder base) → teleports to 38
  const testGame = BoardGameFactory.createTestGame(["TestA", "TestB"], [2, 1]);
  testGame.start();
  testGame.playTurn(); // TestA rolls 2 → cell 2 → ladder → cell 38
  console.log(`TestA position after roll of 2: ${testGame["players"][0].position} (expected 38)`);
}

main();
```

---

## 5. Design Patterns — Deep Dive

### 5.1 Template Method — `BoardGame`

This is the core pattern. `BoardGame.playTurn()` is the fixed skeleton:

```
roll → move → apply cell effect → check win → advance turn
```

Subclasses customise behaviour via three hooks:
- `initialiseBoard()` — **abstract**, must be implemented
- `isWinCondition()` — **overrideable**, default is reach-end
- `onAfterMove()` — **optional hook**, e.g. bonus turn on a double roll in Ludo

**Why not just one `SnakesAndLaddersGame` class?**  
Because in 6 months you're asked to add Ludo. With Template Method, `LudoGame extends BoardGame` and overrides `initialiseBoard()` + adds token capture logic. Zero changes to `BoardGame`.

### 5.2 Strategy — `Dice`

Three axes that can vary independently:
- Number of dice (one, two)
- Fairness (standard, weighted)
- Determinism (real random, scripted for testing)

`DeterministicDice` is the most important one for **unit testing** — you can verify that landing on cell 2 always teleports to 38, reproducibly.

### 5.3 Observer — `GameObserver`

The game loop emits events. Who listens is not the game's concern. Add a `UIRenderer`, `SoundEffect`, or `AnalyticsTracker` by adding one observer — no changes to the game loop.

**Event types:** `ROLL`, `MOVE`, `SNAKE_BITE`, `LADDER_CLIMB`, `WIN`, `MYSTERY`, `SKIP`

### 5.4 Polymorphism on `Cell` (not a formal GoF pattern, but the key OOP insight)

The `cell.land(player, game)` call is the whole secret. The game loop has **zero `if/else` or `switch`** for cell types. It calls `land()` and trusts the cell to do the right thing. This is the Open/Closed Principle in practice:
- **Closed for modification** — adding `BombCell` doesn't touch any existing code
- **Open for extension** — just add a new subclass

---

## 6. The Generic Board Game Framework

Here's what makes this answer stand out — show the interviewer that your design supports more than just Snake & Ladder:

```typescript
// Ludo — four-player, tokens, home column
class LudoGame extends BoardGame {
  protected initialiseBoard(): Board {
    return new Board(56); // Ludo uses a different board structure
  }

  protected isWinCondition(player: Player): boolean {
    return player.position === 56; // home column end
  }

  protected onAfterMove(player: Player, roll: number): void {
    // In Ludo: rolling a 6 gives another turn
    if (roll === 6) {
      console.log(`${player.name} rolled 6 — gets an extra turn!`);
      this.turnManager.revertAdvance(); // bonus turn hook
    }
  }
}

// Chess — completely different board, no dice, different win
class ChessGame extends BoardGame {
  protected initialiseBoard(): Board {
    return new Board(64); // 8×8 = 64 squares
  }

  protected isWinCondition(player: Player): boolean {
    // Chess win = opponent's king is captured — different logic entirely
    return this.isOpponentKingCaptured(player);
  }

  private isOpponentKingCaptured(_player: Player): boolean {
    // Chess-specific logic — not relevant here
    return false;
  }
}
```

The game loop in `BoardGame.run()` is shared across all three games. Only the hooks differ.

---

## 7. Edge Cases & Nuances

| Edge Case | How to Handle |
|-----------|--------------|
| **Overshoot (roll past cell 100)** | Player stays at current position; needs exact roll to win |
| **Snake on cell 100** | Validate in `Board.setCell()` — throw if special cell placed on winning cell |
| **Ladder leading to snake head** | Chain resolution: `LadderCell.land()` moves player → game loop calls `cell.land()` again for the new cell. Cap recursion depth. |
| **All players skip simultaneously** | `TurnManager.allPlayersFinished()` breaks the loop |
| **Two players on same cell** | Allowed in Snake & Ladder; not allowed in Ludo. Override `onAfterMove()` to handle collision. |
| **Player rolls on SKIPPED turn** | `TurnManager.advance()` skips SKIPPED players automatically |
| **Max turns exceeded** | `run(maxTurns)` guard prevents infinite loops in degenerate board configs |
| **Bot player** | `Player.isBot()` flag. AI strategy can be injected as a `BotStrategy` interface — deterministic dice + min-path algorithm. |

### Chain Resolution (Snake → Ladder → Snake):
```typescript
// After landing, apply cell effect, then re-check the NEW cell.
// Cap at a small depth to prevent infinite chain bugs.
private applyCell(player: Player, depth: number = 0): void {
  if (depth > 3) return; // safety cap
  const cell = this.board.getCell(player.position);
  const positionBefore = player.position;
  cell.land(player, this);
  // If position changed (snake/ladder fired), check the new cell too
  if (player.position !== positionBefore) {
    this.applyCell(player, depth + 1);
  }
}
```

---

## 8. Follow-Up Questions Interviewers Ask

1. **"How would you make this multiplayer over a network?"**
   → `GameEvent` objects are already serializable. Add a `WebSocketObserver` that broadcasts events. Client sends `PlayTurnCommand`; server validates and processes. State lives on the server.

2. **"How would you add a leaderboard?"**
   → `LeaderboardObserver implements GameObserver`. On `WIN` event, record `{ playerName, turns: player.rollHistory.length, date }`. Sorted by fewest turns. Stored in a persistent map.

3. **"How would you support saving and resuming a game?"**
   → Serialize game state: `{ players: Player[], boardConfig: CellOverride[], diceType, turnIndex }`. On resume: deserialize and call `start()` with pre-populated state. This is the Memento pattern applied to a game.

4. **"How would you make the board configurable via JSON?"**
   → `BoardGameFactory.createFromConfig(config: BoardConfig)` — reads snakes/ladders/mysteries from a JSON file. `BoardConfig` maps cell IDs to cell types. No code change for new board variants.

5. **"What if a Snake lands on another Snake's tail?"**
   → Chain resolution (see edge case above). Max depth guard prevents infinite recursion in pathological configs. Also: validate board config at startup — no chains longer than N hops.

---

## 9. Key Takeaways

- **Template Method is the answer** to "design a generic board game." The invariant game loop lives in `BoardGame`; the variant behaviour lives in hooks. This is what interviewers mean when they say "extensible design."
- **`Cell.land()` polymorphism eliminates all `if/else` on cell type.** This is the Open/Closed Principle applied to a concrete problem. Adding `BombCell` is a new file, not an edit to an existing one.
- **`DeterministicDice`** is the signal that you think about testability, not just runtime behavior. You can write a unit test that says "if player rolls 2, they should be on cell 38" and have it pass 100% of the time.
- **Observer for game events** decouples the game loop from UI, audio, and analytics. In an interview, mention the `WebSocketObserver` extension — it shows you're thinking distributed from the start.
- **Overshoot rule** is the most common missed edge case. Mention it before the interviewer does.
- **Chain resolution** (snake at top of ladder) shows real-world thinking. Cap the recursion depth — a badly configured board could otherwise spin forever.

---

*Next up: LLD #05 — Library Management System*