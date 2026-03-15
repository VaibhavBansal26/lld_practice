# 8. Restaurant Management System

Code

```python
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime, timedelta
from decimal import Decimal
from enum import Enum
from typing import Dict, List, Optional, Set


# =========================
# Menu / MenuItem
# =========================

class Category(str, Enum):
    MAIN = "MAIN"
    APPETIZER = "APPETIZER"
    DESSERT = "DESSERT"


@dataclass(frozen=True)
class MenuItem:
    name: str
    description: str
    price: Decimal
    category: Category


class Menu:
    def __init__(self):
        self.menu_items: Dict[str, MenuItem] = {}

    def add_item(self, item: MenuItem) -> None:
        self.menu_items[item.name] = item

    def get_item(self, name: str) -> Optional[MenuItem]:
        return self.menu_items.get(name)

    def get_menu_items(self) -> Dict[str, MenuItem]:
        return dict(self.menu_items)


# =========================
# OrderItem
# =========================

class OrderStatus(str, Enum):
    PENDING = "PENDING"
    SENT_TO_KITCHEN = "SENT_TO_KITCHEN"
    DELIVERED = "DELIVERED"
    CANCELED = "CANCELED"


@dataclass
class OrderItem:
    item: MenuItem
    status: OrderStatus = OrderStatus.PENDING

    def send_to_kitchen(self) -> None:
        if self.status == OrderStatus.PENDING:
            self.status = OrderStatus.SENT_TO_KITCHEN

    def deliver_to_customer(self) -> None:
        if self.status == OrderStatus.SENT_TO_KITCHEN:
            self.status = OrderStatus.DELIVERED

    def cancel(self) -> None:
        if self.status in {OrderStatus.PENDING, OrderStatus.SENT_TO_KITCHEN}:
            self.status = OrderStatus.CANCELED


# =========================
# Reservation
# =========================

@dataclass(frozen=True)
class Reservation:
    party_name: str
    party_size: int
    time: datetime
    assigned_table: "Table"


# =========================
# Table
# =========================

class Table:
    def __init__(self, table_id: int, capacity: int):
        self.table_id = table_id
        self.capacity = capacity
        self.reservations: Dict[datetime, Reservation] = {}
        self.ordered_items: Dict[MenuItem, List[OrderItem]] = {}

    def calculate_bill_amount(self) -> Decimal:
        total = Decimal("0.00")
        for order_items in self.ordered_items.values():
            for order_item in order_items:
                if order_item.status != OrderStatus.CANCELED:
                    total += order_item.item.price
        return total

    def add_order(self, item: MenuItem, quantity: int = 1) -> None:
        for _ in range(quantity):
            self._add_single_order(item)

    def _add_single_order(self, item: MenuItem) -> None:
        if item not in self.ordered_items:
            self.ordered_items[item] = []
        self.ordered_items[item].append(OrderItem(item))

    def remove_order(self, item: MenuItem) -> None:
        order_items = self.ordered_items.get(item)
        if order_items:
            order_items.pop(0)
            if not order_items:
                del self.ordered_items[item]

    def is_available_at(self, reservation_time: datetime) -> bool:
        return reservation_time not in self.reservations

    def add_reservation(self, reservation: Reservation) -> None:
        self.reservations[reservation.time] = reservation

    def remove_reservation(self, reservation_time: datetime) -> None:
        self.reservations.pop(reservation_time, None)

    def __repr__(self) -> str:
        return f"Table(id={self.table_id}, capacity={self.capacity})"


# =========================
# Layout
# =========================

class Layout:
    def __init__(self, table_capacities: List[int]):
        self.tables_by_id: Dict[int, Table] = {}
        self.tables_by_capacity: Dict[int, Set[Table]] = {}

        for i, capacity in enumerate(table_capacities):
            table = Table(i, capacity)
            self.tables_by_id[i] = table
            if capacity not in self.tables_by_capacity:
                self.tables_by_capacity[capacity] = set()
            self.tables_by_capacity[capacity].add(table)

    def find_available_table(self, party_size: int, reservation_time: datetime) -> Optional[Table]:
        for capacity in sorted(self.tables_by_capacity.keys()):
            if capacity >= party_size:
                for table in self.tables_by_capacity[capacity]:
                    if table.is_available_at(reservation_time):
                        return table
        return None


# =========================
# ReservationManager
# =========================

class ReservationManager:
    def __init__(self, layout: Layout):
        self.layout = layout
        self.reservations: Set[Reservation] = set()

    def find_available_time_slots(
        self,
        range_start: datetime,
        range_end: datetime,
        party_size: int,
    ) -> List[datetime]:
        current = range_start
        possible_reservations: List[datetime] = []

        while current <= range_end:
            available_table = self.layout.find_available_table(party_size, current)
            if available_table is not None:
                possible_reservations.append(current)
            current = current + timedelta(hours=1)

        return possible_reservations

    def create_reservation(
        self,
        party_name: str,
        party_size: int,
        desired_time: datetime,
    ) -> Reservation:
        desired_time = desired_time.replace(minute=0, second=0, microsecond=0)
        table = self.layout.find_available_table(party_size, desired_time)

        if table is None:
            raise ValueError("No available table for the requested time and party size")

        reservation = Reservation(party_name, party_size, desired_time, table)
        table.add_reservation(reservation)
        self.reservations.add(reservation)
        return reservation

    def remove_reservation(
        self,
        party_name: str,
        party_size: int,
        reservation_time: datetime,
    ) -> None:
        for reservation in list(self.reservations):
            if (
                reservation.time == reservation_time
                and reservation.party_size == party_size
                and reservation.party_name == party_name
            ):
                table = reservation.assigned_table
                table.remove_reservation(reservation_time)
                self.reservations.remove(reservation)
                return


# =========================
# Restaurant
# =========================

class Restaurant:
    def __init__(self, name: str, menu: Menu, layout: Layout):
        self.name = name
        self.menu = menu
        self.layout = layout
        self.reservation_manager = ReservationManager(layout)

    def find_available_time_slots(
        self,
        range_start: datetime,
        range_end: datetime,
        party_size: int,
    ) -> List[datetime]:
        return self.reservation_manager.find_available_time_slots(
            range_start, range_end, party_size
        )

    def create_scheduled_reservation(
        self,
        party_name: str,
        party_size: int,
        time: datetime,
    ) -> Reservation:
        return self.reservation_manager.create_reservation(
            party_name, party_size, time
        )

    def remove_reservation(
        self,
        party_name: str,
        party_size: int,
        reservation_time: datetime,
    ) -> None:
        self.reservation_manager.remove_reservation(
            party_name, party_size, reservation_time
        )

    def create_walk_in_reservation(
        self,
        party_name: str,
        party_size: int,
    ) -> Reservation:
        return self.reservation_manager.create_reservation(
            party_name, party_size, datetime.now()
        )

    def order_item(self, table: Table, item: MenuItem, quantity: int = 1) -> None:
        table.add_order(item, quantity)

    def cancel_item(self, table: Table, item: MenuItem) -> None:
        table.remove_order(item)

    def calculate_table_bill(self, table: Table) -> Decimal:
        return table.calculate_bill_amount()


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    # Menu setup
    menu = Menu()
    pasta = MenuItem("Pasta", "Creamy white sauce pasta", Decimal("12.50"), Category.MAIN)
    soup = MenuItem("Soup", "Tomato basil soup", Decimal("6.00"), Category.APPETIZER)
    cake = MenuItem("Cake", "Chocolate lava cake", Decimal("8.00"), Category.DESSERT)

    menu.add_item(pasta)
    menu.add_item(soup)
    menu.add_item(cake)

    # Layout setup
    layout = Layout([2, 2, 4, 4, 6, 8])

    # Restaurant
    restaurant = Restaurant("Food Palace", menu, layout)

    # Find slots
    now = datetime.now().replace(minute=0, second=0, microsecond=0)
    slots = restaurant.find_available_time_slots(now, now + timedelta(hours=3), 4)
    print("Available slots:", slots)

    # Scheduled reservation
    reservation = restaurant.create_scheduled_reservation("Alice", 4, now + timedelta(hours=1))
    print("Reservation created:", reservation)

    # Walk-in
    walk_in = restaurant.create_walk_in_reservation("Bob", 2)
    print("Walk-in reservation:", walk_in)

    # Orders
    table = reservation.assigned_table
    restaurant.order_item(table, pasta, 2)
    restaurant.order_item(table, soup, 1)
    restaurant.order_item(table, cake, 2)

    print("Bill before cancel:", restaurant.calculate_table_bill(table))

    restaurant.cancel_item(table, cake)
    print("Bill after cancel one cake:", restaurant.calculate_table_bill(table))
```

