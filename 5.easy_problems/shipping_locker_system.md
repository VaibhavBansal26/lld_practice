# 10. Shipping Locker System

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal
from enum import Enum
from typing import Dict, Optional, Set
import uuid


# =========================
# Exceptions
# =========================

class ShippingLockerException(Exception):
    pass


class MaximumStoragePeriodExceededException(ShippingLockerException):
    pass


class NoLockerAvailableException(ShippingLockerException):
    pass


class PackageIncompatibleException(ShippingLockerException):
    pass


# =========================
# Shipping Status
# =========================

class ShippingStatus(str, Enum):
    CREATED = "CREATED"
    IN_LOCKER = "IN_LOCKER"
    RETRIEVED = "RETRIEVED"
    EXPIRED = "EXPIRED"


# =========================
# Locker Size
# =========================

class LockerSize(Enum):
    SMALL = (
        "Small",
        Decimal("5.00"),
        Decimal("10.00"),
        Decimal("10.00"),
        Decimal("10.00"),
    )
    MEDIUM = (
        "Medium",
        Decimal("10.00"),
        Decimal("20.00"),
        Decimal("20.00"),
        Decimal("20.00"),
    )
    LARGE = (
        "Large",
        Decimal("15.00"),
        Decimal("30.00"),
        Decimal("30.00"),
        Decimal("30.00"),
    )

    def __init__(
        self,
        size_name: str,
        daily_charge: Decimal,
        width: Decimal,
        height: Decimal,
        depth: Decimal,
    ):
        self.size_name = size_name
        self.daily_charge = daily_charge
        self.width = width
        self.height = height
        self.depth = depth


# =========================
# Account Policy / Account
# =========================

@dataclass(frozen=True)
class AccountLockerPolicy:
    free_period_days: int
    maximum_period_days: int


class Account:
    def __init__(self, account_id: str, owner_name: str, locker_policy: AccountLockerPolicy):
        self.account_id = account_id
        self.owner_name = owner_name
        self.locker_policy = locker_policy
        self.usage_charges = Decimal("0.00")

    def add_usage_charge(self, amount: Decimal) -> None:
        self.usage_charges += amount

    def get_locker_policy(self) -> AccountLockerPolicy:
        return self.locker_policy

    def __repr__(self) -> str:
        return f"Account(account_id={self.account_id}, owner_name={self.owner_name}, usage_charges={self.usage_charges})"


# =========================
# Shipping Package
# =========================

class ShippingPackage(ABC):
    @abstractmethod
    def get_status(self) -> ShippingStatus:
        pass

    @abstractmethod
    def update_shipping_status(self, status: ShippingStatus) -> None:
        pass

    @abstractmethod
    def get_locker_size(self) -> LockerSize:
        pass

    @abstractmethod
    def get_user(self) -> Account:
        pass


class BasicShippingPackage(ShippingPackage):
    def __init__(
        self,
        order_id: str,
        user: Account,
        width: Decimal,
        height: Decimal,
        depth: Decimal,
    ):
        self.order_id = order_id
        self.user = user
        self.width = width
        self.height = height
        self.depth = depth
        self.status = ShippingStatus.CREATED

    def get_status(self) -> ShippingStatus:
        return self.status

    def update_shipping_status(self, status: ShippingStatus) -> None:
        self.status = status

    def get_user(self) -> Account:
        return self.user

    def get_locker_size(self) -> LockerSize:
        for size in LockerSize:
            if (
                size.width >= self.width
                and size.height >= self.height
                and size.depth >= self.depth
            ):
                return size
        raise PackageIncompatibleException("No locker size available for the package")

    def __repr__(self) -> str:
        return f"BasicShippingPackage(order_id={self.order_id}, status={self.status})"


# =========================
# Locker
# =========================

