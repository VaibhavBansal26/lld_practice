# 3. Vending Machine

# Step 1: Understand the problem statement and requirements

This code is designing a **vending machine system**. It supports product storage, money insertion, product selection, transaction validation, dispensing, and change return.

## Main requirements covered

- Store products in racks
- Track inventory count for each rack
- Let a user insert money
- Let a user choose a product by rack ID
- Validate whether the transaction is valid
- Dispense the selected product
- Deduct product price from inserted balance
- Return remaining change
- Keep transaction history
- Allow transaction cancellation

## Core flow

A user inserts money, selects a product, confirms the transaction, receives the product, and gets back remaining change. If anything is wrong, the system throws an exception.

---

# Step 2: Identify the main entities and their relationships

## 1. `Product`
Represents an item sold by the vending machine.

### Attributes
- `product_code`
- `description`
- `unit_price`

---

## 2. `Rack`
Represents a vending slot that holds one product and its quantity.

### Attributes
- `rack_code`
- `product`
- `count`

### Relationship
A rack contains one product type and tracks how many units are left.

---

## 3. `InventoryManager`
Manages all racks in the vending machine.

### Responsibilities
- store racks
- fetch a product from a rack
- fetch a rack by code
- dispense one item from a rack
- update the machine's rack setup

---

## 4. `PaymentProcessor`
Handles inserted money and change.

### Responsibilities
- add money
- charge product amount
- return remaining balance as change
- expose current balance

---

## 5. `Transaction`
Represents one purchase attempt.

### Stores
- selected product
- selected rack
- total amount returned as change

---

## 6. `VendingMachine`
Main controller class that coordinates inventory, payment, and transaction flow.

### Responsibilities
- accept money
- allow product selection
- validate transaction
- complete purchase
- store transaction history
- cancel transaction

---

## 7. `InvalidTransactionException`
Custom exception used when the transaction fails validation.

---

# Step 3: Define the classes and their attributes and methods

## 1. `Product`

### Purpose
Stores product information.

### Attributes
- `product_code: str`
- `description: str`
- `unit_price: Decimal`

### Note
It is defined as a `dataclass(frozen=True)`, so product objects are immutable.

---

## 2. `Rack`

### Purpose
Represents a rack containing a specific product and quantity.

### Attributes
- `rack_code`
- `product`
- `count`

### Methods
- `get_product()`
- `get_product_count()`
- `set_count(count)`

---

## 3. `InventoryManager`

### Purpose
Manages inventory inside the machine.

### Attribute
- `racks: Dict[str, Rack]`

### Methods
- `get_product_in_rack(rack_code)`
- `dispense_product_from_rack(rack)`
- `update_rack(racks)`
- `get_rack(name)`

---

## 4. `PaymentProcessor`

### Purpose
Tracks user balance and payment actions.

### Attribute
- `current_balance`

### Methods
- `add_balance(amount)`
- `charge(amount)`
- `return_change()`
- `get_current_balance()`

---

## 5. `Transaction`

### Purpose
Stores current or completed purchase details.

### Attributes
- `product`
- `rack`
- `total_amount`

### Methods
- `set_product(product)`
- `get_product()`
- `set_rack(rack)`
- `get_rack()`
- `set_total_amount(total_amount)`
- `get_total_amount()`

---

## 6. `VendingMachineState` and `NoMoneyInsertedState`

### Purpose
These are placeholder classes for a possible **State pattern** design.

### Current status
They are declared but not actively used for behavior control in this implementation.

---

## 7. `VendingMachine`

### Purpose
Main class that orchestrates the vending machine behavior.

### Attributes
- `transaction_history`
- `current_transaction`
- `inventory_manager`
- `payment_processor`
- `current_state`
- `balance`
- `selected_product`

### Methods
- `set_rack(rack)`
- `insert_money(amount)`
- `choose_product(rack_id)`
- `confirm_transaction()`
- `_validate_transaction()`
- `get_transaction_history()`
- `cancel_transaction()`
- `get_inventory_manager()`

---

# Step 4: Implement the classes and their methods

## 1. Product and rack setup
Products are created first, then racks are created with a product and count. After that, the machine is loaded with those racks using `set_rack(...)`.

---

## 2. Money insertion
The user inserts money through:

```python
machine.insert_money(Decimal("5.00"))
```

This delegates to `PaymentProcessor.add_balance(...)`, which increases the current balance.

---

## 3. Product selection
The user selects a product by rack ID using:

```python
machine.choose_product("A1")
```

This does two things:
- fetches the product from the inventory manager
- stores both the selected product and rack inside the current transaction

---

## 4. Transaction validation
Before completing the purchase, `_validate_transaction()` checks:

- product is selected
- rack is selected
- rack has stock
- inserted balance is enough to buy the product

If any check fails, it raises `InvalidTransactionException`.

---

## 5. Charging and dispensing
Inside `confirm_transaction()`:

