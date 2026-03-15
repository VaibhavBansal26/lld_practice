# 5. ATM Machine

Code

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass
from decimal import Decimal
from enum import Enum
from hashlib import md5
from typing import Dict, Optional


# =========================
# Enums
# =========================

class AccountType(str, Enum):
    CHECKING = "CHECKING"
    SAVING = "SAVING"


class TransactionType(str, Enum):
    WITHDRAW = "WITHDRAW"
    DEPOSIT = "DEPOSIT"


# =========================
# Account
# =========================

class Account:
    def __init__(
        self,
        account_number: str,
        account_type: AccountType,
        card_number: str,
        pin: str,
    ):
        self.balance = Decimal("0.00")
        self.account_number = account_number
        self.card_number = card_number
        self.card_pin_hash = self._calculate_md5(pin)
        self.account_type = account_type

    def validate_pin(self, pin_number: str) -> bool:
        entry_pin_hash = self._calculate_md5(pin_number)
        return self.card_pin_hash == entry_pin_hash

    def update_balance_with_transaction(self, balance_change: Decimal) -> None:
        self.balance += balance_change

    def get_balance(self) -> Decimal:
        return self.balance

    def _calculate_md5(self, value: str) -> bytes:
        return md5(value.encode("utf-8")).digest()

    def __repr__(self) -> str:
        return (
            f"Account(account_number={self.account_number}, "
            f"card_number={self.card_number}, "
            f"account_type={self.account_type}, "
            f"balance={self.balance})"
        )


# =========================
# Bank Interface / Bank
# =========================

class BankInterface(ABC):
    @abstractmethod
    def add_account(
        self,
        account_number: str,
        account_type: AccountType,
        card_number: str,
        pin: str,
    ) -> None:
        pass

    @abstractmethod
    def validate_card(self, card_number: str) -> bool:
        pass

    @abstractmethod
    def check_pin(self, card_number: str, pin_number: str) -> bool:
        pass

    @abstractmethod
    def get_account_by_account_number(self, account_number: str) -> Optional[Account]:
        pass

    @abstractmethod
    def get_account_by_card(self, card_number: str) -> Optional[Account]:
        pass

    @abstractmethod
    def withdraw_funds(self, account: Account, amount: Decimal) -> bool:
        pass


class Bank(BankInterface):
    def __init__(self):
        self.accounts: Dict[str, Account] = {}
        self.account_by_card: Dict[str, Account] = {}

    def add_account(
        self,
        account_number: str,
        account_type: AccountType,
        card_number: str,
        pin: str,
    ) -> None:
        new_account = Account(account_number, account_type, card_number, pin)
        self.accounts[new_account.account_number] = new_account
        self.account_by_card[new_account.card_number] = new_account

    def validate_card(self, card_number: str) -> bool:
        return self.get_account_by_card(card_number) is not None

    def check_pin(self, card_number: str, pin_number: str) -> bool:
        account = self.get_account_by_card(card_number)
        if account is not None:
            return account.validate_pin(pin_number)
        return False

    def get_account_by_account_number(self, account_number: str) -> Optional[Account]:
        return self.accounts.get(account_number)

    def get_account_by_card(self, card_number: str) -> Optional[Account]:
        return self.account_by_card.get(card_number)

    def withdraw_funds(self, account: Account, amount: Decimal) -> bool:
        if account.get_balance() >= amount:
            account.update_balance_with_transaction(-amount)
            return True
        return False


# =========================
# Transactions
# =========================

class Transaction(ABC):
    @abstractmethod
    def get_type(self) -> TransactionType:
        pass

    @abstractmethod
    def validate_transaction(self) -> bool:
        pass

    @abstractmethod
    def execute_transaction(self) -> None:
        pass