class Locker:
    def __init__(self, size: LockerSize):
        self.size = size
        self.current_package: Optional[ShippingPackage] = None
        self.assignment_date: Optional[datetime] = None
        self.access_code: Optional[str] = None

    def assign_package(self, pkg: ShippingPackage, date: datetime) -> None:
        self.current_package = pkg
        self.assignment_date = date
        self.access_code = self._generate_access_code()

    def release_locker(self) -> None:
        self.current_package = None
        self.assignment_date = None
        self.access_code = None

    def calculate_storage_charges(self) -> Decimal:
        if self.current_package is None or self.assignment_date is None:
            return Decimal("0.00")

        policy = self.current_package.get_user().get_locker_policy()
        total_days_used = (datetime.now() - self.assignment_date).days

        if total_days_used > policy.maximum_period_days:
            self.current_package.update_shipping_status(ShippingStatus.EXPIRED)
            raise MaximumStoragePeriodExceededException(
                f"Package has exceeded maximum allowed storage period of "
                f"{policy.maximum_period_days} days"
            )

        chargeable_days = max(0, total_days_used - policy.free_period_days)
        return self.size.daily_charge * Decimal(chargeable_days)

    def is_available(self) -> bool:
        return self.current_package is None

    def check_access_code(self, code: str) -> bool:
        return self.access_code is not None and self.access_code == code

    def get_access_code(self) -> Optional[str]:
        return self.access_code

    def get_package(self) -> Optional[ShippingPackage]:
        return self.current_package

    def _generate_access_code(self) -> str:
        return str(uuid.uuid4())[:8]

    def __hash__(self) -> int:
        return id(self)

    def __repr__(self) -> str:
        return f"Locker(size={self.size.name}, available={self.is_available()})"


# =========================
# Site
# =========================

class Site:
    def __init__(self, lockers: Dict[LockerSize, int]):
        self.lockers: Dict[LockerSize, Set[Locker]] = {}

        for size, count in lockers.items():
            locker_set = set()
            for _ in range(count):
                locker_set.add(Locker(size))
            self.lockers[size] = locker_set

    def find_available_locker(self, size: LockerSize) -> Optional[Locker]:
        for locker in self.lockers.get(size, set()):
            if locker.is_available():
                return locker
        return None

    def place_package(self, pkg: ShippingPackage, date: datetime) -> Locker:
        size = pkg.get_locker_size()
        locker = self.find_available_locker(size)
        if locker is not None:
            locker.assign_package(pkg, date)
            pkg.update_shipping_status(ShippingStatus.IN_LOCKER)
            return locker

        raise NoLockerAvailableException(
            f"No locker of size {size.name} is currently available"
        )


# =========================
# Notification
# =========================

class NotificationInterface(ABC):
    @abstractmethod
    def send_notification(self, message: str, account: Account) -> None:
        pass


class ConsoleNotificationService(NotificationInterface):
    def send_notification(self, message: str, account: Account) -> None:
        print(f"[Notification to {account.owner_name}] {message}")


# =========================
# Locker Manager
# =========================

