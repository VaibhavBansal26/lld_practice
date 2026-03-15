# 7. Grocery Store Inventory System

What this Python version includes

Item

Catalog

Inventory

DiscountCampaign

DiscountCriteria

DiscountCalculationStrategy

OrderItem

Order

Checkout

Receipt

GroceryStoreSystem

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from decimal import Decimal
from typing import Dict, List, Optional
import uuid


# =========================
# Item
# =========================

@dataclass
class Item:
    name: str
    barcode: str
    category: str
    price: Decimal


# =========================
# Catalog
# =========================

class Catalog:
    def __init__(self):
        self.items: Dict[str, Item] = {}

    def update_item(self, item: Item) -> None:
        self.items[item.barcode] = item

    def remove_item(self, barcode: str) -> None:
        self.items.pop(barcode, None)

    def get_item(self, barcode: str) -> Optional[Item]:
        return self.items.get(barcode)


# =========================
# Inventory
# =========================

class Inventory:
    def __init__(self):
        self.stock: Dict[str, int] = {}

    def add_stock(self, barcode: str, count: int) -> None:
        self.stock[barcode] = self.stock.get(barcode, 0) + count

    def reduce_stock(self, barcode: str, count: int) -> None:
        self.stock[barcode] = self.stock.get(barcode, 0) - count

    def get_stock(self, barcode: str) -> int:
        return self.stock.get(barcode, 0)


# =========================
# Discount Criteria
# =========================

class DiscountCriteria(ABC):
    @abstractmethod
    def is_applicable(self, item: Item) -> bool:
        pass


class CategoryCriteria(DiscountCriteria):
    def __init__(self, category: str):
        self.category = category

    def is_applicable(self, item: Item) -> bool:
        return item.category == self.category


class BarcodeCriteria(DiscountCriteria):
    def __init__(self, barcode: str):
        self.barcode = barcode

    def is_applicable(self, item: Item) -> bool:
        return item.barcode == self.barcode


# =========================
# Discount Calculation Strategy
# =========================

class DiscountCalculationStrategy(ABC):
    @abstractmethod
    def calculate_discounted_price(self, original_price: Decimal) -> Decimal:
        pass


class PercentageDiscountStrategy(DiscountCalculationStrategy):
    def __init__(self, percentage: Decimal):
        self.percentage = percentage

    def calculate_discounted_price(self, original_price: Decimal) -> Decimal:
        discount_factor = Decimal("1") - (self.percentage / Decimal("100"))
        return original_price * discount_factor


class FixedAmountDiscountStrategy(DiscountCalculationStrategy):
    def __init__(self, amount: Decimal):
        self.amount = amount

    def calculate_discounted_price(self, original_price: Decimal) -> Decimal:
        discounted = original_price - self.amount
        return discounted if discounted > Decimal("0") else Decimal("0")


# =========================
# Discount Campaign
# =========================

class DiscountCampaign:
    def __init__(
        self,
        discount_id: str,
        name: str,
        criteria: DiscountCriteria,
        calculation_strategy: DiscountCalculationStrategy,
    ):
        self.discount_id = discount_id
        self.name = name
        self.criteria = criteria
        self.calculation_strategy = calculation_strategy

    def is_applicable(self, item: Item) -> bool:
        return self.criteria.is_applicable(item)

    def calculate_discount(self, order_item: OrderItem) -> Decimal:
        return self.calculation_strategy.calculate_discounted_price(order_item.calculate_price())


# =========================
# Order Item
# =========================

@dataclass(frozen=True)
class OrderItem:
    item: Item
    quantity: int

    def calculate_price(self) -> Decimal:
        return self.item.price * Decimal(self.quantity)

    def calculate_price_with_discount(self, new_discount: DiscountCampaign) -> Decimal:
        return new_discount.calculate_discount(self)


# =========================
# Order
# =========================

