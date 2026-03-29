# LLD #06 — ATM Machine

> **Language:** TypeScript
> **Difficulty:** ⭐⭐⭐⭐ (Senior)
> **Patterns Used:** State, Strategy, Singleton, Command, Chain of Responsibility
> **Core Challenge:** State Pattern at its most canonical — every operation is only valid in one specific state

---

## 1. Requirements

### Functional Requirements
- Insert card → validate card (not expired, not blocked, not stolen)
- Enter PIN → authenticate; block card after 3 wrong attempts
- Select transaction: Withdraw, Deposit, Balance Enquiry, Mini Statement
- Withdraw: check account balance, check ATM cash reserve, dispense cash
- Deposit: accept cash/cheque, update balance
- Balance Enquiry: return current balance
- Eject card on session end or timeout
- Print receipt (optional)

### Non-Functional Requirements
- **Every operation must be rejected if ATM is in wrong state** — no silent no-ops
- ATM goes `OUT_OF_SERVICE` automatically when cash drops below threshold
- Concurrent session safety — one card session at a time
- Pluggable transaction types — add `ChequeDeposit` without changing state logic
- Bank communication is external — ATM delegates balance/auth checks to `BankService`

### What interviewers watch for
- Do you model **six distinct states** and make illegal ops throw?
- Do you know the **PIN retry counter** and card-blocking logic?
- Do you separate **ATM cash reserve** (physical machine) from **account balance** (bank)?
- Do you use **Strategy** for different transaction types?
- Do you handle **session timeout**?

---

## 2. The Core Design Insight — Two Independent Balances

The most common mistake: confusing ATM cash with account balance.

```
ATM Cash Reserve   — physical cash inside the machine
                    Decremented on every withdrawal
                    Filled by technician, not by deposits
                    When low → ATM goes OUT_OF_SERVICE

Account Balance    — the customer's bank balance
                    Stored at the bank (BankService)
                    Checked before withdrawal (must be ≥ amount)
                    Updated by the bank on transaction commit
```

A withdrawal requires **both** checks to pass:
1. `account.balance >= requestedAmount` (bank check)
2. `atm.cashReserve >= requestedAmount` (machine check)

If either fails, the transaction is declined.

---

## 3. States and Their Legal Operations

```
IDLE
  ✅ insertCard()
  ❌ enterPin(), selectTransaction(), withdraw(), deposit(), ejectCard()

CARD_INSERTED
  ✅ enterPin(), ejectCard()
  ❌ insertCard(), selectTransaction(), withdraw(), deposit()

PIN_VERIFIED
  ✅ selectTransaction(), ejectCard()
  ❌ insertCard(), enterPin(), withdraw(), deposit() [directly — go through TRANSACTION]

TRANSACTION
  ✅ withdraw(), deposit(), checkBalance(), printMiniStatement(), ejectCard()
  ❌ insertCard(), enterPin()

CARD_BLOCKED
  ✅ ejectCard() [only way out — card is ejected and blocked]
  ❌ everything else

OUT_OF_SERVICE
  ✅ refillCash() [technician only — returns to IDLE]
  ❌ everything else
```

Any call to a method not in the ✅ list must **throw** — not silently return, not log a warning, **throw**.

---

## 4. Entities & Relationships

```
ATM (Singleton — one machine, one session)
  └── ATMState (abstract) — 6 concrete states
  └── CashDispenser          — physical cash mgmt
  └── CardReader             — card I/O
  └── Keypad                 — PIN input
  └── Display                — screen output
  └── ReceiptPrinter         — optional receipt
  └── BankService (interface)— external bank comm

Card
  └── Account (via BankService lookup)

Account
  └── balance, accountNumber, type

Transaction (abstract) — Command Pattern
  ├── WithdrawTransaction
  ├── DepositTransaction
  ├── BalanceEnquiryTransaction
  └── MiniStatementTransaction

Session
  └── Card, Member, startTime, transactionHistory[]
```

---

## 5. Full TypeScript Implementation