class WithdrawTransaction(Transaction):
    def __init__(self, account: Account, amount: Decimal):
        self.account = account
        self.amount = amount

        if not self.validate_transaction():
            raise ValueError("Cannot complete withdrawal: Insufficient funds in account")

    def get_type(self) -> TransactionType:
        return TransactionType.WITHDRAW

    def validate_transaction(self) -> bool:
        return self.account is not None and self.account.get_balance() >= self.amount

    def execute_transaction(self) -> None:
        self.account.update_balance_with_transaction(-self.amount)


class DepositTransaction(Transaction):
    def __init__(self, account: Account, amount: Decimal):
        self.account = account
        self.amount = amount

    def get_type(self) -> TransactionType:
        return TransactionType.DEPOSIT

    def validate_transaction(self) -> bool:
        return True

    def execute_transaction(self) -> None:
        self.account.update_balance_with_transaction(self.amount)


# =========================
# Hardware Components
# =========================

class CardProcessor:
    def __init__(self):
        self.card_number: Optional[str] = None

    def insert_card(self, card_number: str) -> None:
        self.card_number = card_number

    def eject_card(self) -> None:
        self.card_number = None

    def get_card_number(self) -> Optional[str]:
        return self.card_number


class DepositBox:
    def collect_deposit(self, amount: Decimal) -> Decimal:
        return amount


class CashDispenser:
    def dispense_cash(self, amount: Decimal) -> None:
        print(f"Dispensing cash: {amount}")


class Keypad:
    def get_input(self, prompt: str = "") -> str:
        return input(prompt)


class Display:
    def show_message(self, message: str) -> None:
        print(message)


# =========================
# ATM State Machine
# =========================

class ATMState:
    @staticmethod
    def _render_default_action(atm_machine: "ATMMachine") -> None:
        atm_machine.get_display().show_message("Invalid action, please try again.")

    def process_card_insertion(self, atm_machine: "ATMMachine", card_number: str) -> None:
        self._render_default_action(atm_machine)

    def process_card_ejection(self, atm_machine: "ATMMachine") -> None:
        self._render_default_action(atm_machine)

    def process_pin_entry(self, atm_machine: "ATMMachine", pin: str) -> None:
        self._render_default_action(atm_machine)

    def process_withdrawal_request(self, atm_machine: "ATMMachine") -> None:
        self._render_default_action(atm_machine)

    def process_deposit_request(self, atm_machine: "ATMMachine") -> None:
        self._render_default_action(atm_machine)

    def process_amount_entry(self, atm_machine: "ATMMachine", amount: Decimal) -> None:
        self._render_default_action(atm_machine)

    def process_deposit_collection(self, atm_machine: "ATMMachine", amount: Decimal) -> None:
        self._render_default_action(atm_machine)


class IdleState(ATMState):
    def process_card_insertion(self, atm_machine: "ATMMachine", card_number: str) -> None:
        if atm_machine.get_bank_interface().validate_card(card_number):
            atm_machine.get_card_processor().insert_card(card_number)
            atm_machine.get_display().show_message("Please enter your PIN")
            atm_machine.transition_to_state(PinEntryState())
        else:
            atm_machine.get_display().show_message("Invalid card. Please try again.")


class PinEntryState(ATMState):
    def process_card_ejection(self, atm_machine: "ATMMachine") -> None:
        atm_machine.get_card_processor().eject_card()
        atm_machine.get_display().show_message("Card ejected")
        atm_machine.transition_to_state(IdleState())

    def process_pin_entry(self, atm_machine: "ATMMachine", pin: str) -> None:
        card_number = atm_machine.get_card_processor().get_card_number()
        if card_number is None:
            atm_machine.get_display().show_message("No card inserted")
            atm_machine.transition_to_state(IdleState())
            return

        if atm_machine.get_bank_interface().check_pin(card_number, pin):
            atm_machine.get_display().show_message("PIN verified. Select transaction.")
            atm_machine.transition_to_state(TransactionSelectionState())
        else:
            atm_machine.get_display().show_message("Invalid PIN. Please try again.")