DEEP DIVE

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from decimal import Decimal
from enum import Enum
from typing import Dict, List, Optional, Set


# =========================
# Menu / MenuItem
# =========================

class Category(str, Enum):
    MAIN = "MAIN"
    APPETIZER = "APPETIZER"
    DESSERT = "DESSERT"


@dataclass(frozen=True)
class MenuItem:
    name: str
    description: str
    price: Decimal
    category: Category


class Menu:
    def __init__(self):
        self.menu_items: Dict[str, MenuItem] = {}

    def add_item(self, item: MenuItem) -> None:
        self.menu_items[item.name] = item

    def get_item(self, name: str) -> Optional[MenuItem]:
        return self.menu_items.get(name)

    def get_menu_items(self) -> Dict[str, MenuItem]:
        return dict(self.menu_items)


# =========================
# OrderItem
# =========================

class OrderStatus(str, Enum):
    PENDING = "PENDING"
    SENT_TO_KITCHEN = "SENT_TO_KITCHEN"
    DELIVERED = "DELIVERED"
    CANCELED = "CANCELED"


@dataclass
class OrderItem:
    item: MenuItem
    status: OrderStatus = OrderStatus.PENDING

    def send_to_kitchen(self) -> None:
        if self.status == OrderStatus.PENDING:
            self.status = OrderStatus.SENT_TO_KITCHEN

    def deliver_to_customer(self) -> None:
        if self.status == OrderStatus.SENT_TO_KITCHEN:
            self.status = OrderStatus.DELIVERED

    def cancel(self) -> None:
        if self.status in {OrderStatus.PENDING, OrderStatus.SENT_TO_KITCHEN}:
            self.status = OrderStatus.CANCELED