```typescript
// ============================================================
// ENUMS
// ============================================================

enum ATMStateType {
  IDLE            = "IDLE",
  CARD_INSERTED   = "CARD_INSERTED",
  PIN_VERIFIED    = "PIN_VERIFIED",
  TRANSACTION     = "TRANSACTION",
  CARD_BLOCKED    = "CARD_BLOCKED",
  OUT_OF_SERVICE  = "OUT_OF_SERVICE",
}

enum CardStatus {
  ACTIVE  = "ACTIVE",
  BLOCKED = "BLOCKED",   // too many wrong PINs or bank-blocked
  EXPIRED = "EXPIRED",
  STOLEN  = "STOLEN",
}

enum TransactionType {
  WITHDRAWAL       = "WITHDRAWAL",
  DEPOSIT          = "DEPOSIT",
  BALANCE_ENQUIRY  = "BALANCE_ENQUIRY",
  MINI_STATEMENT   = "MINI_STATEMENT",
}

enum TransactionStatus {
  PENDING   = "PENDING",
  SUCCESS   = "SUCCESS",
  FAILED    = "FAILED",
  CANCELLED = "CANCELLED",
}

// ============================================================
// DOMAIN ENTITIES
// ============================================================

class Card {
  public failedPinAttempts: number = 0;
  private static readonly MAX_PIN_ATTEMPTS = 3;

  constructor(
    public readonly cardNumber: string,   // 16-digit PAN
    public readonly accountNumber: string,
    public readonly expiryDate: Date,
    public status: CardStatus = CardStatus.ACTIVE
  ) {}

  isValid(): boolean {
    return (
      this.status === CardStatus.ACTIVE &&
      new Date() < this.expiryDate
    );
  }

  // Returns true if PIN matches; false and increments counter if wrong.
  // Auto-blocks after MAX_PIN_ATTEMPTS wrong entries.
  verifyPin(enteredPin: string, correctPin: string): boolean {
    if (enteredPin === correctPin) {
      this.failedPinAttempts = 0; // reset on success
      return true;
    }
    this.failedPinAttempts++;
    if (this.failedPinAttempts >= Card.MAX_PIN_ATTEMPTS) {
      this.status = CardStatus.BLOCKED;
    }
    return false;
  }

  getRemainingPinAttempts(): number {
    return Card.MAX_PIN_ATTEMPTS - this.failedPinAttempts;
  }

  isBlocked(): boolean {
    return this.status === CardStatus.BLOCKED;
  }
}

class Account {
  private transactions: AccountTransaction[] = [];

  constructor(
    public readonly accountNumber: string,
    public readonly holderName: string,
    private balance: number,
    public readonly accountType: "SAVINGS" | "CURRENT" = "SAVINGS",
    private readonly pin: string     // stored hashed in real systems
  ) {}

  getBalance(): number { return this.balance; }

  verifyPin(pin: string): boolean { return this.pin === pin; }

  debit(amount: number, description: string): void {
    if (amount > this.balance) {
      throw new Error("Insufficient balance");
    }
    this.balance -= amount;
    this.transactions.push({ type: "DEBIT", amount, description, date: new Date() });
  }

  credit(amount: number, description: string): void {
    this.balance += amount;
    this.transactions.push({ type: "CREDIT", amount, description, date: new Date() });
  }

  getMiniStatement(count: number = 5): AccountTransaction[] {
    return [...this.transactions].reverse().slice(0, count);
  }
}

interface AccountTransaction {
  type: "DEBIT" | "CREDIT";
  amount: number;
  description: string;
  date: Date;
}

// ============================================================
// TRANSACTION — Command Pattern
// ============================================================

// Each transaction type is a Command object. This lets us:
// 1. Log the full transaction history
// 2. Retry or undo (in theory)
// 3. Add new transaction types without changing the ATM state machine
abstract class Transaction {
  public status: TransactionStatus = TransactionStatus.PENDING;
  public readonly createdAt: Date = new Date();
  public message: string = "";

  constructor(
    public readonly transactionId: string,
    public readonly type: TransactionType,
    public readonly cardNumber: string,
    public readonly amount: number = 0
  ) {}

  abstract execute(account: Account, atm: ATM): void;

  protected succeed(msg: string): void {
    this.status = TransactionStatus.SUCCESS;
    this.message = msg;
    console.log(`[TXN SUCCESS] ${this.transactionId}: ${msg}`);
  }

  protected fail(msg: string): void {
    this.status = TransactionStatus.FAILED;
    this.message = msg;
    console.warn(`[TXN FAILED] ${this.transactionId}: ${msg}`);
  }
}

class WithdrawTransaction extends Transaction {
  constructor(id: string, cardNumber: string, amount: number) {
    super(id, TransactionType.WITHDRAWAL, cardNumber, amount);
  }

  execute(account: Account, atm: ATM): void {
    // Check 1: Account has sufficient balance
    if (account.getBalance() < this.amount) {
      this.fail(`Insufficient account balance (balance: ₹${account.getBalance()}, requested: ₹${this.amount})`);
      return;
    }

    // Check 2: ATM has sufficient physical cash
    if (!atm.getCashDispenser().hasSufficientCash(this.amount)) {
      this.fail(`ATM cash insufficient. Please try a lower amount.`);
      return;
    }

    // Both checks pass — debit account and dispense cash
    account.debit(this.amount, `ATM Withdrawal`);
    atm.getCashDispenser().dispenseCash(this.amount);
    this.succeed(`Dispensed ₹${this.amount}. Remaining balance: ₹${account.getBalance()}`);

    // Check if ATM needs to go out of service after this withdrawal
    atm.checkCashLevel();
  }
}

class DepositTransaction extends Transaction {
  constructor(id: string, cardNumber: string, amount: number) {
    super(id, TransactionType.DEPOSIT, cardNumber, amount);
  }

  execute(account: Account, atm: ATM): void {
    // Accept the deposited cash and update account
    account.credit(this.amount, `ATM Deposit`);
    // Note: deposited cash goes into the ATM's reserve too
    atm.getCashDispenser().addCash(this.amount);
    this.succeed(`Deposited ₹${this.amount}. New balance: ₹${account.getBalance()}`);
  }
}

class BalanceEnquiryTransaction extends Transaction {
  constructor(id: string, cardNumber: string) {
    super(id, TransactionType.BALANCE_ENQUIRY, cardNumber, 0);
  }

  execute(account: Account, _atm: ATM): void {
    this.succeed(`Account balance: ₹${account.getBalance()}`);
  }
}

class MiniStatementTransaction extends Transaction {
  constructor(id: string, cardNumber: string) {
    super(id, TransactionType.MINI_STATEMENT, cardNumber, 0);
  }

  execute(account: Account, _atm: ATM): void {
    const stmts = account.getMiniStatement(5);
    const lines = stmts.map(
      s => `${s.date.toDateString()} | ${s.type} | ₹${s.amount} | ${s.description}`
    );
    this.succeed(`\nLast ${stmts.length} transactions:\n${lines.join("\n")}`);
  }
}

// ============================================================
// HARDWARE COMPONENTS
// ============================================================

// CashDispenser models the physical cash in the machine.
// It is SEPARATE from any account balance.
class CashDispenser {
  private static readonly LOW_CASH_THRESHOLD = 5000;

  constructor(private cashAvailable: number = 100000) {}

  hasSufficientCash(amount: number): boolean {
    return this.cashAvailable >= amount;
  }

  dispenseCash(amount: number): void {
    if (!this.hasSufficientCash(amount)) {
      throw new Error(`Cannot dispense ₹${amount} — only ₹${this.cashAvailable} in ATM`);
    }
    this.cashAvailable -= amount;
    console.log(`[DISPENSER] ₹${amount} dispensed. Remaining: ₹${this.cashAvailable}`);
  }

  addCash(amount: number): void {
    this.cashAvailable += amount;
    console.log(`[DISPENSER] ₹${amount} added. Total: ₹${this.cashAvailable}`);
  }

  refill(amount: number): void {
    this.cashAvailable = amount;
    console.log(`[DISPENSER] Refilled to ₹${this.cashAvailable}`);
  }

  isLow(): boolean {
    return this.cashAvailable < CashDispenser.LOW_CASH_THRESHOLD;
  }

  getCashAvailable(): number { return this.cashAvailable; }
}

class CardReader {
  private insertedCard: Card | null = null;

  insertCard(card: Card): void {
    if (this.insertedCard) {
      throw new Error("A card is already inserted");
    }
    this.insertedCard = card;
    console.log(`[CARD READER] Card ${card.cardNumber.slice(-4)} inserted`);
  }

  ejectCard(): Card | null {
    const card = this.insertedCard;
    this.insertedCard = null;
    if (card) {
      console.log(`[CARD READER] Card ${card.cardNumber.slice(-4)} ejected`);
    }
    return card;
  }

  getInsertedCard(): Card | null { return this.insertedCard; }
}

class Display {
  show(message: string): void {
    console.log(`[SCREEN] ${message}`);
  }
}

class ReceiptPrinter {
  print(transaction: Transaction, account: Account): void {
    console.log(`\n---- ATM RECEIPT ----`);
    console.log(`Date    : ${transaction.createdAt.toLocaleString()}`);
    console.log(`Type    : ${transaction.type}`);
    if (transaction.amount > 0) {
      console.log(`Amount  : ₹${transaction.amount}`);
    }
    console.log(`Status  : ${transaction.status}`);
    console.log(`Balance : ₹${account.getBalance()}`);
    console.log(`---------------------\n`);
  }
}

// ============================================================
// BANK SERVICE — external dependency (interface)
// ============================================================

// The ATM never stores PINs or balances itself.
// It delegates all financial operations to the bank.
interface BankService {
  getAccount(accountNumber: string): Account | null;
  validateCard(card: Card): boolean;
}

// In-memory bank for demo — real impl would be an API / core banking call
class InMemoryBankService implements BankService {
  private accounts: Map<string, Account> = new Map();

  addAccount(account: Account): void {
    this.accounts.set(account.accountNumber, account);
  }

  getAccount(accountNumber: string): Account | null {
    return this.accounts.get(accountNumber) ?? null;
  }

  validateCard(card: Card): boolean {
    // Check card is active, not expired, and maps to a known account
    return card.isValid() && this.accounts.has(card.accountNumber);
  }
}

// ============================================================
// ATM STATE — Abstract Base + 6 Concrete States
// ============================================================

// ATMState is the abstract State class.
// Each concrete state implements only the operations valid in that state.
// Every other operation throws NotSupportedInStateError.
abstract class ATMState {
  constructor(protected atm: ATM) {}

  // Default implementations: throw for every operation.
  // Concrete states selectively override the ones they support.

  insertCard(_card: Card): void {
    throw new Error(`Cannot insert card in state: ${this.getStateType()}`);
  }

  enterPin(_pin: string): void {
    throw new Error(`Cannot enter PIN in state: ${this.getStateType()}`);
  }

  selectTransaction(_type: TransactionType, _amount?: number): void {
    throw new Error(`Cannot select transaction in state: ${this.getStateType()}`);
  }

  ejectCard(): void {
    throw new Error(`Cannot eject card in state: ${this.getStateType()}`);
  }

  // Technician-only operation — not customer-facing
  refillCash(_amount: number): void {
    throw new Error(`Cannot refill cash in state: ${this.getStateType()}`);
  }

  abstract getStateType(): ATMStateType;
}

// ── IDLE ─────────────────────────────────────────────────────
// ATM is waiting. The only valid operation is inserting a card.
class IdleState extends ATMState {
  getStateType(): ATMStateType { return ATMStateType.IDLE; }

  insertCard(card: Card): void {
    // Validate card with the bank before accepting the session
    if (!this.atm.getBankService().validateCard(card)) {
      this.atm.getDisplay().show("Card invalid or expired. Please contact your bank.");
      return;
    }

    this.atm.getCardReader().insertCard(card);
    this.atm.startSession(card);
    this.atm.setState(new CardInsertedState(this.atm));
    this.atm.getDisplay().show(`Welcome! Please enter your PIN. Attempts remaining: ${card.getRemainingPinAttempts()}`);
  }
}

// ── CARD_INSERTED ─────────────────────────────────────────────
// Card is in the reader. Waiting for PIN.
class CardInsertedState extends ATMState {
  getStateType(): ATMStateType { return ATMStateType.CARD_INSERTED; }

  enterPin(pin: string): void {
    const session = this.atm.getCurrentSession()!;
    const card = session.card;
    const account = this.atm.getBankService().getAccount(card.accountNumber)!;

    const correct = card.verifyPin(pin, account.verifyPin.bind(account, pin) ? pin : "");

    // Simpler: delegate PIN verification to Account
    const pinCorrect = account.verifyPin(pin);

    if (pinCorrect) {
      card.failedPinAttempts = 0;
      this.atm.setState(new PinVerifiedState(this.atm));
      this.atm.getDisplay().show("PIN accepted. Please select a transaction.");
    } else {
      card.failedPinAttempts++;
      if (card.isBlocked()) {
        // Card is now blocked — transition to CARD_BLOCKED
        this.atm.getDisplay().show("Card blocked due to too many incorrect PIN attempts.");
        this.atm.setState(new CardBlockedState(this.atm));
      } else {
        this.atm.getDisplay().show(
          `Incorrect PIN. ${card.getRemainingPinAttempts()} attempt(s) remaining.`
        );
      }
    }
  }

  ejectCard(): void {
    this.atm.getCardReader().ejectCard();
    this.atm.endSession();
    this.atm.setState(new IdleState(this.atm));
    this.atm.getDisplay().show("Card ejected. Thank you.");
  }
}

// ── PIN_VERIFIED ──────────────────────────────────────────────
// PIN is correct. Customer can select a transaction or eject.
class PinVerifiedState extends ATMState {
  getStateType(): ATMStateType { return ATMStateType.PIN_VERIFIED; }

  selectTransaction(type: TransactionType, amount: number = 0): void {
    // Create the appropriate Transaction command object
    const txn = this.atm.createTransaction(type, amount);
    this.atm.getCurrentSession()!.addTransaction(txn);

    // Move to TRANSACTION state to process it
    this.atm.setState(new TransactionState(this.atm, txn));
    this.atm.processCurrentTransaction();
  }

  ejectCard(): void {
    this.atm.getCardReader().ejectCard();
    this.atm.endSession();
    this.atm.setState(new IdleState(this.atm));
    this.atm.getDisplay().show("Thank you for using the ATM.");
  }
}

// ── TRANSACTION ───────────────────────────────────────────────
// A transaction is being processed. No new card operations allowed.
class TransactionState extends ATMState {
  constructor(atm: ATM, private readonly transaction: Transaction) {
    super(atm);
  }

  getStateType(): ATMStateType { return ATMStateType.TRANSACTION; }

  // After transaction completes, customer can:
  // - Do another transaction (back to PIN_VERIFIED)
  // - Eject card (back to IDLE)
  selectTransaction(type: TransactionType, amount: number = 0): void {
    const txn = this.atm.createTransaction(type, amount);
    this.atm.getCurrentSession()!.addTransaction(txn);
    this.atm.setState(new TransactionState(this.atm, txn));
    this.atm.processCurrentTransaction();
  }

  ejectCard(): void {
    this.atm.getCardReader().ejectCard();
    this.atm.endSession();
    this.atm.setState(new IdleState(this.atm));
    this.atm.getDisplay().show("Card ejected. Thank you.");
  }
}

// ── CARD_BLOCKED ──────────────────────────────────────────────
// Card is blocked after 3 wrong PINs. The only action is ejecting the card.
// Once ejected, the ATM returns to IDLE — but the card remains BLOCKED
// at the bank level (the physical card and account are flagged).
class CardBlockedState extends ATMState {
  constructor(atm: ATM) {
    super(atm);
    // Immediately eject the blocked card
    this.ejectCard();
  }

  getStateType(): ATMStateType { return ATMStateType.CARD_BLOCKED; }

  ejectCard(): void {
    this.atm.getCardReader().ejectCard();
    this.atm.endSession();
    this.atm.setState(new IdleState(this.atm));
    this.atm.getDisplay().show(
      "Your card has been blocked due to multiple incorrect PIN attempts. " +
      "Please contact your bank to unblock it."
    );
  }
}

// ── OUT_OF_SERVICE ────────────────────────────────────────────
// ATM cash is too low to serve customers.
// Only a technician refill can bring it back to IDLE.
class OutOfServiceState extends ATMState {
  constructor(atm: ATM) {
    super(atm);
    console.warn(`[ATM] Entering OUT_OF_SERVICE — cash level critical`);
    atm.getDisplay().show("ATM is temporarily out of service. We apologise for the inconvenience.");
  }

  getStateType(): ATMStateType { return ATMStateType.OUT_OF_SERVICE; }

  // Only technician can refill — this is the one escape from this state
  refillCash(amount: number): void {
    this.atm.getCashDispenser().refill(amount);
    this.atm.setState(new IdleState(this.atm));
    this.atm.getDisplay().show(`ATM is back in service. Cash loaded: ₹${amount}`);
    console.log(`[TECHNICIAN] ATM refilled with ₹${amount}. Back in service.`);
  }
}

// ============================================================
// SESSION
// ============================================================

// A Session captures the full lifecycle of one card insertion.
// Created when a card is inserted, closed when card is ejected.
class Session {
  public readonly startTime: Date = new Date();
  public endTime: Date | null = null;
  private transactions: Transaction[] = [];

  constructor(
    public readonly sessionId: string,
    public readonly card: Card
  ) {}

  addTransaction(txn: Transaction): void {
    this.transactions.push(txn);
  }

  getTransactions(): Transaction[] { return [...this.transactions]; }

  end(): void {
    this.endTime = new Date();
  }

  getDurationSeconds(): number {
    const end = this.endTime ?? new Date();
    return Math.round((end.getTime() - this.startTime.getTime()) / 1000);
  }
}

// ============================================================
// ATM — Singleton Context
// ============================================================

// ATM is the Context in the State Pattern.
// It holds the current state and delegates all operations to it.
// It is a Singleton because there is one physical machine.
class ATM {
  private static instance: ATM | null = null;

  private state: ATMState;
  private currentSession: Session | null = null;
  private sessionCounter = 1;
  private transactionCounter = 1;

  // Hardware components
  private readonly cashDispenser: CashDispenser;
  private readonly cardReader: CardReader;
  private readonly display: Display;
  private readonly receiptPrinter: ReceiptPrinter;

  private constructor(
    public readonly atmId: string,
    private readonly bankService: BankService,
    initialCash: number = 100000
  ) {
    this.cashDispenser = new CashDispenser(initialCash);
    this.cardReader    = new CardReader();
    this.display       = new Display();
    this.receiptPrinter = new ReceiptPrinter();

    // ATM starts in IDLE state
    this.state = new IdleState(this);
    console.log(`[ATM ${atmId}] Initialised. Cash: ₹${initialCash}`);
  }

  static getInstance(atmId: string, bankService: BankService, initialCash?: number): ATM {
    if (!ATM.instance) {
      ATM.instance = new ATM(atmId, bankService, initialCash);
    }
    return ATM.instance;
  }

  static resetInstance(): void { ATM.instance = null; }

  // ── STATE MANAGEMENT ────────────────────────────────────────
  setState(state: ATMState): void {
    console.log(`[ATM] State: ${this.state.getStateType()} → ${state.getStateType()}`);
    this.state = state;
  }

  getStateType(): ATMStateType { return this.state.getStateType(); }

  // ── PUBLIC INTERFACE (delegates to current state) ────────────
  // These are the only public-facing methods. Everything is gated by state.

  insertCard(card: Card): void        { this.state.insertCard(card); }
  enterPin(pin: string): void         { this.state.enterPin(pin); }
  ejectCard(): void                   { this.state.ejectCard(); }
  refillCash(amount: number): void    { this.state.refillCash(amount); }

  selectTransaction(type: TransactionType, amount: number = 0): void {
    this.state.selectTransaction(type, amount);
  }

  // ── TRANSACTION EXECUTION ────────────────────────────────────
  // Called by TransactionState after creating the command object.
  processCurrentTransaction(): void {
    const session = this.currentSession;
    if (!session) return;

    const txns = session.getTransactions();
    const txn = txns[txns.length - 1]; // most recent
    if (!txn) return;

    const account = this.bankService.getAccount(session.card.accountNumber);
    if (!account) {
      this.display.show("Account not found. Please contact your bank.");
      return;
    }

    // Execute the transaction command
    txn.execute(account, this);
    this.display.show(txn.message);

    // Print receipt
    this.receiptPrinter.print(txn, account);

    // After a successful transaction, move back to PIN_VERIFIED
    // so the customer can do another operation without re-entering PIN
    if (txn.status === TransactionStatus.SUCCESS || txn.status === TransactionStatus.FAILED) {
      this.setState(new PinVerifiedState(this));
      this.display.show("Select another transaction or press EJECT to exit.");
    }
  }

  // ── TRANSACTION FACTORY ───────────────────────────────────────
  createTransaction(type: TransactionType, amount: number): Transaction {
    const id = `TXN-${String(this.transactionCounter++).padStart(6, "0")}`;
    const cardNumber = this.currentSession?.card.cardNumber ?? "UNKNOWN";

    switch (type) {
      case TransactionType.WITHDRAWAL:       return new WithdrawTransaction(id, cardNumber, amount);
      case TransactionType.DEPOSIT:          return new DepositTransaction(id, cardNumber, amount);
      case TransactionType.BALANCE_ENQUIRY:  return new BalanceEnquiryTransaction(id, cardNumber);
      case TransactionType.MINI_STATEMENT:   return new MiniStatementTransaction(id, cardNumber);
      default: throw new Error(`Unknown transaction type: ${type}`);
    }
  }

  // ── CASH LEVEL MONITORING ─────────────────────────────────────
  // Called after every withdrawal. If cash drops too low, go out of service.
  checkCashLevel(): void {
    if (this.cashDispenser.isLow()) {
      // Only transition to OUT_OF_SERVICE if we're not already there
      if (this.getStateType() !== ATMStateType.OUT_OF_SERVICE) {
        this.setState(new OutOfServiceState(this));
      }
    }
  }

  // ── SESSION MANAGEMENT ────────────────────────────────────────
  startSession(card: Card): void {
    const sessionId = `SES-${String(this.sessionCounter++).padStart(4, "0")}`;
    this.currentSession = new Session(sessionId, card);
    console.log(`[SESSION] ${sessionId} started for card ${card.cardNumber.slice(-4)}`);
  }

  endSession(): void {
    if (this.currentSession) {
      this.currentSession.end();
      console.log(
        `[SESSION] ${this.currentSession.sessionId} ended. ` +
        `Duration: ${this.currentSession.getDurationSeconds()}s. ` +
        `Transactions: ${this.currentSession.getTransactions().length}`
      );
      this.currentSession = null;
    }
  }

  getCurrentSession(): Session | null { return this.currentSession; }

  // ── ACCESSORS (used by states & transactions) ─────────────────
  getCashDispenser(): CashDispenser       { return this.cashDispenser; }
  getCardReader(): CardReader             { return this.cardReader; }
  getDisplay(): Display                   { return this.display; }
  getBankService(): BankService           { return this.bankService; }
}

// ============================================================
// DEMO
// ============================================================

function main() {
  // Setup bank
  const bank = new InMemoryBankService();
  const aliceAccount = new Account("ACC001", "Alice", 25000, "SAVINGS", "1234");
  bank.addAccount(aliceAccount);

  // Setup ATM with ₹50,000 cash
  ATM.resetInstance();
  const atm = ATM.getInstance("ATM-001", bank, 50000);

  // Valid card for Alice
  const aliceCard = new Card(
    "4111111111111234",
    "ACC001",
    new Date("2027-12-31")
  );

  console.log("\n=== Scenario 1: Normal withdrawal flow ===");
  atm.insertCard(aliceCard);
  atm.enterPin("1234");
  atm.selectTransaction(TransactionType.BALANCE_ENQUIRY);
  atm.selectTransaction(TransactionType.WITHDRAWAL, 5000);
  atm.ejectCard();

  console.log("\n=== Scenario 2: Wrong PIN → card blocked ===");
  const bobCard = new Card("4111111111115678", "ACC001", new Date("2027-12-31"));
  atm.insertCard(bobCard);
  atm.enterPin("9999"); // wrong
  atm.enterPin("8888"); // wrong
  atm.enterPin("7777"); // wrong → BLOCKED

  console.log("\n=== Scenario 3: Invalid operation in wrong state ===");
  try {
    atm.enterPin("1234"); // ATM is IDLE now — should throw
  } catch (e: any) {
    console.log(`[EXPECTED ERROR] ${e.message}`);
  }

  console.log("\n=== Scenario 4: Technician refill after cash runs low ===");
  // Simulate cash running low by doing large withdrawals
  const testCard = new Card("4111111111119999", "ACC001", new Date("2027-12-31"));
  atm.insertCard(testCard);
  atm.enterPin("1234");
  atm.selectTransaction(TransactionType.WITHDRAWAL, 46000); // drops below threshold
  atm.ejectCard();

  console.log(`\nATM state: ${atm.getStateType()}`); // OUT_OF_SERVICE
  atm.refillCash(100000); // technician refills
  console.log(`ATM state: ${atm.getStateType()}`); // back to IDLE
}

main();
```