class LockerManager:
    def __init__(
        self,
        site: Site,
        accounts: Dict[str, Account],
        notification_service: NotificationInterface,
    ):
        self.site = site
        self.accounts = accounts
        self.notification_service = notification_service
        self.access_code_map: Dict[str, Locker] = {}

    def assign_package(self, pkg: ShippingPackage, date: datetime) -> Locker:
        locker = self.site.place_package(pkg, date)
        access_code = locker.get_access_code()

        if access_code is not None:
            self.access_code_map[access_code] = locker
            self.notification_service.send_notification(
                f"Package assigned to locker. Access code: {access_code}",
                pkg.get_user(),
            )

        return locker

    def pick_up_package(self, access_code: str) -> Optional[Locker]:
        locker = self.access_code_map.get(access_code)
        if locker is not None and locker.check_access_code(access_code):
            try:
                charge = locker.calculate_storage_charges()
                pkg = locker.get_package()

                locker.release_locker()
                self.access_code_map.pop(access_code, None)

                if pkg is not None:
                    pkg.get_user().add_usage_charge(charge)
                    pkg.update_shipping_status(ShippingStatus.RETRIEVED)

                return locker

            except MaximumStoragePeriodExceededException:
                locker.release_locker()
                self.access_code_map.pop(access_code, None)
                return locker

        return None


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    # accounts
    standard_policy = AccountLockerPolicy(free_period_days=2, maximum_period_days=7)
    account_1 = Account("A1", "Alice", standard_policy)
    account_2 = Account("A2", "Bob", standard_policy)

    accounts = {
        account_1.account_id: account_1,
        account_2.account_id: account_2,
    }

    # site with locker counts
    site = Site({
        LockerSize.SMALL: 2,
        LockerSize.MEDIUM: 2,
        LockerSize.LARGE: 1,
    })

    # manager
    notification_service = ConsoleNotificationService()
    locker_manager = LockerManager(site, accounts, notification_service)

    # packages
    pkg_1 = BasicShippingPackage(
        order_id="ORDER-101",
        user=account_1,
        width=Decimal("8.00"),
        height=Decimal("8.00"),
        depth=Decimal("8.00"),
    )

    pkg_2 = BasicShippingPackage(
        order_id="ORDER-102",
        user=account_2,
        width=Decimal("18.00"),
        height=Decimal("18.00"),
        depth=Decimal("18.00"),
    )

    # assign packages
    locker_1 = locker_manager.assign_package(pkg_1, datetime.now())
    locker_2 = locker_manager.assign_package(pkg_2, datetime.now())

    print("Assigned locker 1 access code:", locker_1.get_access_code())
    print("Assigned locker 2 access code:", locker_2.get_access_code())

    # pickup package
    picked = locker_manager.pick_up_package(locker_1.get_access_code())
    print("Locker picked up:", picked)
    print("Account 1 usage charges:", account_1.usage_charges)
    print("Package 1 status:", pkg_1.get_status())
