# LLD #01 — Parking Lot System

> **Language:** TypeScript  
> **Difficulty:** ⭐⭐ (Foundational)  
> **Patterns Used:** Strategy, Singleton, Factory, State, Observer  

---

## 1. Requirements

### Functional Requirements
- Support multiple levels, each with multiple parking spots
- Support different vehicle types: Motorcycle, Car, Truck
- Different spot sizes: Small, Medium, Large
- A vehicle should be parked in the most appropriate spot (smallest that fits)
- Issue a ticket on entry; calculate and collect fee on exit
- Support multiple entry and exit gates
- Track availability of spots in real time

### Non-Functional Requirements
- Thread-safe spot assignment (prevent double booking)
- Extensible pricing strategy (hourly, flat rate, surge, etc.)
- Easy to add new vehicle types or spot types

### What interviewers watch for
- Do you clarify requirements before coding?
- Do you identify the right entities (not just Spot and Vehicle)?
- Do you use design patterns purposefully, not decoratively?
- Do you handle edge cases: full lot, wrong vehicle size, lost ticket?

---

## 2. Entities & Relationships

```
ParkingLot (Singleton)
  └── Level[]
        └── ParkingSpot[]
              └── Vehicle (nullable — null means spot is free)

ParkingLot
  └── EntryGate[] / ExitGate[]
  └── PricingStrategy (pluggable)
  └── Ticket[] (active tickets map)

Ticket
  └── Vehicle
  └── ParkingSpot
  └── entryTime / exitTime
```

### Key Design Decisions
| Decision | Why |
|----------|-----|
| `ParkingLot` as Singleton | One lot instance controls all state |
| `PricingStrategy` as interface | Swap hourly/flat/surge without changing lot logic |
| Spot assignment in `Level` | Each level manages its own spots — SRP |
| `Ticket` as first-class entity | Needed for billing, audit trail, lost ticket handling |

---

## 3. Class Diagram (Text)

```
<<enum>> VehicleType       <<enum>> SpotSize
  MOTORCYCLE                 SMALL
  CAR                        MEDIUM
  TRUCK                      LARGE

<<abstract>> Vehicle
  - licensePlate: string
  - type: VehicleType
  + getSize(): SpotSize       ← each vehicle knows what size it needs

  Motorcycle extends Vehicle  → needs SMALL
  Car        extends Vehicle  → needs MEDIUM
  Truck      extends Vehicle  → needs LARGE

ParkingSpot
  - id: string
  - size: SpotSize
  - vehicle: Vehicle | null   ← null = free
  + isAvailable(): boolean
  + park(vehicle): void
  + removeVehicle(): Vehicle

Level
  - levelNumber: number
  - spots: ParkingSpot[]
  + findSpot(size): ParkingSpot | null   ← finds first available spot of right size
  + freeSpot(spotId): void

<<interface>> PricingStrategy
  + calculateFee(entry: Date, exit: Date, vehicleType): number

  HourlyPricing implements PricingStrategy
  FlatRatePricing implements PricingStrategy

Ticket
  - ticketId: string
  - vehicle: Vehicle
  - spot: ParkingSpot
  - level: Level
  - entryTime: Date
  - exitTime?: Date
  - fee?: number

ParkingLot (Singleton)
  - instance: ParkingLot
  - levels: Level[]
  - activeTickets: Map<ticketId, Ticket>
  - pricingStrategy: PricingStrategy
  + getInstance(): ParkingLot
  + parkVehicle(vehicle): Ticket | null
  + exitVehicle(ticketId): number   ← returns fee
  + getAvailability(): object
```

---

## 4. Full TypeScript Implementation