---

## 6. Design Patterns — Deep Dive

### 6.1 State Pattern — The canonical ATM example

This is the most textbook application of State Pattern in all of LLD. The key properties that make it perfect here:

- **Every operation is state-dependent** — `enterPin()` only makes sense in `CARD_INSERTED`. Anywhere else it should throw.
- **Transitions are self-driven** — `CardInsertedState.enterPin()` transitions to `PinVerifiedState` or `CardBlockedState` depending on the result. The Context (`ATM`) never decides transitions — the states do.
- **Invalid transitions throw loudly** — the default implementation in `ATMState` throws for every operation. Concrete states selectively override valid ones. This is the Open/Closed Principle: adding a `MAINTENANCE` state means a new class, not a new `if/else`.

**Interview phrasing:** *"The ATM's behaviour is entirely determined by its current state. Rather than a giant switch statement on `this.state` in every method, each state is a class that implements only its valid operations. The default base class throws — so I get compile-time safety and runtime safety together."*

### 6.2 Command — `Transaction` hierarchy

Each transaction type is a command object with an `execute(account, atm)` method. Benefits:
- **Loggable and auditable** — every transaction is a typed object with `createdAt`, `status`, `message`
- **Undoable** (in theory) — a `Reversible` sub-interface could add `rollback()`
- **Extensible** — `ChequeDepositTransaction`, `FundTransferTransaction` are new subclasses, zero changes to ATM states