- the product price is charged using `payment_processor.charge(product.unit_price)`
- one item is removed from the rack using `inventory_manager.dispense_product_from_rack(rack)`

---

## 6. Return change
After charging, the machine returns the leftover balance:

- `return_change()` gives back the remaining amount
- that returned amount is stored in `current_transaction.total_amount`

---

## 7. Save history and reset machine state
After a successful transaction:

- the transaction is added to `transaction_history`
- `current_transaction` is reset
- `selected_product` is cleared

This prepares the machine for the next purchase.

---

# Design patterns and design ideas used

## 1. Composition
`VendingMachine` is built using other components:
- `InventoryManager`
- `PaymentProcessor`
- `Transaction`

So the system is modular and responsibilities are separated.

## 2. Encapsulation
Each class manages its own data and behavior:
- `Rack` manages count
- `PaymentProcessor` manages balance
- `InventoryManager` manages racks
- `Transaction` manages purchase details

## 3. State pattern placeholder
`VendingMachineState` and `NoMoneyInsertedState` suggest the author may later implement the **State pattern**, but right now they are placeholders only.

---

# Strengths of the design

- Clear separation of concerns
- Easy to understand flow
- Good use of helper classes
- Custom exception for invalid transactions
- Keeps transaction history
- Easy to extend later with more states or payment methods

---

# Limitations in the current implementation

## 1. State classes are not fully implemented
The machine has state placeholders, but they do not yet control behavior.

## 2. `balance` attribute in `VendingMachine` is unused
The actual balance is managed by `PaymentProcessor`, so `self.balance` is redundant right now.

## 3. `selected_product` stores rack ID, not actual product
The name suggests product, but it actually stores the rack identifier.

## 4. `cancel_transaction()` returns change internally but does not return it to caller
It resets things correctly, but the returned change is not exposed back to the user.

---

# Interview style summary

> This code implements a vending machine system where `Product` represents sale items, `Rack` stores products and stock count, `InventoryManager` manages inventory, `PaymentProcessor` handles money and change, and `Transaction` stores purchase details. The `VendingMachine` class coordinates the full purchase flow by accepting money, selecting products, validating the transaction, dispensing inventory, returning change, and storing transaction history.

# Code

