# LLD #02 — Elevator System

> **Language:** TypeScript  
> **Difficulty:** ⭐⭐⭐ (Intermediate–Senior)  
> **Patterns Used:** State, Strategy, Observer, Singleton, Command  

---

## 1. Requirements

### Functional Requirements
- Support a building with N floors and M elevators
- Users can press an external button (floor panel) to request an elevator: UP or DOWN
- Users inside an elevator press an internal button to select a destination floor
- Each elevator has a state: IDLE, MOVING_UP, MOVING_DOWN, DOORS_OPEN, EMERGENCY
- The dispatcher assigns the best elevator to an external request
- Doors open when the elevator reaches a requested floor, then close after a timeout
- Support an emergency stop button inside the elevator

### Non-Functional Requirements
- Efficient scheduling — minimize total passenger wait time
- Thread-safe: concurrent button presses must not corrupt state
- Easy to swap scheduling algorithms (pluggable dispatcher)
- Extensible: adding a new elevator state should not require changes in caller code

### What interviewers watch for
- Do you model the State Machine explicitly, or leave it as a mess of `if/else`?
- Do you separate dispatch logic (which elevator?) from movement logic (how to move)?
- Do you know the SCAN algorithm and why it's used in real elevators?
- Do you handle edge cases: door held open, no pending requests, emergency mode?

---

## 2. The Core Challenge — State Machine

An elevator is fundamentally a **finite state machine (FSM)**. The five states and their legal transitions:

```
IDLE
  → MOVING_UP    (when a request is above current floor)
  → MOVING_DOWN  (when a request is below current floor)

MOVING_UP
  → DOORS_OPEN   (reached a requested floor)
  → MOVING_UP    (self-loop: still floors to serve above)

MOVING_DOWN
  → DOORS_OPEN   (reached a requested floor)
  → MOVING_DOWN  (self-loop: still floors to serve below)

DOORS_OPEN
  → IDLE          (no more pending requests)
  → MOVING_UP     (more requests above, continue scan)
  → MOVING_DOWN   (more requests below, continue scan)
  → EMERGENCY     (emergency button pressed)

EMERGENCY
  → IDLE          (only via manual reset by technician)
```

**Why use the State Pattern here?**  
Without it, every method becomes:
```typescript
if (state === "MOVING_UP") { ... }
else if (state === "DOORS_OPEN") { ... }
// multiply across every method, every new state, forever
```
With the State Pattern, each state is a class that handles only its own valid transitions. Invalid transitions throw — no silent bugs.

---

## 3. The Core Challenge — Scheduling (SCAN Algorithm)

The most important nuance interviewers test: **how does an elevator decide which floor to go to next?**

### Naive approach (FCFS — First Come, First Served)
Serve requests in the order they arrive.

**Problem:** Floor 1 → Floor 10 → Floor 2 → Floor 9 → Floor 3 — the elevator zigzags constantly, maximizing travel distance.

### SCAN Algorithm (Elevator Algorithm)
The elevator moves in one direction, serving all requests in that direction. Only after it has no more requests in the current direction does it reverse.

Think of it like a disk head scheduling algorithm — the elevator "scans" in one direction.

```
Elevator is at floor 3, moving UP.
Pending requests: [1, 7, 2, 5, 9]

SCAN picks: 5 → 7 → 9 (all above, in order)
Then reverses DOWN: 2 → 1

vs FCFS: 1 → 7 → 2 → 5 → 9 (zigzag)
```

**SCAN stops at the furthest request in its current direction, then reverses.**  
**LOOK** is a variant that reverses early — when there are no more requests in the current direction — instead of going all the way to the top/bottom floor. We'll implement LOOK.

---

## 4. Entities & Class Design

```
Building (Singleton)
  └── Elevator[]
  └── ElevatorController (Dispatcher)
  └── Floor[]

Elevator
  └── ElevatorState (abstract)
        ├── IdleState
        ├── MovingUpState
        ├── MovingDownState
        ├── DoorsOpenState
        └── EmergencyState
  └── pendingFloors: SortedSet<number>   ← floors the elevator must visit
  └── currentFloor: number

Floor
  └── upButton: Button
  └── downButton: Button

ElevatorController (Dispatcher)
  └── assignElevator(floor, direction): Elevator
  └── strategy: DispatchStrategy

<<interface>> DispatchStrategy
  ├── NearestElevatorStrategy
  └── LeastLoadStrategy
```