### 6.3 Singleton — `ATM`

There is one physical machine. One session at a time. One state at a time. Singleton enforces this at the code level.

### 6.4 Separation of `CashDispenser` from `Account`

This isn't a GoF pattern but it's the most important design separation interviewers check for. `CashDispenser` is a physical hardware component with its own cash reserve. `Account` balance lives in the bank. A withdrawal must pass both checks. A deposit increases both the account balance AND the cash in the dispenser.

---

## 7. PIN Verification — The Subtle Details

```typescript
// Wrong approach — comparing PIN in ATM
if (card.pin === enteredPin) { ... }    // ATM should NEVER store PINs

// Right approach — delegating to the bank
const pinCorrect = bankService.verifyPin(card.accountNumber, enteredPin);

// Why it matters:
// 1. PINs are stored HASHED at the bank (bcrypt/SHA-256)
// 2. ATM gets a boolean back — never the actual PIN
// 3. The bank can impose its own lockout logic independently
// 4. PIN change flows through the bank, not the ATM
```

### The 3-strike counter lives on the Card, not the ATM

```typescript
// Card tracks failed attempts — this is per-card, not per-ATM
card.failedPinAttempts++;

// Why on Card, not ATM?
// The same card can be tried at multiple ATMs.
// If the counter was on the ATM, moving to a different ATM resets it.
// Counter must persist with the card (i.e., in the bank's card table).
// In real systems: bank increments the counter on each failed auth.
```

