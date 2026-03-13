# 12. Amazon Prime


Code

It covers things like:

subscription plans

trial plan

membership status

renew

cancel

payment tracking

benefit access checks

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from decimal import Decimal
from enum import Enum
from typing import Dict, List, Optional
import uuid


# =========================
# Enums
# =========================

class MembershipStatus(str, Enum):
    PENDING = "PENDING"
    ACTIVE = "ACTIVE"
    EXPIRED = "EXPIRED"
    CANCELLED = "CANCELLED"
    TRIAL = "TRIAL"
    PAUSED = "PAUSED"


class PaymentStatus(str, Enum):
    INITIATED = "INITIATED"
    SUCCESS = "SUCCESS"
    FAILED = "FAILED"
    REFUNDED = "REFUNDED"


class BenefitType(str, Enum):
    SHIPPING = "SHIPPING"
    VIDEO = "VIDEO"
    MUSIC = "MUSIC"
    DEALS = "DEALS"
    EARLY_ACCESS = "EARLY_ACCESS"


# =========================
# Core Entities
# =========================

@dataclass(frozen=True)
class User:
    user_id: str
    name: str
    email: str


@dataclass(frozen=True)
class PrimePlan:
    plan_id: str
    name: str
    duration_days: int
    price: Decimal
    is_trial: bool = False


@dataclass
class Payment:
    payment_id: str
    amount: Decimal
    status: PaymentStatus
    created_at: datetime
    membership_id: str


@dataclass
class Membership:
    membership_id: str
    user: User
    plan: PrimePlan
    status: MembershipStatus
    start_date: datetime
    end_date: datetime
    auto_renew: bool = True
    payments: List[Payment] = field(default_factory=list)

    def is_active(self) -> bool:
        now = datetime.now()
        return (
            self.status in {MembershipStatus.ACTIVE, MembershipStatus.TRIAL}
            and self.start_date <= now <= self.end_date
        )

    def renew(self) -> None:
        current_time = datetime.now()

        if self.end_date < current_time:
            self.start_date = current_time
            self.end_date = current_time + timedelta(days=self.plan.duration_days)
        else:
            self.start_date = self.end_date
            self.end_date = self.end_date + timedelta(days=self.plan.duration_days)

        self.status = MembershipStatus.ACTIVE

    def cancel(self) -> None:
        self.status = MembershipStatus.CANCELLED

    def expire_if_needed(self) -> None:
        if (
            datetime.now() > self.end_date
            and self.status not in {MembershipStatus.CANCELLED, MembershipStatus.EXPIRED}
        ):
            self.status = MembershipStatus.EXPIRED


# =========================
# Benefits
# =========================

class Benefit(ABC):
    def __init__(self, benefit_type: BenefitType, description: str):
        self.benefit_type = benefit_type
        self.description = description

    @abstractmethod
    def is_available_for_membership(self, membership: Membership) -> bool:
        pass


class ShippingBenefit(Benefit):
    def __init__(self):
        super().__init__(BenefitType.SHIPPING, "Free fast delivery")

    def is_available_for_membership(self, membership: Membership) -> bool:
        return membership.is_active()


class PrimeVideoBenefit(Benefit):
    def __init__(self):
        super().__init__(BenefitType.VIDEO, "Prime Video access")

    def is_available_for_membership(self, membership: Membership) -> bool:
        return membership.is_active()


class PrimeMusicBenefit(Benefit):
    def __init__(self):
        super().__init__(BenefitType.MUSIC, "Prime Music access")

    def is_available_for_membership(self, membership: Membership) -> bool:
        return membership.is_active()


class DealsBenefit(Benefit):
    def __init__(self):
        super().__init__(BenefitType.DEALS, "Exclusive Prime deals")

    def is_available_for_membership(self, membership: Membership) -> bool:
        return membership.is_active()


class EarlyAccessBenefit(Benefit):
    def __init__(self):
        super().__init__(BenefitType.EARLY_ACCESS, "Early access to sales")

    def is_available_for_membership(self, membership: Membership) -> bool:
        return membership.is_active()


# =========================
# Payment Processor
# =========================