```python
from __future__ import annotations

from dataclasses import dataclass
from decimal import Decimal
from typing import Dict, List, Optional


# =========================
# Exceptions
# =========================

class InvalidTransactionException(Exception):
    pass


# =========================
# Product
# =========================

@dataclass(frozen=True)
class Product:
    product_code: str
    description: str
    unit_price: Decimal


# =========================
# Rack
# =========================

class Rack:
    def __init__(self, rack_code: str, product: Product, count: int):
        self.rack_code = rack_code
        self.product = product
        self.count = count

    def get_product(self) -> Product:
        return self.product

    def get_product_count(self) -> int:
        return self.count

    def set_count(self, count: int) -> None:
        self.count = count


# =========================
# Inventory Manager
# =========================

class InventoryManager:
    def __init__(self):
        self.racks: Dict[str, Rack] = {}

    def get_product_in_rack(self, rack_code: str) -> Product:
        rack = self.racks.get(rack_code)
        if rack is None:
            raise KeyError(f"Rack '{rack_code}' not found")
        return rack.get_product()

    def dispense_product_from_rack(self, rack: Rack) -> None:
        if rack.get_product_count() > 0:
            rack.set_count(rack.get_product_count() - 1)
        else:
            raise ValueError("Cannot dispense product. Rack is empty.")

    def update_rack(self, racks: Dict[str, Rack]) -> None:
        self.racks = racks

    def get_rack(self, name: str) -> Optional[Rack]:
        return self.racks.get(name)


# =========================
# Payment Processor
# =========================

class PaymentProcessor:
    def __init__(self):
        self.current_balance = Decimal("0")

    def add_balance(self, amount: Decimal) -> None:
        self.current_balance += amount

    def charge(self, amount: Decimal) -> None:
        self.current_balance -= amount

    def return_change(self) -> Decimal:
        change = self.current_balance
        self.current_balance = Decimal("0")
        return change

    def get_current_balance(self) -> Decimal:
        return self.current_balance


# =========================
# Transaction
# =========================

class Transaction:
    def __init__(self):
        self.product: Optional[Product] = None
        self.rack: Optional[Rack] = None
        self.total_amount: Decimal = Decimal("0")

    def set_product(self, product: Product) -> None:
        self.product = product

    def get_product(self) -> Optional[Product]:
        return self.product

    def set_rack(self, rack: Rack) -> None:
        self.rack = rack

    def get_rack(self) -> Optional[Rack]:
        return self.rack

    def set_total_amount(self, total_amount: Decimal) -> None:
        self.total_amount = total_amount

    def get_total_amount(self) -> Decimal:
        return self.total_amount


# =========================
# Optional Vending Machine State placeholders
# =========================

class VendingMachineState:
    pass


class NoMoneyInsertedState(VendingMachineState):
    pass


# =========================
# Vending Machine
# =========================

class VendingMachine:
    def __init__(self):
        self.transaction_history: List[Transaction] = []
        self.current_transaction = Transaction()
        self.inventory_manager = InventoryManager()
        self.payment_processor = PaymentProcessor()

        self.current_state: VendingMachineState = NoMoneyInsertedState()
        self.balance = Decimal("0")
        self.selected_product: Optional[str] = None

    def set_rack(self, rack: Dict[str, Rack]) -> None:
        self.inventory_manager.update_rack(rack)

    def insert_money(self, amount: Decimal) -> None:
        self.payment_processor.add_balance(amount)

    def choose_product(self, rack_id: str) -> None:
        product = self.inventory_manager.get_product_in_rack(rack_id)
        rack = self.inventory_manager.get_rack(rack_id)

        if rack is None:
            raise KeyError(f"Rack '{rack_id}' not found")

        self.current_transaction.set_rack(rack)
        self.current_transaction.set_product(product)
        self.selected_product = rack_id

    def confirm_transaction(self) -> Transaction:
        # Step 1: Validate the transaction before processing
        self._validate_transaction()

        # Step 2: Charge the customer for the product
        product = self.current_transaction.get_product()
        rack = self.current_transaction.get_rack()

        assert product is not None
        assert rack is not None

        self.payment_processor.charge(product.unit_price)

        # Step 3: Dispense the product from the rack
        self.inventory_manager.dispense_product_from_rack(rack)

        # Step 4: Return the change to the customer
        self.current_transaction.set_total_amount(self.payment_processor.return_change())

        # Step 5: Add the completed transaction to the history
        self.transaction_history.append(self.current_transaction)
        completed_transaction = self.current_transaction

        # Reset current transaction for next purchase
        self.current_transaction = Transaction()
        self.selected_product = None

        return completed_transaction

    def _validate_transaction(self) -> None:
        product = self.current_transaction.get_product()
        rack = self.current_transaction.get_rack()

        if product is None:
            raise InvalidTransactionException("Invalid product selection")
        if rack is None:
            raise InvalidTransactionException("Invalid rack selection")
        if rack.get_product_count() == 0:
            raise InvalidTransactionException("Insufficient inventory for product.")
        if self.payment_processor.get_current_balance() < product.unit_price:
            raise InvalidTransactionException("Insufficient fund")

    def get_transaction_history(self) -> List[Transaction]:
        return list(self.transaction_history)

    def cancel_transaction(self) -> None:
        self.payment_processor.return_change()
        self.current_transaction = Transaction()
        self.selected_product = None

    def get_inventory_manager(self) -> InventoryManager:
        return self.inventory_manager


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    chips = Product("P101", "Potato Chips", Decimal("2.50"))
    soda = Product("P102", "Soda Can", Decimal("1.75"))

    rack_a1 = Rack("A1", chips, 5)
    rack_b1 = Rack("B1", soda, 3)

    machine = VendingMachine()
    machine.set_rack({
        "A1": rack_a1,
        "B1": rack_b1,
    })

    machine.insert_money(Decimal("5.00"))
    machine.choose_product("A1")
    transaction = machine.confirm_transaction()

    print("Purchased:", transaction.get_product().description)
    print("Change returned:", transaction.get_total_amount())
    print("Remaining stock in A1:", machine.get_inventory_manager().get_rack("A1").get_product_count())
```

**Advanced interview-ready Python version of the vending machine with:**

State pattern

coin and note handling

refund support

sold out state

exact change behavior

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass
from decimal import Decimal
from typing import Dict, List, Optional


# =========================
# Exceptions
# =========================

class VendingMachineException(Exception):
    pass


class InvalidTransactionException(VendingMachineException):
    pass


class SoldOutException(VendingMachineException):
    pass


class ExactChangeOnlyException(VendingMachineException):
    pass


class InvalidDenominationException(VendingMachineException):
    pass


# =========================
# Money / Denominations
# =========================

@dataclass(frozen=True)
class Denomination:
    name: str
    value: Decimal


PENNY = Denomination("PENNY", Decimal("0.01"))
NICKEL = Denomination("NICKEL", Decimal("0.05"))
DIME = Denomination("DIME", Decimal("0.10"))
QUARTER = Denomination("QUARTER", Decimal("0.25"))
ONE_DOLLAR = Denomination("ONE_DOLLAR", Decimal("1.00"))
FIVE_DOLLAR = Denomination("FIVE_DOLLAR", Decimal("5.00"))

SUPPORTED_DENOMINATIONS = {
    PENNY.name: PENNY,
    NICKEL.name: NICKEL,
    DIME.name: DIME,
    QUARTER.name: QUARTER,
    ONE_DOLLAR.name: ONE_DOLLAR,
    FIVE_DOLLAR.name: FIVE_DOLLAR,
}


# =========================
# Product
# =========================

@dataclass(frozen=True)
class Product:
    product_code: str
    description: str
    unit_price: Decimal