# =========================
# Reservation
# =========================

@dataclass(frozen=True)
class Reservation:
    party_name: str
    party_size: int
    time: datetime
    assigned_table: "Table"


# =========================
# Table
# =========================

class Table:
    def __init__(self, table_id: int, capacity: int):
        self.table_id = table_id
        self.capacity = capacity
        self.reservations: Dict[datetime, Reservation] = {}
        self.ordered_items: Dict[MenuItem, List[OrderItem]] = {}

    def calculate_bill_amount(self) -> Decimal:
        total = Decimal("0.00")
        for order_items in self.ordered_items.values():
            for order_item in order_items:
                if order_item.status != OrderStatus.CANCELED:
                    total += order_item.item.price
        return total

    def add_order(self, item: MenuItem, quantity: int = 1) -> None:
        for _ in range(quantity):
            self._add_single_order(item)

    def _add_single_order(self, item: MenuItem) -> None:
        if item not in self.ordered_items:
            self.ordered_items[item] = []
        self.ordered_items[item].append(OrderItem(item))

    def remove_order(self, item: MenuItem) -> None:
        order_items = self.ordered_items.get(item)
        if order_items:
            order_items.pop(0)
            if not order_items:
                del self.ordered_items[item]

    def get_ordered_items(self) -> Dict[MenuItem, List[OrderItem]]:
        return self.ordered_items

    def is_available_at(self, reservation_time: datetime) -> bool:
        return reservation_time not in self.reservations

    def add_reservation(self, reservation: Reservation) -> None:
        self.reservations[reservation.time] = reservation

    def remove_reservation(self, reservation_time: datetime) -> None:
        self.reservations.pop(reservation_time, None)

    def __repr__(self) -> str:
        return f"Table(id={self.table_id}, capacity={self.capacity})"


# =========================
# Layout
# =========================