class Order:
    def __init__(self):
        self.order_id = str(uuid.uuid4())
        self.items: List[OrderItem] = []
        self.applied_discounts: Dict[OrderItem, DiscountCampaign] = {}
        self.payment_amount: Decimal = Decimal("0")

    def add_item(self, item: OrderItem) -> None:
        self.items.append(item)

    def calculate_subtotal(self) -> Decimal:
        total = Decimal("0")
        for item in self.items:
            total += item.calculate_price()
        return total

    def calculate_total(self) -> Decimal:
        total = Decimal("0")
        for item in self.items:
            discount = self.applied_discounts.get(item)
            if discount is not None:
                total += item.calculate_price_with_discount(discount)
            else:
                total += item.calculate_price()
        return total

    def apply_discount(self, item: OrderItem, discount: DiscountCampaign) -> None:
        self.applied_discounts[item] = discount

    def calculate_change(self) -> Decimal:
        return self.payment_amount - self.calculate_total()

    def set_payment(self, payment_amount: Decimal) -> None:
        self.payment_amount = payment_amount

    def get_applied_discounts(self) -> Dict[OrderItem, DiscountCampaign]:
        return self.applied_discounts


# =========================
# Receipt
# =========================

@dataclass
class Receipt:
    order: Order

    def generate_summary(self) -> dict:
        return {
            "order_id": self.order.order_id,
            "subtotal": self.order.calculate_subtotal(),
            "total": self.order.calculate_total(),
            "payment_amount": self.order.payment_amount,
            "change": self.order.calculate_change(),
        }


# =========================
# Checkout
# =========================

class Checkout:
    def __init__(self, active_discounts: List[DiscountCampaign]):
        self.active_discounts = active_discounts
        self.start_new_order()

    def start_new_order(self) -> None:
        self.current_order = Order()

    def process_payment(self, payment_amount: Decimal) -> Decimal:
        self.current_order.set_payment(payment_amount)
        return self.current_order.calculate_change()

    def add_item_to_order(self, item: Item, quantity: int) -> None:
        order_item = OrderItem(item, quantity)
        self.current_order.add_item(order_item)

        for new_discount in self.active_discounts:
            if new_discount.is_applicable(item):
                if order_item in self.current_order.get_applied_discounts():
                    existing_discount = self.current_order.get_applied_discounts()[order_item]

                    # lower discounted price is better for customer
                    if (
                        order_item.calculate_price_with_discount(new_discount)
                        < order_item.calculate_price_with_discount(existing_discount)
                    ):
                        self.current_order.apply_discount(order_item, new_discount)
                else:
                    self.current_order.apply_discount(order_item, new_discount)

    def get_receipt(self) -> Receipt:
        return Receipt(self.current_order)

    def get_order_total(self) -> Decimal:
        return self.current_order.calculate_total()


# =========================
# Grocery Store System
# =========================

class GroceryStoreSystem:
    def __init__(self):
        self.catalog = Catalog()
        self.inventory = Inventory()
        self.active_discounts: List[DiscountCampaign] = []
        self.checkout = Checkout(self.active_discounts)

    def add_or_update_item(self, item: Item) -> None:
        self.catalog.update_item(item)

    def update_inventory(self, barcode: str, count: int) -> None:
        self.inventory.add_stock(barcode, count)

    def add_discount_campaign(self, discount: DiscountCampaign) -> None:
        self.active_discounts.append(discount)

    def get_item_by_barcode(self, barcode: str) -> Optional[Item]:
        return self.catalog.get_item(barcode)

    def remove_item(self, barcode: str) -> None:
        self.catalog.remove_item(barcode)


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    system = GroceryStoreSystem()

    apple = Item("Apple", "111", "Fruit", Decimal("2.50"))
    milk = Item("Milk", "222", "Dairy", Decimal("4.00"))
    bread = Item("Bread", "333", "Bakery", Decimal("3.00"))

    system.add_or_update_item(apple)
    system.add_or_update_item(milk)
    system.add_or_update_item(bread)

    system.update_inventory("111", 20)
    system.update_inventory("222", 10)
    system.update_inventory("333", 15)

    fruit_discount = DiscountCampaign(
        discount_id="D1",
        name="10% off Fruit",
        criteria=CategoryCriteria("Fruit"),
        calculation_strategy=PercentageDiscountStrategy(Decimal("10")),
    )

    bread_discount = DiscountCampaign(
        discount_id="D2",
        name="$1 off Bread",
        criteria=BarcodeCriteria("333"),
        calculation_strategy=FixedAmountDiscountStrategy(Decimal("1.00")),
    )

    system.add_discount_campaign(fruit_discount)
    system.add_discount_campaign(bread_discount)

    system.checkout.add_item_to_order(apple, 2)
    system.checkout.add_item_to_order(milk, 1)
    system.checkout.add_item_to_order(bread, 1)

    print("Subtotal:", system.checkout.current_order.calculate_subtotal())
    print("Total:", system.checkout.get_order_total())

    change = system.checkout.process_payment(Decimal("20.00"))
    print("Change:", change)

    receipt = system.checkout.get_receipt()
    print("Receipt:", receipt.generate_summary())