```typescript
// ============================================================
// ENUMS
// ============================================================

// VehicleType is used to drive pricing and display logic
enum VehicleType {
  MOTORCYCLE = "MOTORCYCLE",
  CAR = "CAR",
  TRUCK = "TRUCK",
}

// SpotSize determines which vehicles can fit where.
// A LARGE spot can hold any vehicle; a SMALL spot only fits a motorcycle.
enum SpotSize {
  SMALL = 1,   // numeric values allow size comparison: vehicle.size <= spot.size
  MEDIUM = 2,
  LARGE = 3,
}

// ============================================================
// VEHICLE HIERARCHY
// ============================================================

// Abstract base — every vehicle knows its type and what spot size it needs.
// This is the "Tell, Don't Ask" principle: the vehicle reports its own size
// rather than external code doing instanceof checks.
abstract class Vehicle {
  constructor(
    public readonly licensePlate: string,
    public readonly type: VehicleType
  ) {}

  // Each subclass declares the minimum spot size it requires
  abstract getRequiredSpotSize(): SpotSize;
}

class Motorcycle extends Vehicle {
  constructor(licensePlate: string) {
    super(licensePlate, VehicleType.MOTORCYCLE);
  }

  getRequiredSpotSize(): SpotSize {
    return SpotSize.SMALL; // motorcycles can fit in any spot, but prefer SMALL
  }
}

class Car extends Vehicle {
  constructor(licensePlate: string) {
    super(licensePlate, VehicleType.CAR);
  }

  getRequiredSpotSize(): SpotSize {
    return SpotSize.MEDIUM;
  }
}

class Truck extends Vehicle {
  constructor(licensePlate: string) {
    super(licensePlate, VehicleType.TRUCK);
  }

  getRequiredSpotSize(): SpotSize {
    return SpotSize.LARGE;
  }
}

// ============================================================
// PARKING SPOT
// ============================================================

class ParkingSpot {
  private vehicle: Vehicle | null = null; // null means the spot is free

  constructor(
    public readonly spotId: string,
    public readonly size: SpotSize,
    public readonly levelNumber: number
  ) {}

  isAvailable(): boolean {
    return this.vehicle === null;
  }

  // Can this spot accommodate the given vehicle?
  // A larger spot can always fit a smaller vehicle.
  canFit(vehicle: Vehicle): boolean {
    return this.isAvailable() && vehicle.getRequiredSpotSize() <= this.size;
  }

  // Park a vehicle — called after spot is selected
  park(vehicle: Vehicle): void {
    if (!this.isAvailable()) {
      throw new Error(`Spot ${this.spotId} is already occupied`);
    }
    this.vehicle = vehicle;
  }

  // Free the spot on exit — returns the vehicle that was parked
  removeVehicle(): Vehicle {
    if (!this.vehicle) {
      throw new Error(`Spot ${this.spotId} is already empty`);
    }
    const parked = this.vehicle;
    this.vehicle = null;
    return parked;
  }

  getVehicle(): Vehicle | null {
    return this.vehicle;
  }
}

// ============================================================
// LEVEL
// ============================================================

// Each level manages its own collection of spots.
// The level is responsible for finding an appropriate spot — SRP.
class Level {
  private spots: ParkingSpot[] = [];

  constructor(public readonly levelNumber: number) {}

  addSpot(spot: ParkingSpot): void {
    this.spots.push(spot);
  }

  // Find the *smallest available spot* that fits the vehicle.
  // This is a greedy strategy: prefer tightest fit to preserve larger spots
  // for vehicles that actually need them (a car shouldn't take a truck spot).
  findSpot(vehicle: Vehicle): ParkingSpot | null {
    const requiredSize = vehicle.getRequiredSpotSize();

    // Filter spots that can fit the vehicle, then sort by size ascending
    // so we always assign the most compact viable spot
    const candidates = this.spots
      .filter((spot) => spot.canFit(vehicle))
      .sort((a, b) => a.size - b.size);

    return candidates.length > 0 ? candidates[0] : null;
  }

  freeSpot(spotId: string): void {
    const spot = this.spots.find((s) => s.spotId === spotId);
    if (!spot) throw new Error(`Spot ${spotId} not found on level ${this.levelNumber}`);
    spot.removeVehicle();
  }

  // Availability summary for this level
  getAvailability(): { total: number; available: number; bySize: Record<string, number> } {
    const bySize: Record<string, number> = { SMALL: 0, MEDIUM: 0, LARGE: 0 };
    let available = 0;

    for (const spot of this.spots) {
      if (spot.isAvailable()) {
        available++;
        // Map SpotSize enum value back to label
        const label = SpotSize[spot.size] as keyof typeof bySize;
        bySize[label]++;
      }
    }

    return { total: this.spots.length, available, bySize };
  }
}

// ============================================================
// TICKET
// ============================================================

// Ticket is a first-class entity — it's the contract between
// the lot and the driver. It holds everything needed for billing.
class Ticket {
  public exitTime?: Date;
  public fee?: number;

  constructor(
    public readonly ticketId: string,
    public readonly vehicle: Vehicle,
    public readonly spot: ParkingSpot,
    public readonly level: Level,
    public readonly entryTime: Date = new Date()
  ) {}

  isActive(): boolean {
    return this.exitTime === undefined;
  }
}

// ============================================================
// PRICING STRATEGY — Strategy Pattern
// ============================================================

// The Strategy interface decouples pricing logic from the ParkingLot.
// New pricing models (surge, weekend rate, EV discount) are added
// by implementing this interface — no changes to ParkingLot needed.
interface PricingStrategy {
  calculateFee(entryTime: Date, exitTime: Date, vehicleType: VehicleType): number;
}

// Hourly pricing with different rates per vehicle type
class HourlyPricingStrategy implements PricingStrategy {
  // Rates per hour per vehicle type (in currency units)
  private readonly rates: Record<VehicleType, number> = {
    [VehicleType.MOTORCYCLE]: 20,
    [VehicleType.CAR]: 40,
    [VehicleType.TRUCK]: 80,
  };

  calculateFee(entryTime: Date, exitTime: Date, vehicleType: VehicleType): number {
    // Calculate duration in hours, rounding UP to the next hour
    const durationMs = exitTime.getTime() - entryTime.getTime();
    const durationHours = Math.ceil(durationMs / (1000 * 60 * 60));

    // Minimum charge of 1 hour even for very short stays
    const billableHours = Math.max(1, durationHours);
    return billableHours * this.rates[vehicleType];
  }
}

// Flat rate — same fee regardless of duration (e.g., event parking)
class FlatRatePricingStrategy implements PricingStrategy {
  private readonly flatFee: number;

  constructor(flatFee: number = 100) {
    this.flatFee = flatFee;
  }

  calculateFee(_entryTime: Date, _exitTime: Date, _vehicleType: VehicleType): number {
    return this.flatFee;
  }
}

// ============================================================
// PARKING LOT — Singleton
// ============================================================

// ParkingLot is the central orchestrator.
// It's a Singleton because there's only one physical lot.
// It delegates spot-finding to Levels and fee calculation to PricingStrategy.
class ParkingLot {
  private static instance: ParkingLot | null = null;

  private levels: Level[] = [];
  private activeTickets: Map<string, Ticket> = new Map();
  private pricingStrategy: PricingStrategy;
  private ticketCounter: number = 1;

  // Private constructor enforces Singleton — no `new ParkingLot()` from outside
  private constructor(pricingStrategy: PricingStrategy = new HourlyPricingStrategy()) {
    this.pricingStrategy = pricingStrategy;
  }

  // Lazy initialization — instance created only on first call
  static getInstance(pricingStrategy?: PricingStrategy): ParkingLot {
    if (!ParkingLot.instance) {
      ParkingLot.instance = new ParkingLot(pricingStrategy);
    }
    return ParkingLot.instance;
  }

  // For testing — reset the singleton between test cases
  static resetInstance(): void {
    ParkingLot.instance = null;
  }

  addLevel(level: Level): void {
    this.levels.push(level);
  }

  // Allow hot-swapping the pricing strategy (e.g., switch to surge pricing at peak hours)
  setPricingStrategy(strategy: PricingStrategy): void {
    this.pricingStrategy = strategy;
  }

  // ── ENTRY ─────────────────────────────────────────────────
  // Attempts to park a vehicle. Returns a Ticket if successful, null if lot is full.
  // In production, this method would be synchronized / use a mutex for thread safety.
  parkVehicle(vehicle: Vehicle): Ticket | null {
    // Scan levels in order — level 0 fills up first (ground floor preference)
    for (const level of this.levels) {
      const spot = level.findSpot(vehicle);

      if (spot) {
        // Park the vehicle in the found spot
        spot.park(vehicle);

        // Generate a ticket and record it
        const ticketId = `TKT-${String(this.ticketCounter++).padStart(6, "0")}`;
        const ticket = new Ticket(ticketId, vehicle, spot, level);
        this.activeTickets.set(ticketId, ticket);

        console.log(
          `[ENTRY] ${vehicle.licensePlate} parked at Level ${level.levelNumber}, Spot ${spot.spotId} | Ticket: ${ticketId}`
        );
        return ticket;
      }
    }

    // No suitable spot found across all levels
    console.warn(`[FULL] No spot available for ${vehicle.type} (${vehicle.licensePlate})`);
    return null;
  }

  // ── EXIT ──────────────────────────────────────────────────
  // Processes vehicle exit. Returns the fee charged.
  exitVehicle(ticketId: string): number {
    const ticket = this.activeTickets.get(ticketId);

    if (!ticket) {
      // Handle lost ticket — charge a penalty or max daily rate
      throw new Error(`Ticket ${ticketId} not found. Lost ticket policy applies.`);
    }

    if (!ticket.isActive()) {
      throw new Error(`Ticket ${ticketId} has already been processed.`);
    }

    // Mark exit time and calculate fee
    ticket.exitTime = new Date();
    ticket.fee = this.pricingStrategy.calculateFee(
      ticket.entryTime,
      ticket.exitTime,
      ticket.vehicle.type
    );

    // Free the spot
    ticket.level.freeSpot(ticket.spot.spotId);

    // Remove from active tickets (move to archived in a real system)
    this.activeTickets.delete(ticketId);

    console.log(
      `[EXIT] ${ticket.vehicle.licensePlate} exited | Duration: ${this.getDurationLabel(ticket)} | Fee: ₹${ticket.fee}`
    );

    return ticket.fee;
  }

  // ── AVAILABILITY ──────────────────────────────────────────
  getAvailability(): void {
    console.log("\n=== Parking Lot Availability ===");
    for (const level of this.levels) {
      const avail = level.getAvailability();
      console.log(
        `Level ${level.levelNumber}: ${avail.available}/${avail.total} available` +
        ` | Small: ${avail.bySize.SMALL}, Medium: ${avail.bySize.MEDIUM}, Large: ${avail.bySize.LARGE}`
      );
    }
    console.log("================================\n");
  }

  // Helper to format duration for display
  private getDurationLabel(ticket: Ticket): string {
    if (!ticket.exitTime) return "N/A";
    const ms = ticket.exitTime.getTime() - ticket.entryTime.getTime();
    const mins = Math.floor(ms / 60000);
    return mins < 60 ? `${mins}m` : `${Math.floor(mins / 60)}h ${mins % 60}m`;
  }
}

// ============================================================
// FACTORY — ParkingLotFactory
// ============================================================

// Factory pattern to build a configured lot without polluting client code
// with construction details. In real scenarios, this reads from a config file.
class ParkingLotFactory {
  static createStandardLot(): ParkingLot {
    const lot = ParkingLot.getInstance(new HourlyPricingStrategy());

    // Level 0: Ground floor — mostly car spots, a few motorcycle & truck
    const level0 = new Level(0);
    for (let i = 1; i <= 10; i++) level0.addSpot(new ParkingSpot(`L0-S${i}`, SpotSize.SMALL, 0));
    for (let i = 1; i <= 30; i++) level0.addSpot(new ParkingSpot(`L0-M${i}`, SpotSize.MEDIUM, 0));
    for (let i = 1; i <= 5; i++)  level0.addSpot(new ParkingSpot(`L0-L${i}`, SpotSize.LARGE, 0));

    // Level 1: Upper floor — overflow
    const level1 = new Level(1);
    for (let i = 1; i <= 5; i++)  level1.addSpot(new ParkingSpot(`L1-S${i}`, SpotSize.SMALL, 1));
    for (let i = 1; i <= 20; i++) level1.addSpot(new ParkingSpot(`L1-M${i}`, SpotSize.MEDIUM, 1));
    for (let i = 1; i <= 3; i++)  level1.addSpot(new ParkingSpot(`L1-L${i}`, SpotSize.LARGE, 1));

    lot.addLevel(level0);
    lot.addLevel(level1);

    return lot;
  }
}

// ============================================================
// DEMO / DRIVER CODE
// ============================================================

function main() {
  // Build the lot using factory
  const lot = ParkingLotFactory.createStandardLot();

  lot.getAvailability();

  // Entry: Three vehicles arrive
  const bike = new Motorcycle("MH-01-AB-1234");
  const car1 = new Car("DL-05-CD-5678");
  const truck = new Truck("KA-09-EF-9999");
  const car2 = new Car("MH-14-GH-4321");

  const bikeTicket  = lot.parkVehicle(bike);
  const car1Ticket  = lot.parkVehicle(car1);
  const truckTicket = lot.parkVehicle(truck);
  const car2Ticket  = lot.parkVehicle(car2);

  lot.getAvailability();

  // Simulate passage of time for billing demo
  if (car1Ticket) {
    // Manually backdate entry time to simulate 2.5 hours ago
    (car1Ticket as any).entryTime = new Date(Date.now() - 2.5 * 60 * 60 * 1000);
  }

  // Exit: Car 1 leaves
  if (car1Ticket) {
    const fee = lot.exitVehicle(car1Ticket.ticketId);
    console.log(`Car 1 paid: ₹${fee}`); // Should be ₹120 (3 hours * ₹40)
  }

  // Switch to flat rate pricing (e.g., for a concert event)
  lot.setPricingStrategy(new FlatRatePricingStrategy(150));

  // Truck exits under new pricing
  if (truckTicket) {
    const fee = lot.exitVehicle(truckTicket.ticketId);
    console.log(`Truck paid: ₹${fee}`); // Should be ₹150 flat
  }

  lot.getAvailability();
}

main();
```