---

## 5. Full TypeScript Implementation

```typescript
// ============================================================
// ENUMS
// ============================================================

// Direction of travel or button press
enum Direction {
  UP = "UP",
  DOWN = "DOWN",
  NONE = "NONE", // used when elevator is IDLE
}

// All legal elevator states
enum ElevatorStatus {
  IDLE = "IDLE",
  MOVING_UP = "MOVING_UP",
  MOVING_DOWN = "MOVING_DOWN",
  DOORS_OPEN = "DOORS_OPEN",
  EMERGENCY = "EMERGENCY",
}

// ============================================================
// ELEVATOR REQUEST
// ============================================================

// Represents a single request — either an external hall call or
// an internal cabin call. Encapsulating it as an object (Command Pattern)
// makes it easy to queue, log, and replay requests.
class ElevatorRequest {
  constructor(
    public readonly floor: number,
    public readonly direction: Direction, // NONE for internal (cabin) requests
    public readonly timestamp: Date = new Date()
  ) {}

  // Internal requests (from inside the cabin) have no direction preference
  isInternal(): boolean {
    return this.direction === Direction.NONE;
  }
}

// ============================================================
// STATE PATTERN — Abstract Base + Concrete States
// ============================================================

// The abstract state class declares all events an elevator can receive.
// Each concrete state implements only the transitions it allows.
// Illegal transitions throw — making invalid state changes loud, not silent.
abstract class ElevatorState {
  constructor(protected elevator: Elevator) {}

  // Called when the elevator is asked to move to the next target floor
  abstract processNextFloor(): void;

  // Called when the elevator physically reaches a floor
  abstract onFloorReached(floor: number): void;

  // Called when doors finish closing
  abstract onDoorsClosed(): void;

  // Emergency button pressed
  onEmergency(): void {
    // Emergency is valid from ANY state — always transition to EMERGENCY
    console.warn(`[EMERGENCY] Elevator ${this.elevator.id} — emergency triggered!`);
    this.elevator.setState(new EmergencyState(this.elevator));
  }

  abstract getStatus(): ElevatorStatus;
}

// ── IDLE STATE ────────────────────────────────────────────────
// The elevator is stationary with doors closed. It waits for a request.
class IdleState extends ElevatorState {
  getStatus(): ElevatorStatus { return ElevatorStatus.IDLE; }

  processNextFloor(): void {
    const next = this.elevator.getNextTarget();
    if (next === null) return; // truly nothing to do

    if (next > this.elevator.currentFloor) {
      this.elevator.setState(new MovingUpState(this.elevator));
    } else if (next < this.elevator.currentFloor) {
      this.elevator.setState(new MovingDownState(this.elevator));
    } else {
      // Target is the current floor — open doors immediately
      this.elevator.setState(new DoorsOpenState(this.elevator));
    }
  }

  onFloorReached(_floor: number): void {
    // Not moving, so this shouldn't fire. No-op defensively.
  }

  onDoorsClosed(): void {
    // Already closed. No-op.
  }
}

// ── MOVING UP STATE ───────────────────────────────────────────
// The elevator is ascending. On each tick it checks if the current floor
// matches any pending request. If yes, stop and open doors.
class MovingUpState extends ElevatorState {
  getStatus(): ElevatorStatus { return ElevatorStatus.MOVING_UP; }

  processNextFloor(): void {
    // Move the elevator one floor up
    this.elevator.currentFloor++;
    console.log(`[MOVE] Elevator ${this.elevator.id} → Floor ${this.elevator.currentFloor} ↑`);
    this.onFloorReached(this.elevator.currentFloor);
  }

  onFloorReached(floor: number): void {
    if (this.elevator.shouldStopAt(floor)) {
      // This floor has a pending request — stop and open doors
      this.elevator.removePendingFloor(floor);
      this.elevator.setState(new DoorsOpenState(this.elevator));
    } else {
      // Keep going — check if there's still a reason to move up
      const next = this.elevator.getNextTarget();
      if (next === null || next < this.elevator.currentFloor) {
        // No more requests above — reverse or idle
        this.elevator.setState(
          next !== null ? new MovingDownState(this.elevator) : new IdleState(this.elevator)
        );
      }
      // else continue ascending next tick
    }
  }

  onDoorsClosed(): void {
    // Doors shouldn't be open while moving. No-op.
  }
}

// ── MOVING DOWN STATE ─────────────────────────────────────────
// Mirror of MovingUpState, but descending.
class MovingDownState extends ElevatorState {
  getStatus(): ElevatorStatus { return ElevatorStatus.MOVING_DOWN; }

  processNextFloor(): void {
    this.elevator.currentFloor--;
    console.log(`[MOVE] Elevator ${this.elevator.id} → Floor ${this.elevator.currentFloor} ↓`);
    this.onFloorReached(this.elevator.currentFloor);
  }

  onFloorReached(floor: number): void {
    if (this.elevator.shouldStopAt(floor)) {
      this.elevator.removePendingFloor(floor);
      this.elevator.setState(new DoorsOpenState(this.elevator));
    } else {
      const next = this.elevator.getNextTarget();
      if (next === null || next > this.elevator.currentFloor) {
        this.elevator.setState(
          next !== null ? new MovingUpState(this.elevator) : new IdleState(this.elevator)
        );
      }
    }
  }

  onDoorsClosed(): void {
    // No-op.
  }
}

// ── DOORS OPEN STATE ──────────────────────────────────────────
// Doors are open for boarding/alighting. A timer triggers door close.
// During this state, new internal requests can be added.
class DoorsOpenState extends ElevatorState {
  private doorTimer: ReturnType<typeof setTimeout> | null = null;
  private static readonly DOOR_OPEN_DURATION_MS = 3000; // 3 seconds

  constructor(elevator: Elevator) {
    super(elevator);
    console.log(`[DOORS] Elevator ${elevator.id} — doors OPEN at floor ${elevator.currentFloor}`);
    this.startDoorTimer();
  }

  getStatus(): ElevatorStatus { return ElevatorStatus.DOORS_OPEN; }

  // Start a countdown to close the doors automatically
  private startDoorTimer(): void {
    this.doorTimer = setTimeout(() => {
      this.onDoorsClosed();
    }, DoorsOpenState.DOOR_OPEN_DURATION_MS);
  }

  // Door-open button pressed inside the cabin — reset the timer
  reopenDoors(): void {
    if (this.doorTimer) clearTimeout(this.doorTimer);
    console.log(`[DOORS] Elevator ${this.elevator.id} — door timer reset`);
    this.startDoorTimer();
  }

  processNextFloor(): void {
    // Cannot move while doors are open
    console.log(`[WARN] Cannot move while doors are open — Elevator ${this.elevator.id}`);
  }

  onFloorReached(_floor: number): void {
    // Already at the floor. No-op.
  }

  onDoorsClosed(): void {
    if (this.doorTimer) clearTimeout(this.doorTimer);
    console.log(`[DOORS] Elevator ${this.elevator.id} — doors CLOSED at floor ${this.elevator.currentFloor}`);

    // After doors close, decide next action using SCAN logic
    const next = this.elevator.getNextTarget();

    if (next === null) {
      this.elevator.setState(new IdleState(this.elevator));
    } else if (next > this.elevator.currentFloor) {
      this.elevator.setState(new MovingUpState(this.elevator));
    } else {
      this.elevator.setState(new MovingDownState(this.elevator));
    }
  }
}

// ── EMERGENCY STATE ───────────────────────────────────────────
// Elevator is stopped. All pending requests are cleared.
// Only a technician reset can move it back to IDLE.
class EmergencyState extends ElevatorState {
  constructor(elevator: Elevator) {
    super(elevator);
    elevator.clearAllRequests(); // Drop all pending floors — safety first
    console.error(`[EMERGENCY] Elevator ${elevator.id} halted. Manual reset required.`);
  }

  getStatus(): ElevatorStatus { return ElevatorStatus.EMERGENCY; }

  processNextFloor(): void {
    throw new Error(`Elevator ${this.elevator.id} is in EMERGENCY mode. Cannot move.`);
  }

  onFloorReached(_floor: number): void {}

  onDoorsClosed(): void {}

  onEmergency(): void {
    // Already in emergency, no-op
  }

  // Manual reset — only called by maintenance
  reset(): void {
    console.log(`[RESET] Elevator ${this.elevator.id} — emergency reset by technician`);
    this.elevator.setState(new IdleState(this.elevator));
  }
}

// ============================================================
// ELEVATOR
// ============================================================

// The Elevator class is the Context in the State Pattern.
// It delegates all behavior to its current state object.
// It owns the pending floors set and the SCAN scheduling logic.
class Elevator {
  public currentFloor: number;
  private state: ElevatorState;

  // We use two sorted sets to implement the SCAN/LOOK algorithm:
  // floorsAbove: floors to visit while moving up (ascending order)
  // floorsBelow: floors to visit while moving down (descending order)
  // This avoids re-sorting on every tick.
  private floorsAbove: number[] = []; // sorted ascending
  private floorsBelow: number[] = []; // sorted descending

  // Current sweep direction (for SCAN continuity across DOORS_OPEN)
  private sweepDirection: Direction = Direction.NONE;

  constructor(
    public readonly id: string,
    initialFloor: number = 1,
    public readonly minFloor: number = 1,
    public readonly maxFloor: number = 20
  ) {
    this.currentFloor = initialFloor;
    this.state = new IdleState(this); // Start in IDLE
  }

  // ── STATE MANAGEMENT ────────────────────────────────────────

  setState(newState: ElevatorState): void {
    this.state = newState;
  }

  getStatus(): ElevatorStatus {
    return this.state.getStatus();
  }

  // ── REQUEST HANDLING ─────────────────────────────────────────

  // Add a floor to the pending set (from dispatcher or internal button)
  addRequest(floor: number): void {
    if (floor < this.minFloor || floor > this.maxFloor) {
      throw new Error(`Floor ${floor} out of range [${this.minFloor}–${this.maxFloor}]`);
    }
    if (floor === this.currentFloor && this.getStatus() === ElevatorStatus.DOORS_OPEN) {
      return; // Already here and doors are open — nothing to do
    }

    if (floor > this.currentFloor) {
      if (!this.floorsAbove.includes(floor)) {
        this.floorsAbove.push(floor);
        this.floorsAbove.sort((a, b) => a - b); // ascending
      }
    } else {
      if (!this.floorsBelow.includes(floor)) {
        this.floorsBelow.push(floor);
        this.floorsBelow.sort((a, b) => b - a); // descending
      }
    }

    console.log(`[REQUEST] Elevator ${this.id} — floor ${floor} added to queue`);

    // If idle, immediately trigger movement
    if (this.getStatus() === ElevatorStatus.IDLE) {
      this.state.processNextFloor();
    }
  }

  // SCAN/LOOK: get the next floor to visit
  // While moving UP: pick the lowest floor above current
  // While moving DOWN: pick the highest floor below current
  // When switching direction: reverse
  getNextTarget(): number | null {
    const status = this.getStatus();

    if (status === ElevatorStatus.MOVING_UP || this.sweepDirection === Direction.UP) {
      // Prefer continuing upward
      if (this.floorsAbove.length > 0) {
        this.sweepDirection = Direction.UP;
        return this.floorsAbove[0]; // smallest floor above
      }
      if (this.floorsBelow.length > 0) {
        this.sweepDirection = Direction.DOWN;
        return this.floorsBelow[0]; // highest floor below (sorted desc)
      }
    } else if (status === ElevatorStatus.MOVING_DOWN || this.sweepDirection === Direction.DOWN) {
      // Prefer continuing downward
      if (this.floorsBelow.length > 0) {
        this.sweepDirection = Direction.DOWN;
        return this.floorsBelow[0]; // highest floor below
      }
      if (this.floorsAbove.length > 0) {
        this.sweepDirection = Direction.UP;
        return this.floorsAbove[0]; // lowest floor above
      }
    }

    // IDLE or no direction — pick nearest
    const nearest = this.getNearestFloor();
    if (nearest !== null) {
      this.sweepDirection = nearest > this.currentFloor ? Direction.UP : Direction.DOWN;
    }
    return nearest;
  }

  // Get the closest pending floor regardless of direction (used from IDLE)
  private getNearestFloor(): number | null {
    const candidates = [...this.floorsAbove, ...this.floorsBelow];
    if (candidates.length === 0) return null;
    return candidates.reduce((closest, floor) =>
      Math.abs(floor - this.currentFloor) < Math.abs(closest - this.currentFloor) ? floor : closest
    );
  }

  // Should the elevator stop at this floor? (was it requested?)
  shouldStopAt(floor: number): boolean {
    return this.floorsAbove.includes(floor) || this.floorsBelow.includes(floor);
  }

  removePendingFloor(floor: number): void {
    this.floorsAbove = this.floorsAbove.filter(f => f !== floor);
    this.floorsBelow = this.floorsBelow.filter(f => f !== floor);
  }

  clearAllRequests(): void {
    this.floorsAbove = [];
    this.floorsBelow = [];
    this.sweepDirection = Direction.NONE;
  }

  // ── PUBLIC ACTIONS ───────────────────────────────────────────

  // Simulate one "tick" of movement (called by the building loop)
  tick(): void {
    this.state.processNextFloor();
  }

  // Emergency button pressed inside cabin
  triggerEmergency(): void {
    this.state.onEmergency();
  }

  // Peek at pending floors for dispatcher scoring
  getPendingCount(): number {
    return this.floorsAbove.length + this.floorsBelow.length;
  }

  // Distance from current floor to a target (useful for dispatcher)
  distanceTo(floor: number): number {
    return Math.abs(this.currentFloor - floor);
  }
}

// ============================================================
// DISPATCH STRATEGY — Strategy Pattern
// ============================================================

// The dispatcher decides which elevator handles an external hall call.
// Multiple strategies can be swapped without touching the building logic.
interface DispatchStrategy {
  selectElevator(elevators: Elevator[], request: ElevatorRequest): Elevator | null;
}

// Nearest elevator — minimize wait time for this one request
class NearestElevatorStrategy implements DispatchStrategy {
  selectElevator(elevators: Elevator[], request: ElevatorRequest): Elevator | null {
    // Filter out elevators in emergency mode
    const available = elevators.filter(e => e.getStatus() !== ElevatorStatus.EMERGENCY);
    if (available.length === 0) return null;

    // Score: distance to requested floor
    // Prefer idle elevators that are closest
    return available.reduce((best, elevator) => {
      const bestScore = this.score(best, request);
      const currScore = this.score(elevator, request);
      return currScore < bestScore ? elevator : best;
    });
  }

  private score(elevator: Elevator, request: ElevatorRequest): number {
    const distance = elevator.distanceTo(request.floor);
    // Idle elevators are more efficient to redirect (no direction bias)
    const idleBonus = elevator.getStatus() === ElevatorStatus.IDLE ? -3 : 0;
    return distance + idleBonus;
  }
}

// Least-load elevator — distribute work evenly (good for high-traffic buildings)
class LeastLoadStrategy implements DispatchStrategy {
  selectElevator(elevators: Elevator[], _request: ElevatorRequest): Elevator | null {
    const available = elevators.filter(e => e.getStatus() !== ElevatorStatus.EMERGENCY);
    if (available.length === 0) return null;

    // Pick the elevator with the fewest pending requests
    return available.reduce((best, elevator) =>
      elevator.getPendingCount() < best.getPendingCount() ? elevator : best
    );
  }
}

// ============================================================
// ELEVATOR CONTROLLER (DISPATCHER)
// ============================================================

// The central dispatcher. When a hall button is pressed on a floor,
// the controller selects the best elevator and assigns it the request.
class ElevatorController {
  constructor(
    private elevators: Elevator[],
    private strategy: DispatchStrategy = new NearestElevatorStrategy()
  ) {}

  setStrategy(strategy: DispatchStrategy): void {
    this.strategy = strategy;
  }

  // External hall call — user pressed UP or DOWN on a floor panel
  handleHallCall(floor: number, direction: Direction): void {
    const request = new ElevatorRequest(floor, direction);
    const selected = this.strategy.selectElevator(this.elevators, request);

    if (!selected) {
      console.warn(`[DISPATCH] No elevator available for floor ${floor} ${direction}`);
      return;
    }

    console.log(`[DISPATCH] Floor ${floor} ${direction} → Elevator ${selected.id}`);
    selected.addRequest(floor);
  }

  // Internal cabin call — passenger pressed a floor button inside the elevator
  handleCabinCall(elevatorId: string, floor: number): void {
    const elevator = this.elevators.find(e => e.id === elevatorId);
    if (!elevator) throw new Error(`Elevator ${elevatorId} not found`);
    elevator.addRequest(floor);
  }
}

// ============================================================
// BUILDING — Singleton
// ============================================================

// The Building owns all elevators and the controller.
// It runs the simulation loop (tick-based, simulating time passing).
class Building {
  private static instance: Building | null = null;
  private controller: ElevatorController;

  private constructor(
    public readonly name: string,
    public readonly totalFloors: number,
    private elevators: Elevator[]
  ) {
    this.controller = new ElevatorController(elevators);
  }

  static getInstance(name: string, totalFloors: number, numElevators: number): Building {
    if (!Building.instance) {
      const elevators = Array.from(
        { length: numElevators },
        (_, i) => new Elevator(`E${i + 1}`, 1, 1, totalFloors)
      );
      Building.instance = new Building(name, totalFloors, elevators);
    }
    return Building.instance;
  }

  static resetInstance(): void {
    Building.instance = null;
  }

  getController(): ElevatorController {
    return this.controller;
  }

  // Advance all elevators by one tick (one floor of movement)
  tick(): void {
    for (const elevator of this.elevators) {
      const status = elevator.getStatus();
      if (status === ElevatorStatus.MOVING_UP || status === ElevatorStatus.MOVING_DOWN) {
        elevator.tick();
      }
    }
  }

  // Run the simulation for N ticks
  simulate(ticks: number): void {
    for (let i = 0; i < ticks; i++) {
      this.tick();
    }
  }

  printStatus(): void {
    console.log("\n=== Elevator Status ===");
    for (const e of this.elevators) {
      console.log(`Elevator ${e.id}: Floor ${e.currentFloor} | ${e.getStatus()}`);
    }
    console.log("======================\n");
  }
}

// ============================================================
// DEMO
// ============================================================

function main() {
  const building = Building.getInstance("TechPark Tower", 20, 3);
  const controller = building.getController();

  // External hall calls
  controller.handleHallCall(8, Direction.UP);   // Someone on floor 8 wants to go up
  controller.handleHallCall(3, Direction.DOWN);  // Someone on floor 3 wants to go down
  controller.handleHallCall(15, Direction.UP);   // Someone on floor 15 wants to go up

  // Internal cabin calls (once inside, passengers select destinations)
  controller.handleCabinCall("E1", 12); // Passenger in E1 wants floor 12
  controller.handleCabinCall("E2", 1);  // Passenger in E2 wants floor 1

  building.printStatus();

  // Simulate 20 ticks of movement
  building.simulate(20);

  building.printStatus();
}

main();
```