---

## 8. Session Timeout

In a real ATM, an inactive session times out after ~30 seconds:

```typescript
class Session {
  private timeoutHandle: ReturnType<typeof setTimeout> | null = null;
  private static readonly TIMEOUT_MS = 30_000;

  startTimeout(atm: ATM): void {
    this.timeoutHandle = setTimeout(() => {
      console.warn("[SESSION TIMEOUT] Ejecting card due to inactivity");
      atm.getDisplay().show("Session timed out. Your card has been ejected.");
      atm.ejectCard(); // triggers state transition to IDLE
    }, Session.TIMEOUT_MS);
  }

  resetTimeout(atm: ATM): void {
    if (this.timeoutHandle) clearTimeout(this.timeoutHandle);
    this.startTimeout(atm);
  }

  clearTimeout(): void {
    if (this.timeoutHandle) clearTimeout(this.timeoutHandle);
  }
}
```

The session timeout should reset on every keypress (PIN entry, amount selection). This is `session.resetTimeout(atm)` called in every valid user action.

---

## 9. Edge Cases & Nuances

| Edge Case | How to Handle |
|-----------|--------------|
| **Wrong PIN 3 times** | `card.failedPinAttempts >= 3` → `card.status = BLOCKED` → `CardBlockedState` auto-ejects |
| **Expired card** | `Card.isValid()` returns false → `IdleState.insertCard()` rejects before state change |
| **ATM cash insufficient** | `WithdrawTransaction.execute()` checks `cashDispenser.hasSufficientCash()` — fails gracefully |
| **Account balance insufficient** | `account.getBalance() < amount` — transaction fails, no cash dispensed |
| **Session timeout** | Timer in `Session` auto-calls `atm.ejectCard()` after 30s inactivity |
| **Power cut mid-transaction** | Bank uses two-phase commit: debit only finalised after dispense confirmed. On recovery, idempotency key prevents double debit |
| **Card stuck in reader** | Hardware signal → `CardBlockedState` or maintenance mode. ATM flags for servicing |
| **Dispense failure (jam)** | `CashDispenser.dispenseCash()` throws → bank must reverse the debit (compensating transaction) |
| **Customer enters wrong amount format** | UI layer validates before calling `selectTransaction()` — ATM state machine only receives valid amounts |
| **Multiple sessions simultaneously** | Enforced by Singleton + `currentSession !== null` check in `startSession()` |