---

## 5. Design Patterns — Deep Dive

### 5.1 Singleton — `ParkingLot`
**Why:** There is exactly one physical parking lot. All gates, all levels, all tickets must share the same state. If two instances existed, they'd have inconsistent availability data.

**Interview gotcha:** Interviewers often ask — *"Is Singleton an anti-pattern?"*  
**Answer:** It can be, because it makes testing hard (global state). That's why we expose `resetInstance()` for tests and inject `PricingStrategy` as a dependency rather than hardcoding it inside.

### 5.2 Strategy — `PricingStrategy`
**Why:** Pricing rules change frequently — time of day, events, vehicle type, membership. The Strategy pattern lets you swap algorithms at runtime without touching `ParkingLot`.

**Extension example:** Add `MembershipPricingStrategy` that gives 20% discount to registered members — zero changes to `ParkingLot`.

### 5.3 Factory — `ParkingLotFactory`
**Why:** Building a `ParkingLot` requires creating levels, spots, and strategies. The factory encapsulates this construction logic. Client code just calls `createStandardLot()`.

### 5.4 Template Method (implicit) — `Vehicle`
`Vehicle` is an abstract class with a concrete interface but an abstract `getRequiredSpotSize()`. Each subclass fills in the template. This avoids `if (vehicle instanceof Car)` checks everywhere.