```

**DEEP DIVE**

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass
from decimal import Decimal
from typing import Dict, List, Optional
import uuid


# =========================
# Item
# =========================

@dataclass
class Item:
    name: str
    barcode: str
    category: str
    price: Decimal


# =========================
# Catalog
# =========================

class Catalog:
    def __init__(self):
        self.items: Dict[str, Item] = {}

    def update_item(self, item: Item) -> None:
        self.items[item.barcode] = item

    def remove_item(self, barcode: str) -> None:
        self.items.pop(barcode, None)

    def get_item(self, barcode: str) -> Optional[Item]:
        return self.items.get(barcode)


# =========================
# Inventory
# =========================

class Inventory:
    def __init__(self):
        self.stock: Dict[str, int] = {}

    def add_stock(self, barcode: str, count: int) -> None:
        self.stock[barcode] = self.stock.get(barcode, 0) + count

    def reduce_stock(self, barcode: str, count: int) -> None:
        self.stock[barcode] = self.stock.get(barcode, 0) - count

    def get_stock(self, barcode: str) -> int:
        return self.stock.get(barcode, 0)


# =========================
# Discount Criteria
# =========================

class DiscountCriteria(ABC):
    @abstractmethod
    def is_applicable(self, item: Item, order_item: Optional["OrderItem"] = None) -> bool:
        pass


class CategoryCriteria(DiscountCriteria):
    def __init__(self, category: str):
        self.category = category

    def is_applicable(self, item: Item, order_item: Optional["OrderItem"] = None) -> bool:
        return item.category == self.category


class BarcodeCriteria(DiscountCriteria):
    def __init__(self, barcode: str):
        self.barcode = barcode

    def is_applicable(self, item: Item, order_item: Optional["OrderItem"] = None) -> bool:
        return item.barcode == self.barcode


class MinPriceCriteria(DiscountCriteria):
    def __init__(self, min_price: Decimal):
        self.min_price = min_price

    def is_applicable(self, item: Item, order_item: Optional["OrderItem"] = None) -> bool:
        return item.price >= self.min_price


class MinQuantityCriteria(DiscountCriteria):
    def __init__(self, min_quantity: int):
        self.min_quantity = min_quantity

    def is_applicable(self, item: Item, order_item: Optional["OrderItem"] = None) -> bool:
        return order_item is not None and order_item.quantity >= self.min_quantity


class CompositeCriteria(DiscountCriteria):
    def __init__(self, criteria_list: List[DiscountCriteria]):
        self.criteria_list = list(criteria_list)

    def is_applicable(self, item: Item, order_item: Optional["OrderItem"] = None) -> bool:
        return all(criteria.is_applicable(item, order_item) for criteria in self.criteria_list)

    def add_criteria(self, criteria: DiscountCriteria) -> None:
        self.criteria_list.append(criteria)

    def remove_criteria(self, criteria: DiscountCriteria) -> None:
        if criteria in self.criteria_list:
            self.criteria_list.remove(criteria)


class OrCriteria(DiscountCriteria):
    def __init__(self, criteria_list: List[DiscountCriteria]):
        self.criteria_list = list(criteria_list)

    def is_applicable(self, item: Item, order_item: Optional["OrderItem"] = None) -> bool:
        return any(criteria.is_applicable(item, order_item) for criteria in self.criteria_list)


# =========================
# Discount Calculation Strategy
# =========================

class DiscountCalculationStrategy(ABC):
    @abstractmethod
    def calculate_discounted_price(self, original_price: Decimal) -> Decimal:
        pass


class NoDiscountStrategy(DiscountCalculationStrategy):
    def calculate_discounted_price(self, original_price: Decimal) -> Decimal:
        return original_price


class PercentageDiscountStrategy(DiscountCalculationStrategy):
    def __init__(self, percentage: Decimal):
        self.percentage = percentage

    def calculate_discounted_price(self, original_price: Decimal) -> Decimal:
        discount_factor = Decimal("1") - (self.percentage / Decimal("100"))
        return (original_price * discount_factor).quantize(Decimal("0.01"))


class FixedAmountDiscountStrategy(DiscountCalculationStrategy):
    def __init__(self, amount: Decimal):
        self.amount = amount

    def calculate_discounted_price(self, original_price: Decimal) -> Decimal:
        discounted = original_price - self.amount
        return max(discounted, Decimal("0.00")).quantize(Decimal("0.01"))


# =========================
# Decorator Pattern for sequential discount calculation
# =========================

class FixedDiscountDecorator(DiscountCalculationStrategy):
    def __init__(self, strategy: DiscountCalculationStrategy, fixed_amount: Decimal):
        self.strategy = strategy
        self.fixed_amount = fixed_amount

    def calculate_discounted_price(self, original_price: Decimal) -> Decimal:
        base_price = self.strategy.calculate_discounted_price(original_price)
        return max(base_price - self.fixed_amount, Decimal("0.00")).quantize(Decimal("0.01"))


class PercentageDiscountDecorator(DiscountCalculationStrategy):
    def __init__(self, strategy: DiscountCalculationStrategy, additional_percentage: Decimal):
        self.strategy = strategy
        self.additional_percentage = additional_percentage

    def calculate_discounted_price(self, original_price: Decimal) -> Decimal:
        base_discounted_price = self.strategy.calculate_discounted_price(original_price)
        factor = Decimal("1") - (self.additional_percentage / Decimal("100"))
        return (base_discounted_price * factor).quantize(Decimal("0.01"))


# =========================
# Discount Campaign
# =========================

class DiscountCampaign:
    def __init__(
        self,
        discount_id: str,
        name: str,
        criteria: DiscountCriteria,
        calculation_strategy: DiscountCalculationStrategy,
    ):
        self.discount_id = discount_id
        self.name = name
        self.criteria = criteria
        self.calculation_strategy = calculation_strategy

    def is_applicable(self, item: Item, order_item: Optional["OrderItem"] = None) -> bool:
        return self.criteria.is_applicable(item, order_item)

    def calculate_discount(self, order_item: "OrderItem") -> Decimal:
        return self.calculation_strategy.calculate_discounted_price(order_item.calculate_price())


# =========================
# Order Item
# =========================

@dataclass(frozen=True)
class OrderItem:
    item: Item
    quantity: int

    def calculate_price(self) -> Decimal:
        return (self.item.price * Decimal(self.quantity)).quantize(Decimal("0.01"))

    def calculate_price_with_discount(self, new_discount: DiscountCampaign) -> Decimal:
        return new_discount.calculate_discount(self)


# =========================
# Order
# =========================

class Order:
    def __init__(self):
        self.order_id = str(uuid.uuid4())
        self.items: List[OrderItem] = []
        self.applied_discounts: Dict[OrderItem, DiscountCampaign] = {}
        self.payment_amount: Decimal = Decimal("0.00")

    def add_item(self, item: OrderItem) -> None:
        self.items.append(item)

    def calculate_subtotal(self) -> Decimal:
        total = Decimal("0.00")
        for item in self.items:
            total += item.calculate_price()
        return total.quantize(Decimal("0.01"))

    def calculate_total(self) -> Decimal:
        total = Decimal("0.00")
        for item in self.items:
            discount = self.applied_discounts.get(item)
            if discount is not None:
                total += item.calculate_price_with_discount(discount)
            else:
                total += item.calculate_price()
        return total.quantize(Decimal("0.01"))

    def apply_discount(self, item: OrderItem, discount: DiscountCampaign) -> None:
        self.applied_discounts[item] = discount

    def calculate_change(self) -> Decimal:
        return (self.payment_amount - self.calculate_total()).quantize(Decimal("0.01"))

    def set_payment(self, payment_amount: Decimal) -> None:
        self.payment_amount = payment_amount.quantize(Decimal("0.01"))

    def get_applied_discounts(self) -> Dict[OrderItem, DiscountCampaign]:
        return self.applied_discounts


# =========================
# Receipt
# =========================

@dataclass
class Receipt:
    order: Order

    def generate_summary(self) -> dict:
        return {
            "order_id": self.order.order_id,
            "subtotal": self.order.calculate_subtotal(),
            "total": self.order.calculate_total(),
            "payment_amount": self.order.payment_amount,
            "change": self.order.calculate_change(),
        }


# =========================
# Checkout
# =========================

class Checkout:
    def __init__(self, active_discounts: List[DiscountCampaign]):
        self.active_discounts = active_discounts
        self.start_new_order()

    def start_new_order(self) -> None:
        self.current_order = Order()

    def process_payment(self, payment_amount: Decimal) -> Decimal:
        self.current_order.set_payment(payment_amount)
        return self.current_order.calculate_change()

    def add_item_to_order(self, item: Item, quantity: int) -> None:
        order_item = OrderItem(item, quantity)
        self.current_order.add_item(order_item)

        for new_discount in self.active_discounts:
            if new_discount.is_applicable(item, order_item):
                if order_item in self.current_order.get_applied_discounts():
                    existing_discount = self.current_order.get_applied_discounts()[order_item]

                    # choose better discount for customer -> lower final price
                    if order_item.calculate_price_with_discount(new_discount) < order_item.calculate_price_with_discount(existing_discount):
                        self.current_order.apply_discount(order_item, new_discount)
                else:
                    self.current_order.apply_discount(order_item, new_discount)

    def get_receipt(self) -> Receipt:
        return Receipt(self.current_order)

    def get_order_total(self) -> Decimal:
        return self.current_order.calculate_total()


# =========================
# Grocery Store System
# =========================

class GroceryStoreSystem:
    def __init__(self):
        self.catalog = Catalog()
        self.inventory = Inventory()
        self.active_discounts: List[DiscountCampaign] = []
        self.checkout = Checkout(self.active_discounts)

    def add_or_update_item(self, item: Item) -> None:
        self.catalog.update_item(item)

    def update_inventory(self, barcode: str, count: int) -> None:
        self.inventory.add_stock(barcode, count)

    def add_discount_campaign(self, discount: DiscountCampaign) -> None:
        self.active_discounts.append(discount)

    def get_item_by_barcode(self, barcode: str) -> Optional[Item]:
        return self.catalog.get_item(barcode)

    def remove_item(self, barcode: str) -> None:
        self.catalog.remove_item(barcode)


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    system = GroceryStoreSystem()

    laptop = Item("Laptop", "101", "Electronics", Decimal("250.00"))
    phone = Item("Phone", "102", "Electronics", Decimal("150.00"))
    bread = Item("Bread", "103", "Food", Decimal("3.00"))
    apple = Item("Apple", "104", "Food", Decimal("2.00"))

    system.add_or_update_item(laptop)
    system.add_or_update_item(phone)
    system.add_or_update_item(bread)
    system.add_or_update_item(apple)

    system.update_inventory("101", 5)
    system.update_inventory("102", 10)
    system.update_inventory("103", 20)
    system.update_inventory("104", 50)

    # Simple category discount
    fruit_discount = DiscountCampaign(
        discount_id="D1",
        name="10% off Food",
        criteria=CategoryCriteria("Food"),
        calculation_strategy=PercentageDiscountStrategy(Decimal("10")),
    )

    # Composite criteria:
    # Electronics item AND unit price >= 200
    electronics_high_price_discount = DiscountCampaign(
        discount_id="D2",
        name="20% off premium electronics",
        criteria=CompositeCriteria([
            CategoryCriteria("Electronics"),
            MinPriceCriteria(Decimal("200.00")),
        ]),
        calculation_strategy=PercentageDiscountStrategy(Decimal("20")),
    )

    # Buy at least 3 items in food category
    food_bulk_discount = DiscountCampaign(
        discount_id="D3",
        name="Food bulk special",
        criteria=CompositeCriteria([
            CategoryCriteria("Food"),
            MinQuantityCriteria(3),
        ]),
        calculation_strategy=FixedAmountDiscountStrategy(Decimal("2.00")),
    )

    # Decorator based layered discount:
    # first apply 10% then subtract $5
    layered_discount_strategy = FixedDiscountDecorator(
        PercentageDiscountStrategy(Decimal("10")),
        Decimal("5.00"),
    )

    layered_discount = DiscountCampaign(
        discount_id="D4",
        name="Layered electronics promo",
        criteria=CategoryCriteria("Electronics"),
        calculation_strategy=layered_discount_strategy,
    )

    system.add_discount_campaign(fruit_discount)
    system.add_discount_campaign(electronics_high_price_discount)
    system.add_discount_campaign(food_bulk_discount)
    system.add_discount_campaign(layered_discount)

    # Add items to order
    system.checkout.add_item_to_order(laptop, 1)
    system.checkout.add_item_to_order(bread, 3)
    system.checkout.add_item_to_order(apple, 4)

    print("Subtotal:", system.checkout.current_order.calculate_subtotal())
    print("Total:", system.checkout.get_order_total())

    change = system.checkout.process_payment(Decimal("400.00"))
    print("Change:", change)

    receipt = system.checkout.get_receipt()
    print("Receipt:", receipt.generate_summary())

    print("\nApplied discounts:")
    for order_item, campaign in system.checkout.current_order.get_applied_discounts().items():
        print(f"{order_item.item.name} x {order_item.quantity} -> {campaign.name}")
```

