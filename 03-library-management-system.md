# LLD #05 — Library Management System

> **Language:** TypeScript
> **Difficulty:** ⭐⭐⭐ (Intermediate–Senior)
> **Patterns Used:** State, Strategy, Observer, Factory, Iterator
> **Core Challenge:** `Book` vs `BookItem` distinction + Loan state machine + Reservation queue

---

## 1. Requirements

### Functional Requirements
- Catalogue management: add/remove/search books (by title, author, ISBN, genre)
- Member management: register members, manage library cards
- Checkout (issue) a book copy to a member
- Return a book copy; auto-trigger fine if overdue
- Reserve a book when all copies are currently loaned out
- Auto-notify the next member in the reservation queue when a copy becomes available
- Fine calculation: configurable daily rate per book category
- Librarian vs Member roles with different permissions
- Search: by title, author, ISBN, subject

### Non-Functional Requirements
- Fine policy is pluggable — new rules (weekend multiplier, category caps) without changing core logic
- Notification delivery is pluggable — email, SMS, push without coupling to loan logic
- State transitions on `BookItem` must be explicit and validated
- Easy to add new member types (Student, Faculty, Premium) with different borrow limits

### What interviewers watch for
- Do you separate `Book` (catalogue) from `BookItem` (physical copy)?
- Do you model the `BookItem` **state machine** explicitly?
- Do you handle the **reservation queue** and its auto-notify trigger?
- Do you use **Strategy** for fine calculation and notifications?
- Do you think about **role-based access** — Librarian vs Member?

---

## 2. The Core Insight — `Book` vs `BookItem`

This is the defining design decision. Almost every weak candidate conflates them.

```
❌ Wrong model:
   Book has status: AVAILABLE | LOANED | OVERDUE
   Problem: "Harry Potter" exists as 5 physical copies.
   If copy #3 is loaned, does the Book become LOANED?
   What about the other 4 copies?

✅ Correct model:
   Book     — the catalogue entry. Title, author, ISBN, genre.
              Exists once. Has NO status field.
   BookItem — one physical copy of a Book.
              Has a barcode, rack location, condition, and a STATUS.
              "Harry Potter" → 5 BookItems, each with independent status.
```

The DB relationship: `Book` 1 → N `BookItem`. A `Loan` is always against a `BookItem`, never against a `Book`.

---

## 3. `BookItem` State Machine

```
AVAILABLE → RESERVED   (member reserves when all copies loaned)
AVAILABLE → LOANED     (direct checkout, no reservation pending)
RESERVED  → LOANED     (reserved member checks it out)
RESERVED  → AVAILABLE  (reservation expires or member cancels)
LOANED    → AVAILABLE  (returned on time)
LOANED    → OVERDUE    (return date passes, background job fires)
OVERDUE   → AVAILABLE  (returned late, fine calculated)
```

Invalid transitions (e.g., `OVERDUE → RESERVED`) must throw — not silently pass.

---

## 4. Entities & Relationships

```
Library (Singleton)
  ├── Catalogue
  │     └── Book[]
  │           └── BookItem[]
  ├── MemberRegistry
  │     └── Member[]
  ├── LoanService
  │     ├── Loan[]
  │     └── FineStrategy
  ├── ReservationService
  │     └── Reservation[] (queue per Book)
  └── NotificationService
        └── NotificationChannel[]

Book
  └── BookItem[]
        └── BookItemStatus: AVAILABLE | RESERVED | LOANED | OVERDUE

Member (abstract)
  ├── RegularMember  — max 3 books, 14-day loan
  ├── StudentMember  — max 5 books, 21-day loan
  └── PremiumMember  — max 10 books, 30-day loan

Loan
  └── BookItem, Member, issueDate, dueDate, returnDate?, Fine?

Reservation
  └── Book, Member, reservedAt, expiresAt, status

Fine
  └── Loan, daysOverdue, amount, FineStatus: PENDING | PAID | WAIVED
```

---

## 5. Full TypeScript Implementation