```

**DEEP DIVE**

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal
from enum import Enum
from typing import Dict, List, Optional, Set
import uuid


# =========================
# Exceptions
# =========================

class ShippingLockerException(Exception):
    pass


class MaximumStoragePeriodExceededException(ShippingLockerException):
    pass


class NoLockerAvailableException(ShippingLockerException):
    pass


class PackageIncompatibleException(ShippingLockerException):
    pass


# =========================
# Shipping Status
# =========================

class ShippingStatus(str, Enum):
    CREATED = "CREATED"
    IN_LOCKER = "IN_LOCKER"
    RETRIEVED = "RETRIEVED"
    EXPIRED = "EXPIRED"


# =========================
# Locker Size
# =========================

class LockerSize(Enum):
    SMALL = (
        "Small",
        Decimal("5.00"),
        Decimal("10.00"),
        Decimal("10.00"),
        Decimal("10.00"),
    )
    MEDIUM = (
        "Medium",
        Decimal("10.00"),
        Decimal("20.00"),
        Decimal("20.00"),
        Decimal("20.00"),
    )
    LARGE = (
        "Large",
        Decimal("15.00"),
        Decimal("30.00"),
        Decimal("30.00"),
        Decimal("30.00"),
    )
    XLARGE = (
        "XLarge",
        Decimal("20.00"),
        Decimal("40.00"),
        Decimal("40.00"),
        Decimal("40.00"),
    )

    def __init__(
        self,
        size_name: str,
        daily_charge: Decimal,
        width: Decimal,
        height: Decimal,
        depth: Decimal,
    ):
        self.size_name = size_name
        self.daily_charge = daily_charge
        self.width = width
        self.height = height
        self.depth = depth


# =========================
# Factory Pattern
# =========================

class LockerFactory:
    @staticmethod
    def create_locker(size: LockerSize) -> "Locker":
        if size == LockerSize.SMALL:
            return Locker(LockerSize.SMALL)
        if size == LockerSize.MEDIUM:
            return Locker(LockerSize.MEDIUM)
        if size == LockerSize.LARGE:
            return Locker(LockerSize.LARGE)
        if size == LockerSize.XLARGE:
            return Locker(LockerSize.XLARGE)
        raise ValueError(f"Unsupported locker size: {size}")


# =========================
# Account Policy / Account
# =========================

@dataclass(frozen=True)
class AccountLockerPolicy:
    free_period_days: int
    maximum_period_days: int


class Account:
    def __init__(self, account_id: str, owner_name: str, locker_policy: AccountLockerPolicy):
        self.account_id = account_id
        self.owner_name = owner_name
        self.locker_policy = locker_policy
        self.usage_charges = Decimal("0.00")

    def add_usage_charge(self, amount: Decimal) -> None:
        self.usage_charges += amount

    def get_locker_policy(self) -> AccountLockerPolicy:
        return self.locker_policy

    def __repr__(self) -> str:
        return f"Account(account_id={self.account_id}, owner_name={self.owner_name}, usage_charges={self.usage_charges})"


# =========================
# Shipping Package
# =========================

class ShippingPackage(ABC):
    @abstractmethod
    def get_status(self) -> ShippingStatus:
        pass

    @abstractmethod
    def update_shipping_status(self, status: ShippingStatus) -> None:
        pass

    @abstractmethod
    def get_locker_size(self) -> LockerSize:
        pass

    @abstractmethod
    def get_user(self) -> Account:
        pass


class BasicShippingPackage(ShippingPackage):
    def __init__(
        self,
        order_id: str,
        user: Account,
        width: Decimal,
        height: Decimal,
        depth: Decimal,
    ):
        self.order_id = order_id
        self.user = user
        self.width = width
        self.height = height
        self.depth = depth
        self.status = ShippingStatus.CREATED

    def get_status(self) -> ShippingStatus:
        return self.status

    def update_shipping_status(self, status: ShippingStatus) -> None:
        self.status = status

    def get_user(self) -> Account:
        return self.user

    def get_locker_size(self) -> LockerSize:
        for size in LockerSize:
            if (
                size.width >= self.width
                and size.height >= self.height
                and size.depth >= self.depth
            ):
                return size
        raise PackageIncompatibleException("No locker size available for the package")

    def __repr__(self) -> str:
        return f"BasicShippingPackage(order_id={self.order_id}, status={self.status})"


# =========================
# Locker
# =========================

class Locker:
    def __init__(self, size: LockerSize):
        self.size = size
        self.current_package: Optional[ShippingPackage] = None
        self.assignment_date: Optional[datetime] = None
        self.access_code: Optional[str] = None

    def assign_package(self, pkg: ShippingPackage, date: datetime) -> None:
        self.current_package = pkg
        self.assignment_date = date
        self.access_code = self._generate_access_code()

    def release_locker(self) -> None:
        self.current_package = None
        self.assignment_date = None
        self.access_code = None

    def calculate_storage_charges(self) -> Decimal:
        if self.current_package is None or self.assignment_date is None:
            return Decimal("0.00")

        policy = self.current_package.get_user().get_locker_policy()
        total_days_used = (datetime.now() - self.assignment_date).days

        if total_days_used > policy.maximum_period_days:
            self.current_package.update_shipping_status(ShippingStatus.EXPIRED)
            raise MaximumStoragePeriodExceededException(
                f"Package has exceeded maximum allowed storage period of "
                f"{policy.maximum_period_days} days"
            )

        chargeable_days = max(0, total_days_used - policy.free_period_days)
        return self.size.daily_charge * Decimal(chargeable_days)

    def is_available(self) -> bool:
        return self.current_package is None

    def check_access_code(self, code: str) -> bool:
        return self.access_code is not None and self.access_code == code

    def get_access_code(self) -> Optional[str]:
        return self.access_code

    def get_package(self) -> Optional[ShippingPackage]:
        return self.current_package

    def _generate_access_code(self) -> str:
        return str(uuid.uuid4())[:8]

    def __hash__(self) -> int:
        return id(self)

    def __repr__(self) -> str:
        return f"Locker(size={self.size.name}, available={self.is_available()})"


# =========================
# Site
# =========================

class Site:
    def __init__(self, lockers: Dict[LockerSize, int]):
        self.lockers: Dict[LockerSize, Set[Locker]] = {}

        for size, count in lockers.items():
            locker_set = set()
            for _ in range(count):
                locker_set.add(LockerFactory.create_locker(size))
            self.lockers[size] = locker_set

    def find_available_locker(self, size: LockerSize) -> Optional[Locker]:
        for locker in self.lockers.get(size, set()):
            if locker.is_available():
                return locker
        return None

    def place_package(self, pkg: ShippingPackage, date: datetime) -> Locker:
        size = pkg.get_locker_size()
        locker = self.find_available_locker(size)
        if locker is not None:
            locker.assign_package(pkg, date)
            pkg.update_shipping_status(ShippingStatus.IN_LOCKER)
            return locker

        raise NoLockerAvailableException(
            f"No locker of size {size.name} is currently available"
        )


# =========================
# Observer Pattern for locker events
# =========================

class LockerEventObserver(ABC):
    @abstractmethod
    def update(self, message: str, account: Account) -> None:
        pass


class EmailNotification(LockerEventObserver):
    def update(self, message: str, account: Account) -> None:
        print(f"[Email to {account.owner_name}] {message}")


class SmsNotification(LockerEventObserver):
    def update(self, message: str, account: Account) -> None:
        print(f"[SMS to {account.owner_name}] {message}")


class AnalyticsObserver(LockerEventObserver):
    def update(self, message: str, account: Account) -> None:
        print(f"[Analytics] account={account.account_id}, event='{message}'")


# =========================
# Locker Manager
# =========================

class LockerManager:
    def __init__(
        self,
        site: Site,
        accounts: Dict[str, Account],
    ):
        self.site = site
        self.accounts = accounts
        self.access_code_map: Dict[str, Locker] = {}
        self.observers: List[LockerEventObserver] = []

    def add_observer(self, observer: LockerEventObserver) -> None:
        self.observers.append(observer)

    def remove_observer(self, observer: LockerEventObserver) -> None:
        if observer in self.observers:
            self.observers.remove(observer)

    def _notify_observers(self, message: str, account: Account) -> None:
        for observer in self.observers:
            observer.update(message, account)

    def assign_package(self, pkg: ShippingPackage, date: datetime) -> Locker:
        locker = self.site.place_package(pkg, date)
        access_code = locker.get_access_code()

        if access_code is not None:
            self.access_code_map[access_code] = locker
            self._notify_observers(
                f"Package assigned to locker. Access code: {access_code}",
                pkg.get_user(),
            )

        return locker

    def pick_up_package(self, access_code: str) -> Optional[Locker]:
        locker = self.access_code_map.get(access_code)
        if locker is not None and locker.check_access_code(access_code):
            pkg = locker.get_package()
            if pkg is None:
                return None

            try:
                charge = locker.calculate_storage_charges()
                locker.release_locker()
                self.access_code_map.pop(access_code, None)

                pkg.get_user().add_usage_charge(charge)
                pkg.update_shipping_status(ShippingStatus.RETRIEVED)

                self._notify_observers(
                    f"Package retrieved successfully. Charges applied: {charge}",
                    pkg.get_user(),
                )
                return locker

            except MaximumStoragePeriodExceededException as e:
                locker.release_locker()
                self.access_code_map.pop(access_code, None)

                self._notify_observers(
                    f"Package expired and locker released. Reason: {str(e)}",
                    pkg.get_user(),
                )
                return locker

        return None


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    # accounts
    standard_policy = AccountLockerPolicy(free_period_days=2, maximum_period_days=7)
    account_1 = Account("A1", "Alice", standard_policy)
    account_2 = Account("A2", "Bob", standard_policy)

    accounts = {
        account_1.account_id: account_1,
        account_2.account_id: account_2,
    }

    # site with locker counts
    site = Site({
        LockerSize.SMALL: 2,
        LockerSize.MEDIUM: 2,
        LockerSize.LARGE: 1,
        LockerSize.XLARGE: 1,
    })

    # locker manager
    locker_manager = LockerManager(site, accounts)

    # register observers
    locker_manager.add_observer(EmailNotification())
    locker_manager.add_observer(SmsNotification())
    locker_manager.add_observer(AnalyticsObserver())

    # packages
    pkg_1 = BasicShippingPackage(
        order_id="ORDER-101",
        user=account_1,
        width=Decimal("8.00"),
        height=Decimal("8.00"),
        depth=Decimal("8.00"),
    )

    pkg_2 = BasicShippingPackage(
        order_id="ORDER-102",
        user=account_2,
        width=Decimal("35.00"),
        height=Decimal("35.00"),
        depth=Decimal("35.00"),
    )

    # assign packages
    locker_1 = locker_manager.assign_package(pkg_1, datetime.now())
    locker_2 = locker_manager.assign_package(pkg_2, datetime.now())

    print("Assigned locker 1 access code:", locker_1.get_access_code())
    print("Assigned locker 2 access code:", locker_2.get_access_code())

    # pickup package
    picked = locker_manager.pick_up_package(locker_1.get_access_code())
    print("Locker picked up:", picked)
    print("Account 1 usage charges:", account_1.usage_charges)
    print("Package 1 status:", pkg_1.get_status())
```