## Design Patterns, SOLID Principles, and OOP Concepts Used in the Advanced Grocery Store LLD

# 1. Design Patterns Used

## 1.1 Strategy Pattern

**Where used**
- `DiscountCalculationStrategy`
- `NoDiscountStrategy`
- `PercentageDiscountStrategy`
- `FixedAmountDiscountStrategy`

**Why we used it**
Different discount campaigns may calculate discounts in different ways.

Examples:
- percentage based discount
- fixed amount discount
- no discount
- future slab based discount
- future tiered pricing discount

Instead of hardcoding all discount logic into one class, we encapsulate each pricing behavior inside a strategy.

**Benefits**
- flexible discount calculation
- easy to test each pricing rule independently
- easy to add new discount styles later

**Simple reason**  
When calculation behavior can vary independently, Strategy is a strong fit.

---

## 1.2 Composite Pattern

**Where used**
- `DiscountCriteria`
- `CompositeCriteria`
- `OrCriteria`
- `CategoryCriteria`
- `BarcodeCriteria`
- `MinPriceCriteria`
- `MinQuantityCriteria`

**Why we used it**
Real discount rules are often made up of multiple smaller conditions.

Examples:
- category is electronics **and** price is at least 200
- category is food **and** quantity is at least 3
- category is fruit **or** barcode matches a specific item

