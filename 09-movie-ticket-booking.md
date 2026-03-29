# LLD #03 — Movie Ticket Booking System (BookMyShow)

> **Language:** TypeScript  
> **Difficulty:** ⭐⭐⭐⭐ (Senior)  
> **Patterns Used:** Strategy, Observer, Singleton, Factory, Template Method  
> **Core Challenge:** Concurrency — preventing double-booking under race conditions

---

## 1. Requirements

### Functional Requirements
- Browse movies, cinemas, and shows
- View seat layout for a show; seats have categories (RECLINER, PREMIUM, REGULAR)
- Select seats → temporarily hold them for a limited time (10 min TTL)
- Complete payment to confirm booking
- On payment failure or TTL expiry → seats released back to AVAILABLE
- Generate a ticket (with QR/booking ID) on successful booking
- Apply discount coupons at checkout
- Support cancellations with a refund policy

### Non-Functional Requirements
- **Concurrency-safe seat booking** — two users cannot book the same seat
- Seat hold TTL must auto-expire (background job or TTL mechanism)
- Seat availability must be real-time — stale reads lead to bad UX
- Payment gateway is external — system must handle gateway failures gracefully

### What interviewers watch for
- Do you model the **seat lifecycle** explicitly with statuses?
- Do you know the **CAS (Compare-And-Swap)** pattern for atomic seat locking?
- Do you understand why **optimistic vs pessimistic locking** matters here?
- Do you handle TTL expiry — the silent race that everyone forgets?
- Do you separate **booking flow** from **payment** correctly?

---

## 2. The Core Challenge — Concurrency & Seat Locking

### The Race Condition

```
Time 0:   User A and User B both see seat A5 as AVAILABLE
Time 1ms: User A requests hold on A5
Time 1ms: User B requests hold on A5 (simultaneous)
Time 2ms: Both succeed ← DOUBLE BOOKING. System is broken.
```

### Solution: Compare-And-Swap (CAS) at the DB layer

The only reliable fix is an **atomic conditional update**:

```sql
-- This SQL must execute as a single atomic statement
UPDATE seats
SET status = 'HELD', held_by = :userId, hold_expires_at = NOW() + INTERVAL 10 MINUTE
WHERE show_id = :showId
  AND seat_id = :seatId
  AND status = 'AVAILABLE'   -- ← the CAS condition
```

If **0 rows affected** → someone else grabbed it first. Return error to the user.  
If **1 row affected** → hold confirmed. Proceed to payment.

### Optimistic vs Pessimistic Locking

| Strategy | Mechanism | Use When |
|----------|-----------|----------|
| **Pessimistic** | `SELECT ... FOR UPDATE` (row lock) | High contention, short transactions |
| **Optimistic** | CAS with version column | Low contention, longer transactions |
| **Application-level lock** | Redis `SETNX` with TTL | Distributed systems, cache-first |

For BookMyShow:
- For **popular shows** (IPL, new blockbuster): pessimistic or Redis lock — many users hammering the same seats
- For **off-peak shows**: optimistic CAS is sufficient and cheaper

We implement **CAS** in our model (the most interview-relevant pattern).

---

## 3. Entities & Relationships

```
Movie
  └── Show[]  (a movie can have multiple shows across cinemas)

Cinema
  └── Screen[]
        └── Seat[]   (physical seats, never change)

Show  (a Movie playing at a Screen at a DateTime)
  └── ShowSeat[]  (SeatStatus snapshot per show — resets for each show)
        └── SeatStatus: AVAILABLE | HELD | BOOKED

Booking
  └── User
  └── Show
  └── BookingSeat[]  (which seats were booked)
  └── Payment
  └── BookingStatus: PENDING | CONFIRMED | CANCELLED | FAILED

Payment
  └── amount, method, gatewayRef
  └── PaymentStatus: PENDING | SUCCESS | FAILED | REFUNDED

Coupon
  └── DiscountStrategy (Strategy Pattern for discount calculation)
```

### Key Design Decision: `Seat` vs `ShowSeat`

This is the most common mistake candidates make — conflating physical seats with per-show seat availability.

- `Seat` — a physical row/column in a screen. It exists forever. It has a category (RECLINER, etc.).
- `ShowSeat` — the status of that seat **for one specific show**. When a show ends, these reset. A seat can be AVAILABLE for the 3 PM show and BOOKED for the 6 PM show simultaneously.

---

## 4. Seat Status Lifecycle