```typescript
// ============================================================
// ENUMS
// ============================================================

enum BookItemStatus {
  AVAILABLE = "AVAILABLE",
  RESERVED  = "RESERVED",   // held for a specific member
  LOANED    = "LOANED",     // checked out
  OVERDUE   = "OVERDUE",    // past due date and not returned
  LOST      = "LOST",       // reported lost
  DAMAGED   = "DAMAGED",    // under repair, not loanable
}

enum LoanStatus {
  ACTIVE    = "ACTIVE",     // currently loaned out
  RETURNED  = "RETURNED",   // returned on time
  OVERDUE   = "OVERDUE",    // past due, not yet returned
  RETURNED_LATE = "RETURNED_LATE",
}

enum FineStatus {
  PENDING = "PENDING",
  PAID    = "PAID",
  WAIVED  = "WAIVED",
}

enum ReservationStatus {
  PENDING   = "PENDING",    // waiting for a copy to become available
  READY     = "READY",      // a copy is available, member notified
  FULFILLED = "FULFILLED",  // member checked out the reserved copy
  CANCELLED = "CANCELLED",  // member cancelled or reservation expired
  EXPIRED   = "EXPIRED",    // member did not collect within the hold window
}

enum MemberStatus {
  ACTIVE    = "ACTIVE",
  SUSPENDED = "SUSPENDED",  // outstanding fines or policy violations
  EXPIRED   = "EXPIRED",    // membership lapsed
}

// Book genres for search/categorisation
enum Genre {
  FICTION      = "FICTION",
  NON_FICTION  = "NON_FICTION",
  SCIENCE      = "SCIENCE",
  TECHNOLOGY   = "TECHNOLOGY",
  HISTORY      = "HISTORY",
  BIOGRAPHY    = "BIOGRAPHY",
  REFERENCE    = "REFERENCE",
}

// ============================================================
// AUTHOR
// ============================================================

class Author {
  constructor(
    public readonly authorId: string,
    public readonly name: string,
    public readonly bio?: string
  ) {}
}

// ============================================================
// BOOK — the catalogue entry
// ============================================================

// Book represents the intellectual work — not any physical copy.
// It has metadata and a list of physical BookItems.
class Book {
  private items: BookItem[] = [];

  constructor(
    public readonly bookId: string,
    public readonly isbn: string,
    public readonly title: string,
    public readonly authors: Author[],
    public readonly genre: Genre,
    public readonly subject: string,
    public readonly publisher: string,
    public readonly publicationYear: number,
    public readonly language: string = "English"
  ) {}

  addItem(item: BookItem): void {
    this.items.push(item);
  }

  // Returns all physical copies with a given status
  getItemsByStatus(status: BookItemStatus): BookItem[] {
    return this.items.filter(i => i.status === status);
  }

  getAvailableItem(): BookItem | null {
    return this.getItemsByStatus(BookItemStatus.AVAILABLE)[0] ?? null;
  }

  getTotalCopies(): number { return this.items.length; }

  getAvailableCopies(): number {
    return this.getItemsByStatus(BookItemStatus.AVAILABLE).length;
  }

  // Used for reservation display and search results
  isAvailable(): boolean {
    return this.getAvailableCopies() > 0;
  }
}

// ============================================================
// BOOK ITEM — one physical copy
// ============================================================

// BookItem is a single physical copy on the shelf.
// It has a barcode (unique per copy), a rack location, and a STATUS.
// The State Pattern lives here — all status transitions are validated.
class BookItem {
  private _status: BookItemStatus = BookItemStatus.AVAILABLE;
  private reservedForMemberId: string | null = null;

  constructor(
    public readonly itemId: string,
    public readonly barcode: string,     // unique physical identifier
    public readonly bookId: string,      // FK to Book catalogue entry
    public rackLocation: string,         // e.g. "A-12-3" (aisle-shelf-position)
    public condition: string = "Good"
  ) {}

  get status(): BookItemStatus { return this._status; }

  // ── STATE TRANSITIONS ─────────────────────────────────────
  // Every transition is explicit. Invalid ones throw immediately.
  // This is the State Pattern applied without subclasses —
  // appropriate here because transitions are simple validations,
  // not complex behaviour-per-state.

  reserve(memberId: string): void {
    this.assertStatus(
      [BookItemStatus.AVAILABLE],
      `Cannot reserve a book in ${this._status} state`
    );
    this._status = BookItemStatus.RESERVED;
    this.reservedForMemberId = memberId;
  }

  checkout(memberId: string): void {
    // Allow checkout from AVAILABLE (walk-in) or RESERVED (for this member)
    if (this._status === BookItemStatus.RESERVED) {
      if (this.reservedForMemberId !== memberId) {
        throw new Error(
          `BookItem ${this.itemId} is reserved for a different member`
        );
      }
    } else {
      this.assertStatus(
        [BookItemStatus.AVAILABLE],
        `Cannot checkout a book in ${this._status} state`
      );
    }
    this._status = BookItemStatus.LOANED;
    this.reservedForMemberId = null;
  }

  markOverdue(): void {
    this.assertStatus(
      [BookItemStatus.LOANED],
      `Cannot mark overdue — book is not LOANED`
    );
    this._status = BookItemStatus.OVERDUE;
  }

  returnBook(): void {
    this.assertStatus(
      [BookItemStatus.LOANED, BookItemStatus.OVERDUE],
      `Cannot return a book in ${this._status} state`
    );
    this._status = BookItemStatus.AVAILABLE;
    this.reservedForMemberId = null;
  }

  cancelReservation(): void {
    this.assertStatus(
      [BookItemStatus.RESERVED],
      `Cannot cancel reservation — book is not RESERVED`
    );
    this._status = BookItemStatus.AVAILABLE;
    this.reservedForMemberId = null;
  }

  markLost(): void {
    this._status = BookItemStatus.LOST;
  }

  markDamaged(): void {
    this._status = BookItemStatus.DAMAGED;
  }

  private assertStatus(allowed: BookItemStatus[], errorMsg: string): void {
    if (!allowed.includes(this._status)) {
      throw new Error(errorMsg);
    }
  }
}

// ============================================================
// MEMBER — Abstract base with borrow limits
// ============================================================

// Member is an abstract class — different member types have different
// borrow limits and loan durations. Template Method pattern.
abstract class Member {
  private activeLoans: Loan[] = [];
  public status: MemberStatus = MemberStatus.ACTIVE;
  public outstandingFines: number = 0;

  constructor(
    public readonly memberId: string,
    public readonly name: string,
    public readonly email: string,
    public readonly phone: string,
    public readonly membershipExpiry: Date
  ) {}

  // Subclasses define their borrow policies
  abstract getMaxBooksAllowed(): number;
  abstract getLoanDurationDays(): number;
  abstract getMemberType(): string;

  canBorrow(): boolean {
    return (
      this.status === MemberStatus.ACTIVE &&
      new Date() <= this.membershipExpiry &&
      this.activeLoans.length < this.getMaxBooksAllowed() &&
      this.outstandingFines === 0   // blocked if unpaid fines
    );
  }

  addLoan(loan: Loan): void {
    this.activeLoans.push(loan);
  }

  removeLoan(loanId: string): void {
    this.activeLoans = this.activeLoans.filter(l => l.loanId !== loanId);
  }

  getActiveLoans(): Loan[] {
    return [...this.activeLoans];
  }

  hasLoan(itemId: string): boolean {
    return this.activeLoans.some(l => l.bookItem.itemId === itemId);
  }

  payFine(amount: number): void {
    this.outstandingFines = Math.max(0, this.outstandingFines - amount);
    if (this.outstandingFines === 0 && this.status === MemberStatus.SUSPENDED) {
      this.status = MemberStatus.ACTIVE; // auto-reinstate after paying
    }
  }

  suspend(): void {
    this.status = MemberStatus.SUSPENDED;
  }
}

// Regular public library card holder
class RegularMember extends Member {
  getMaxBooksAllowed(): number    { return 3; }
  getLoanDurationDays(): number   { return 14; }
  getMemberType(): string         { return "Regular"; }
}

// Student with extended privileges
class StudentMember extends Member {
  getMaxBooksAllowed(): number    { return 5; }
  getLoanDurationDays(): number   { return 21; }
  getMemberType(): string         { return "Student"; }
}

// Premium/Faculty member
class PremiumMember extends Member {
  getMaxBooksAllowed(): number    { return 10; }
  getLoanDurationDays(): number   { return 30; }
  getMemberType(): string         { return "Premium"; }
}

// ============================================================
// FINE — Fine entity
// ============================================================

class Fine {
  public status: FineStatus = FineStatus.PENDING;
  public readonly amount: number;

  constructor(
    public readonly fineId: string,
    public readonly loanId: string,
    public readonly memberId: string,
    public readonly daysOverdue: number,
    dailyRate: number
  ) {
    this.amount = daysOverdue * dailyRate;
  }

  pay(): void {
    if (this.status !== FineStatus.PENDING) {
      throw new Error(`Fine ${this.fineId} is already ${this.status}`);
    }
    this.status = FineStatus.PAID;
  }

  waive(reason: string): void {
    console.log(`[FINE] ${this.fineId} waived. Reason: ${reason}`);
    this.status = FineStatus.WAIVED;
  }
}

// ============================================================
// LOAN
// ============================================================

// Loan is the central transaction entity.
// It ties Member ↔ BookItem for the duration of a checkout.
class Loan {
  public status: LoanStatus = LoanStatus.ACTIVE;
  public returnDate: Date | null = null;
  public fine: Fine | null = null;

  public readonly dueDate: Date;

  constructor(
    public readonly loanId: string,
    public readonly member: Member,
    public readonly bookItem: BookItem,
    public readonly issueDate: Date = new Date(),
    loanDurationDays: number
  ) {
    // Due date = issue date + member's allowed loan duration
    this.dueDate = new Date(issueDate);
    this.dueDate.setDate(this.dueDate.getDate() + loanDurationDays);
  }

  isOverdue(): boolean {
    return this.status === LoanStatus.ACTIVE && new Date() > this.dueDate;
  }

  getDaysOverdue(): number {
    if (!this.isOverdue()) return 0;
    const msOverdue = Date.now() - this.dueDate.getTime();
    return Math.floor(msOverdue / (1000 * 60 * 60 * 24));
  }

  markReturned(returnDate: Date = new Date()): void {
    this.returnDate = returnDate;
    this.status = returnDate > this.dueDate
      ? LoanStatus.RETURNED_LATE
      : LoanStatus.RETURNED;
  }

  attachFine(fine: Fine): void {
    this.fine = fine;
  }
}

// ============================================================
// RESERVATION
// ============================================================

class Reservation {
  public status: ReservationStatus = ReservationStatus.PENDING;
  public reservedItemId: string | null = null; // set when a copy becomes available
  public readonly expiresAt: Date;
  private static readonly HOLD_WINDOW_DAYS = 3; // member has 3 days to collect

  constructor(
    public readonly reservationId: string,
    public readonly book: Book,
    public readonly member: Member,
    public readonly reservedAt: Date = new Date()
  ) {
    // Reservation expires if member doesn't collect within the hold window
    this.expiresAt = new Date(reservedAt);
    this.expiresAt.setDate(this.expiresAt.getDate() + Reservation.HOLD_WINDOW_DAYS);
  }

  markReady(itemId: string): void {
    if (this.status !== ReservationStatus.PENDING) {
      throw new Error(`Reservation ${this.reservationId} is not PENDING`);
    }
    this.reservedItemId = itemId;
    this.status = ReservationStatus.READY;
  }

  fulfil(): void {
    if (this.status !== ReservationStatus.READY) {
      throw new Error(`Reservation ${this.reservationId} is not READY`);
    }
    this.status = ReservationStatus.FULFILLED;
  }

  cancel(): void {
    this.status = ReservationStatus.CANCELLED;
  }

  expire(): void {
    this.status = ReservationStatus.EXPIRED;
  }

  isExpired(): boolean {
    return this.status === ReservationStatus.READY && new Date() > this.expiresAt;
  }
}

// ============================================================
// FINE STRATEGY — Strategy Pattern
// ============================================================

// Fine calculation is a plug-in concern.
// Different libraries have different policies:
//   public library: ₹2/day, no cap
//   university:     ₹5/day for tech books, ₹2/day for fiction
//   premium service: 0 fines, just suspension
interface FineStrategy {
  calculateDailyRate(bookItem: BookItem): number;
  getDescription(): string;
}

// Flat rate — same for all books
class FlatRateFineStrategy implements FineStrategy {
  constructor(private readonly ratePerDay: number = 2) {}

  calculateDailyRate(_bookItem: BookItem): number {
    return this.ratePerDay;
  }

  getDescription(): string {
    return `Flat ₹${this.ratePerDay}/day`;
  }
}

// Category-based rate — tech books cost more per overdue day
class CategoryBasedFineStrategy implements FineStrategy {
  private readonly rates: Partial<Record<string, number>> = {
    TECHNOLOGY: 5,
    REFERENCE:  10,  // Reference books are not supposed to leave the library
    FICTION:    1,
    default:    2,
  };

  calculateDailyRate(bookItem: BookItem): number {
    // To get the genre we'd look up via bookId — simplified here
    return this.rates["default"]!;
  }

  getDescription(): string {
    return "Category-based fine (₹1–₹10/day)";
  }
}

// ============================================================
// NOTIFICATION CHANNEL — Strategy Pattern
// ============================================================

// Notifications are another pluggable strategy.
// The loan/reservation services emit events; channels deliver them.
interface NotificationChannel {
  notify(member: Member, subject: string, message: string): void;
}

class EmailChannel implements NotificationChannel {
  notify(member: Member, subject: string, message: string): void {
    console.log(`[EMAIL → ${member.email}] ${subject}: ${message}`);
  }
}

class SMSChannel implements NotificationChannel {
  notify(member: Member, _subject: string, message: string): void {
    console.log(`[SMS → ${member.phone}] ${message}`);
  }
}

// ============================================================
// NOTIFICATION SERVICE — Observer target
// ============================================================

class NotificationService {
  private channels: NotificationChannel[] = [];

  addChannel(channel: NotificationChannel): void {
    this.channels.push(channel);
  }

  send(member: Member, subject: string, message: string): void {
    for (const ch of this.channels) {
      try {
        ch.notify(member, subject, message);
      } catch (err) {
        console.error(`[NOTIFY ERROR] Channel failed:`, err);
        // Don't let a failed notification break the loan flow
      }
    }
  }
}

// ============================================================
// CATALOGUE
// ============================================================

class Catalogue {
  private books: Map<string, Book> = new Map(); // bookId → Book

  addBook(book: Book): void {
    this.books.set(book.bookId, book);
  }

  removeBook(bookId: string): void {
    this.books.delete(bookId);
  }

  getBookById(bookId: string): Book | null {
    return this.books.get(bookId) ?? null;
  }

  getBookByIsbn(isbn: string): Book | null {
    return [...this.books.values()].find(b => b.isbn === isbn) ?? null;
  }

  // Full-text search across title, author names, subject
  search(query: string): Book[] {
    const q = query.toLowerCase();
    return [...this.books.values()].filter(book =>
      book.title.toLowerCase().includes(q) ||
      book.authors.some(a => a.name.toLowerCase().includes(q)) ||
      book.subject.toLowerCase().includes(q) ||
      book.isbn.includes(q)
    );
  }

  searchByGenre(genre: Genre): Book[] {
    return [...this.books.values()].filter(b => b.genre === genre);
  }
}

// ============================================================
// LOAN SERVICE
// ============================================================

class LoanService {
  private loans: Map<string, Loan> = new Map();
  private loanCounter = 1;
  private fineCounter = 1;

  constructor(
    private readonly fineStrategy: FineStrategy,
    private readonly notificationService: NotificationService
  ) {}

  // ── CHECKOUT ─────────────────────────────────────────────────
  // Issue a BookItem to a Member.
  // If the book was reserved for this member, the reservation is fulfilled.
  checkout(member: Member, bookItem: BookItem, reservation?: Reservation): Loan {
    // Guard: member must be eligible
    if (!member.canBorrow()) {
      throw new Error(
        `Member ${member.memberId} cannot borrow: ` +
        `status=${member.status}, outstanding fines=₹${member.outstandingFines}`
      );
    }

    // BookItem.checkout() validates the state transition
    bookItem.checkout(member.memberId);

    // Fulfil reservation if this was a reserved copy
    if (reservation) {
      reservation.fulfil();
    }

    const loanId = `LN-${String(this.loanCounter++).padStart(6, "0")}`;
    const loan = new Loan(loanId, member, bookItem, new Date(), member.getLoanDurationDays());

    this.loans.set(loanId, loan);
    member.addLoan(loan);

    console.log(
      `[CHECKOUT] ${member.name} checked out "${bookItem.itemId}" | Due: ${loan.dueDate.toDateString()}`
    );

    this.notificationService.send(
      member,
      "Book Checked Out",
      `You have checked out item ${bookItem.barcode}. Due date: ${loan.dueDate.toDateString()}.`
    );

    return loan;
  }

  // ── RETURN ────────────────────────────────────────────────────
  // Return a BookItem. Calculate fine if overdue.
  // Returns the Fine object if applicable, null otherwise.
  returnBook(loan: Loan): Fine | null {
    const returnDate = new Date();
    const daysOverdue = loan.getDaysOverdue();

    // Return the physical item first
    loan.bookItem.returnBook();
    loan.markReturned(returnDate);
    member_cleanup: {
      loan.member.removeLoan(loan.loanId);
    }

    let fine: Fine | null = null;

    if (daysOverdue > 0) {
      // Calculate fine based on pluggable strategy
      const dailyRate = this.fineStrategy.calculateDailyRate(loan.bookItem);
      const fineId = `FN-${String(this.fineCounter++).padStart(6, "0")}`;
      fine = new Fine(fineId, loan.loanId, loan.member.memberId, daysOverdue, dailyRate);

      loan.attachFine(fine);
      loan.member.outstandingFines += fine.amount;

      // Auto-suspend if fines exceed threshold (₹50)
      if (loan.member.outstandingFines > 50) {
        loan.member.suspend();
        console.warn(`[SUSPEND] Member ${loan.member.memberId} suspended — outstanding fines: ₹${loan.member.outstandingFines}`);
      }

      this.notificationService.send(
        loan.member,
        "Overdue Fine",
        `Your book was returned ${daysOverdue} day(s) late. Fine: ₹${fine.amount}. ${this.fineStrategy.getDescription()}.`
      );

      console.log(`[RETURN LATE] Loan ${loan.loanId} | Overdue: ${daysOverdue} days | Fine: ₹${fine.amount}`);
    } else {
      console.log(`[RETURN] Loan ${loan.loanId} returned on time`);
    }

    return fine;
  }

  // ── OVERDUE JOB ───────────────────────────────────────────────
  // Background job (runs daily). Marks all overdue loans + BookItems.
  processOverdueLoans(): void {
    let count = 0;
    for (const loan of this.loans.values()) {
      if (loan.status === LoanStatus.ACTIVE && loan.isOverdue()) {
        loan.status = LoanStatus.OVERDUE;
        loan.bookItem.markOverdue();
        count++;

        this.notificationService.send(
          loan.member,
          "Overdue Notice",
          `Your loan of item ${loan.bookItem.barcode} was due on ${loan.dueDate.toDateString()}. Please return it immediately.`
        );
      }
    }
    if (count > 0) {
      console.log(`[OVERDUE JOB] Marked ${count} loan(s) as overdue`);
    }
  }
}

// ============================================================
// RESERVATION SERVICE
// ============================================================

class ReservationService {
  // Each Book has its own FIFO queue of reservations
  private queues: Map<string, Reservation[]> = new Map(); // bookId → queue
  private reservationCounter = 1;

  constructor(private readonly notificationService: NotificationService) {}

  // ── RESERVE ───────────────────────────────────────────────────
  // Add a member to the waitlist for a Book.
  // Called when all copies are currently loaned out.
  reserve(member: Member, book: Book): Reservation {
    if (book.isAvailable()) {
      throw new Error(
        `Book "${book.title}" has available copies — no need to reserve. Just checkout.`
      );
    }

    // Prevent duplicate reservation by same member for same book
    const queue = this.getQueue(book.bookId);
    const alreadyQueued = queue.some(
      r => r.member.memberId === member.memberId &&
           r.status === ReservationStatus.PENDING
    );
    if (alreadyQueued) {
      throw new Error(`Member ${member.memberId} already has a pending reservation for "${book.title}"`);
    }

    const reservationId = `RES-${String(this.reservationCounter++).padStart(6, "0")}`;
    const reservation = new Reservation(reservationId, book, member);
    queue.push(reservation);

    const position = queue.length;
    console.log(`[RESERVE] ${member.name} is #${position} in queue for "${book.title}"`);

    this.notificationService.send(
      member,
      "Reservation Confirmed",
      `You are #${position} in the waitlist for "${book.title}". We will notify you when a copy is available.`
    );

    return reservation;
  }

  // ── NOTIFY NEXT IN QUEUE ─────────────────────────────────────
  // Called by LoanService when a BookItem is returned (status → AVAILABLE).
  // Assigns the copy to the next member in the reservation queue.
  notifyNextInQueue(book: Book, returnedItem: BookItem): Reservation | null {
    const queue = this.getQueue(book.bookId);
    const pendingReservations = queue.filter(r => r.status === ReservationStatus.PENDING);

    if (pendingReservations.length === 0) return null;

    // FIFO: the first person in the queue gets the copy
    const nextReservation = pendingReservations[0];

    // Reserve the physical copy for this specific member
    returnedItem.reserve(nextReservation.member.memberId);
    nextReservation.markReady(returnedItem.itemId);

    this.notificationService.send(
      nextReservation.member,
      "Book Available for Collection",
      `"${book.title}" is now available for you. Please collect within ${
        nextReservation.expiresAt.toDateString()
      } or your reservation will expire.`
    );

    console.log(
      `[QUEUE] "${book.title}" reserved for ${nextReservation.member.name} | Expires: ${nextReservation.expiresAt.toDateString()}`
    );

    return nextReservation;
  }

  // ── EXPIRE STALE RESERVATIONS ─────────────────────────────────
  // Background job: members who didn't collect in time lose their spot
  processExpiredReservations(): void {
    for (const queue of this.queues.values()) {
      for (const res of queue) {
        if (res.isExpired()) {
          res.expire();
          // Release the reserved BookItem back to AVAILABLE
          if (res.reservedItemId) {
            // In a real system, look up the BookItem and call cancelReservation()
            console.log(`[EXPIRE] Reservation ${res.reservationId} expired for member ${res.member.memberId}`);
          }
          this.notificationService.send(
            res.member,
            "Reservation Expired",
            `Your reservation for "${res.book.title}" has expired as you did not collect it within the hold window.`
          );
        }
      }
    }
  }

  cancelReservation(reservationId: string): void {
    for (const queue of this.queues.values()) {
      const res = queue.find(r => r.reservationId === reservationId);
      if (res) {
        res.cancel();
        console.log(`[CANCEL] Reservation ${reservationId} cancelled`);
        return;
      }
    }
    throw new Error(`Reservation ${reservationId} not found`);
  }

  getQueueLength(bookId: string): number {
    return this.getQueue(bookId).filter(r => r.status === ReservationStatus.PENDING).length;
  }

  private getQueue(bookId: string): Reservation[] {
    if (!this.queues.has(bookId)) this.queues.set(bookId, []);
    return this.queues.get(bookId)!;
  }
}