class TransactionSelectionState(ATMState):
    def process_card_ejection(self, atm_machine: "ATMMachine") -> None:
        atm_machine.get_card_processor().eject_card()
        atm_machine.get_display().show_message("Session ended. Card ejected.")
        atm_machine.transition_to_state(IdleState())

    def process_withdrawal_request(self, atm_machine: "ATMMachine") -> None:
        atm_machine.get_display().show_message("Please enter withdrawal amount")
        atm_machine.transition_to_state(WithdrawAmountEntryState())

    def process_deposit_request(self, atm_machine: "ATMMachine") -> None:
        atm_machine.get_display().show_message("Please insert deposit amount")
        atm_machine.transition_to_state(DepositCollectionState())


class WithdrawAmountEntryState(ATMState):
    def process_card_ejection(self, atm_machine: "ATMMachine") -> None:
        atm_machine.get_card_processor().eject_card()
        atm_machine.get_display().show_message("Transaction cancelled, card ejected")
        atm_machine.transition_to_state(IdleState())

    def process_amount_entry(self, atm_machine: "ATMMachine", amount: Decimal) -> None:
        card_number = atm_machine.get_card_processor().get_card_number()
        if card_number is None:
            atm_machine.get_display().show_message("No card inserted")
            atm_machine.transition_to_state(IdleState())
            return

        account = atm_machine.get_bank_interface().get_account_by_card(card_number)
        if account is None:
            atm_machine.get_display().show_message("Account not found")
            atm_machine.transition_to_state(IdleState())
            return

        is_success = atm_machine.get_bank_interface().withdraw_funds(account, amount)

        if is_success:
            atm_machine.get_cash_dispenser().dispense_cash(amount)
            atm_machine.get_display().show_message("Please take your cash.")
        else:
            atm_machine.get_display().show_message("Insufficient funds, please try again.")

        atm_machine.transition_to_state(TransactionSelectionState())


class DepositCollectionState(ATMState):
    def process_card_ejection(self, atm_machine: "ATMMachine") -> None:
        atm_machine.get_card_processor().eject_card()
        atm_machine.get_display().show_message("Transaction cancelled, card ejected")
        atm_machine.transition_to_state(IdleState())

    def process_deposit_collection(self, atm_machine: "ATMMachine", amount: Decimal) -> None:
        card_number = atm_machine.get_card_processor().get_card_number()
        if card_number is None:
            atm_machine.get_display().show_message("No card inserted")
            atm_machine.transition_to_state(IdleState())
            return

        account = atm_machine.get_bank_interface().get_account_by_card(card_number)
        if account is None:
            atm_machine.get_display().show_message("Account not found")
            atm_machine.transition_to_state(IdleState())
            return

        collected_amount = atm_machine.get_deposit_box().collect_deposit(amount)
        transaction = DepositTransaction(account, collected_amount)
        transaction.execute_transaction()

        atm_machine.get_display().show_message("Deposit successful.")
        atm_machine.transition_to_state(TransactionSelectionState())


# =========================
# ATM Machine
# =========================