The Composite pattern allows us to combine simple criteria into larger logical rules without hardcoding nested `if else` checks.

**Benefits**
- reusable criteria building blocks
- support for nested logical conditions
- more expressive promotion rules

**Simple reason**  
Composite is useful when multiple small conditions must be combined into larger business rules.

---

## 1.3 Decorator Pattern

**Where used**
- `FixedDiscountDecorator`
- `PercentageDiscountDecorator`

**Why we used it**
Sometimes a discount needs to be applied in multiple steps.

Examples:
- apply 10 percent off first
- then subtract 5 dollars
- future cases like coupon + member discount + campaign discount

Instead of creating a separate class for every possible discount combination, we wrap one discount strategy inside another.

**Benefits**
- supports sequential discount application
- avoids explosion of subclasses
- easy to layer pricing rules dynamically

**Simple reason**  
Decorator helps when behavior needs to be added on top of an existing strategy in layers.

---

## 1.4 Facade-like System Coordinator

**Where used**
- `GroceryStoreSystem`

**Why we used it**
`GroceryStoreSystem` provides a simple interface to the rest of the system.

Clients do not directly coordinate:
- `Catalog`
- `Inventory`
- `Checkout`
- active discount campaigns

Instead, they use:
- `add_or_update_item()`
- `update_inventory()`
- `add_discount_campaign()`
- `get_item_by_barcode()`
- `remove_item()`