// ============================================================
// LIBRARY — Singleton Facade
// ============================================================

class Library {
  private static instance: Library | null = null;

  public readonly catalogue: Catalogue;
  public readonly loanService: LoanService;
  public readonly reservationService: ReservationService;
  public readonly notificationService: NotificationService;

  private members: Map<string, Member> = new Map();
  private memberCounter = 1;

  private constructor(
    public readonly name: string,
    fineStrategy: FineStrategy = new FlatRateFineStrategy()
  ) {
    this.catalogue = new Catalogue();

    this.notificationService = new NotificationService();
    this.notificationService.addChannel(new EmailChannel());
    this.notificationService.addChannel(new SMSChannel());

    this.loanService = new LoanService(fineStrategy, this.notificationService);
    this.reservationService = new ReservationService(this.notificationService);
  }

  static getInstance(name: string, fineStrategy?: FineStrategy): Library {
    if (!Library.instance) {
      Library.instance = new Library(name, fineStrategy);
    }
    return Library.instance;
  }

  static resetInstance(): void { Library.instance = null; }

  // ── MEMBER REGISTRATION ───────────────────────────────────────
  registerMember(
    name: string,
    email: string,
    phone: string,
    type: "Regular" | "Student" | "Premium",
    expiryYears: number = 1
  ): Member {
    const memberId = `MBR-${String(this.memberCounter++).padStart(4, "0")}`;
    const expiry = new Date();
    expiry.setFullYear(expiry.getFullYear() + expiryYears);

    const member: Member = type === "Student"
      ? new StudentMember(memberId, name, email, phone, expiry)
      : type === "Premium"
      ? new PremiumMember(memberId, name, email, phone, expiry)
      : new RegularMember(memberId, name, email, phone, expiry);

    this.members.set(memberId, member);
    console.log(`[REGISTER] ${type} member "${name}" (${memberId}) registered`);
    return member;
  }

