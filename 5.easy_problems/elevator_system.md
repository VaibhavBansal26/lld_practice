# 6. Elevator System

Code

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from collections import deque
from dataclasses import dataclass
from enum import Enum
from typing import List, Optional
import random


# =========================
# Enums / Status
# =========================

class Direction(str, Enum):
    UP = "UP"
    DOWN = "DOWN"
    IDLE = "IDLE"


@dataclass(frozen=True)
class ElevatorStatus:
    current_floor: int
    direction: Direction


# =========================
# Elevator Car
# =========================

class ElevatorCar:
    def __init__(self, starting_floor: int):
        self.status = ElevatorStatus(starting_floor, Direction.IDLE)
        self.target_floors = deque()

    def get_status(self) -> ElevatorStatus:
        return self.status

    def get_current_floor(self) -> int:
        return self.status.current_floor

    def get_current_direction(self) -> Direction:
        return self.status.direction

    def add_floor_request(self, floor: int) -> None:
        if floor not in self.target_floors:
            self.target_floors.append(floor)
            self._update_direction(floor)

    def is_idle(self) -> bool:
        return len(self.target_floors) == 0

    def _update_direction(self, target_floor: int) -> None:
        if self.status.current_floor < target_floor:
            self.status = ElevatorStatus(self.status.current_floor, Direction.UP)
        elif self.status.current_floor > target_floor:
            self.status = ElevatorStatus(self.status.current_floor, Direction.DOWN)
        else:
            self.status = ElevatorStatus(self.status.current_floor, Direction.IDLE)

    def move_one_step(self) -> None:
        if not self.target_floors:
            self.status = ElevatorStatus(self.status.current_floor, Direction.IDLE)
            return

        target_floor = self.target_floors[0]
        current_floor = self.status.current_floor

        if current_floor < target_floor:
            current_floor += 1
            self.status = ElevatorStatus(current_floor, Direction.UP)
        elif current_floor > target_floor:
            current_floor -= 1
            self.status = ElevatorStatus(current_floor, Direction.DOWN)
        else:
            # reached target floor
            self.target_floors.popleft()
            if self.target_floors:
                self._update_direction(self.target_floors[0])
            else:
                self.status = ElevatorStatus(current_floor, Direction.IDLE)

    def __repr__(self) -> str:
        return f"ElevatorCar(floor={self.get_current_floor()}, direction={self.get_current_direction()}, targets={list(self.target_floors)})"


# =========================
# Dispatching Strategy
# =========================

class DispatchingStrategy(ABC):
    @abstractmethod
    def select_elevator(
        self,
        elevators: List[ElevatorCar],
        floor: int,
        direction: Direction,
    ) -> Optional[ElevatorCar]:
        pass


class FirstComeFirstServeStrategy(DispatchingStrategy):
    def select_elevator(
        self,
        elevators: List[ElevatorCar],
        floor: int,
        direction: Direction,
    ) -> Optional[ElevatorCar]:
        for elevator in elevators:
            if elevator.is_idle() or elevator.get_current_direction() == direction:
                return elevator

        return random.choice(elevators) if elevators else None


class ShortestSeekTimeFirstStrategy(DispatchingStrategy):
    def select_elevator(
        self,
        elevators: List[ElevatorCar],
        floor: int,
        direction: Direction,
    ) -> Optional[ElevatorCar]:
        best_elevator = None
        shortest_distance = float("inf")

        for elevator in elevators:
            distance = abs(elevator.get_current_floor() - floor)
            if (elevator.is_idle() or elevator.get_current_direction() == direction) and distance < shortest_distance:
                best_elevator = elevator
                shortest_distance = distance

        return best_elevator


# =========================
# Elevator Dispatch
# =========================

class ElevatorDispatch:
    def __init__(self, strategy: DispatchingStrategy):
        self.strategy = strategy

    def dispatch_elevator_car(
        self,
        floor: int,
        direction: Direction,
        elevators: List[ElevatorCar],
    ) -> None:
        selected_elevator = self.strategy.select_elevator(elevators, floor, direction)
        if selected_elevator is not None:
            selected_elevator.add_floor_request(floor)


# =========================
# Elevator System
# =========================