# =========================
# Rack
# =========================

class Rack:
    def __init__(self, rack_code: str, product: Product, count: int):
        self.rack_code = rack_code
        self.product = product
        self.count = count

    def get_product(self) -> Product:
        return self.product

    def get_product_count(self) -> int:
        return self.count

    def set_count(self, count: int) -> None:
        self.count = count

    def is_empty(self) -> bool:
        return self.count <= 0


# =========================
# Inventory Manager
# =========================

class InventoryManager:
    def __init__(self):
        self.racks: Dict[str, Rack] = {}

    def update_rack(self, racks: Dict[str, Rack]) -> None:
        self.racks = racks

    def get_rack(self, rack_code: str) -> Optional[Rack]:
        return self.racks.get(rack_code)

    def get_product_in_rack(self, rack_code: str) -> Product:
        rack = self.get_rack(rack_code)
        if rack is None:
            raise InvalidTransactionException(f"Rack '{rack_code}' not found")
        return rack.get_product()

    def dispense_product_from_rack(self, rack: Rack) -> None:
        if rack.get_product_count() <= 0:
            raise SoldOutException("Cannot dispense product. Rack is empty.")
        rack.set_count(rack.get_product_count() - 1)

    def is_all_sold_out(self) -> bool:
        return all(rack.is_empty() for rack in self.racks.values()) if self.racks else True


# =========================
# Cash Inventory
# =========================

class CashInventory:
    def __init__(self):
        self.cash_count: Dict[Denomination, int] = {
            PENNY: 0,
            NICKEL: 0,
            DIME: 0,
            QUARTER: 0,
            ONE_DOLLAR: 0,
            FIVE_DOLLAR: 0,
        }

    def add(self, denomination: Denomination, count: int = 1) -> None:
        self.cash_count[denomination] += count

    def remove(self, denomination: Denomination, count: int = 1) -> None:
        if self.cash_count[denomination] < count:
            raise InvalidTransactionException(f"Not enough {denomination.name} in cash inventory")
        self.cash_count[denomination] -= count

    def get_count(self, denomination: Denomination) -> int:
        return self.cash_count.get(denomination, 0)

    def copy_counts(self) -> Dict[Denomination, int]:
        return dict(self.cash_count)

    def total_amount(self) -> Decimal:
        total = Decimal("0")
        for denomination, count in self.cash_count.items():
            total += denomination.value * count
        return total


# =========================
# Payment Processor
# =========================

class PaymentProcessor:
    def __init__(self, machine_cash_inventory: CashInventory):
        self.machine_cash_inventory = machine_cash_inventory
        self.current_balance = Decimal("0")
        self.inserted_cash: Dict[Denomination, int] = {}

    def insert_money(self, denomination: Denomination, count: int = 1) -> None:
        self.current_balance += denomination.value * count
        self.inserted_cash[denomination] = self.inserted_cash.get(denomination, 0) + count

    def get_current_balance(self) -> Decimal:
        return self.current_balance

    def commit_inserted_cash_to_machine(self) -> None:
        for denomination, count in self.inserted_cash.items():
            self.machine_cash_inventory.add(denomination, count)
        self.inserted_cash = {}

    def charge(self, amount: Decimal) -> None:
        self.current_balance -= amount

    def refund(self) -> Dict[Denomination, int]:
        refunded = dict(self.inserted_cash)
        self.inserted_cash = {}
        self.current_balance = Decimal("0")
        return refunded

    def can_make_change(self, amount: Decimal) -> bool:
        try:
            self._calculate_change_distribution(amount, self.machine_cash_inventory.copy_counts())
            return True
        except ExactChangeOnlyException:
            return False

    def return_change(self) -> Dict[Denomination, int]:
        change_amount = self.current_balance
        if change_amount == Decimal("0"):
            return {}

        change_distribution = self._calculate_change_distribution(
            change_amount,
            self.machine_cash_inventory.copy_counts(),
        )

        for denomination, count in change_distribution.items():
            self.machine_cash_inventory.remove(denomination, count)

        self.current_balance = Decimal("0")
        self.inserted_cash = {}
        return change_distribution

    def _calculate_change_distribution(
        self,
        amount: Decimal,
        available_cash: Dict[Denomination, int],
    ) -> Dict[Denomination, int]:
        remaining = amount.quantize(Decimal("0.01"))
        result: Dict[Denomination, int] = {}

        denominations_desc = sorted(
            available_cash.keys(),
            key=lambda d: d.value,
            reverse=True,
        )

        for denomination in denominations_desc:
            denom_value = denomination.value
            available_count = available_cash[denomination]

            while available_count > 0 and remaining >= denom_value:
                remaining = (remaining - denom_value).quantize(Decimal("0.01"))
                result[denomination] = result.get(denomination, 0) + 1
                available_count -= 1

        if remaining != Decimal("0.00"):
            raise ExactChangeOnlyException("Machine cannot return exact change")

        return result