## Design Patterns, SOLID Principles, and OOP Concepts Used in the Advanced Shipping Locker LLD

# 1. Design Patterns Used

## 1.1 Factory Pattern

**Where used**
- `LockerFactory`

**Why we used it**
Locker creation was originally happening directly using constructors like:

- `Locker(LockerSize.SMALL)`
- `Locker(LockerSize.MEDIUM)`
- `Locker(LockerSize.LARGE)`

That works for a small system, but it becomes rigid when new locker sizes or specialized locker types need to be introduced.

Examples:
- `XLARGE` lockers
- temperature controlled lockers
- fragile item lockers
- refrigerated lockers

By introducing `LockerFactory`, we centralize locker creation in one place.

**Benefits**
- centralized object creation
- easier extensibility
- cleaner code in `Site`
- lower maintenance cost when locker types change

**Simple reason**  
Factory is useful when object creation logic should be centralized and future locker types may vary.

---

## 1.2 Observer Pattern

**Where used**
- `LockerEventObserver`
- `EmailNotification`
- `SmsNotification`
- `AnalyticsObserver`
- `LockerManager`

**Why we used it**
Originally, `LockerManager` directly called the notification service. That tightly couples locker operations to one notification mechanism.

With the Observer pattern:
- `LockerManager` becomes the subject
- observer implementations subscribe to locker events
- multiple systems can react when an event occurs