  // ── SEARCH ────────────────────────────────────────────────────
  searchBooks(query: string): Book[] {
    return this.catalogue.search(query);
  }

  // ── CHECKOUT FLOW ─────────────────────────────────────────────
  // Convenience method: pick the first available copy of a Book,
  // then delegate to LoanService.
  issueBook(member: Member, book: Book): Loan {
    const availableItem = book.getAvailableItem();

    if (!availableItem) {
      throw new Error(
        `No available copies of "${book.title}". ` +
        `Queue length: ${this.reservationService.getQueueLength(book.bookId)}. ` +
        `Consider reserving it.`
      );
    }

    return this.loanService.checkout(member, availableItem);
  }

  // ── RETURN FLOW ───────────────────────────────────────────────
  // Return a book, calculate fines, then notify the reservation queue.
  returnBook(loan: Loan): Fine | null {
    const fine = this.loanService.returnBook(loan);

    // After a return, check if anyone is waiting for this Book
    const book = this.catalogue.getBookById(loan.bookItem.bookId);
    if (book) {
      this.reservationService.notifyNextInQueue(book, loan.bookItem);
    }

    return fine;
  }
}

// ============================================================
// MEMBER FACTORY — Factory Pattern
// ============================================================

// Encapsulates member creation with validation
class MemberFactory {
  static create(
    type: "Regular" | "Student" | "Premium",
    memberId: string,
    name: string,
    email: string,
    phone: string,
    expiryYears: number = 1
  ): Member {
    const expiry = new Date();
    expiry.setFullYear(expiry.getFullYear() + expiryYears);
    switch (type) {
      case "Student": return new StudentMember(memberId, name, email, phone, expiry);
      case "Premium": return new PremiumMember(memberId, name, email, phone, expiry);
      default:        return new RegularMember(memberId, name, email, phone, expiry);
    }
  }
}