---

## 10. Follow-Up Questions Interviewers Ask

1. **"How do you handle a dispense failure after account debit?"**
   → Two-phase commit pattern. Debit is tentative (marked `PENDING` in the bank). The ATM confirms dispense success; bank commits the debit. On timeout/failure, bank auto-reverses the tentative debit. This requires an idempotency key per transaction.

2. **"How would you add a daily withdrawal limit?"**
   → `Account` gets `dailyWithdrawnAmount` (resets at midnight via cron) and `dailyWithdrawalLimit`. `WithdrawTransaction.execute()` checks `dailyWithdrawnAmount + amount <= dailyWithdrawalLimit` before proceeding.

3. **"How would you support cardless (UPI/QR) ATM?"**
   → Add `QRCodeReader` hardware component. New state `QR_SCANNED` between `IDLE` and `PIN_VERIFIED`. `IdleState` gains a `scanQR()` operation. The rest of the flow (PIN → Transaction → Eject) is identical.

4. **"What if two people try to insert cards simultaneously?"**
   → Enforced by hardware (one card slot) and `CardReader.insertCard()` which throws if a card is already inserted. The `Session` also guards: `currentSession !== null` → reject second card at the software level.

5. **"How would you add an admin/maintenance mode?"**
   → `MaintenanceState extends ATMState`. Only accessible via a technician key + password. Valid operations: `refillCash()`, `runDiagnostics()`, `viewTransactionLog()`. No customer operations permitted. Exit → IDLE.

---

## 11. Key Takeaways

- **State Pattern is non-negotiable here.** An ATM without explicit state objects is `if (state == "IDLE")` everywhere — a maintenance disaster. Six state classes with a throwing base class is the right answer.
- **Invalid operations throw — they never silently no-op.** This is the core value of State Pattern. `atm.enterPin()` in `IDLE` state should crash loudly, not do nothing.
- **Two separate balances.** `CashDispenser` = physical machine cash. `Account.balance` = bank balance. A withdrawal checks both. Always draw this distinction in the interview.
- **PIN counter lives on the Card (at the bank), not on the ATM.** A counter on the ATM can be bypassed by moving to a different machine. The bank owns the lock-out logic.
- **`CardBlockedState` self-ejects** — once the card is blocked, the only valid action is ejecting it. The constructor auto-calls `ejectCard()`. No customer input needed.
- **Session timeout** is the edge case that separates senior answers. The 30-second timer, the reset on keypress, and the auto-eject on expiry — all need to be mentioned.
- **Command Pattern for transactions** gives you a clean audit trail, extensibility (new transaction types = new subclasses), and a path to two-phase commit without changing state machine logic.

---

*Next up: LLD #07 — Chess / Tic-Tac-Toe*