Examples of events:
- package assigned to locker
- package retrieved
- package expired
- locker released

Examples of observers:
- email notification
- SMS notification
- analytics system
- audit logging system

**Benefits**
- decouples event source from event handlers
- allows multiple observers
- easy to add new reactions without modifying locker logic
- better scalability and maintainability

**Simple reason**  
Observer is useful when one component produces events and multiple other components may need to react.

---

## 1.3 Facade Pattern

**Where used**
- `LockerManager`

**Why we used it**
`LockerManager` acts as the main entry point for the locker workflow.

Clients do not directly coordinate:
- `Site`
- `Locker`
- access code mapping
- observer notification
- package status transitions
- account charge updates

Instead, they use:
- `assign_package()`
- `pick_up_package()`

This hides the complexity of the lower level components and presents a simpler interface.

**Simple reason**  
Facade makes a multi-component system easier to use.

---

## 1.4 Composition Over Inheritance

**Where used**
- `LockerManager` contains `Site` and observer list
- `Site` contains lockers grouped by size
- `Locker` contains package assignment state
- `BasicShippingPackage` contains `Account`
- `Account` contains `AccountLockerPolicy`

**Why we used it**
Instead of using deep inheritance hierarchies, the design combines smaller focused objects.

Examples:
- `LockerManager` does not inherit notification behavior, it keeps observer references
- `Site` does not inherit locker creation logic, it uses `LockerFactory`
- `Account` does not inherit policy logic, it contains a locker policy object