class Layout:
    def __init__(self, table_capacities: List[int]):
        self.tables_by_id: Dict[int, Table] = {}
        self.tables_by_capacity: Dict[int, Set[Table]] = {}

        for i, capacity in enumerate(table_capacities):
            table = Table(i, capacity)
            self.tables_by_id[i] = table
            if capacity not in self.tables_by_capacity:
                self.tables_by_capacity[capacity] = set()
            self.tables_by_capacity[capacity].add(table)

    def find_available_table(self, party_size: int, reservation_time: datetime) -> Optional[Table]:
        for capacity in sorted(self.tables_by_capacity.keys()):
            if capacity >= party_size:
                for table in self.tables_by_capacity[capacity]:
                    if table.is_available_at(reservation_time):
                        return table
        return None


# =========================
# ReservationManager
# =========================

class ReservationManager:
    def __init__(self, layout: Layout):
        self.layout = layout
        self.reservations: Set[Reservation] = set()

    def find_available_time_slots(
        self,
        range_start: datetime,
        range_end: datetime,
        party_size: int,
    ) -> List[datetime]:
        current = range_start
        possible_reservations: List[datetime] = []

        while current <= range_end:
            available_table = self.layout.find_available_table(party_size, current)
            if available_table is not None:
                possible_reservations.append(current)
            current = current + timedelta(hours=1)

        return possible_reservations

    def create_reservation(
        self,
        party_name: str,
        party_size: int,
        desired_time: datetime,
    ) -> Reservation:
        desired_time = desired_time.replace(minute=0, second=0, microsecond=0)
        table = self.layout.find_available_table(party_size, desired_time)

        if table is None:
            raise ValueError("No available table for the requested time and party size")

        reservation = Reservation(party_name, party_size, desired_time, table)
        table.add_reservation(reservation)
        self.reservations.add(reservation)
        return reservation

    def remove_reservation(
        self,
        party_name: str,
        party_size: int,
        reservation_time: datetime,
    ) -> None:
        for reservation in list(self.reservations):
            if (
                reservation.time == reservation_time
                and reservation.party_size == party_size
                and reservation.party_name == party_name
            ):
                table = reservation.assigned_table
                table.remove_reservation(reservation_time)
                self.reservations.remove(reservation)
                return


# =========================
# Command Pattern for order actions
# =========================

class OrderCommand(ABC):
    @abstractmethod
    def execute(self) -> None:
        pass


class SendToKitchenCommand(OrderCommand):
    def __init__(self, order_item: OrderItem):
        self.order_item = order_item

    def execute(self) -> None:
        self.order_item.send_to_kitchen()


class DeliverCommand(OrderCommand):
    def __init__(self, order_item: OrderItem):
        self.order_item = order_item

    def execute(self) -> None:
        self.order_item.deliver_to_customer()


class CancelCommand(OrderCommand):
    def __init__(self, order_item: OrderItem):
        self.order_item = order_item

    def execute(self) -> None:
        self.order_item.cancel()


class OrderManager:
    def __init__(self):
        self.command_queue: List[OrderCommand] = []

    def add_command(self, command: OrderCommand) -> None:
        self.command_queue.append(command)

    def execute_commands(self) -> None:
        for command in self.command_queue:
            command.execute()
        self.command_queue.clear()


# =========================
# Restaurant
# =========================