# =========================
# Transaction
# =========================

class Transaction:
    def __init__(self):
        self.product: Optional[Product] = None
        self.rack: Optional[Rack] = None
        self.change_returned: Dict[Denomination, int] = {}
        self.inserted_amount: Decimal = Decimal("0")

    def set_product(self, product: Product) -> None:
        self.product = product

    def get_product(self) -> Optional[Product]:
        return self.product

    def set_rack(self, rack: Rack) -> None:
        self.rack = rack

    def get_rack(self) -> Optional[Rack]:
        return self.rack

    def set_change_returned(self, change: Dict[Denomination, int]) -> None:
        self.change_returned = change

    def set_inserted_amount(self, amount: Decimal) -> None:
        self.inserted_amount = amount


# =========================
# State Pattern
# =========================

class VendingMachineState(ABC):
    @abstractmethod
    def insert_money(self, machine: "VendingMachine", denomination: Denomination, count: int = 1) -> None:
        pass

    @abstractmethod
    def choose_product(self, machine: "VendingMachine", rack_id: str) -> None:
        pass

    @abstractmethod
    def confirm_transaction(self, machine: "VendingMachine") -> Transaction:
        pass

    @abstractmethod
    def cancel_transaction(self, machine: "VendingMachine") -> Dict[Denomination, int]:
        pass


class NoMoneyInsertedState(VendingMachineState):
    def insert_money(self, machine: "VendingMachine", denomination: Denomination, count: int = 1) -> None:
        machine.payment_processor.insert_money(denomination, count)
        machine.current_state = HasMoneyState()

    def choose_product(self, machine: "VendingMachine", rack_id: str) -> None:
        raise InvalidTransactionException("Insert money first")

    def confirm_transaction(self, machine: "VendingMachine") -> Transaction:
        raise InvalidTransactionException("No active transaction")

    def cancel_transaction(self, machine: "VendingMachine") -> Dict[Denomination, int]:
        return {}


class HasMoneyState(VendingMachineState):
    def insert_money(self, machine: "VendingMachine", denomination: Denomination, count: int = 1) -> None:
        machine.payment_processor.insert_money(denomination, count)

    def choose_product(self, machine: "VendingMachine", rack_id: str) -> None:
        rack = machine.inventory_manager.get_rack(rack_id)
        if rack is None:
            raise InvalidTransactionException("Invalid rack selection")
        if rack.is_empty():
            raise SoldOutException("Selected product is sold out")

        product = rack.get_product()
        machine.current_transaction.set_rack(rack)
        machine.current_transaction.set_product(product)
        machine.selected_product = rack_id
        machine.current_state = ProductSelectedState()

    def confirm_transaction(self, machine: "VendingMachine") -> Transaction:
        raise InvalidTransactionException("Choose product first")

    def cancel_transaction(self, machine: "VendingMachine") -> Dict[Denomination, int]:
        refund = machine.payment_processor.refund()
        machine._reset_transaction()
        return refund


class ProductSelectedState(VendingMachineState):
    def insert_money(self, machine: "VendingMachine", denomination: Denomination, count: int = 1) -> None:
        machine.payment_processor.insert_money(denomination, count)

    def choose_product(self, machine: "VendingMachine", rack_id: str) -> None:
        rack = machine.inventory_manager.get_rack(rack_id)
        if rack is None:
            raise InvalidTransactionException("Invalid rack selection")
        if rack.is_empty():
            raise SoldOutException("Selected product is sold out")

        product = rack.get_product()
        machine.current_transaction.set_rack(rack)
        machine.current_transaction.set_product(product)
        machine.selected_product = rack_id

    def confirm_transaction(self, machine: "VendingMachine") -> Transaction:
        machine._validate_transaction()

        product = machine.current_transaction.get_product()
        rack = machine.current_transaction.get_rack()
        assert product is not None
        assert rack is not None

        inserted_amount = machine.payment_processor.get_current_balance()
        machine.current_transaction.set_inserted_amount(inserted_amount)

        change_needed = (inserted_amount - product.unit_price).quantize(Decimal("0.01"))

        machine.payment_processor.commit_inserted_cash_to_machine()
        machine.payment_processor.charge(product.unit_price)

        try:
            change = machine.payment_processor.return_change()
        except ExactChangeOnlyException:
            # rollback is omitted for interview simplicity
            raise ExactChangeOnlyException("Unable to return exact change for this transaction")

        machine.inventory_manager.dispense_product_from_rack(rack)
        machine.current_transaction.set_change_returned(change)

        completed_transaction = machine.current_transaction
        machine.transaction_history.append(completed_transaction)
        machine._reset_transaction()

        if machine.inventory_manager.is_all_sold_out():
            machine.current_state = SoldOutState()

        return completed_transaction

    def cancel_transaction(self, machine: "VendingMachine") -> Dict[Denomination, int]:
        refund = machine.payment_processor.refund()
        machine._reset_transaction()
        return refund