// ============================================================
// DEMO
// ============================================================

function main() {
  Library.resetInstance();
  const library = Library.getInstance("City Central Library", new FlatRateFineStrategy(2));

  // Add a book with 2 copies
  const authorA = new Author("AU001", "Robert C. Martin");
  const cleanCode = new Book("BK001", "978-0132350884", "Clean Code", [authorA], Genre.TECHNOLOGY, "Software Engineering", "Prentice Hall", 2008);
  const item1 = new BookItem("IT001", "BC-0001", "BK001", "T-12-A");
  const item2 = new BookItem("IT002", "BC-0002", "BK001", "T-12-B");
  cleanCode.addItem(item1);
  cleanCode.addItem(item2);
  library.catalogue.addBook(cleanCode);

  // Register members
  const alice = library.registerMember("Alice", "alice@example.com", "9001", "Regular");
  const bob   = library.registerMember("Bob",   "bob@example.com",   "9002", "Student");
  const carol = library.registerMember("Carol", "carol@example.com", "9003", "Regular");

  console.log("\n=== Scenario 1: Alice and Bob both checkout ===");
  const loan1 = library.issueBook(alice, cleanCode);
  const loan2 = library.issueBook(bob, cleanCode);

  console.log("\n=== Scenario 2: Carol tries to checkout (no copies available) ===");
  try {
    library.issueBook(carol, cleanCode);
  } catch (e: any) {
    console.log(`[EXPECTED ERROR] ${e.message}`);
    // Carol should reserve instead
    library.reservationService.reserve(carol, cleanCode);
  }

  console.log("\n=== Scenario 3: Alice returns late (simulate overdue) ===");
  // Backdate the due date to simulate an overdue return
  (loan1 as any).dueDate = new Date(Date.now() - 3 * 24 * 60 * 60 * 1000); // 3 days ago
  const fine = library.returnBook(loan1);
  // Returning triggers Carol's reservation notification automatically

  console.log("\n=== Scenario 4: Carol collects the reserved copy ===");
  const carolReservation = library.reservationService["queues"].get("BK001")?.[0];
  if (carolReservation && carolReservation.status === ReservationStatus.READY) {
    const reservedItem = cleanCode.getItemsByStatus(BookItemStatus.RESERVED)[0];
    if (reservedItem) {
      library.loanService.checkout(carol, reservedItem, carolReservation);
    }
  }

  console.log(`\n=== Final State ===`);
  console.log(`Available copies: ${cleanCode.getAvailableCopies()}`);
  console.log(`Alice's outstanding fines: ₹${alice.outstandingFines}`);
}