**Simple reason**  
Composition keeps the design modular and flexible.

---

## 1.5 Manager / Coordinator Pattern

**Where used**
- `LockerManager`
- `Site`

**Why we used it**
Some classes coordinate workflows instead of representing only raw entities.

### `LockerManager`
Coordinates:
- package assignment
- package pickup
- observer notifications
- access code mapping
- account charge update
- package status update

### `Site`
Coordinates:
- locker inventory by size
- finding available locker
- placing package into a locker

This keeps entity classes like `Locker` and `Account` simpler.

**Simple reason**  
Coordinator classes help organize business workflows cleanly.

---

# 2. SOLID Principles Used

## 2.1 Single Responsibility Principle (SRP)

**Meaning**
A class should have one reason to change.

**Where used**

### `Locker`
Only manages one locker’s state:
- assigned package
- assignment date
- access code
- charge calculation
- release logic

### `LockerFactory`
Only handles locker creation.

### `Site`
Only manages locker inventory and placement logic.

### `BasicShippingPackage`
Only models package data and locker size compatibility.

### `Account`
Only models account data and usage charge tracking.

### `AccountLockerPolicy`
Only defines storage policy rules.

### `LockerManager`
Only coordinates high level locker workflows.

### Observer classes
Each observer only handles its own event response logic.

**Why this is good**
Responsibilities are clearly separated, which makes the code easier to maintain, test, and extend.

---

## 2.2 Open Closed Principle (OCP)

**Meaning**
Software should be open for extension but closed for modification.

**Where used**
The design allows adding new behavior without changing stable existing code.

Examples:
- add new locker sizes like `XXLARGE`
- add new locker subclasses through the factory
- add new observers like `PushNotificationObserver`
- add new package types
- add new policy rules

The observer based event system and factory based creation both support this well.

**Why this is good**
The system can evolve without repeatedly modifying core business logic.

---

## 2.3 Liskov Substitution Principle (LSP)

**Meaning**
Subclasses or implementations should be replaceable for their base type.

**Where used**
Any implementation of `LockerEventObserver` can replace another wherever an observer is expected.

Examples:
- `EmailNotification`
- `SmsNotification`
- `AnalyticsObserver`

Also, any implementation of `ShippingPackage` can replace `BasicShippingPackage` if it follows the same contract.

**Why this is good**
The system can use different implementations without breaking client behavior.

---

## 2.4 Interface Segregation Principle (ISP)

**Meaning**
Clients should not be forced to depend on methods they do not use.

**Where used**

### `ShippingPackage`
Defines only the methods needed by the locker system:
- `get_status()`
- `update_shipping_status()`
- `get_locker_size()`
- `get_user()`

### `LockerEventObserver`
Defines only:
- `update(message, account)`

These interfaces are small, focused, and easy to implement.

**Why this is good**
Implementations only depend on behavior they actually need.

---

## 2.5 Dependency Inversion Principle (DIP)

**Meaning**
High level modules should depend on abstractions, not concrete implementations.

**Where used**
- `LockerManager` depends on `LockerEventObserver` abstraction instead of concrete notification classes
- `LockerManager` works with `ShippingPackage` abstraction instead of only `BasicShippingPackage`