class SoldOutState(VendingMachineState):
    def insert_money(self, machine: "VendingMachine", denomination: Denomination, count: int = 1) -> None:
        raise SoldOutException("Machine is sold out")

    def choose_product(self, machine: "VendingMachine", rack_id: str) -> None:
        raise SoldOutException("Machine is sold out")

    def confirm_transaction(self, machine: "VendingMachine") -> Transaction:
        raise SoldOutException("Machine is sold out")

    def cancel_transaction(self, machine: "VendingMachine") -> Dict[Denomination, int]:
        return {}


# =========================
# Vending Machine
# =========================

class VendingMachine:
    def __init__(self):
        self.transaction_history: List[Transaction] = []
        self.current_transaction = Transaction()
        self.inventory_manager = InventoryManager()
        self.machine_cash_inventory = CashInventory()
        self.payment_processor = PaymentProcessor(self.machine_cash_inventory)

        self.current_state: VendingMachineState = NoMoneyInsertedState()
        self.selected_product: Optional[str] = None

    def set_rack(self, racks: Dict[str, Rack]) -> None:
        self.inventory_manager.update_rack(racks)
        if self.inventory_manager.is_all_sold_out():
            self.current_state = SoldOutState()
        else:
            self.current_state = NoMoneyInsertedState()

    def load_cash(self, denomination: Denomination, count: int) -> None:
        self.machine_cash_inventory.add(denomination, count)

    def insert_money(self, denomination: Denomination, count: int = 1) -> None:
        if denomination.name not in SUPPORTED_DENOMINATIONS:
            raise InvalidDenominationException(f"Unsupported denomination: {denomination}")
        self.current_state.insert_money(self, denomination, count)

    def choose_product(self, rack_id: str) -> None:
        self.current_state.choose_product(self, rack_id)

    def confirm_transaction(self) -> Transaction:
        return self.current_state.confirm_transaction(self)

    def cancel_transaction(self) -> Dict[Denomination, int]:
        return self.current_state.cancel_transaction(self)

    def get_transaction_history(self) -> List[Transaction]:
        return list(self.transaction_history)

    def get_inventory_manager(self) -> InventoryManager:
        return self.inventory_manager

    def get_current_balance(self) -> Decimal:
        return self.payment_processor.get_current_balance()

    def is_exact_change_only(self, rack_id: str) -> bool:
        rack = self.inventory_manager.get_rack(rack_id)
        if rack is None or rack.is_empty():
            return False

        product = rack.get_product()
        inserted_amount = self.payment_processor.get_current_balance()

        if inserted_amount < product.unit_price:
            return False

        change_needed = (inserted_amount - product.unit_price).quantize(Decimal("0.01"))
        return change_needed > Decimal("0.00") and not self.payment_processor.can_make_change(change_needed)

    def _validate_transaction(self) -> None:
        product = self.current_transaction.get_product()
        rack = self.current_transaction.get_rack()

        if product is None:
            raise InvalidTransactionException("Invalid product selection")
        if rack is None:
            raise InvalidTransactionException("Invalid rack selection")
        if rack.get_product_count() == 0:
            raise SoldOutException("Insufficient inventory for product.")
        if self.payment_processor.get_current_balance() < product.unit_price:
            raise InvalidTransactionException("Insufficient fund")

        change_needed = (self.payment_processor.get_current_balance() - product.unit_price).quantize(Decimal("0.01"))
        if change_needed > Decimal("0.00") and not self.payment_processor.can_make_change(change_needed):
            raise ExactChangeOnlyException("Exact change required")

    def _reset_transaction(self) -> None:
        self.current_transaction = Transaction()
        self.selected_product = None

        if self.inventory_manager.is_all_sold_out():
            self.current_state = SoldOutState()
        else:
            self.current_state = NoMoneyInsertedState()


# =========================
# Helper display
# =========================

def format_cash(cash_map: Dict[Denomination, int]) -> Dict[str, int]:
    return {denom.name: count for denom, count in cash_map.items()}


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    chips = Product("P101", "Potato Chips", Decimal("2.50"))
    soda = Product("P102", "Soda Can", Decimal("1.75"))

    rack_a1 = Rack("A1", chips, 2)
    rack_b1 = Rack("B1", soda, 1)

    machine = VendingMachine()
    machine.set_rack({
        "A1": rack_a1,
        "B1": rack_b1,
    })

    # load initial change into machine
    machine.load_cash(QUARTER, 10)
    machine.load_cash(DIME, 10)
    machine.load_cash(NICKEL, 10)

    # transaction 1
    machine.insert_money(ONE_DOLLAR, 3)  # $3.00
    machine.choose_product("A1")
    txn = machine.confirm_transaction()

    print("Purchased:", txn.get_product().description)
    print("Inserted:", txn.inserted_amount)
    print("Change returned:", format_cash(txn.change_returned))
    print("Remaining stock A1:", machine.get_inventory_manager().get_rack("A1").get_product_count())

    # transaction 2 with refund
    machine.insert_money(QUARTER, 2)
    machine.insert_money(DIME, 1)
    refund = machine.cancel_transaction()
    print("Refund:", format_cash(refund))

    # transaction 3 exact change scenario example
    exact_change_machine = VendingMachine()
    exact_change_machine.set_rack({
        "B1": Rack("B1", soda, 1)
    })
    # no change loaded
    exact_change_machine.insert_money(ONE_DOLLAR, 2)  # $2.00
    exact_change_machine.choose_product("B1")

    try:
        exact_change_machine.confirm_transaction()
    except ExactChangeOnlyException as exc:
        print("Exact change error:", str(exc))