class ElevatorSystem:
    def __init__(self, elevators: List[ElevatorCar], strategy: DispatchingStrategy):
        self.elevators = elevators
        self.dispatch_controller = ElevatorDispatch(strategy)

    def get_all_elevator_statuses(self) -> List[ElevatorStatus]:
        return [elevator.get_status() for elevator in self.elevators]

    def request_elevator(self, current_floor: int, direction: Direction) -> None:
        self.dispatch_controller.dispatch_elevator_car(current_floor, direction, self.elevators)

    def select_floor(self, car: ElevatorCar, destination_floor: int) -> None:
        car.add_floor_request(destination_floor)

    def step(self) -> None:
        for elevator in self.elevators:
            elevator.move_one_step()


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    elevators = [
        ElevatorCar(starting_floor=0),
        ElevatorCar(starting_floor=5),
        ElevatorCar(starting_floor=10),
    ]

    strategy = ShortestSeekTimeFirstStrategy()
    system = ElevatorSystem(elevators, strategy)

    print("Initial statuses:")
    for status in system.get_all_elevator_statuses():
        print(status)

    print("\nHall request from floor 3 going UP")
    system.request_elevator(3, Direction.UP)

    print("\nAfter dispatch:")
    for elevator in elevators:
        print(elevator)

    print("\nSimulating movement:")
    for i in range(5):
        system.step()
        print(f"Step {i + 1}:")
        for elevator in elevators:
            print(elevator)

    print("\nInside selected elevator, request floor 8")
    elevators[0].add_floor_request(8)

    for i in range(6):
        system.step()
        print(f"Step {i + 6}:")
        for elevator in elevators:
            print(elevator)