**Where it can improve**
`Site` currently directly uses `LockerFactory`, and storage is in memory.  
A more advanced version could inject:
- locker repository abstraction
- event bus abstraction
- billing service abstraction

**Why this is still good**
The event system and package abstraction already show strong DIP thinking.

---

# 3. OOP Concepts Used

## 3.1 Encapsulation

**Where used**
Each class bundles data and behavior together.

Examples:
- `Locker` encapsulates assignment, access code, storage duration, and release logic
- `Account` encapsulates usage charge updates
- `BasicShippingPackage` encapsulates size compatibility logic
- `Site` encapsulates locker grouping and lookup

**Why this is useful**
Internal details are hidden inside the correct class and managed through methods.

---

## 3.2 Abstraction

**Where used**
- `ShippingPackage`
- `LockerEventObserver`

These abstractions define what behavior is required without exposing implementation details.

**Why this is useful**
The rest of the system can work with generic package and observer behavior instead of one hardcoded class.

---

## 3.3 Inheritance

**Where used**
Concrete observer classes implement `LockerEventObserver`.

Examples:
- `EmailNotification`
- `SmsNotification`
- `AnalyticsObserver`

Also, future package types can implement `ShippingPackage`.

**Why this is useful**
It allows multiple implementations to share the same contract.

---

## 3.4 Polymorphism

**Where used**
The system can call:

- `observer.update(...)`
- `pkg.get_locker_size()`
- `pkg.update_shipping_status(...)`

without caring which concrete implementation is behind the abstraction.

Examples:
- different observer types react differently to the same event
- future package types can calculate compatibility differently

**Why this is useful**
The same method call can behave differently depending on the concrete object.

---

## 3.5 Composition

**Where used**
- `LockerManager` has `Site` and observers
- `Site` has lockers
- `Locker` has a package
- `Account` has a locker policy
- `BasicShippingPackage` has a user account

**Why this is useful**
The design is assembled from reusable pieces instead of relying on large inheritance hierarchies.

---

# 4. Additional Good Design Choices

## 4.1 Precision with Decimal

**Where used**
- locker daily charges
- dimensions
- account usage charges

**Why**
Financial calculations and dimension checks should avoid floating point precision issues.

---

## 4.2 Atomic Locker Placement Flow

**Where used**
- `Site.place_package()`

**Why**
The method encapsulates:
- find locker
- assign package
- update package status

This avoids scattered logic and keeps placement consistent.

---

## 4.3 Access Code Mapping for Efficient Retrieval

**Where used**
- `LockerManager.access_code_map`

**Why**
It allows O(1) style lookup of locker by access code, which is useful during package pickup.

---

## 4.4 Policy Driven Charging

**Where used**
- `AccountLockerPolicy`
- `Locker.calculate_storage_charges()`

**Why**
This cleanly separates:
- policy rules
- locker usage calculation
- account charging

and makes the design easier to customize later.

---

# 5. Final Summary

## Design Patterns Used
- **Factory Pattern**
- **Observer Pattern**
- **Facade Pattern**
- **Composition over inheritance**
- **Manager / Coordinator Pattern**

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

In this advanced shipping locker LLD, the most important patterns are **Factory** and **Observer**.  
**Factory** is used through `LockerFactory` so locker creation is centralized and future locker sizes or specialized locker types can be added without changing the rest of the system.  
**Observer** is used to make locker event handling event driven, so package assignment, pickup, and expiration can notify multiple systems like email, SMS, or analytics without tightly coupling `LockerManager` to any one notification service.

The design also follows strong OOP principles like **encapsulation**, **abstraction**, **inheritance**, **polymorphism**, and **composition**. It aligns well with SOLID, especially **SRP**, **OCP**, and **DIP**, because responsibilities are separated clearly, the system is extensible, and high level workflows depend on abstractions instead of hardcoded implementations.