---

## 6. Design Patterns — Deep Dive

### 6.1 State Pattern — `ElevatorState`
**Why:** An elevator's behavior changes dramatically based on its current state. `processNextFloor()` does completely different things when IDLE (find a target, set direction) vs MOVING_UP (increment floor, check if we should stop) vs DOORS_OPEN (refuse movement, start timer).

**Without State Pattern:** Every method is a switch statement. Adding the `MAINTENANCE` state requires touching every switch statement.

**With State Pattern:** Add `MaintenanceState extends ElevatorState`, wire up its transitions, done. Zero changes to `Elevator` or caller code.

**Key insight:** State objects hold a reference back to the Context (`this.elevator`) so they can call `elevator.setState(new MovingUpState(...))` — the state transitions itself out.

### 6.2 Strategy — `DispatchStrategy`
**Why:** Dispatching is an independent concern from elevator movement. A high-traffic morning might use `LeastLoadStrategy`. An empty building at night uses `NearestElevatorStrategy`. Hot-swap at runtime.

### 6.3 Command — `ElevatorRequest`
**Why:** Encapsulating a request as an object (not just a floor number) allows:
- Timestamp for analytics ("average wait time")
- Direction metadata for smarter dispatch
- Request queuing and retry

### 6.4 Singleton — `Building`
**Why:** Same rationale as Parking Lot — one physical building, one source of truth for elevator state.