```
AVAILABLE → HELD      (atomic CAS — only one user wins)
HELD      → BOOKED    (payment confirmed)
HELD      → AVAILABLE (TTL expired OR payment failed)
BOOKED    → AVAILABLE (cancellation with refund)
```

**The TTL release** is handled by a background job (or Redis key expiry callback) that runs periodically:

```sql
UPDATE show_seats
SET status = 'AVAILABLE', held_by = NULL, hold_expires_at = NULL
WHERE status = 'HELD' AND hold_expires_at < NOW()
```

---

## 5. Full TypeScript Implementation

```typescript
// ============================================================
// ENUMS
// ============================================================

enum SeatCategory {
  RECLINER = "RECLINER",   // Premium tier — highest price
  PREMIUM = "PREMIUM",     // Mid tier
  REGULAR = "REGULAR",     // Economy tier
}

// The critical status enum — seat availability is per-show, not per-seat
enum SeatStatus {
  AVAILABLE = "AVAILABLE",
  HELD = "HELD",       // Temporarily locked for a user — has a TTL
  BOOKED = "BOOKED",   // Confirmed booking
}

enum BookingStatus {
  PENDING = "PENDING",       // Hold placed, awaiting payment
  CONFIRMED = "CONFIRMED",   // Payment successful, ticket issued
  CANCELLED = "CANCELLED",   // User cancelled
  FAILED = "FAILED",         // Payment failed
}

enum PaymentStatus {
  PENDING = "PENDING",
  SUCCESS = "SUCCESS",
  FAILED = "FAILED",
  REFUNDED = "REFUNDED",
}

enum PaymentMethod {
  CREDIT_CARD = "CREDIT_CARD",
  UPI = "UPI",
  NET_BANKING = "NET_BANKING",
  WALLET = "WALLET",
}

// ============================================================
// CORE DOMAIN ENTITIES
// ============================================================

// A physical seat in a screen — exists independently of any show
class Seat {
  constructor(
    public readonly seatId: string,
    public readonly row: string,       // e.g. "A", "B", "C"
    public readonly column: number,    // e.g. 1, 2, 3
    public readonly category: SeatCategory,
    public readonly screenId: string
  ) {}

  // Human-readable label like "A5"
  getLabel(): string {
    return `${this.row}${this.column}`;
  }
}

// A physical screen inside a cinema — contains seats
class Screen {
  private seats: Seat[] = [];

  constructor(
    public readonly screenId: string,
    public readonly cinemaId: string,
    public readonly name: string,        // e.g. "Audi 1"
    public readonly totalCapacity: number
  ) {}

  addSeat(seat: Seat): void {
    this.seats.push(seat);
  }

  getSeats(): Seat[] {
    return [...this.seats];
  }
}

// A cinema/multiplex with multiple screens
class Cinema {
  private screens: Screen[] = [];

  constructor(
    public readonly cinemaId: string,
    public readonly name: string,       // e.g. "PVR Juhu"
    public readonly city: string,
    public readonly address: string
  ) {}

  addScreen(screen: Screen): void {
    this.screens.push(screen);
  }

  getScreens(): Screen[] {
    return [...this.screens];
  }
}

// A movie — contains metadata only
class Movie {
  constructor(
    public readonly movieId: string,
    public readonly title: string,
    public readonly genre: string[],
    public readonly durationMinutes: number,
    public readonly language: string,
    public readonly rating: string        // e.g. "U/A", "A"
  ) {}
}

// A show = a Movie playing on a Screen at a specific time
// This is the booking unit — users book seats for a Show
class Show {
  private showSeats: Map<string, ShowSeat> = new Map(); // seatId → ShowSeat

  constructor(
    public readonly showId: string,
    public readonly movie: Movie,
    public readonly screen: Screen,
    public readonly startTime: Date,
    public readonly endTime: Date,
    public readonly priceByCategory: Map<SeatCategory, number> // base prices
  ) {
    // Initialize all ShowSeats from the screen's physical seats
    for (const seat of screen.getSeats()) {
      const price = priceByCategory.get(seat.category) ?? 0;
      this.showSeats.set(seat.seatId, new ShowSeat(seat, this.showId, price));
    }
  }

  getShowSeat(seatId: string): ShowSeat | undefined {
    return this.showSeats.get(seatId);
  }

  getAvailableSeats(): ShowSeat[] {
    return Array.from(this.showSeats.values()).filter(
      (ss) => ss.status === SeatStatus.AVAILABLE
    );
  }

  // Returns all show seats grouped by category for the seat map UI
  getSeatMap(): Map<SeatCategory, ShowSeat[]> {
    const map = new Map<SeatCategory, ShowSeat[]>();
    for (const ss of this.showSeats.values()) {
      const category = ss.seat.category;
      if (!map.has(category)) map.set(category, []);
      map.get(category)!.push(ss);
    }
    return map;
  }
}

// ============================================================
// SHOW SEAT — the heart of the concurrency problem
// ============================================================

// ShowSeat represents a single seat's status for a specific show.
// This is NOT the physical seat — it's an availability record.
// One Seat can have many ShowSeats across different shows.
class ShowSeat {
  public status: SeatStatus = SeatStatus.AVAILABLE;
  public heldByUserId: string | null = null;
  public holdExpiresAt: Date | null = null;

  constructor(
    public readonly seat: Seat,
    public readonly showId: string,
    public readonly basePrice: number
  ) {}

  // ── CAS HOLD ────────────────────────────────────────────────
  // This is the critical section. In a real system this is a single
  // atomic DB UPDATE with WHERE status = 'AVAILABLE'.
  // Here we simulate it with an in-memory check-and-set.
  // The return value signals whether the hold was granted.
  tryHold(userId: string, ttlMinutes: number = 10): boolean {
    // The CAS condition: only transition if currently AVAILABLE
    if (this.status !== SeatStatus.AVAILABLE) {
      // In a real DB: UPDATE affected 0 rows → hold denied
      return false;
    }

    // Atomically transition to HELD
    // In DB: UPDATE show_seats SET status='HELD', held_by=:userId,
    //        hold_expires_at=NOW()+INTERVAL :ttl MINUTE
    //        WHERE show_id=:showId AND seat_id=:seatId AND status='AVAILABLE'
    this.status = SeatStatus.HELD;
    this.heldByUserId = userId;
    this.holdExpiresAt = new Date(Date.now() + ttlMinutes * 60 * 1000);
    return true;
  }

  // Confirm the seat after successful payment
  confirm(): void {
    if (this.status !== SeatStatus.HELD) {
      throw new Error(`Cannot confirm seat ${this.seat.seatId} — not in HELD state`);
    }
    this.status = SeatStatus.BOOKED;
    this.heldByUserId = null;
    this.holdExpiresAt = null;
  }

  // Release the hold — called on payment failure or TTL expiry
  release(): void {
    if (this.status === SeatStatus.BOOKED) {
      throw new Error(`Cannot release a BOOKED seat — must cancel booking first`);
    }
    this.status = SeatStatus.AVAILABLE;
    this.heldByUserId = null;
    this.holdExpiresAt = null;
  }

  // Cancel a confirmed booking (e.g. user-initiated or show cancelled)
  cancel(): void {
    if (this.status !== SeatStatus.BOOKED) {
      throw new Error(`Seat ${this.seat.seatId} is not BOOKED — cannot cancel`);
    }
    this.status = SeatStatus.AVAILABLE;
  }

  // Has the hold expired? Used by the TTL cleanup job
  isHoldExpired(): boolean {
    return (
      this.status === SeatStatus.HELD &&
      this.holdExpiresAt !== null &&
      this.holdExpiresAt < new Date()
    );
  }
}

// ============================================================
// DISCOUNT STRATEGY — Strategy Pattern
// ============================================================

// Discounts are an independently varying concern — new offer types
// (festival, membership, bank cashback) are added by implementing this
// interface without changing the booking flow.
interface DiscountStrategy {
  apply(originalAmount: number): number;
  getDescription(): string;
}

// Percentage-based discount (most common — "20% off")
class PercentageDiscount implements DiscountStrategy {
  constructor(private readonly percentage: number) {}

  apply(amount: number): number {
    return amount * (1 - this.percentage / 100);
  }

  getDescription(): string {
    return `${this.percentage}% off`;
  }
}

// Flat amount discount ("₹100 off")
class FlatDiscount implements DiscountStrategy {
  constructor(private readonly amount: number) {}

  apply(originalAmount: number): number {
    return Math.max(0, originalAmount - this.amount); // floor at 0
  }

  getDescription(): string {
    return `₹${this.amount} off`;
  }
}

// Buy-one-get-one: second ticket is free
class Bogo2Discount implements DiscountStrategy {
  apply(originalAmount: number): number {
    // Halve the total (effectively one ticket free)
    // Real impl would need ticket count — simplified here
    return originalAmount / 2;
  }

  getDescription(): string {
    return "Buy 1 Get 1 Free";
  }
}

// A coupon wraps a strategy + validity metadata
class Coupon {
  constructor(
    public readonly code: string,
    public readonly strategy: DiscountStrategy,
    public readonly validUntil: Date,
    public readonly minOrderValue: number = 0
  ) {}

  isValid(orderAmount: number): boolean {
    return new Date() <= this.validUntil && orderAmount >= this.minOrderValue;
  }

  applyDiscount(amount: number): number {
    return this.strategy.apply(amount);
  }
}

// ============================================================
// PAYMENT GATEWAY — Strategy Pattern
// ============================================================

// Swappable payment providers (Razorpay, Stripe, PayU)
interface PaymentGateway {
  processPayment(amount: number, method: PaymentMethod): Promise<{ success: boolean; gatewayRef: string }>;
  refund(gatewayRef: string, amount: number): Promise<boolean>;
}

// Simulated gateway for demo — real impl would hit external API
class SimulatedPaymentGateway implements PaymentGateway {
  async processPayment(
    amount: number,
    _method: PaymentMethod
  ): Promise<{ success: boolean; gatewayRef: string }> {
    // Simulate ~90% success rate
    const success = Math.random() > 0.1;
    const gatewayRef = `TXN-${Date.now()}-${Math.random().toString(36).slice(2, 8).toUpperCase()}`;
    console.log(`[PAYMENT] ₹${amount} — ${success ? "SUCCESS" : "FAILED"} (ref: ${gatewayRef})`);
    return { success, gatewayRef };
  }

  async refund(gatewayRef: string, amount: number): Promise<boolean> {
    console.log(`[REFUND] ₹${amount} for txn ${gatewayRef}`);
    return true; // Always succeeds in simulation
  }
}

// ============================================================
// PAYMENT
// ============================================================

class Payment {
  public status: PaymentStatus = PaymentStatus.PENDING;
  public gatewayRef: string | null = null;

  constructor(
    public readonly paymentId: string,
    public readonly bookingId: string,
    public readonly amount: number,
    public readonly method: PaymentMethod,
    public readonly createdAt: Date = new Date()
  ) {}

  markSuccess(gatewayRef: string): void {
    this.status = PaymentStatus.SUCCESS;
    this.gatewayRef = gatewayRef;
  }

  markFailed(): void {
    this.status = PaymentStatus.FAILED;
  }

  markRefunded(): void {
    this.status = PaymentStatus.REFUNDED;
  }
}

// ============================================================
// BOOKING
// ============================================================

// Booking is the central transaction entity.
// It ties together: User + Show + Seats + Payment + Coupon
class Booking {
  public status: BookingStatus = BookingStatus.PENDING;
  public payment: Payment | null = null;
  public coupon: Coupon | null = null;
  public finalAmount: number;

  constructor(
    public readonly bookingId: string,
    public readonly userId: string,
    public readonly show: Show,
    public readonly showSeats: ShowSeat[],  // the seats being booked
    public readonly createdAt: Date = new Date()
  ) {
    // Compute base amount from sum of seat prices
    this.finalAmount = showSeats.reduce((sum, ss) => sum + ss.basePrice, 0);
  }

  // Apply a coupon — adjusts finalAmount via strategy
  applyCoupon(coupon: Coupon): void {
    if (!coupon.isValid(this.finalAmount)) {
      throw new Error(`Coupon ${coupon.code} is not valid for this order`);
    }
    this.coupon = coupon;
    this.finalAmount = coupon.applyDiscount(this.finalAmount);
    console.log(
      `[COUPON] ${coupon.code} applied (${coupon.strategy.getDescription()}) → ₹${this.finalAmount.toFixed(2)}`
    );
  }

  confirm(payment: Payment): void {
    this.status = BookingStatus.CONFIRMED;
    this.payment = payment;
    // Confirm all ShowSeats → BOOKED
    for (const ss of this.showSeats) {
      ss.confirm();
    }
  }

  fail(payment: Payment): void {
    this.status = BookingStatus.FAILED;
    this.payment = payment;
    // Release all held seats back to AVAILABLE
    for (const ss of this.showSeats) {
      ss.release();
    }
  }

  cancel(): void {
    if (this.status !== BookingStatus.CONFIRMED) {
      throw new Error(`Cannot cancel a ${this.status} booking`);
    }
    this.status = BookingStatus.CANCELLED;
    // Release all booked seats
    for (const ss of this.showSeats) {
      ss.cancel();
    }
  }

  generateTicketId(): string {
    // Format: BMS-<bookingId>-<showId>
    return `BMS-${this.bookingId}-${this.show.showId}`;
  }
}

// ============================================================
// OBSERVER — Booking Event Notifications
// ============================================================

// BookingObserver lets other systems react to booking events
// without coupling them to the BookingService.
// Use cases: email confirmation, analytics, loyalty points, seat map update
interface BookingObserver {
  onBookingConfirmed(booking: Booking): void;
  onBookingFailed(booking: Booking): void;
  onBookingCancelled(booking: Booking): void;
}

// Example: Email notification observer
class EmailNotificationObserver implements BookingObserver {
  onBookingConfirmed(booking: Booking): void {
    console.log(
      `[EMAIL] Booking confirmed! Ticket ${booking.generateTicketId()} sent to user ${booking.userId}`
    );
  }

  onBookingFailed(booking: Booking): void {
    console.log(`[EMAIL] Payment failed for booking ${booking.bookingId}. Seats released.`);
  }

  onBookingCancelled(booking: Booking): void {
    console.log(`[EMAIL] Booking ${booking.bookingId} cancelled. Refund initiated.`);
  }
}

// Example: Analytics observer
class AnalyticsObserver implements BookingObserver {
  onBookingConfirmed(booking: Booking): void {
    console.log(`[ANALYTICS] Revenue: ₹${booking.finalAmount} | Movie: ${booking.show.movie.title}`);
  }

  onBookingFailed(_booking: Booking): void {}

  onBookingCancelled(booking: Booking): void {
    console.log(`[ANALYTICS] Cancellation logged for booking ${booking.bookingId}`);
  }
}

// ============================================================
// BOOKING SERVICE — Orchestrator
// ============================================================

// BookingService is the main application service.
// It coordinates: seat selection → hold → payment → confirm/fail.
// It is NOT a Singleton — it's a stateless service (state lives in entities).
class BookingService {
  private observers: BookingObserver[] = [];
  private bookingCounter = 1;
  private paymentCounter = 1;

  constructor(private readonly paymentGateway: PaymentGateway) {}

  addObserver(observer: BookingObserver): void {
    this.observers.push(observer);
  }

  // ── STEP 1: HOLD SEATS ───────────────────────────────────────
  // Atomically hold the requested seats. If ANY seat fails to hold,
  // we roll back all previously held seats in this request — all-or-nothing.
  holdSeats(userId: string, show: Show, seatIds: string[]): Booking | null {
    const showSeats: ShowSeat[] = [];
    const heldSeats: ShowSeat[] = []; // track for rollback

    for (const seatId of seatIds) {
      const showSeat = show.getShowSeat(seatId);
      if (!showSeat) {
        // Rollback: release all seats we already held in this loop
        heldSeats.forEach((ss) => ss.release());
        throw new Error(`Seat ${seatId} not found for this show`);
      }

      // THE CAS — only one user wins per seat
      const granted = showSeat.tryHold(userId);
      if (!granted) {
        // This seat was grabbed by another user — rollback and fail fast
        heldSeats.forEach((ss) => ss.release());
        console.warn(`[HOLD FAILED] Seat ${seatId} is no longer available`);
        return null;
      }

      heldSeats.push(showSeat);
      showSeats.push(showSeat);
    }

    // All seats successfully held — create the booking in PENDING state
    const bookingId = `BKG-${String(this.bookingCounter++).padStart(6, "0")}`;
    const booking = new Booking(bookingId, userId, show, showSeats);

    console.log(
      `[HOLD] Booking ${bookingId} created for ${showSeats.length} seat(s) | ₹${booking.finalAmount}`
    );
    return booking;
  }

  // ── STEP 2 (OPTIONAL): APPLY COUPON ─────────────────────────
  applyCoupon(booking: Booking, coupon: Coupon): void {
    booking.applyCoupon(coupon);
  }

  // ── STEP 3: PROCESS PAYMENT ──────────────────────────────────
  // Payment is made AFTER the hold, not before.
  // If payment fails, we release the hold immediately.
  async processPayment(booking: Booking, method: PaymentMethod): Promise<boolean> {
    const paymentId = `PAY-${String(this.paymentCounter++).padStart(6, "0")}`;
    const payment = new Payment(paymentId, booking.bookingId, booking.finalAmount, method);

    try {
      const result = await this.paymentGateway.processPayment(booking.finalAmount, method);

      if (result.success) {
        payment.markSuccess(result.gatewayRef);
        booking.confirm(payment);
        this.notifyObservers("confirmed", booking);
        console.log(`[BOOKING] ${booking.bookingId} CONFIRMED | Ticket: ${booking.generateTicketId()}`);
        return true;
      } else {
        payment.markFailed();
        booking.fail(payment); // This releases the held seats
        this.notifyObservers("failed", booking);
        return false;
      }
    } catch (err) {
      // Network error, gateway timeout, etc.
      // Still release the hold — don't leave seats stuck in HELD
      payment.markFailed();
      booking.fail(payment);
      this.notifyObservers("failed", booking);
      console.error(`[PAYMENT ERROR] Gateway exception:`, err);
      return false;
    }
  }

  // ── STEP 4 (OPTIONAL): CANCEL BOOKING ───────────────────────
  async cancelBooking(booking: Booking): Promise<void> {
    if (booking.status !== BookingStatus.CONFIRMED) {
      throw new Error(`Cannot cancel booking in state: ${booking.status}`);
    }

    booking.cancel();

    // Initiate refund via payment gateway
    if (booking.payment?.gatewayRef) {
      const refunded = await this.paymentGateway.refund(
        booking.payment.gatewayRef,
        booking.finalAmount
      );
      if (refunded) {
        booking.payment.markRefunded();
      }
    }

    this.notifyObservers("cancelled", booking);
    console.log(`[CANCEL] Booking ${booking.bookingId} cancelled. Seats released.`);
  }

  private notifyObservers(event: "confirmed" | "failed" | "cancelled", booking: Booking): void {
    for (const obs of this.observers) {
      if (event === "confirmed") obs.onBookingConfirmed(booking);
      else if (event === "failed") obs.onBookingFailed(booking);
      else obs.onBookingCancelled(booking);
    }
  }
}

// ============================================================
// TTL EXPIRY BACKGROUND JOB
// ============================================================

// In production, this runs as a scheduled job (cron) or is triggered
// by Redis key-expiry events. It prevents seats from being stuck in
// HELD state when users abandon the checkout page.
class SeatHoldExpiryJob {
  constructor(private readonly show: Show) {}

  // Scan all ShowSeats and release expired holds
  run(): void {
    let released = 0;

    // In a real DB: UPDATE show_seats SET status='AVAILABLE'
    //               WHERE status='HELD' AND hold_expires_at < NOW()
    for (const showSeat of this.getHeldSeats()) {
      if (showSeat.isHoldExpired()) {
        console.log(
          `[TTL] Releasing expired hold on seat ${showSeat.seat.getLabel()} for show ${showSeat.showId}`
        );
        showSeat.release();
        released++;
      }
    }

    if (released > 0) {
      console.log(`[TTL JOB] Released ${released} expired seat hold(s)`);
    }
  }

  private getHeldSeats(): ShowSeat[] {
    // In real system: query DB for HELD seats with expired TTL
    // Here we extract from the show's internal map (not ideal — for demo only)
    return this.show.getAvailableSeats(); // placeholder — real impl queries DB
  }
}

// ============================================================
// BOOKING FACADE — Public API
// ============================================================

// Facade simplifies the multi-step booking flow for callers.
// Without it, every client would need to know: holdSeats → applyCoupon → processPayment.
class BookingFacade {
  private service: BookingService;

  constructor() {
    const gateway = new SimulatedPaymentGateway();
    this.service = new BookingService(gateway);

    // Register observers
    this.service.addObserver(new EmailNotificationObserver());
    this.service.addObserver(new AnalyticsObserver());
  }

  // One-shot booking flow — the only method most callers need
  async bookSeats(
    userId: string,
    show: Show,
    seatIds: string[],
    paymentMethod: PaymentMethod,
    coupon?: Coupon
  ): Promise<{ success: boolean; ticketId?: string; error?: string }> {
    // Step 1: Hold seats (CAS)
    const booking = this.service.holdSeats(userId, show, seatIds);
    if (!booking) {
      return { success: false, error: "One or more seats are no longer available" };
    }

    // Step 2: Apply coupon if provided
    if (coupon) {
      try {
        this.service.applyCoupon(booking, coupon);
      } catch (err: any) {
        // Coupon failure is non-fatal — proceed without discount
        console.warn(`[COUPON] ${err.message} — proceeding without discount`);
      }
    }

    // Step 3: Process payment
    const paid = await this.service.processPayment(booking, paymentMethod);
    if (!paid) {
      return { success: false, error: "Payment failed. Please try again." };
    }

    return { success: true, ticketId: booking.generateTicketId() };
  }
}

// ============================================================
// DEMO
// ============================================================

async function main() {
  // Build the domain
  const movie = new Movie("M001", "Kalki 2898 AD", ["Sci-Fi", "Action"], 180, "Telugu", "U/A");

  const screen = new Screen("SCR001", "CIN001", "Audi 1", 100);
  // Add some seats
  const priceMap = new Map<SeatCategory, number>([
    [SeatCategory.RECLINER, 600],
    [SeatCategory.PREMIUM, 350],
    [SeatCategory.REGULAR, 200],
  ]);
  ["A1", "A2", "A3"].forEach((id, i) =>
    screen.addSeat(new Seat(id, "A", i + 1, SeatCategory.RECLINER, "SCR001"))
  );
  ["B1", "B2", "B3"].forEach((id, i) =>
    screen.addSeat(new Seat(id, "B", i + 1, SeatCategory.PREMIUM, "SCR001"))
  );

  const showTime = new Date("2025-01-15T18:00:00");
  const show = new Show("SH001", movie, screen, showTime, new Date("2025-01-15T21:00:00"), priceMap);

  const facade = new BookingFacade();

  // Coupon: 20% off
  const coupon = new Coupon(
    "KALKI20",
    new PercentageDiscount(20),
    new Date("2025-12-31"),
    200
  );

  console.log("\n=== Scenario 1: Normal booking with coupon ===");
  const result1 = await facade.bookSeats("USER001", show, ["A1", "A2"], PaymentMethod.UPI, coupon);
  console.log("Result:", result1);

  console.log("\n=== Scenario 2: Race condition — User B tries to book A1 (already booked) ===");
  const result2 = await facade.bookSeats("USER002", show, ["A1", "B1"], PaymentMethod.CREDIT_CARD);
  console.log("Result:", result2);

  console.log("\n=== Scenario 3: Available seats only ===");
  const result3 = await facade.bookSeats("USER002", show, ["B1", "B2"], PaymentMethod.UPI);
  console.log("Result:", result3);

  // Check availability
  console.log(`\nAvailable seats: ${show.getAvailableSeats().length}`);
}

main().catch(console.error);
```