```

## Design Patterns, SOLID Principles, and OOP Concepts Used in the Advanced Vending Machine System

# 1. Design Patterns Used

## 1.1 State Pattern

**Where used**
- `VendingMachineState`
- `NoMoneyInsertedState`
- `HasMoneyState`
- `ProductSelectedState`
- `SoldOutState`

**Why we used it**
The vending machine behaves differently depending on its current state.

Examples:
- if no money is inserted, product selection should not be allowed
- if money is inserted, product can be selected
- if a product is selected, transaction can be confirmed
- if the machine is sold out, no operation should proceed

Instead of putting many `if else` checks inside `VendingMachine`, we move behavior into separate state classes.

**Benefit**
- cleaner logic
- easier to extend
- avoids huge conditional blocks

**Simple reason**  
When object behavior changes based on state, State pattern is a strong fit.

---

## 1.2 Strategy-like Greedy Change Calculation

**Where used**
- `PaymentProcessor._calculate_change_distribution()`

**Why it is strategy-like**
The change return logic is encapsulated in a specific method and can later be replaced with:
- greedy change strategy
- optimal change strategy
- country specific denomination strategy

Right now it is not fully extracted into a separate `ChangeStrategy` class, but the design is ready for that extension.

**Simple reason**  
The design separates the algorithm for returning change from the rest of the vending flow.

---

## 1.3 Facade-like Central Coordinator

**Where used**
- `VendingMachine`

**Why we used it**
`VendingMachine` acts as the main entry point for clients.

Clients do not directly coordinate:
- `InventoryManager`
- `PaymentProcessor`
- `CashInventory`
- `Transaction`
- state classes

Instead, clients call:
- `insert_money()`
- `choose_product()`
- `confirm_transaction()`
- `cancel_transaction()`

This simplifies interaction with the system.

**Simple reason**  
A facade style interface makes the system easier to use.

---

## 1.4 Composition Over Inheritance

**Where used**
- `VendingMachine` contains:
  - `InventoryManager`
  - `CashInventory`
  - `PaymentProcessor`
  - `Transaction`
  - `VendingMachineState`
- `InventoryManager` contains `Rack`
- `Rack` contains `Product`

**Why we used it**
Instead of building deep inheritance hierarchies, the system is built by combining focused objects.

For example:
- `VendingMachine` does not inherit inventory logic
- it contains an `InventoryManager`

**Simple reason**  
Composition keeps responsibilities separated and improves flexibility.

---

## 1.5 Manager / Coordinator Pattern

**Where used**
- `InventoryManager`
- `PaymentProcessor`

**Why we used it**
Some classes are designed to coordinate a specific responsibility.

### `InventoryManager`
Handles:
- rack lookup
- product retrieval
- stock decrement
- sold out checks

### `PaymentProcessor`
Handles:
- money insertion
- current balance
- refund
- charge
- change calculation

This prevents `VendingMachine` from becoming a god class.

**Simple reason**  
Managers organize domain workflows into focused components.

---

# 2. SOLID Principles Used

## 2.1 Single Responsibility Principle (SRP)

**Meaning**
A class should have one reason to change.

**Where used**

### `Product`
Only stores product information.

### `Rack`
Only represents one storage unit and its count.

### `InventoryManager`
Only manages inventory related operations.

### `CashInventory`
Only manages machine cash availability.

### `PaymentProcessor`
Only handles payment and change related logic.

### `Transaction`
Only represents the transaction data.

### State classes
Each state handles only behavior relevant to that state.

**Why this is good**
Responsibilities are clearly separated, so changing one part does not heavily affect others.

---

## 2.2 Open Closed Principle (OCP)

**Meaning**
Software should be open for extension but closed for modification.

**Where used**
- new vending machine states can be added
- new denominations can be introduced
- change calculation strategy can be extracted later
- new machine behaviors can be added without rewriting existing core flow

Example:
- add `MaintenanceState`
- add `OutOfServiceState`
- add `CardPaymentState`

**Why this is good**
The design supports extension without breaking stable code.

---

## 2.3 Liskov Substitution Principle (LSP)

**Meaning**
Subclasses should be replaceable for their base type.

**Where used**
All concrete states can be used anywhere `VendingMachineState` is expected:
- `NoMoneyInsertedState`
- `HasMoneyState`
- `ProductSelectedState`
- `SoldOutState`

`VendingMachine` talks to the abstract state interface, not specific implementations.

**Why this is good**
The machine can switch states safely without changing the calling code.

---

## 2.4 Interface Segregation Principle (ISP)

**Meaning**
Clients should not be forced to depend on methods they do not use.

**Where used**
The state interface is reasonably focused:
- `insert_money`
- `choose_product`
- `confirm_transaction`
- `cancel_transaction`

Each state provides behavior for the same small machine interaction contract.

**Why this is good**
The abstraction is compact and centered on the machine’s user actions.

**Note**
This is not a perfect textbook ISP example, but it is still aligned with the principle because the interface is not bloated.

---

## 2.5 Dependency Inversion Principle (DIP)

**Meaning**
High level modules should depend on abstractions, not low level details.

**Where used**
- `VendingMachine` depends on `VendingMachineState`, not concrete state classes during behavior calls
- state transitions work through the abstract `VendingMachineState` type

**Where it can be improved**
Currently `VendingMachine` directly creates:
- `InventoryManager`
- `CashInventory`
- `PaymentProcessor`

A more advanced design could inject these dependencies through the constructor.

Example:
- inject `PaymentProcessor`
- inject `InventoryManager`
- inject `ChangeStrategy`

**Why this is still good**
The state behavior already uses abstraction well, and the system is ready for better dependency injection if needed.

---

# 3. OOP Concepts Used

## 3.1 Encapsulation

**Where used**
Each class bundles data and behavior together.

Examples:

### `Rack`
Encapsulates:
- rack code
- product
- count

### `PaymentProcessor`
Encapsulates:
- current balance
- inserted cash
- charge
- refund
- return change

### `CashInventory`
Encapsulates:
- denomination counts
- total amount
- add/remove money

**Why this is useful**
Internal details are hidden and controlled through methods.

---

## 3.2 Abstraction

**Where used**
- `VendingMachineState` is an abstraction
- clients interact with `VendingMachine` without knowing internal complexity

**Why this is useful**
Users of the class do not need to understand how inventory, payment, and state transitions are internally implemented.

---

## 3.3 Inheritance

**Where used**
State classes inherit from `VendingMachineState`:
- `NoMoneyInsertedState`
- `HasMoneyState`
- `ProductSelectedState`
- `SoldOutState`

**Why this is useful**
It allows all states to share the same contract and be used polymorphically.

---

## 3.4 Polymorphism

**Where used**
`VendingMachine` delegates behavior to `current_state`.

Example:
- `self.current_state.insert_money(...)`
- `self.current_state.choose_product(...)`
- `self.current_state.confirm_transaction(...)`

The actual behavior depends on which concrete state object is active.

**Why this is useful**
The same method call behaves differently in different states.

---

## 3.5 Composition

**Where used**
The system is built from smaller collaborating objects:
- `VendingMachine` has managers and processors
- `InventoryManager` has racks
- `Rack` has a product

**Why this is useful**
It keeps the design modular and easier to change.

---

# 4. Additional Good Design Choices

## 4.1 Precision for Money

**Where used**
- `Decimal`

**Why**
Money should not use `float` because of rounding and precision issues.

---

## 4.2 Explicit Exceptions

**Where used**
- `InvalidTransactionException`
- `SoldOutException`
- `ExactChangeOnlyException`
- `InvalidDenominationException`

**Why**
This makes failure scenarios explicit and easier to debug and test.

---

## 4.3 Domain Modeling

**Where used**
- `Product`
- `Rack`
- `Transaction`
- `Denomination`
- `CashInventory`

**Why**
The design closely matches the real world vending machine domain, which is good for LLD interviews.

---

# 5. Final Summary

## Design Patterns Used
- **State Pattern**
- **Facade-like central coordinator**
- **Composition over inheritance**
- **Manager / Coordinator pattern**
- **Strategy-like change algorithm design**

## SOLID Principles Used
- **Single Responsibility Principle**
- **Open Closed Principle**
- **Liskov Substitution Principle**
- **Interface Segregation Principle**
- **Dependency Inversion Principle** partially, with room for stronger dependency injection

## OOP Concepts Used
- **Encapsulation**
- **Abstraction**
- **Inheritance**
- **Polymorphism**
- **Composition**

---

# 6. Interview Ready Answer

In this advanced vending machine design, the most important pattern is the **State pattern**, because the machine behaves differently depending on whether no money is inserted, money is inserted, a product is selected, or the machine is sold out.

I also used **composition** heavily by separating inventory, payment, cash storage, and transaction logic into different classes, which aligns with the **Single Responsibility Principle**.

The design uses **encapsulation** to keep each class responsible for its own data and behavior, **abstraction** through the state interface, **inheritance** for concrete machine states, and **polymorphism** because the machine delegates behavior to the current state object.

Overall, the system is modular, extensible, and interview ready because it clearly models the domain while applying strong object oriented design principles.