class Restaurant:
    def __init__(self, name: str, menu: Menu, layout: Layout):
        self.name = name
        self.menu = menu
        self.layout = layout
        self.reservation_manager = ReservationManager(layout)
        self.order_manager = OrderManager()

    def find_available_time_slots(
        self,
        range_start: datetime,
        range_end: datetime,
        party_size: int,
    ) -> List[datetime]:
        return self.reservation_manager.find_available_time_slots(
            range_start, range_end, party_size
        )

    def create_scheduled_reservation(
        self,
        party_name: str,
        party_size: int,
        time: datetime,
    ) -> Reservation:
        return self.reservation_manager.create_reservation(
            party_name, party_size, time
        )

    def remove_reservation(
        self,
        party_name: str,
        party_size: int,
        reservation_time: datetime,
    ) -> None:
        self.reservation_manager.remove_reservation(
            party_name, party_size, reservation_time
        )

    def create_walk_in_reservation(
        self,
        party_name: str,
        party_size: int,
    ) -> Reservation:
        return self.reservation_manager.create_reservation(
            party_name, party_size, datetime.now()
        )

    def order_item(self, table: Table, item: MenuItem, quantity: int = 1) -> None:
        for _ in range(quantity):
            table.add_order(item, 1)
            order_items = table.get_ordered_items().get(item)
            if order_items:
                last_order = order_items[-1]
                send_to_kitchen = SendToKitchenCommand(last_order)
                self.order_manager.add_command(send_to_kitchen)
                self.order_manager.execute_commands()

    def cancel_item(self, table: Table, item: MenuItem) -> None:
        order_items = table.get_ordered_items().get(item)
        if order_items:
            last_order = order_items[-1]
            cancel_order = CancelCommand(last_order)
            self.order_manager.add_command(cancel_order)
            self.order_manager.execute_commands()
            table.remove_order(item)

    def deliver_item(self, table: Table, item: MenuItem) -> None:
        order_items = table.get_ordered_items().get(item)
        if order_items:
            last_order = order_items[-1]
            deliver_order = DeliverCommand(last_order)
            self.order_manager.add_command(deliver_order)
            self.order_manager.execute_commands()

    def calculate_table_bill(self, table: Table) -> Decimal:
        return table.calculate_bill_amount()


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    # Menu setup
    menu = Menu()
    pasta = MenuItem("Pasta", "Creamy white sauce pasta", Decimal("12.50"), Category.MAIN)
    soup = MenuItem("Soup", "Tomato basil soup", Decimal("6.00"), Category.APPETIZER)
    cake = MenuItem("Cake", "Chocolate lava cake", Decimal("8.00"), Category.DESSERT)

    menu.add_item(pasta)
    menu.add_item(soup)
    menu.add_item(cake)

    # Layout setup
    layout = Layout([2, 2, 4, 4, 6, 8])

    # Restaurant
    restaurant = Restaurant("Food Palace", menu, layout)

    # Find slots
    now = datetime.now().replace(minute=0, second=0, microsecond=0)
    slots = restaurant.find_available_time_slots(now, now + timedelta(hours=3), 4)
    print("Available slots:", slots)

    # Scheduled reservation
    reservation = restaurant.create_scheduled_reservation("Alice", 4, now + timedelta(hours=1))
    print("Reservation created:", reservation)

    # Walk-in
    walk_in = restaurant.create_walk_in_reservation("Bob", 2)
    print("Walk-in reservation:", walk_in)

    # Orders
    table = reservation.assigned_table
    restaurant.order_item(table, pasta, 2)
    restaurant.order_item(table, soup, 1)
    restaurant.order_item(table, cake, 2)

    print("\nOrder statuses after sending to kitchen:")
    for menu_item, order_list in table.get_ordered_items().items():
        print(menu_item.name, [order.status.value for order in order_list])

    restaurant.deliver_item(table, pasta)
    restaurant.cancel_item(table, cake)

    print("\nOrder statuses after one pasta delivered and one cake canceled:")
    for menu_item, order_list in table.get_ordered_items().items():
        print(menu_item.name, [order.status.value for order in order_list])

    print("\nBill:", restaurant.calculate_table_bill(table))