---

## 6. Design Patterns — Deep Dive

### 6.1 CAS (Compare-And-Swap) — Not a pattern, a primitive
This is the mechanism, not the pattern. `tryHold()` is an atomic check-and-set. In a real DB it maps to a single `UPDATE ... WHERE status = 'AVAILABLE'`. The affected row count tells you whether you won or lost the race. This is the **most important concept in this entire LLD**.

### 6.2 Strategy — `DiscountStrategy` & `PaymentGateway`
Two independent Strategy hierarchies. Discounts change based on offers; payment providers change based on business. Neither affects booking flow logic. Interviewers love seeing you identify two independent strategy axes in one system.

### 6.3 Observer — `BookingObserver`
Email, analytics, loyalty points, and seat-map push updates all need to react to booking events. Observer decouples them completely. The booking service fires events; observers do their own thing. Adding a new observer (e.g., WhatsApp notifications) is zero-change to the service.

### 6.4 Facade — `BookingFacade`
The multi-step flow (hold → coupon → payment) is complex. The Facade exposes one clean `bookSeats()` method. Client code (API controllers) doesn't need to know the sequence.

### 6.5 Template Method (implicit) — Booking flow sequence
The sequence hold → coupon → payment → confirm/fail is a fixed template. Only the payment step varies (different gateways). This is Template Method in spirit even without a formal abstract class.