class PaymentProcessor:
    def process_payment(self, membership_id: str, amount: Decimal) -> Payment:
        return Payment(
            payment_id=str(uuid.uuid4()),
            amount=amount,
            status=PaymentStatus.SUCCESS,
            created_at=datetime.now(),
            membership_id=membership_id,
        )


# =========================
# Repository
# =========================

class MembershipRepository:
    def __init__(self):
        self.memberships: Dict[str, Membership] = {}

    def save(self, membership: Membership) -> None:
        self.memberships[membership.membership_id] = membership

    def get_by_membership_id(self, membership_id: str) -> Optional[Membership]:
        return self.memberships.get(membership_id)

    def get_by_user_id(self, user_id: str) -> Optional[Membership]:
        for membership in self.memberships.values():
            if membership.user.user_id == user_id:
                return membership
        return None

    def get_all_memberships(self) -> List[Membership]:
        return list(self.memberships.values())


# =========================
# Prime Service
# =========================

class PrimeService:
    def __init__(self, membership_repo: MembershipRepository, payment_processor: PaymentProcessor):
        self.membership_repo = membership_repo
        self.payment_processor = payment_processor

    def subscribe(self, user: User, plan: PrimePlan) -> Membership:
        existing_membership = self.membership_repo.get_by_user_id(user.user_id)
        if existing_membership and existing_membership.status in {
            MembershipStatus.ACTIVE,
            MembershipStatus.TRIAL,
        }:
            raise ValueError("User already has an active Prime membership")

        membership_id = str(uuid.uuid4())
        start_date = datetime.now()
        end_date = start_date + timedelta(days=plan.duration_days)

        status = MembershipStatus.TRIAL if plan.is_trial else MembershipStatus.ACTIVE

        membership = Membership(
            membership_id=membership_id,
            user=user,
            plan=plan,
            status=status,
            start_date=start_date,
            end_date=end_date,
            auto_renew=True,
        )

        if plan.price > Decimal("0.00"):
            payment = self.payment_processor.process_payment(membership_id, plan.price)
            membership.payments.append(payment)

        self.membership_repo.save(membership)
        return membership

    def renew_membership(self, user_id: str) -> Membership:
        membership = self.membership_repo.get_by_user_id(user_id)
        if membership is None:
            raise ValueError("Membership not found")

        if membership.status == MembershipStatus.CANCELLED:
            raise ValueError("Cancelled membership cannot be renewed directly")

        payment = self.payment_processor.process_payment(
            membership.membership_id,
            membership.plan.price,
        )
        membership.payments.append(payment)
        membership.renew()
        self.membership_repo.save(membership)
        return membership

    def cancel_membership(self, user_id: str) -> Membership:
        membership = self.membership_repo.get_by_user_id(user_id)
        if membership is None:
            raise ValueError("Membership not found")

        membership.cancel()
        self.membership_repo.save(membership)
        return membership

    def pause_membership(self, user_id: str) -> Membership:
        membership = self.membership_repo.get_by_user_id(user_id)
        if membership is None:
            raise ValueError("Membership not found")

        if membership.status not in {MembershipStatus.ACTIVE, MembershipStatus.TRIAL}:
            raise ValueError("Only active or trial memberships can be paused")

        membership.status = MembershipStatus.PAUSED
        self.membership_repo.save(membership)
        return membership

    def resume_membership(self, user_id: str) -> Membership:
        membership = self.membership_repo.get_by_user_id(user_id)
        if membership is None:
            raise ValueError("Membership not found")

        if membership.status != MembershipStatus.PAUSED:
            raise ValueError("Only paused memberships can be resumed")

        membership.expire_if_needed()
        if membership.status == MembershipStatus.EXPIRED:
            raise ValueError("Paused membership already expired and cannot be resumed")

        membership.status = MembershipStatus.ACTIVE
        self.membership_repo.save(membership)
        return membership

    def check_benefit_access(self, user_id: str, benefit: Benefit) -> bool:
        membership = self.membership_repo.get_by_user_id(user_id)
        if membership is None:
            return False

        membership.expire_if_needed()
        self.membership_repo.save(membership)
        return benefit.is_available_for_membership(membership)

    def get_membership(self, user_id: str) -> Optional[Membership]:
        membership = self.membership_repo.get_by_user_id(user_id)
        if membership is not None:
            membership.expire_if_needed()
            self.membership_repo.save(membership)
        return membership

    def process_auto_renewals(self) -> None:
        for membership in self.membership_repo.get_all_memberships():
            membership.expire_if_needed()

            if (
                membership.status == MembershipStatus.EXPIRED
                and membership.auto_renew
                and membership.plan.price > Decimal("0.00")
            ):
                payment = self.payment_processor.process_payment(
                    membership.membership_id,
                    membership.plan.price,
                )
                membership.payments.append(payment)
                membership.renew()

            self.membership_repo.save(membership)