main();
```

---

## 6. Design Patterns — Deep Dive

### 6.1 State Transitions on `BookItem` (Lightweight State Pattern)
Rather than full State Pattern subclasses, `BookItem` uses explicit transition methods with `assertStatus()` guards. This is appropriate when transitions have validation logic but minimal per-state behaviour. Each method is a named transition: `reserve()`, `checkout()`, `markOverdue()`, `returnBook()`. Adding a new state (e.g., `UNDER_REPAIR`) is one new method and a new entry in the relevant guards.

### 6.2 Template Method — `Member`
`Member` is abstract. The game loop equivalent is `canBorrow()` — always the same check — but the limits it checks (`getMaxBooksAllowed()`, `getLoanDurationDays()`) are deferred to subclasses. Adding `FacultyMember` with 15-book/60-day policy = one subclass, zero changes to `Member`.

### 6.3 Strategy — `FineStrategy` and `NotificationChannel`
Two completely independent axes of variation, both plug into the system via interfaces. `FineStrategy` varies by library policy; `NotificationChannel` varies by delivery mechanism. Both are injected at construction time, never hardcoded.

### 6.4 Observer — `NotificationService`
The loan and reservation services don't care how notifications are delivered. They call `notificationService.send()` and forget. `NotificationService` fans out to registered channels (`EmailChannel`, `SMSChannel`). Adding WhatsApp = one new class + `addChannel()` call.

### 6.5 Factory — `MemberFactory`
Encapsulates the switch between `RegularMember`, `StudentMember`, `PremiumMember`. Client code passes a string type; the factory returns the right concrete class. Follows the Factory Method pattern without requiring a full abstract factory.

---

## 7. The Reservation Queue — Key Interview Nuance

The reservation system is a **FIFO queue per Book** (not per `BookItem`). When a `BookItem` is returned:

1. `LoanService.returnBook()` calls `bookItem.returnBook()` → status = AVAILABLE
2. It then calls `reservationService.notifyNextInQueue(book, returnedItem)`
3. `ReservationService` finds the first PENDING reservation in the queue
4. It calls `returnedItem.reserve(nextMember.memberId)` → status = RESERVED
5. It sets `reservation.status = READY` and sends a notification
6. The member has 3 days to collect — a background job runs `processExpiredReservations()`

**The coupling between return and reservation notification is deliberate and important.** The moment a copy is returned, the next person in queue should be notified. This is the Observer trigger that most candidates miss.

---

## 8. `Book` vs `BookItem` — The Interview Differentiator (again)

| Aspect | Book | BookItem |
|--------|------|----------|
| What it represents | Intellectual work | Physical object |
| Count | One per title | Many per title |
| Has status? | No | Yes (AVAILABLE, LOANED…) |
| Has location? | No | Yes (rack, aisle) |
| What gets loaned? | Never | Always |
| DB primary key | ISBN or UUID | Barcode or UUID |
| Relationship | Parent | Child (FK to Book) |

---

## 9. Edge Cases & Nuances

| Edge Case | How to Handle |
|-----------|--------------|
| **Member at borrow limit** | `canBorrow()` checks `activeLoans.length < getMaxBooksAllowed()` |
| **Member has unpaid fines** | `canBorrow()` blocks if `outstandingFines > 0` |
| **Expired membership** | `canBorrow()` checks `new Date() <= membershipExpiry` |
| **Duplicate reservation** | `ReservationService.reserve()` checks for existing PENDING entry |
| **Reservation expires** | Background job runs `processExpiredReservations()` → releases reserved copy |
| **Lost book** | `bookItem.markLost()` — removed from circulation; member charged replacement cost |
| **Same member reserves twice** | Guard in `reserve()` — throws if already PENDING for same member+book |
| **Fine waiver** | `fine.waive(reason)` — librarian-only action; deducts from member's outstanding balance |
| **Membership renewal** | Update `member.membershipExpiry`; if EXPIRED status, reactivate |
| **Book edition mismatch** | Each edition has its own `Book` entry (different ISBN). Not merged. |

---

## 10. Follow-Up Questions Interviewers Ask

1. **"How would you handle a book that is in multiple branches?"**
   → `BookItem` gets a `branchId` FK. `Catalogue.search()` accepts an optional `branchId` filter. `LoanService` validates the member's home branch or allows inter-branch loans with extra days.

2. **"How would you add a search by availability?"**
   → `Catalogue.searchAvailable(query)` filters `search()` results to only those where `book.getAvailableCopies() > 0`. For performance, maintain a denormalised `availableCount` column in DB, updated on every checkout/return.

3. **"How would you implement a due-date reminder?"**
   → Background job runs daily. For each `ACTIVE` loan where `dueDate - today <= 2 days`, call `notificationService.send()`. This is `LoanService.sendDueDateReminders()` — pure iteration over active loans.

4. **"How would you add e-books with a simultaneous borrow limit?"**
   → `EBook extends Book` with a `maxSimultaneousLoans` field. `EBookItem` has no physical location. `LoanService` checks active loans across ALL members for that EBook against the limit before issuing.

5. **"How would you model a librarian vs member role?"**
   → `Account` class with `role: Role.LIBRARIAN | Role.MEMBER`. Wrap `LoanService` and `ReservationService` in a `LibraryController` that checks role before calling fine waiver, book removal, or member suspension.

---

## 11. Key Takeaways

- **`Book` ≠ `BookItem`** — this is the single most important design decision. State lives on `BookItem`, not `Book`. Every loan, every reservation, every status change is against a `BookItem`.
- **Named transition methods on `BookItem`** — `reserve()`, `checkout()`, `markOverdue()`, `returnBook()` — are cleaner and safer than setting `status` directly. Guards make invalid transitions loud.
- **Return triggers reservation notification** — this coupling is intentional and must be explicit. The moment a copy becomes available, the head of the queue is notified. Don't make this a polling job.
- **Two Strategy axes** — fine calculation and notification delivery are independently pluggable. Calling them both out shows strong SRP instincts.
- **Abstract `Member` with borrow limits in subclasses** — adding `SeniorCitizenMember` with free loans is one new class. Never hardcode limits in `LoanService`.
- **Fine suspension threshold** — auto-suspending a member when fines exceed a threshold is a real-world detail that signals you've thought about operations, not just the happy path.

---

*Next up: LLD #06 — ATM Machine*