---

## 7. The `Seat` vs `ShowSeat` Distinction — Interview Differentiator

This is the single most common mistake candidates make:

```
❌ Wrong model:
   Seat has a `status` field → AVAILABLE / BOOKED
   Problem: A seat in Screen A can only be in one state globally.
            The 3PM show and 6PM show share the same status.

✅ Correct model:
   Seat — physical, stateless, permanent (row, column, category)
   ShowSeat — status record per (Show, Seat) pair
   The 3PM ShowSeat for A1 can be BOOKED
   The 6PM ShowSeat for A1 is AVAILABLE
   These are independent records.
```

The DB table for `show_seats` has the composite primary key: `(show_id, seat_id)`.

---

## 8. Optimistic vs Pessimistic Locking — When to Use What

### Pessimistic Locking (DB Row Lock)
```sql
BEGIN TRANSACTION;
SELECT * FROM show_seats
WHERE show_id = ? AND seat_id = ?
FOR UPDATE;              -- ← locks the row. Other transactions wait.

UPDATE show_seats SET status = 'HELD' WHERE ...;
COMMIT;
```
- Pro: Guaranteed no conflict
- Con: Row lock holds for entire transaction duration; poor throughput under high load (e.g., Avengers opening night)

### Optimistic Locking (CAS / Version Column)
```sql
UPDATE show_seats
SET status = 'HELD', version = version + 1
WHERE show_id = ? AND seat_id = ? AND status = 'AVAILABLE' AND version = ?
```
- Pro: No locking, high throughput
- Con: Requires retry logic on conflict; bad for very hot seats (many retries)