# =========================
# Demo
# =========================

if __name__ == "__main__":
    repo = MembershipRepository()
    payment_processor = PaymentProcessor()
    prime_service = PrimeService(repo, payment_processor)

    user = User("u1", "Vaibhav", "vaibhav@example.com")

    monthly_plan = PrimePlan(
        plan_id="p1",
        name="Prime Monthly",
        duration_days=30,
        price=Decimal("14.99"),
        is_trial=False,
    )

    trial_plan = PrimePlan(
        plan_id="p2",
        name="Prime Trial",
        duration_days=30,
        price=Decimal("0.00"),
        is_trial=True,
    )

    shipping_benefit = ShippingBenefit()
    video_benefit = PrimeVideoBenefit()
    music_benefit = PrimeMusicBenefit()
    deals_benefit = DealsBenefit()
    early_access_benefit = EarlyAccessBenefit()

    print("=== Subscribe user to Prime Monthly ===")
    membership = prime_service.subscribe(user, monthly_plan)
    print("Membership ID:", membership.membership_id)
    print("Status:", membership.status)
    print("Start Date:", membership.start_date)
    print("End Date:", membership.end_date)
    print("Payments:", len(membership.payments))

    print("\n=== Check benefit access ===")
    print("Shipping:", prime_service.check_benefit_access(user.user_id, shipping_benefit))
    print("Video:", prime_service.check_benefit_access(user.user_id, video_benefit))
    print("Music:", prime_service.check_benefit_access(user.user_id, music_benefit))
    print("Deals:", prime_service.check_benefit_access(user.user_id, deals_benefit))
    print("Early Access:", prime_service.check_benefit_access(user.user_id, early_access_benefit))

    print("\n=== Cancel membership ===")
    cancelled_membership = prime_service.cancel_membership(user.user_id)
    print("Status after cancel:", cancelled_membership.status)

    print("\n=== Try access after cancel ===")
    print("Shipping:", prime_service.check_benefit_access(user.user_id, shipping_benefit))

    print("\n=== Subscribe another user to trial plan ===")
    trial_user = User("u2", "Alex", "alex@example.com")
    trial_membership = prime_service.subscribe(trial_user, trial_plan)
    print("Trial membership status:", trial_membership.status)
    print("Trial video access:", prime_service.check_benefit_access(trial_user.user_id, video_benefit))