class ATMMachine:
    def __init__(
        self,
        bank: Bank,
        card_processor: CardProcessor,
        deposit_box: DepositBox,
        cash_dispenser: CashDispenser,
        keypad: Keypad,
        display: Display,
    ):
        self.bank = bank
        self.card_processor = card_processor
        self.deposit_box = deposit_box
        self.cash_dispenser = cash_dispenser
        self.keypad = keypad
        self.display = display
        self.state: ATMState = IdleState()

    def insert_card(self, card_number: str) -> None:
        self.state.process_card_insertion(self, card_number)

    def eject_card(self) -> None:
        self.state.process_card_ejection(self)

    def enter_pin(self, pin: str) -> None:
        self.state.process_pin_entry(self, pin)

    def withdraw_request(self) -> None:
        self.state.process_withdrawal_request(self)

    def deposit_request(self) -> None:
        self.state.process_deposit_request(self)

    def enter_amount(self, amount: Decimal) -> None:
        self.state.process_amount_entry(self, amount)

    def collect_deposit(self, amount: Decimal) -> None:
        self.state.process_deposit_collection(self, amount)

    def get_display(self) -> Display:
        return self.display

    def get_cash_dispenser(self) -> CashDispenser:
        return self.cash_dispenser

    def get_bank_interface(self) -> BankInterface:
        return self.bank

    def get_card_processor(self) -> CardProcessor:
        return self.card_processor

    def get_keypad(self) -> Keypad:
        return self.keypad

    def get_deposit_box(self) -> DepositBox:
        return self.deposit_box

    def transition_to_state(self, next_state: ATMState) -> None:
        self.state = next_state

    def get_current_state(self) -> ATMState:
        return self.state


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    bank = Bank()
    bank.add_account("ACC1001", AccountType.CHECKING, "CARD1001", "1234")
    bank.add_account("ACC1002", AccountType.SAVING, "CARD1002", "5678")

    # preload some money
    account_1 = bank.get_account_by_card("CARD1001")
    if account_1:
        account_1.update_balance_with_transaction(Decimal("1000.00"))

    atm = ATMMachine(
        bank=bank,
        card_processor=CardProcessor(),
        deposit_box=DepositBox(),
        cash_dispenser=CashDispenser(),
        keypad=Keypad(),
        display=Display(),
    )

    print("=== ATM Session Start ===")
    atm.insert_card("CARD1001")
    atm.enter_pin("1234")
    atm.withdraw_request()
    atm.enter_amount(Decimal("150.00"))
    atm.deposit_request()
    atm.collect_deposit(Decimal("50.00"))
    atm.eject_card()

    print("\n=== Final Account Balance ===")
    updated_account = bank.get_account_by_card("CARD1001")
    if updated_account:
        print(updated_account.get_balance())