### Redis-based Distributed Lock
```
SETNX seat:{showId}:{seatId} {userId} EX 600
-- Returns 1 = lock granted, 0 = already locked
```
- Pro: Works across multiple app servers; TTL is native; O(1)
- Con: Cache and DB can go out of sync if Redis crashes after lock granted but before DB write

**Interview answer:** Use CAS for most cases. Add Redis distributed locking for premiere shows with thousands of concurrent users.

---

## 9. Edge Cases & Nuances

| Edge Case | How to Handle |
|-----------|--------------|
| **Two users request same seat** | CAS — only 1 DB row update succeeds; other gets 0 affected rows → error |
| **Hold expires while user is paying** | TTL job releases the seat; payment may still succeed → conflict! Solution: check hold validity before confirming payment |
| **Payment gateway timeout** | Treat as failure; release hold. Use idempotency keys to prevent double-charge on retry |
| **Partial seat selection fails** | All-or-nothing hold: roll back all seats in current request on first failure |
| **User tries to book same seat twice** | `tryHold()` returns false if already HELD (by same or different user) |
| **Show cancelled** | Broadcast `ShowCancelledEvent`; release all HOLDs and BOOKINGs; trigger refunds |
| **Coupon used twice** | Coupon entity tracks `usageCount` and `maxUsage`; validate at service layer |
| **Seat hold during high traffic (flash sale)** | Redis lock + queue-based seat assignment; rate limit per user |
| **Network failure after payment success** | Payment gateway webhook confirms success; idempotent booking confirmation |