```

DEEP Dive

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from collections import deque
from dataclasses import dataclass
from enum import Enum
from typing import List, Optional, Set
import random


# =========================
# Enums / Status
# =========================

class Direction(str, Enum):
    UP = "UP"
    DOWN = "DOWN"
    IDLE = "IDLE"


@dataclass(frozen=True)
class ElevatorStatus:
    current_floor: int
    direction: Direction


# =========================
# Observer Pattern
# =========================

class ElevatorObserver(ABC):
    @abstractmethod
    def update(self, floor: int, direction: Direction) -> None:
        pass


class HallwayButtonPanel:
    def __init__(self, floor: int):
        self.floor = floor
        self.observers: List[ElevatorObserver] = []

    def press_button(self, direction: Direction) -> None:
        self._notify_observers(direction)

    def add_observer(self, observer: ElevatorObserver) -> None:
        self.observers.append(observer)

    def _notify_observers(self, direction: Direction) -> None:
        for observer in self.observers:
            observer.update(self.floor, direction)


# =========================
# Elevator Car
# =========================

class ElevatorCar:
    def __init__(self, starting_floor: int, accessible_floors: Set[int]):
        self.status = ElevatorStatus(starting_floor, Direction.IDLE)
        self.target_floors = deque()
        self.accessible_floors = set(accessible_floors)

        if starting_floor not in self.accessible_floors:
            raise ValueError("Starting floor must be in accessible floors")

    def get_status(self) -> ElevatorStatus:
        return self.status

    def get_current_floor(self) -> int:
        return self.status.current_floor

    def get_current_direction(self) -> Direction:
        return self.status.direction

    def get_accessible_floors(self) -> Set[int]:
        return self.accessible_floors

    def can_access_floor(self, floor: int) -> bool:
        return floor in self.accessible_floors

    def add_floor_request(self, floor: int) -> None:
        if floor in self.accessible_floors and floor not in self.target_floors:
            self.target_floors.append(floor)
            self._update_direction(floor)

    def is_idle(self) -> bool:
        return len(self.target_floors) == 0

    def _update_direction(self, target_floor: int) -> None:
        if self.status.current_floor < target_floor:
            self.status = ElevatorStatus(self.status.current_floor, Direction.UP)
        elif self.status.current_floor > target_floor:
            self.status = ElevatorStatus(self.status.current_floor, Direction.DOWN)
        else:
            self.status = ElevatorStatus(self.status.current_floor, Direction.IDLE)

    def move_one_step(self) -> None:
        if not self.target_floors:
            self.status = ElevatorStatus(self.status.current_floor, Direction.IDLE)
            return

        target_floor = self.target_floors[0]
        current_floor = self.status.current_floor

        if current_floor < target_floor:
            current_floor += 1
            self.status = ElevatorStatus(current_floor, Direction.UP)
        elif current_floor > target_floor:
            current_floor -= 1
            self.status = ElevatorStatus(current_floor, Direction.DOWN)
        else:
            self.target_floors.popleft()
            if self.target_floors:
                self._update_direction(self.target_floors[0])
            else:
                self.status = ElevatorStatus(current_floor, Direction.IDLE)

    def __repr__(self) -> str:
        return (
            f"ElevatorCar("
            f"floor={self.get_current_floor()}, "
            f"direction={self.get_current_direction()}, "
            f"targets={list(self.target_floors)}, "
            f"accessible={sorted(self.accessible_floors)})"
        )


# =========================
# Dispatching Strategy
# =========================

class DispatchingStrategy(ABC):
    @abstractmethod
    def select_elevator(
        self,
        elevators: List[ElevatorCar],
        floor: int,
        direction: Direction,
    ) -> Optional[ElevatorCar]:
        pass


class FirstComeFirstServeStrategy(DispatchingStrategy):
    def select_elevator(
        self,
        elevators: List[ElevatorCar],
        floor: int,
        direction: Direction,
    ) -> Optional[ElevatorCar]:
        for elevator in elevators:
            if elevator.can_access_floor(floor) and (
                elevator.is_idle() or elevator.get_current_direction() == direction
            ):
                return elevator

        eligible = [e for e in elevators if e.can_access_floor(floor)]
        return random.choice(eligible) if eligible else None


class ShortestSeekTimeFirstStrategy(DispatchingStrategy):
    def select_elevator(
        self,
        elevators: List[ElevatorCar],
        floor: int,
        direction: Direction,
    ) -> Optional[ElevatorCar]:
        best_elevator = None
        shortest_distance = float("inf")

        for elevator in elevators:
            distance = abs(elevator.get_current_floor() - floor)

            if (
                elevator.can_access_floor(floor)
                and (elevator.is_idle() or elevator.get_current_direction() == direction)
                and distance < shortest_distance
            ):
                best_elevator = elevator
                shortest_distance = distance

        return best_elevator


# =========================
# Elevator Dispatch Controller
# Observer implementation
# =========================

class ElevatorDispatchController(ElevatorObserver):
    def __init__(self, strategy: DispatchingStrategy, elevators: List[ElevatorCar]):
        self.strategy = strategy
        self.elevators = elevators

    def update(self, floor: int, direction: Direction) -> None:
        self.dispatch_elevator_car(floor, direction)

    def dispatch_elevator_car(self, floor: int, direction: Direction) -> None:
        selected_elevator = self.strategy.select_elevator(self.elevators, floor, direction)
        if selected_elevator is not None:
            selected_elevator.add_floor_request(floor)


# =========================
# Elevator System
# =========================

class ElevatorSystem:
    def __init__(
        self,
        elevators: List[ElevatorCar],
        strategy: DispatchingStrategy,
        hallway_panels: List[HallwayButtonPanel],
    ):
        self.elevators = elevators
        self.dispatch_controller = ElevatorDispatchController(strategy, elevators)
        self.hallway_panels = hallway_panels

        for panel in self.hallway_panels:
            panel.add_observer(self.dispatch_controller)

    def get_all_elevator_statuses(self) -> List[ElevatorStatus]:
        return [elevator.get_status() for elevator in self.elevators]

    def request_elevator(self, current_floor: int, direction: Direction) -> None:
        panel = self._get_hallway_panel(current_floor)
        if panel is None:
            raise ValueError(f"No hallway panel found for floor {current_floor}")
        panel.press_button(direction)

    def select_floor(self, car: ElevatorCar, destination_floor: int) -> None:
        car.add_floor_request(destination_floor)

    def step(self) -> None:
        for elevator in self.elevators:
            elevator.move_one_step()

    def _get_hallway_panel(self, floor: int) -> Optional[HallwayButtonPanel]:
        for panel in self.hallway_panels:
            if panel.floor == floor:
                return panel
        return None


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    # building floors: 1 to 20
    all_floors = set(range(1, 21))
    limited_floors = {1, 5, 10, 15, 20}

    elevators = [
        ElevatorCar(starting_floor=1, accessible_floors=limited_floors),
        ElevatorCar(starting_floor=3, accessible_floors=all_floors),
        ElevatorCar(starting_floor=12, accessible_floors=all_floors),
    ]

    hallway_panels = [HallwayButtonPanel(floor) for floor in range(1, 21)]

    strategy = ShortestSeekTimeFirstStrategy()
    system = ElevatorSystem(elevators, strategy, hallway_panels)

    print("Initial statuses:")
    for elevator in elevators:
        print(elevator)

    print("\nHallway request from floor 3 going UP")
    system.request_elevator(3, Direction.UP)

    print("\nAfter dispatch:")
    for elevator in elevators:
        print(elevator)

    print("\nSimulating movement for 4 steps:")
    for i in range(4):
        system.step()
        print(f"Step {i + 1}:")
        for elevator in elevators:
            print(elevator)

    print("\nInside elevator 2, select floor 18")
    system.select_floor(elevators[1], 18)

    for i in range(6):
        system.step()
        print(f"Step {i + 5}:")
        for elevator in elevators:
            print(elevator)

    print("\nHallway request from floor 15 going DOWN")
    system.request_elevator(15, Direction.DOWN)

    print("\nAfter second dispatch:")
    for elevator in elevators:
        print(elevator)
```