```

## Design Patterns, SOLID Principles, and OOP Concepts Used in the Amazon Prime Membership System

# 1. Design Patterns Used

## 1.1 Strategy Pattern

**Where used**
- `Benefit`
- `ShippingBenefit`
- `PrimeVideoBenefit`
- `PrimeMusicBenefit`
- `DealsBenefit`
- `EarlyAccessBenefit`

**Why we used it**
Different Prime benefits can have different access rules.

Examples:
- shipping benefit may require active membership
- video benefit may require active or trial membership
- some benefits may be restricted for paused or cancelled users
- future benefits may have plan specific logic

Instead of hardcoding all benefit access checks inside `PrimeService`, we encapsulate each benefit’s access logic in its own class.

**Benefit**
- easy to add new benefits
- keeps benefit logic modular
- avoids large `if else` blocks

**Simple reason**  
When access rules can vary independently, Strategy is a good fit.

---

## 1.2 Facade-like Service Pattern

**Where used**
- `PrimeService`

**Why we used it**
`PrimeService` acts as the main entry point for membership related operations.

Clients do not directly coordinate:
- repository calls
- payment processing
- membership lifecycle transitions
- benefit validation

Instead, they use:
- `subscribe()`
- `renew_membership()`
- `cancel_membership()`
- `pause_membership()`
- `resume_membership()`
- `check_benefit_access()`
- `process_auto_renewals()`

This hides complexity and provides a clean interface.

**Simple reason**  
A central service makes the system easier to use and reason about.

---

## 1.3 Repository Pattern

**Where used**
- `MembershipRepository`

**Why we used it**
The repository encapsulates persistence and lookup logic for memberships.

Instead of letting `PrimeService` directly manage raw dictionaries or storage details, the repository provides methods like:
- `save()`
- `get_by_membership_id()`
- `get_by_user_id()`
- `get_all_memberships()`

**Benefit**
- separates storage from business logic
- easier to swap in memory storage with DB later
- cleaner service layer

**Simple reason**  
Repository abstracts data access from business operations.

---

## 1.4 Composition Over Inheritance

**Where used**
- `Membership` contains `User`, `PrimePlan`, and `Payment`
- `PrimeService` contains `MembershipRepository` and `PaymentProcessor`

**Why we used it**
Instead of creating deep inheritance hierarchies, the system is built by combining focused objects.

Examples:
- `Membership` does not inherit payment behavior, it stores payments
- `PrimeService` does not inherit repository or payment logic, it uses those components

**Simple reason**  
Composition keeps the design modular and flexible.

---

## 1.5 Manager / Coordinator Pattern

**Where used**
- `PrimeService`
- `PaymentProcessor`

**Why we used it**
Some classes coordinate workflows instead of representing domain entities.

### `PrimeService`
Coordinates:
- subscription creation
- renewal
- cancellation
- pause and resume
- benefit checks
- auto renewal processing

### `PaymentProcessor`
Handles:
- payment creation
- payment success tracking

This prevents entity classes like `Membership` from becoming too large.

**Simple reason**  
Coordinator classes keep workflows organized and clean.

---

# 2. SOLID Principles Used

## 2.1 Single Responsibility Principle (SRP)

**Meaning**
A class should have one reason to change.

**Where used**

### `User`
Only stores user information.

### `PrimePlan`
Only stores plan details like price, duration, and trial flag.

### `Membership`
Only manages membership state and lifecycle behaviors like renew, cancel, and expire.

### `Payment`
Only stores payment details.

### `PaymentProcessor`
Only handles payment processing.

### `MembershipRepository`
Only handles storage and retrieval of memberships.

### `PrimeService`
Only coordinates membership related business workflows.

### Benefit classes
Only handle benefit eligibility rules.

**Why this is good**
Each class is focused and easier to maintain.

---

## 2.2 Open Closed Principle (OCP)

**Meaning**
Software should be open for extension but closed for modification.

**Where used**
- new benefit types can be added easily
- new plans can be introduced
- new payment handling logic can be added
- new membership statuses can be introduced

Examples:
- `PrimeGamingBenefit`
- `PrimeReadingBenefit`
- `StudentPrimePlan`
- region specific benefit access rules

These extensions can be added without changing stable existing code.

**Why this is good**
The design grows safely without rewriting core logic.

---

## 2.3 Liskov Substitution Principle (LSP)

**Meaning**
Subclasses should be replaceable for their base type.

**Where used**
All concrete benefit classes can be used anywhere `Benefit` is expected:
- `ShippingBenefit`
- `PrimeVideoBenefit`
- `PrimeMusicBenefit`
- `DealsBenefit`
- `EarlyAccessBenefit`

`PrimeService` works with the abstract `Benefit` type, not with specific concrete classes.

**Why this is good**
Any new benefit class can be plugged in without breaking the service.

---

## 2.4 Interface Segregation Principle (ISP)

**Meaning**
Clients should not be forced to depend on methods they do not use.

**Where used**
The `Benefit` abstraction is minimal and focused:

- `is_available_for_membership()`

This keeps the interface small and purpose specific.

**Why this is good**
Benefit classes only implement the behavior they actually need.

---

## 2.5 Dependency Inversion Principle (DIP)

**Meaning**
High level modules should depend on abstractions, not low level details.

**Where used**
`PrimeService` depends on:
- `MembershipRepository`
- `PaymentProcessor`

These are injected into the constructor rather than being hardcoded inside the service.

**Where it can improve**
Right now `PaymentProcessor` and `MembershipRepository` are still concrete classes.  
A more advanced design could define interfaces like:
- `MembershipRepositoryInterface`
- `PaymentProcessorInterface`

and have `PrimeService` depend on those abstractions.

**Why this is still good**
Constructor injection already moves in the right direction and improves testability.

---

# 3. OOP Concepts Used

## 3.1 Encapsulation

**Where used**
Each class groups related data and behavior together.

Examples:

### `Membership`
Encapsulates:
- membership status
- start date
- end date
- renew logic
- cancel logic
- expiration logic

### `PaymentProcessor`
Encapsulates payment creation logic.

### `MembershipRepository`
Encapsulates membership storage and retrieval.

**Why this is useful**
Internal details are controlled inside the class, making the system easier to manage.

---

## 3.2 Abstraction

**Where used**
- `Benefit` is an abstraction
- clients interact with `PrimeService` without needing to know internal storage or payment flow

**Why this is useful**
The system exposes only what is necessary and hides implementation details.

---

## 3.3 Inheritance

**Where used**
Concrete benefit classes inherit from `Benefit`:
- `ShippingBenefit`
- `PrimeVideoBenefit`
- `PrimeMusicBenefit`
- `DealsBenefit`
- `EarlyAccessBenefit`

**Why this is useful**
It allows shared structure while supporting different implementations.

---

## 3.4 Polymorphism

**Where used**
`PrimeService.check_benefit_access()` accepts a `Benefit` object.

That means the same method can work with:
- shipping benefit
- video benefit
- music benefit
- any future benefit

Each concrete class provides its own implementation of:

- `is_available_for_membership()`

**Why this is useful**
The same interface can behave differently depending on the benefit type.

---

## 3.5 Composition

**Where used**
- `Membership` contains `User`, `PrimePlan`, and `Payment`
- `PrimeService` contains repository and payment processor

**Why this is useful**
The system is built from smaller parts instead of using large inheritance trees.

---

# 4. Additional Good Design Choices

## 4.1 Precision for Money

**Where used**
- `Decimal`

**Why**
Membership prices and payments should not use `float` because of rounding issues.

---

## 4.2 Clear Domain Modeling

**Where used**
- `User`
- `PrimePlan`
- `Membership`
- `Payment`
- `Benefit`

**Why**
The design clearly reflects the real Prime membership domain, which is good for LLD interviews.

---

## 4.3 Lifecycle Management

**Where used**
- `MembershipStatus`
- `expire_if_needed()`
- `renew()`
- `cancel()`
- `pause_membership()`
- `resume_membership()`

**Why**
Membership systems are workflow driven, so explicit lifecycle modeling makes the code more correct and easier to explain.

---

# 5. Final Summary

## Design Patterns Used
- **Strategy Pattern**
- **Facade-like Service Pattern**
- **Repository Pattern**
- **Composition over inheritance**
- **Manager / Coordinator Pattern**

## SOLID Principles Used
- **Single Responsibility Principle**
- **Open Closed Principle**
- **Liskov Substitution Principle**
- **Interface Segregation Principle**
- **Dependency Inversion Principle** partially, with room for more abstraction

## OOP Concepts Used
- **Encapsulation**
- **Abstraction**
- **Inheritance**
- **Polymorphism**
- **Composition**

---

# 6. Interview Ready Answer

In this Amazon Prime membership design, the main pattern used is the **Strategy pattern** through the `Benefit` hierarchy, because different Prime benefits can have different access rules. I also used a **Facade-like service** through `PrimeService`, which provides a clean interface for subscription, renewal, cancellation, and benefit access checks.

The design follows **SRP** by keeping membership lifecycle, payment processing, repository logic, and benefit rules in separate classes. It follows **OCP** because new benefit types and plans can be added without modifying stable existing code. It also uses core OOP concepts like **encapsulation**, **abstraction**, **inheritance**, **polymorphism**, and **composition** to keep the system modular and extensible.