---

## 10. Follow-Up Questions Interviewers Ask

1. **"How do you handle 50,000 concurrent users for a Coldplay concert?"**  
   → Virtual queue (fair queue, not first-click-wins). Rate limit seat hold requests. Redis SETNX as the atomic lock. DB write happens after Redis lock granted. Horizontal scaling of the booking service.

2. **"How do you prevent the seat hold from expiring during payment?"**  
   → Before confirming payment, re-check `hold_expires_at > NOW()`. If expired: abort, return error, let user retry. Alternatively, extend the TTL when payment is initiated (user clicked "Pay Now").

3. **"How would you design the seat map for real-time availability?"**  
   → WebSocket push updates when seat status changes. Or Server-Sent Events. Client subscribes to `show:{showId}:seats`. On status change event, update seat color in UI.

4. **"How would you add group/family booking?"**  
   → `Booking` already holds `showSeats[]` — it's naturally multi-seat. Add `minGroupSize` / `maxGroupSize` constraints in `BookingService.holdSeats()`.

5. **"How would you model dynamic pricing (surge pricing)?"**  
   → `ShowSeat.basePrice` becomes dynamic: a `PricingService` computes price based on demand (seats sold / total) + time to show. Add a `PricingStrategy` interface. Price is locked at the time the hold is created, not at payment time.

---

## 11. Key Takeaways

- **CAS is the answer to the race condition** — not locks, not application-level checks. A single atomic `UPDATE ... WHERE status = 'AVAILABLE'` is the source of truth.
- **`Seat` ≠ `ShowSeat`** — this distinction is worth 30% of the interview score by itself. Physical seats are permanent; availability is per-show.
- **All-or-nothing hold** — if you're booking 4 seats and seat 3 fails, release seats 1 and 2 immediately. Never leave partial holds.
- **TTL release is mandatory** — without it, abandoned checkouts permanently block seats. Mention the background job or Redis TTL approach.
- **Observer for post-booking events** — email, analytics, loyalty points should never be in the booking transaction path. Fire and forget via observer.
- **Payment failure releases the hold** — the booking service is responsible for cleanup. Never rely on the client to clean up.

---

*Next up: LLD #04 — LRU Cache System*