## Design Patterns, SOLID Principles, and OOP Concepts Used in the Advanced Elevator LLD

# 1. Design Patterns Used

## 1.1 Strategy Pattern

**Where used**
- `DispatchingStrategy`
- `FirstComeFirstServeStrategy`
- `ShortestSeekTimeFirstStrategy`

**Why we used it**
Elevator selection logic can vary depending on the dispatching algorithm.

Examples:
- first available elevator
- shortest seek time first
- future load balancing strategy
- future traffic aware strategy

Instead of hardcoding dispatch logic inside `ElevatorDispatchController`, we encapsulate each selection algorithm in its own strategy class.

**Benefits**
- easy to swap dispatch logic
- easy to test different policies
- easy to extend for future building traffic patterns

**Simple reason**  
When a behavior can vary independently, Strategy is a strong fit.

---

## 1.2 Observer Pattern

**Where used**
- `HallwayButtonPanel`
- `ElevatorObserver`
- `ElevatorDispatchController`

**Why we used it**
We want hallway button presses to notify the dispatch controller in an event driven way.

Flow:
- hallway button is pressed
- hallway panel notifies observers
- dispatch controller receives the event
- dispatch controller assigns an elevator

This removes tight coupling between:
- hallway buttons
- dispatch logic

**Benefits**
- cleaner event driven design
- easier to test
- easier to extend with more observers later
- better separation between UI input and dispatch logic

**Simple reason**  
Observer is useful when one component emits events and other components react to them.

---

## 1.3 Facade-like Coordination

**Where used**
- `ElevatorSystem`

**Why we used it**
`ElevatorSystem` acts as the main entry point for the application.

Clients do not directly coordinate:
- hallway button panels
- dispatch controller
- elevator cars
- strategy logic

Instead, they use:
- `request_elevator()`
- `select_floor()`
- `step()`
- `get_all_elevator_statuses()`

This hides lower level complexity and makes the system easier to use.

**Simple reason**  
A facade style entry point simplifies interaction with a multi component system.

---

## 1.4 Composition Over Inheritance

**Where used**
- `ElevatorSystem` contains elevators and hallway panels
- `ElevatorDispatchController` contains a dispatching strategy and elevator list
- `ElevatorCar` contains status and target floor queue

**Why we used it**
Instead of building deep inheritance trees, the system is assembled from smaller focused objects.

Examples:
- `ElevatorSystem` does not inherit dispatch behavior, it contains a dispatch controller
- `ElevatorDispatchController` does not inherit strategy behavior, it uses a strategy object

**Simple reason**  
Composition keeps the design modular and flexible.

---

## 1.5 Coordinator / Manager Pattern

**Where used**
- `ElevatorSystem`
- `ElevatorDispatchController`

**Why we used it**
Some classes coordinate workflows instead of representing domain entities.

### `ElevatorSystem`
Coordinates:
- system level requests
- hallway panels
- elevator stepping
- status collection

### `ElevatorDispatchController`
Coordinates:
- request reception
- strategy based elevator selection
- assigning floor requests

This prevents domain classes like `ElevatorCar` from becoming too heavy.

**Simple reason**  
Coordinator classes keep workflow logic organized.

---

# 2. SOLID Principles Used

## 2.1 Single Responsibility Principle (SRP)

**Meaning**
A class should have one reason to change.

**Where used**

### `ElevatorCar`
Only manages one elevator’s state, direction, accessible floors, and pending stops.

### `HallwayButtonPanel`
Only handles hallway button events and observer notification.

### `ElevatorDispatchController`
Only handles dispatching of hallway requests.

### `DispatchingStrategy`
Only defines elevator selection logic.

### `ElevatorSystem`
Only coordinates top level elevator system operations.

**Why this is good**
Each class has a focused responsibility, making the system easier to maintain and extend.

---

## 2.2 Open Closed Principle (OCP)

**Meaning**
Software should be open for extension but closed for modification.

**Where used**
You can add new dispatching strategies without changing the rest of the system.

Examples:
- `NearestIdleElevatorStrategy`
- `LoadBalancingStrategy`
- `PeakHourStrategy`

You can also add more observers later without changing hallway button behavior.

**Why this is good**
The system is flexible and ready for future features.

---

## 2.3 Liskov Substitution Principle (LSP)