---

## 6. Edge Cases & Nuances

| Edge Case | How to Handle |
|-----------|--------------|
| **Lot is full** | `parkVehicle()` returns `null` — caller shows "Lot Full" message |
| **Wrong spot assignment** | `canFit()` checks both availability AND size compatibility |
| **Lost ticket** | `exitVehicle()` throws; in real system, charge max daily rate |
| **Double exit** | `isActive()` check on ticket prevents processing twice |
| **Truck in car spot** | `SpotSize` numeric comparison prevents this — `canFit()` returns false |
| **Motorcycle in large spot** | Allowed but suboptimal — `findSpot()` prefers smallest fit to prevent waste |
| **Concurrent entry** | In production: use a mutex/lock around `findSpot()` + `park()` — these two must be atomic |
| **Pricing strategy change mid-day** | `setPricingStrategy()` is live — tickets already issued use the rate at exit time, not entry time |

---

## 7. Thread Safety Note

In a real TypeScript/Node.js server, the `parkVehicle` method would look like:

```typescript
// Using a lock library like 'async-mutex'
import { Mutex } from 'async-mutex';

class ParkingLot {
  private mutex = new Mutex();

  async parkVehicle(vehicle: Vehicle): Promise<Ticket | null> {
    // Acquire lock — only one vehicle can be assigned a spot at a time
    const release = await this.mutex.acquire();
    try {
      // ... same logic as before
    } finally {
      release(); // always release, even if an error occurs
    }
  }
}
```