```

## Design Patterns, SOLID Principles, and OOP Concepts Used in the ATM LLD

# 1. Design Patterns Used

## 1.1 State Pattern

**Where used**
- `ATMState`
- `IdleState`
- `PinEntryState`
- `TransactionSelectionState`
- `WithdrawAmountEntryState`
- `DepositCollectionState`

**Why we used it**
The ATM behaves differently depending on the current step of the user session.

Examples:
- in `IdleState`, only card insertion is valid
- in `PinEntryState`, PIN entry is valid
- in `TransactionSelectionState`, deposit or withdrawal can be selected
- in `WithdrawAmountEntryState`, amount entry is expected

Instead of putting all ATM behavior inside one large class with many `if else` checks, the logic is separated into state classes.

**Benefits**
- cleaner control flow
- easier to extend
- easier to test each stage independently

**Simple reason**  
When object behavior changes based on state, State pattern is a strong fit.

---

## 1.2 Facade Pattern

**Where used**
- `ATMMachine`

**Why we used it**
`ATMMachine` acts as the central interface for the whole ATM flow.

Clients do not directly coordinate:
- `Bank`
- `CardProcessor`
- `CashDispenser`
- `DepositBox`
- `Display`
- state objects

Instead, they use methods like:
- `insert_card()`
- `enter_pin()`
- `withdraw_request()`
- `deposit_request()`
- `enter_amount()`
- `collect_deposit()`
- `eject_card()`

This hides internal complexity and provides a single entry point.

**Simple reason**  
Facade makes a complex subsystem easier to use.

---

## 1.3 Strategy-like Transaction Abstraction

**Where used**
- `Transaction`
- `WithdrawTransaction`
- `DepositTransaction`

**Why we used it**
Different transaction types share a common contract:
- get transaction type
- validate transaction
- execute transaction

This allows transaction behavior to be encapsulated in separate classes.

Examples:
- withdrawal checks balance before execution
- deposit is always valid and simply adds funds

This is close to a Strategy style design because the behavior varies by transaction type but follows a common interface.

**Simple reason**  
A common transaction contract allows different financial operations to be handled uniformly.

---

## 1.4 Interface-based Bank Abstraction

**Where used**
- `BankInterface`
- `Bank`

**Why we used it**
The ATM should depend on banking operations, not one concrete bank implementation.

This makes it possible to replace:
- local in-memory bank
- networked bank service
- mock bank for tests

without changing the ATM flow logic.

**Simple reason**  
Abstraction helps decouple ATM logic from bank implementation details.

---

## 1.5 Manager / Coordinator Pattern

**Where used**
- `ATMMachine`
- `Bank`

**Why we used it**

### `ATMMachine`
Coordinates:
- hardware interaction
- state transitions
- user flow
- calls to bank operations

### `Bank`
Coordinates:
- account storage
- card validation
- PIN checks
- withdrawals

These classes organize workflows rather than just storing raw data.

**Simple reason**  
Coordinator classes keep multi-step workflows clean and centralized.

---

# 2. SOLID Principles Used

## 2.1 Single Responsibility Principle (SRP)

**Meaning**
A class should have one reason to change.

**Where used**

### `Account`
Only handles account data, PIN validation, and balance updates.

### `Bank`
Only handles banking operations like account lookup, card validation, and withdrawals.

### `WithdrawTransaction`
Only handles withdrawal transaction rules.

### `DepositTransaction`
Only handles deposit transaction rules.

### `CardProcessor`
Only handles inserted card data.

### `CashDispenser`
Only handles cash dispensing.

### `DepositBox`
Only handles deposit collection.

### `Display`
Only handles user messages.

### State classes
Each state only handles one part of the ATM user flow.

### `ATMMachine`
Only coordinates the ATM workflow and hardware components.

**Why this is good**
Each class remains focused and easier to understand, test, and maintain.

---

## 2.2 Open Closed Principle (OCP)

**Meaning**
Software should be open for extension but closed for modification.

**Where used**
The design allows adding new functionality without rewriting stable code.

Examples:
- add a `TransferTransaction`
- add a `BalanceInquiryState`
- add a `ChangePinState`
- add a different `BankInterface` implementation

Because state logic and transaction logic are abstracted, new behaviors can be added with minimal change to the core ATM structure.

**Why this is good**
The system is easier to evolve over time.

---

## 2.3 Liskov Substitution Principle (LSP)

**Meaning**
Subclasses should be replaceable for their base type.

**Where used**
Any subclass of `ATMState` can replace another where `ATMState` is expected.

Examples:
- `IdleState`
- `PinEntryState`
- `TransactionSelectionState`
- `WithdrawAmountEntryState`
- `DepositCollectionState`

Also, any implementation of `Transaction` can be used wherever a `Transaction` is expected.

Examples:
- `WithdrawTransaction`
- `DepositTransaction`

And any implementation of `BankInterface` can replace `Bank`.

**Why this is good**
The rest of the system works correctly with any valid implementation of the abstraction.

---

## 2.4 Interface Segregation Principle (ISP)

**Meaning**
Clients should not be forced to depend on methods they do not use.

**Where used**
The interfaces are small and focused.

### `Transaction`
Only defines:
- `get_type()`
- `validate_transaction()`
- `execute_transaction()`

### `BankInterface`
Only defines operations the ATM actually needs.

The ATM hardware classes are also split into focused components instead of one giant hardware interface.

**Why this is good**
Each class only depends on the behavior it actually uses.

---

## 2.5 Dependency Inversion Principle (DIP)

**Meaning**
High level modules should depend on abstractions, not concrete implementations.

**Where used**
- `ATMMachine` depends on `BankInterface` conceptually
- transaction logic depends on `Transaction` abstraction
- state transitions depend on `ATMState` abstraction

**Where it can improve**
In the current code, `ATMMachine` constructor takes `Bank` directly rather than `BankInterface` as the type.  
A stronger DIP version would type the dependency as `BankInterface`.

Still, the design already moves in that direction by defining the bank abstraction separately.

**Why this is good**
It reduces coupling and improves replaceability and testability.

---

# 3. OOP Concepts Used

## 3.1 Encapsulation

**Where used**
Each class bundles related data and behavior together.

Examples:
- `Account` encapsulates balance, card details, and PIN validation
- `Bank` encapsulates account storage and banking logic
- `ATMMachine` encapsulates state, hardware, and ATM workflow
- `CashDispenser` encapsulates cash dispensing behavior

**Why this is useful**
Internal details are controlled inside the right class instead of being spread across the system.

---

## 3.2 Abstraction

**Where used**
- `BankInterface`
- `Transaction`
- `ATMState`

These abstractions define what a component should do, without exposing how every implementation works.

**Why this is useful**
The rest of the system can work with generic behavior instead of hardcoded concrete classes.

---

## 3.3 Inheritance

**Where used**
- concrete states inherit from `ATMState`
- concrete transactions implement the `Transaction` abstraction
- `Bank` implements `BankInterface`

**Why this is useful**
It allows shared contracts while supporting different implementations.

---

## 3.4 Polymorphism

**Where used**
The ATM delegates to the current state using calls like:
- `state.process_card_insertion(...)`
- `state.process_pin_entry(...)`
- `state.process_withdrawal_request(...)`

The actual behavior depends on which state object is active.

Similarly:
- `transaction.execute_transaction()` behaves differently for withdrawal and deposit
- `bank_interface` methods can behave differently depending on implementation

**Why this is useful**
The same method call can behave differently depending on the concrete object.

---

## 3.5 Composition

**Where used**
- `ATMMachine` contains:
  - `CardProcessor`
  - `DepositBox`
  - `CashDispenser`
  - `Keypad`
  - `Display`
  - `Bank`
  - `ATMState`
- `Bank` contains account maps
- `WithdrawTransaction` and `DepositTransaction` contain an `Account`

**Why this is useful**
The system is built by assembling smaller focused objects instead of relying on deep inheritance trees.

---

# 4. Additional Good Design Choices

## 4.1 Money Handling with Decimal

**Where used**
- account balances
- withdrawal amounts
- deposit amounts

**Why**
Financial systems should not use floating point due to rounding issues.

---

## 4.2 Hashed PIN Storage

**Where used**
- `Account.card_pin_hash`

**Why**
PINs should not be stored in plain text. Even though MD5 is not ideal for production security today, the design intent of hashing is correct for interview level discussion.

---

## 4.3 Separate Hardware Components

**Where used**
- `CardProcessor`
- `CashDispenser`
- `DepositBox`
- `Keypad`
- `Display`

**Why**
This models the ATM as a real machine with separate hardware responsibilities, which makes the design cleaner and closer to the real world.

---

## 4.4 Clear User Flow Modeling

**Where used**
- ATM state machine

**Why**
ATM systems are workflow driven, so state based modeling makes the interaction sequence explicit and easy to reason about.

---

# 5. Final Summary

## Design Patterns Used
- **State Pattern**
- **Facade Pattern**
- **Strategy-like transaction abstraction**
- **Interface-based bank abstraction**
- **Manager / Coordinator Pattern**

## SOLID Principles Used
- **Single Responsibility Principle**
- **Open Closed Principle**
- **Liskov Substitution Principle**
- **Interface Segregation Principle**
- **Dependency Inversion Principle** partially, with room for stronger abstraction in the ATM constructor

## OOP Concepts Used
- **Encapsulation**
- **Abstraction**
- **Inheritance**
- **Polymorphism**
- **Composition**

---

# 6. Interview Ready Answer

In this ATM LLD, the most important pattern is the **State pattern**, because the ATM behaves differently depending on whether it is waiting for a card, taking a PIN, selecting a transaction, entering a withdrawal amount, or collecting a deposit. I also used the **Facade pattern** through `ATMMachine`, which gives a simple interface over the ATM’s hardware and workflow.

The design follows strong OOP principles like **encapsulation**, **abstraction**, **inheritance**, **polymorphism**, and **composition**. It aligns well with SOLID, especially **SRP**, **OCP**, and **LSP**, because responsibilities are clearly separated, the system is extensible, and different states and transaction types can be added without changing the core flow too much.