```

## Design Patterns, SOLID Principles, and OOP Concepts Used in the Advanced Restaurant LLD

# 1. Design Patterns Used

## 1.1 Facade Pattern

**Where used**
- `Restaurant`

**Why we used it**
`Restaurant` acts as the main entry point for the whole restaurant system.

Clients do not directly coordinate:
- `Menu`
- `Layout`
- `ReservationManager`
- `Table`
- `OrderManager`

Instead, they use:
- `find_available_time_slots()`
- `create_scheduled_reservation()`
- `remove_reservation()`
- `create_walk_in_reservation()`
- `order_item()`
- `cancel_item()`
- `deliver_item()`
- `calculate_table_bill()`

This hides internal complexity and provides a simpler interface.

**Simple reason**  
Facade makes a multi-component system easier to use.

---

## 1.2 Command Pattern

**Where used**
- `OrderCommand`
- `SendToKitchenCommand`
- `DeliverCommand`
- `CancelCommand`
- `OrderManager`

**Why we used it**
Order related actions need to be represented as distinct executable tasks.

Examples:
- send item to kitchen
- deliver item to customer
- cancel item

Instead of directly modifying `OrderItem` status from different places, we wrap each action into a command object.

**Benefits**
- decouples action request from execution
- supports centralized queueing
- easier to extend with retry, audit, batching, or delay support later
- cleaner coordination during peak restaurant hours

**Simple reason**  
Command is useful when actions should be treated as objects and centrally managed.

---

## 1.3 Manager / Coordinator Pattern

**Where used**
- `ReservationManager`
- `OrderManager`

**Why we used it**

### `ReservationManager`
Coordinates:
- available slot discovery
- reservation creation
- reservation removal
- table assignment for reservations

### `OrderManager`
Coordinates:
- collecting order commands
- centralized command execution
- kitchen flow tracking

These classes organize workflows instead of overloading entity classes like `Table`.

**Simple reason**  
Coordinator classes keep business workflows organized and modular.

---

## 1.4 Composition Over Inheritance

**Where used**
- `Restaurant` contains:
  - `Menu`
  - `Layout`
  - `ReservationManager`
  - `OrderManager`
- `Layout` contains tables
- `Table` contains reservations and ordered items
- `OrderItem` contains a `MenuItem`

**Why we used it**
Instead of building large inheritance hierarchies, the design combines smaller focused objects.

Examples:
- `Restaurant` does not inherit reservation logic, it contains `ReservationManager`
- `Restaurant` does not inherit kitchen command handling, it contains `OrderManager`

**Simple reason**  
Composition makes the system easier to extend and maintain.

---

## 1.5 Strategy-ready Table Assignment Logic

**Where it is partially visible**
- `Layout.find_available_table()`

**Why it is strategy-ready**
The current design uses a fixed rule:
- find the smallest available table that fits the party

This logic can later be extracted into a strategy if needed.

Examples:
- smallest fit strategy
- VIP optimized strategy
- least recently used table strategy
- waiter balanced assignment strategy

**Simple reason**  
The current design is ready for future strategy extraction if table allocation rules become more complex.

---

# 2. SOLID Principles Used

## 2.1 Single Responsibility Principle (SRP)

**Meaning**
A class should have one reason to change.

**Where used**

### `MenuItem`
Only stores menu item details.

### `Menu`
Only manages menu items.

### `OrderItem`
Only represents a single ordered item and its status.

### `Table`
Only manages one table’s reservations and orders.

### `Layout`
Only manages restaurant table arrangement and lookup.

### `Reservation`
Only stores reservation details.

### `ReservationManager`
Only handles reservation workflows.

### `OrderManager`
Only handles queued order command execution.

### `Restaurant`
Only coordinates the restaurant system at a high level.

**Why this is good**
Each class is focused, which makes the system easier to understand and change.

---

## 2.2 Open Closed Principle (OCP)

**Meaning**
Software should be open for extension but closed for modification.

**Where used**
The design makes it easy to extend without rewriting stable existing code.

Examples:
- add new order commands
- add new menu categories
- add new table assignment policies
- add new reservation validation rules
- add new kitchen workflow actions

New command types can be added by creating new `OrderCommand` implementations.

**Why this is good**
The system can evolve with minimal disruption to core logic.

---

## 2.3 Liskov Substitution Principle (LSP)

**Meaning**
Subclasses should be replaceable for their base type.

**Where used**
Any concrete command can replace `OrderCommand`.

Examples:
- `SendToKitchenCommand`
- `DeliverCommand`
- `CancelCommand`

`OrderManager` works with the abstraction and does not depend on specific command implementations.

**Why this is good**
The system remains flexible and correct regardless of which valid command type is used.

---

## 2.4 Interface Segregation Principle (ISP)

**Meaning**
Clients should not be forced to depend on methods they do not use.

**Where used**
The command abstraction is very small and focused.

### `OrderCommand`
Only defines:
- `execute()`

This makes command implementations clean and easy to understand.

**Why this is good**
Each implementation only depends on behavior it actually needs.

---

## 2.5 Dependency Inversion Principle (DIP)

**Meaning**
High level modules should depend on abstractions, not concrete implementations.

**Where used**
The order execution workflow depends on the abstract `OrderCommand` instead of concrete commands.

**Where it can improve**
The overall design still directly uses concrete classes like `ReservationManager`, `Layout`, and `OrderManager`.  
A more advanced version could inject interfaces for:
- reservation service
- table assignment policy
- kitchen queue service

**Why this is still good**
The command based order flow already shows good abstraction and decoupling.

---

# 3. OOP Concepts Used

## 3.1 Encapsulation

**Where used**
Each class bundles related data and behavior together.

Examples:
- `Table` encapsulates reservations, ordered items, and billing logic
- `OrderItem` encapsulates item status transitions
- `ReservationManager` encapsulates booking workflows
- `OrderManager` encapsulates queued order execution

**Why this is useful**
The system keeps business logic in the right class instead of scattering it everywhere.

---

## 3.2 Abstraction

**Where used**
- `OrderCommand`

The system uses `OrderCommand` as an abstraction for executable order actions.

**Why this is useful**
The rest of the system can work with generic order actions rather than hardcoding specific action types.

---

## 3.3 Inheritance

**Where used**
Concrete command classes implement the `OrderCommand` abstraction:
- `SendToKitchenCommand`
- `DeliverCommand`
- `CancelCommand`

**Why this is useful**
It allows all order actions to follow the same contract while having different behavior.

---

## 3.4 Polymorphism

**Where used**
`OrderManager` executes commands through:

- `command.execute()`

The actual behavior depends on which concrete command object is in the queue.

Examples:
- sending an item to kitchen
- delivering an item
- canceling an item

**Why this is useful**
The same method call behaves differently depending on the concrete command implementation.

---

## 3.5 Composition

**Where used**
- `Restaurant` has menu, layout, reservation manager, and order manager
- `Layout` has tables
- `Table` has ordered items and reservations
- `OrderItem` has a menu item

**Why this is useful**
The system is built from smaller reusable parts instead of relying on deep inheritance trees.

---

# 4. Additional Good Design Choices

## 4.1 Efficient Menu Lookup

**Where used**
- `Menu.menu_items`

**Why**
Using a map keyed by item name allows quick item retrieval during ordering.

---

## 4.2 Smallest Fit Table Assignment

**Where used**
- `Layout.find_available_table()`

**Why**
The layout finds the smallest suitable table, which improves table utilization and reduces waste of larger tables.

---

## 4.3 Reservation Time Normalization

**Where used**
- `ReservationManager.create_reservation()`

**Why**
Truncating reservation times to the hour keeps scheduling logic simpler and consistent.

---

## 4.4 Centralized Kitchen Order Flow

**Where used**
- `OrderManager`

**Why**
This provides better visibility into order progression and makes peak-time coordination easier than fully decentralized table-only state changes.

---

# 5. Final Summary

## Design Patterns Used
- **Facade Pattern**
- **Command Pattern**
- **Manager / Coordinator Pattern**
- **Composition over inheritance**
- **Strategy-ready table allocation design**

## SOLID Principles Used
- **Single Responsibility Principle**
- **Open Closed Principle**
- **Liskov Substitution Principle**
- **Interface Segregation Principle**
- **Dependency Inversion Principle** partially, with room for more service abstractions

## OOP Concepts Used
- **Encapsulation**
- **Abstraction**
- **Inheritance**
- **Polymorphism**
- **Composition**

---

# 6. Interview Ready Answer

In this advanced restaurant LLD, the most important patterns are **Facade** and **Command**.  
`Restaurant` acts as a facade over reservations, layout, menu, and order handling.  
The order processing enhancement uses the **Command pattern**, where actions like sending an order to the kitchen, delivering it, or canceling it are represented as separate command objects and executed through a centralized `OrderManager`.

The design also follows strong OOP principles like **encapsulation**, **abstraction**, **inheritance**, **polymorphism**, and **composition**. It aligns well with SOLID, especially **SRP**, **OCP**, and **LSP**, because each class has a focused role, the system is extensible, and new order actions can be added without modifying existing command handling logic.