**Meaning**
Subclasses should be replaceable for their base type.

**Where used**
Any concrete dispatching strategy can replace `DispatchingStrategy`.

Examples:
- `FirstComeFirstServeStrategy`
- `ShortestSeekTimeFirstStrategy`

Also, any observer implementation can replace `ElevatorObserver`.

**Why this is good**
The rest of the system works correctly regardless of which valid implementation is used.

---

## 2.4 Interface Segregation Principle (ISP)

**Meaning**
Clients should not be forced to depend on methods they do not use.

**Where used**
The interfaces are small and focused.

### `DispatchingStrategy`
Only has:
- `select_elevator()`

### `ElevatorObserver`
Only has:
- `update()`

This keeps interfaces simple and purpose specific.

**Why this is good**
Implementations only depend on the methods they actually need.

---

## 2.5 Dependency Inversion Principle (DIP)

**Meaning**
High level modules should depend on abstractions, not low level details.

**Where used**
- `ElevatorDispatchController` depends on `DispatchingStrategy`, not a concrete strategy
- `HallwayButtonPanel` depends on `ElevatorObserver`, not a concrete dispatch controller

This means high level behavior is decoupled from low level implementation details.

**Why this is good**
It improves flexibility, testability, and extensibility.

---

# 3. OOP Concepts Used

## 3.1 Encapsulation

**Where used**
Each class bundles its own data and behavior.

Examples:
- `ElevatorCar` manages floor requests, direction updates, and movement
- `HallwayButtonPanel` manages its observers and notification flow
- `ElevatorDispatchController` manages dispatch behavior
- `ElevatorSystem` manages system level orchestration

**Why this is useful**
Internal details are controlled within each class.

---

## 3.2 Abstraction

**Where used**
- `DispatchingStrategy`
- `ElevatorObserver`

These abstractions define what behavior is expected without exposing implementation details.

**Why this is useful**
The system depends on abstract behavior, which makes it easier to replace implementations.

---

## 3.3 Inheritance

**Where used**
Concrete dispatching strategies inherit from `DispatchingStrategy`:
- `FirstComeFirstServeStrategy`
- `ShortestSeekTimeFirstStrategy`

Also, observer implementations inherit from `ElevatorObserver`.

**Why this is useful**
It allows different implementations to share a common contract.

---

## 3.4 Polymorphism

**Where used**
The dispatch controller calls:

- `strategy.select_elevator(...)`

The actual behavior depends on which strategy object is passed.

Similarly, hallway button panels call:

- `observer.update(...)`

and the actual reaction depends on the observer implementation.

**Why this is useful**
The same method call can behave differently based on the concrete object.

---

## 3.5 Composition

**Where used**
- `ElevatorSystem` contains elevators and hallway panels
- `ElevatorDispatchController` contains strategy and elevators
- `ElevatorCar` contains status and a queue of target floors

**Why this is useful**
The system is built by combining simpler parts instead of relying on deep inheritance structures.

---

# 4. Additional Good Design Choices

## 4.1 Accessible Floors Modeling

**Where used**
- `accessible_floors` inside `ElevatorCar`

**Why**
This allows elevators to serve only specific floors, which matches real world systems.

Examples:
- express elevators
- service elevators
- restricted access elevators

This makes the design more realistic and extensible.

---

## 4.2 Queue Based Request Handling

**Where used**
- `target_floors` in `ElevatorCar`

**Why**
A queue preserves request order and keeps request handling simple and fair.

This is a reasonable LLD choice and easy to explain.

---

## 4.3 Event Driven Request Flow

**Where used**
- hallway button request handling through observer notifications

**Why**
This avoids direct coupling between button press logic and dispatch execution, making the system cleaner under high request volume.

---

# 5. Final Summary

## Design Patterns Used
- **Strategy Pattern**
- **Observer Pattern**
- **Facade-like system coordinator**
- **Composition over inheritance**
- **Coordinator / Manager pattern**

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

In this advanced elevator LLD, the most important patterns are **Strategy** and **Observer**.  
**Strategy** is used for dispatching logic so we can switch between algorithms like first come first serve or shortest seek time first without changing the rest of the system.  
**Observer** is used to make hallway button requests event driven, so button panels notify the dispatch controller without being tightly coupled to it.

The design also follows strong OOP principles like **encapsulation**, **abstraction**, **inheritance**, **polymorphism**, and **composition**. It aligns well with SOLID, especially **SRP**, **OCP**, and **DIP**, because each class has a focused job, the system is extensible, and high level modules depend on abstractions rather than concrete implementations.