This makes client interaction simpler and cleaner.

**Simple reason**  
A facade style entry point hides system complexity and gives a cleaner API.

---

## 1.5 Manager / Coordinator Pattern

**Where used**
- `Checkout`
- `GroceryStoreSystem`

**Why we used it**
Some classes coordinate workflows instead of directly representing domain entities.

### `Checkout`
Coordinates:
- adding order items
- evaluating applicable discounts
- selecting the best discount
- processing payment
- generating receipt data

### `GroceryStoreSystem`
Coordinates:
- catalog updates
- inventory updates
- discount campaign registration
- access to checkout flow

**Simple reason**  
Coordinator classes organize business workflows cleanly and prevent entity classes from becoming too large.

---

## 1.6 Composition Over Inheritance

**Where used**
- `DiscountCampaign` contains:
  - `DiscountCriteria`
  - `DiscountCalculationStrategy`
- `Order` contains:
  - `OrderItem`
  - applied discount mapping
- `Checkout` contains:
  - current order
  - active discount campaigns
- `GroceryStoreSystem` contains:
  - `Catalog`
  - `Inventory`
  - `Checkout`

**Why we used it**
Instead of building deep inheritance trees, the design uses smaller cooperating objects.

**Simple reason**  
Composition keeps the design modular, flexible, and easier to change.