**Why it matters:** Without a lock, two cars could read the same spot as available, both try to park there, and one would overwrite the other.

---

## 8. Follow-Up Questions Interviewers Ask

1. **"How would you handle EV charging spots?"**  
   → Add `SpotType` enum (REGULAR, EV_CHARGING). `ParkingSpot` gets a `spotType` field. `EVCar` overrides `getRequiredSpotSize()` and also declares a required spot type.

2. **"How would you support monthly passes?"**  
   → Add a `Pass` entity with expiry. `ExitGate` checks if vehicle has a valid pass before calling `pricingStrategy`.

3. **"How would you design for 1000 entry/exit gates?"**  
   → The lot itself is stateless per-request once the spot map is locked. Use a distributed lock (Redis SETNX) and a shared spot availability store.

4. **"How would you add a waitlist when lot is full?"**  
   → Observer pattern: vehicles register as observers. When a spot is freed (`removeVehicle()`), the lot notifies the first waiter.

5. **"How would you add analytics?"**  
   → Observer on `Ticket` creation and exit. Analytics service subscribes and receives events asynchronously.

---

## 9. Key Takeaways

- Start with **requirements** — don't jump to classes. Interviewers reward this.
- Model **real-world entities** as classes, not data bags. `Vehicle` knows its size, `ParkingSpot` knows if it can fit a vehicle.
- Apply **Strategy** for anything that changes independently of the core domain (pricing, scheduling, assignment).
- **Singleton** is fine for true one-instance concepts but always inject dependencies so tests can swap them.
- The **"tightest fit" assignment** is a deliberate optimization — call it out in the interview.
- Always mention **concurrency** even if you don't implement it fully — it signals senior thinking.

---

*Next up: LLD #02 — Elevator System*