---

## 7. SCAN vs LOOK vs SSTF — Algorithm Comparison

| Algorithm | Strategy | Best For | Worst Case |
|-----------|----------|----------|------------|
| **FCFS** | Serve in arrival order | Trivial to implement | High zigzag on busy buildings |
| **SSTF** | Shortest Seek Time First — pick nearest request | Low avg wait for moderate traffic | Starvation: far floors never served |
| **SCAN** | Move in one direction until top/bottom, then reverse | Heavy uniform traffic | Floors at turnaround wait the longest |
| **LOOK** | Like SCAN but reverse early (when no more in direction) | Real-world buildings | Slightly more complex to implement |
| **C-SCAN** | Circular — always sweeps same direction, jumps back to start | Uniform wait time across all floors | One direction only; inefficient return trip |

**We implement LOOK** because:
1. Real elevators use it
2. It avoids the waste of SCAN going to top/bottom when no one needs it
3. It prevents starvation unlike SSTF (you'll always eventually get served on a reverse sweep)

---

## 8. Edge Cases & Nuances

| Edge Case | How to Handle |
|-----------|--------------|
| **Door held open** | `DoorsOpenState.reopenDoors()` resets the timer |
| **All elevators in emergency** | `selectElevator()` returns `null`; log + notify monitoring |
| **Request for current floor while IDLE** | `addRequest()` immediately opens doors (no movement needed) |
| **Request added while DOORS_OPEN** | Added to pending set; processed after doors close |
| **Same floor pressed multiple times** | Set semantics on `floorsAbove/floorsBelow` — no duplicates |
| **Elevator overshoot** | `shouldStopAt()` checked on every floor increment — can't overshoot |
| **Building power failure** | Transition all elevators to `EmergencyState` via event broadcast |
| **Passenger presses both UP and DOWN** | Two separate `ElevatorRequest` objects; dispatcher handles independently |
| **VIP/Priority floors** | Inject a `PriorityDispatchStrategy` that scores against a priority floor list |

---

## 9. Thread Safety Note

In a real TypeScript server (e.g., managing physical elevators via IoT):

```typescript
import { Mutex } from "async-mutex";

class Elevator {
  private mutex = new Mutex();

  // addRequest and tick must not run concurrently —
  // a tick could read stale floor state while a new request modifies it
  async addRequest(floor: number): Promise<void> {
    const release = await this.mutex.acquire();
    try {
      // ... same logic
    } finally {
      release();
    }
  }

  async tick(): Promise<void> {
    const release = await this.mutex.acquire();
    try {
      this.state.processNextFloor();
    } finally {
      release();
    }
  }
}
```

**Why it matters:** Without locking, a `tick()` could advance the elevator past a floor at the same moment `addRequest()` is inserting that floor into `floorsAbove`. The stop would be missed.

---

## 10. Follow-Up Questions Interviewers Ask

1. **"What if two elevators are assigned the same external request?"**  
   → Requests are assigned by the dispatcher to exactly one elevator. The dispatcher is the single point of assignment. For distributed systems, use a distributed lock (Redis) on the floor+direction key.

2. **"How would you add a weight sensor?"**  
   → `Elevator` gets a `currentWeight` and `maxWeight` property. `DoorsOpenState` polls the sensor. If overloaded, it refuses to close doors and alerts the passenger. `NearestElevatorStrategy` filters out near-full elevators.

3. **"How would you support express elevators (skip-floor service)?"**  
   → `Elevator` gets a `Set<number> of servicedFloors`. `addRequest()` validates against it and throws if the floor is not serviced. The dispatcher only assigns express elevators to requests on their served floors.

4. **"How would you design for 100 elevators in a skyscraper?"**  
   → Zone-based dispatch: floors 1–20 → elevator bank A, 21–40 → bank B. Each bank has its own `ElevatorController`. A `ZoneRouter` sits above them to handle cross-zone sky-lobby transfers.

5. **"How do you test the SCAN algorithm?"**  
   → Unit test `getNextTarget()` directly. Feed specific floor arrays to `floorsAbove` and `floorsBelow`, assert the returned sequence. No need to simulate full elevator movement — the algorithm is isolated to one method.

---

## 11. Key Takeaways

- The **State Pattern** is non-negotiable here. An elevator without explicit states is a maintenance nightmare. Name the states, draw the transitions, make illegal transitions throw.
- **SCAN/LOOK** is the algorithm interviewers are testing. Know it by heart: "continue in current direction, stop at every requested floor, reverse when no more in that direction."
- **Two sorted lists** (`floorsAbove` ascending, `floorsBelow` descending) is the efficient SCAN data structure. No re-sorting on every tick.
- **Strategy on the dispatcher** is the senior signal. Separating *which elevator?* (dispatcher) from *how to move?* (state machine) shows you understand SRP deeply.
- **Door timer management** is an edge case that reveals real-world thinking. The `DoorsOpenState` owns its own timer and can reset it — the elevator doesn't manage timers.
- Always mention **floor range validation** — an elevator cannot go to floor 25 in a 20-floor building.

---

*Next up: LLD #03 — Movie Ticket Booking System (BookMyShow)*