---

# 2. SOLID Principles Used

## 2.1 Single Responsibility Principle (SRP)

**Meaning**
A class should have one reason to change.

**Where used**

### `Item`
Only stores product data.

### `Catalog`
Only manages item lookup and storage.

### `Inventory`
Only tracks stock counts.

### `OrderItem`
Only models one purchased item with quantity and price calculation.

### `Order`
Only manages transaction level item collection, discount application, and price totals.

### `DiscountCampaign`
Only defines applicability and discounted price behavior for a campaign.

### `Checkout`
Only handles current order processing and discount application flow.

### `GroceryStoreSystem`
Only coordinates top level grocery system operations.

**Why this is good**
Each class stays focused and easier to maintain.

---

## 2.2 Open Closed Principle (OCP)

**Meaning**
Software should be open for extension but closed for modification.

**Where used**
You can add new criteria or discount strategies without changing existing stable logic.

Examples:
- `CustomerTypeCriteria`
- `DayOfWeekCriteria`
- `BuyOneGetOneStrategy`
- `TieredDiscountStrategy`

Also, you can build richer promotions by composing criteria and decorators.

**Why this is good**
The system grows easily as business rules become more complex.

---

## 2.3 Liskov Substitution Principle (LSP)

**Meaning**
Subclasses should be replaceable for their base type.

**Where used**
Any concrete `DiscountCalculationStrategy` can replace another wherever a calculation strategy is expected.

Examples:
- `PercentageDiscountStrategy`
- `FixedAmountDiscountStrategy`
- `FixedDiscountDecorator`
- `PercentageDiscountDecorator`

Any concrete `DiscountCriteria` can also replace another wherever a criteria is expected.

Examples:
- `CategoryCriteria`
- `BarcodeCriteria`
- `CompositeCriteria`
- `OrCriteria`

**Why this is good**
The rest of the system works correctly no matter which valid implementation is plugged in.

---

## 2.4 Interface Segregation Principle (ISP)

**Meaning**
Clients should not be forced to depend on methods they do not use.

**Where used**
The abstraction interfaces are small and focused.

### `DiscountCriteria`
Only has:
- `is_applicable()`

### `DiscountCalculationStrategy`
Only has:
- `calculate_discounted_price()`

This makes them easy to implement and understand.

**Why this is good**
Implementations only depend on behavior they actually need.

---

## 2.5 Dependency Inversion Principle (DIP)

**Meaning**
High level modules should depend on abstractions, not low level details.

**Where used**
- `DiscountCampaign` depends on `DiscountCriteria` abstraction
- `DiscountCampaign` depends on `DiscountCalculationStrategy` abstraction

This means campaign logic does not depend on concrete criteria or pricing implementations.

**Where it can improve**
`Checkout` and `GroceryStoreSystem` still directly use concrete classes like `Catalog` and `Inventory`.  
A more advanced version could inject interfaces for repositories and pricing engines.

**Why this is still good**
The discount architecture already shows strong DIP usage around campaign composition.

---

# 3. OOP Concepts Used

## 3.1 Encapsulation

**Where used**
Each class bundles its own data and behavior.

Examples:
- `Order` encapsulates subtotal, total, change, payment, and discount mapping
- `Inventory` encapsulates stock counts and update logic
- `Checkout` encapsulates current order processing
- `DiscountCampaign` encapsulates both applicability and calculation flow

**Why this is useful**
The internal details are managed inside the right class.

---

## 3.2 Abstraction

**Where used**
- `DiscountCriteria`
- `DiscountCalculationStrategy`

These abstractions define what needs to happen without exposing implementation details.

**Why this is useful**
The rest of the system can work with generic discount behavior, not specific implementations.

---

## 3.3 Inheritance

**Where used**
- concrete criteria inherit from `DiscountCriteria`
- concrete calculation strategies inherit from `DiscountCalculationStrategy`
- decorators also implement `DiscountCalculationStrategy`

**Why this is useful**
It allows different implementations to share the same contract.

---

## 3.4 Polymorphism

**Where used**
The system can call:

- `criteria.is_applicable(...)`
- `strategy.calculate_discounted_price(...)`

without caring which specific subclass is being used.

Examples:
- category based criteria
- barcode based criteria
- composite criteria
- percentage strategy
- fixed amount strategy
- decorated layered strategy

**Why this is useful**
The same method call behaves differently depending on the plugged in implementation.

---

## 3.5 Composition

**Where used**
- `DiscountCampaign` has one criteria and one calculation strategy
- `Order` has many order items
- `Checkout` has one order and many active discounts
- `GroceryStoreSystem` has catalog, inventory, and checkout

**Why this is useful**
The design is assembled from reusable components rather than relying on large inheritance hierarchies.

---

# 4. Additional Good Design Choices

## 4.1 Money Handling with Decimal

**Where used**
- `Item.price`
- discount amounts
- totals
- payment values

**Why**
Currency should not use `float` due to precision issues.

---

## 4.2 Best Discount Selection

**Where used**
- `Checkout.add_item_to_order()`

**Why**
If multiple campaigns apply, the system picks the one that gives the customer the lower final price.

This models realistic checkout behavior and improves customer experience.

---

## 4.3 Explicit Order Model

**Where used**
- `Order`
- `OrderItem`
- `Receipt`

**Why**
This makes checkout flow easier to reason about, test, and extend later with taxes, loyalty points, or coupons.

---

# 5. Final Summary

## Design Patterns Used
- **Strategy Pattern**
- **Composite Pattern**
- **Decorator Pattern**
- **Facade-like system coordinator**
- **Manager / Coordinator Pattern**
- **Composition over inheritance**

## SOLID Principles Used
- **Single Responsibility Principle**
- **Open Closed Principle**
- **Liskov Substitution Principle**
- **Interface Segregation Principle**
- **Dependency Inversion Principle**

## OOP Concepts Used
- **Encapsulation**
- **Abstraction**
- **Inheritance**
- **Polymorphism**
- **Composition**

---

# 6. Interview Ready Answer

In this advanced grocery store LLD, the most important patterns are **Strategy**, **Composite**, and **Decorator**.  
**Strategy** is used for discount calculation so different pricing rules like percentage and fixed discounts can be plugged in easily.  
**Composite** is used for combining multiple discount criteria into more expressive business rules.  
**Decorator** is used to layer discount calculations, such as applying a percentage discount and then a fixed discount sequentially.

The design also follows strong OOP principles like **encapsulation**, **abstraction**, **inheritance**, **polymorphism**, and **composition**. It aligns well with SOLID, especially **SRP**, **OCP**, and **DIP**, because responsibilities are separated cleanly, the system is extensible, and discount campaigns depend on abstractions rather than concrete implementations.