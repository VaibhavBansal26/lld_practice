# 1. Parking Lot

**Goal:**

```text
Design a parking lot system that can manage parking spaces, vehicles, and parking tickets.
```

# Step 1: Understand the problem statement and requirements

This code solves a **parking lot system design** problem.

The system should support:

- Different types of vehicles: **Bike, Car, Truck**
- Different types of parking spots: **Compact, Regular, Oversized, Handicapped**
- Parking and unparking vehicles
- Assigning a suitable parking spot to a vehicle
- Generating a ticket when a vehicle enters
- Calculating parking fare when a vehicle leaves
- Supporting flexible fare calculation using multiple pricing strategies

## Functional requirements covered by the code

1. A vehicle enters the parking lot
2. The system finds an available parking spot based on vehicle size
3. The system parks the vehicle and generates a ticket
4. When the vehicle exits, the system calculates duration and fare
5. The system frees the parking spot

## Design goals achieved

- Good use of **object oriented design**
- Extensible fare calculation using the **Strategy pattern**
- Clear separation of concerns between parking logic, ticketing, and fare calculation
- Easy to extend with new vehicle types or pricing rules later

---

# Step 2: Identify the main entities and their relationships

## Main entities

### 1. Vehicle
Represents a vehicle entering the parking lot.

Subtypes:
- `Bike`
- `Car`
- `Truck`

Each vehicle has:
- `license_plate`
- `size`

---

### 2. ParkingSpot
Represents a parking space.

Subtypes:
- `CompactSpot`
- `RegularSpot`
- `OversizedSpot`
- `HandicappedSpot`

Each spot has:
- `spot_number`
- `vehicle` currently parked
- methods to check availability, occupy, and vacate the spot

---

### 3. ParkingManager
Responsible for:
- finding a suitable parking spot
- parking a vehicle
- unparking a vehicle
- maintaining mappings such as:
  - vehicle to spot
  - spot to vehicle

---

### 4. Ticket
Represents a parking session.

Contains:
- `ticket_id`
- `vehicle`
- `parking_spot`
- `entry_time`
- `exit_time`

Also helps calculate parking duration.

---

### 5. FareStrategy
An abstraction for pricing logic.

Implementations:
- `BaseFareStrategy`
- `PeakHoursFareStrategy`

---

### 6. FareCalculator
Applies one or more fare strategies in sequence.

---

### 7. ParkingLot
The top level coordinator that:
- accepts vehicle entry
- generates ticket
- processes vehicle exit
- calculates final fare

---

# Step 3: Define the classes and their attributes and methods

## 1. `Vehicle` abstract class

### Purpose
Base class for all vehicle types.

### Attributes
- `license_plate`
- `size`

### Methods
- `get_size()`
- `__hash__()`
- `__eq__()`

### Child classes
- `Bike` → `SMALL`
- `Car` → `MEDIUM`
- `Truck` → `LARGE`

---

## 2. `ParkingSpot` abstract class

### Purpose
Defines the common contract for all parking spots.

### Methods
- `is_available()`
- `occupy(vehicle)`
- `vacate()`
- `get_spot_number()`
- `get_size()`

### Child classes
- `CompactSpot`
- `RegularSpot`
- `OversizedSpot`
- `HandicappedSpot`

Each child class stores:
- `spot_number`
- `vehicle`

Each child class defines its own parking size.

---

## 3. `ParkingManager`

### Purpose
Handles spot allocation and release.

### Attributes
- `available_spots`
- `vehicle_to_spot_map`
- `spot_to_vehicle_map`

### Methods
- `find_spot_for_vehicle(vehicle)`
- `park_vehicle(vehicle)`
- `unpark_vehicle(vehicle)`
- `find_vehicle_spot(vehicle)`
- `find_spot_vehicle(spot)`

### Responsibility
This is the core parking allocation engine.

---

## 4. `Ticket`

### Purpose
Tracks the parking session for a vehicle.

### Attributes
- `ticket_id`
- `vehicle`
- `parking_spot`
- `entry_time`
- `exit_time`

### Methods
- `calculate_parking_duration()`
- `get_vehicle()`
- `get_entry_time()`
- `get_exit_time()`
- `set_exit_time(exit_time)`

---

## 5. `FareStrategy`

### Purpose
Defines the interface for all pricing rules.

### Method
- `calculate_fare(ticket, input_fare)`

This allows multiple pricing rules to be chained together.

---

## 6. `BaseFareStrategy`

### Purpose
Applies the base parking charge depending on vehicle size.

### Rates
- Small: `1.0`
- Medium: `2.0`
- Large: `3.0`

### Method
- `calculate_fare(ticket, input_fare)`

---

## 7. `PeakHoursFareStrategy`

### Purpose
Adds extra fare during peak hours.

### Rule
Peak hours:
- `7 to 10`
- `16 to 19`

### Methods
- `calculate_fare(ticket, input_fare)`
- `is_peak_hours(time)`

---

## 8. `FareCalculator`

### Purpose
Applies all fare strategies one after another.

### Attribute
- `fare_strategies`

### Method
- `calculate_fare(ticket)`

---

## 9. `ParkingLot`

### Purpose
Main interface used by the client.

### Attributes
- `parking_manager`
- `fare_calculator`

### Methods
- `generate_ticket_id()`
- `enter_vehicle(vehicle)`
- `leave_vehicle(ticket)`

This class connects parking flow and billing flow.

---

# Step 4: Implement the classes and their methods

## 1. Vehicle and parking spot types are created
The design starts by defining enums and subclasses for:
- vehicle sizes
- parking spot sizes

This makes the model clear and extensible.

---

## 2. `ParkingManager` handles spot allocation
When a vehicle enters:

- `ParkingLot.enter_vehicle(vehicle)` is called
- it delegates to `ParkingManager.park_vehicle(vehicle)`
- `ParkingManager.find_spot_for_vehicle(vehicle)` searches for an available matching spot
- if found, the spot is occupied and mappings are updated

---

## 3. Ticket is generated
After successful parking:

- a `Ticket` object is created
- entry time is stored
- a unique ticket id is generated using UUID

---

## 4. Fare calculation is strategy based
When the vehicle leaves:

- exit time is set in the ticket
- the vehicle is unparked
- `FareCalculator.calculate_fare(ticket)` applies:
  - `BaseFareStrategy`
  - then `PeakHoursFareStrategy`

This makes pricing flexible and extendable.

---

## 5. Parking spot becomes available again
After exit:

- the parked spot is vacated
- it becomes available for the next vehicle

---

# Design patterns used

## 1. Strategy Pattern
Used in:
- `FareStrategy`
- `BaseFareStrategy`
- `PeakHoursFareStrategy`

### Why it is used
It allows the system to add or change pricing rules without modifying existing logic.

### Future extensions
You can easily add:
- weekend pricing
- holiday pricing
- discount pricing
- flat first hour pricing

---

## 2. Inheritance
Used in:
- `Vehicle` → `Bike`, `Car`, `Truck`
- `ParkingSpot` → `CompactSpot`, `RegularSpot`, `OversizedSpot`, `HandicappedSpot`

### Why it is used
It helps reuse common behavior while allowing each subtype to define its specific characteristics.

---

## 3. Abstraction
Used in:
- `Vehicle`
- `ParkingSpot`
- `FareStrategy`

### Why it is used
It defines clean contracts and improves extensibility and readability.

---

# Strengths of the design

- Modular and easy to read
- Good separation of concerns
- Pricing logic is decoupled from parking logic
- Easy to extend with new rules
- Interview friendly low level design

---

# Limitations in the current implementation

## 1. Spot matching is exact only
Currently:
- small vehicle gets only small spot
- medium vehicle gets only medium spot
- large vehicle gets only large spot

In real systems:
- a bike may fit in a medium or large spot
- a car may fit in a large spot if medium is unavailable

So this logic can be improved.

---

## 2. Handicapped spot logic is incomplete
The code has a `HandicappedSpot`, but there is no special handling for:
- handicapped permits
- reserved access
- priority rules

---

## 3. Duplicate code in parking spot classes
`CompactSpot`, `RegularSpot`, `OversizedSpot`, and `HandicappedSpot` contain very similar logic.

This could be simplified by using one reusable concrete base class with configurable size.

---

## 4. No ticket repository
The system does not store:
- all active tickets
- parking history
- completed transactions

A real system would usually have a ticket store or database.

---

## 5. No gate or payment abstractions
A more complete parking lot design may also include:
- entry gate
- exit gate
- payment processor
- display board
- admin panel

---

# Interview style summary

You can explain this design like this:

> This is a parking lot low level design where vehicles are modeled using inheritance, parking spots are abstracted using a common interface, and parking allocation is handled by a ParkingManager. A Ticket stores parking session details, and fare calculation uses the Strategy pattern so multiple pricing rules can be applied independently. The ParkingLot acts as the orchestration layer for vehicle entry and exit.

---

# Very concise interview answer

## Problem
Design a parking lot that supports parking, unparking, ticket generation, and fare calculation.

## Entities
- Vehicle
- ParkingSpot
- ParkingManager
- Ticket
- FareStrategy
- FareCalculator
- ParkingLot

## Design
- Inheritance for vehicles and spots
- Strategy pattern for fare rules
- ParkingManager for spot allocation
- ParkingLot for orchestration

## Flow
Vehicle enters → spot assigned → ticket created → vehicle exits → fare calculated → spot released

**Step 5: Test the implementation with different scenarios**

```python
from abc import ABC, abstractmethod
from datetime import datetime
from decimal import Decimal
from enum import Enum
import uuid


class VehicleSize(str, Enum):
    SMALL = "SMALL"
    MEDIUM = "MEDIUM"
    LARGE = "LARGE"


class Vehicle(ABC):
    def __init__(self, license_plate: str, size: VehicleSize):
        self.license_plate = license_plate
        self.size = size

    def get_size(self) -> VehicleSize:
        return self.size

    def __hash__(self):
        return hash(self.license_plate)

    def __eq__(self, other):
        return isinstance(other, Vehicle) and self.license_plate == other.license_plate


class Bike(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleSize.SMALL)


class Car(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleSize.MEDIUM)


class Truck(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleSize.LARGE)


class ParkingSpot(ABC):
    @abstractmethod
    def is_available(self) -> bool:
        pass

    @abstractmethod
    def occupy(self, vehicle: Vehicle) -> None:
        pass

    @abstractmethod
    def vacate(self) -> None:
        pass

    @abstractmethod
    def get_spot_number(self) -> int:
        pass

    @abstractmethod
    def get_size(self) -> VehicleSize:
        pass


class CompactSpot(ParkingSpot):
    def __init__(self, spot_number: int):
        self.spot_number = spot_number
        self.vehicle = None

    def get_spot_number(self) -> int:
        return self.spot_number

    def is_available(self) -> bool:
        return self.vehicle is None

    def occupy(self, vehicle: Vehicle) -> None:
        if self.is_available():
            self.vehicle = vehicle
        else:
            raise ValueError("Spot is already occupied.")

    def vacate(self) -> None:
        self.vehicle = None

    def get_size(self) -> VehicleSize:
        return VehicleSize.SMALL


class RegularSpot(ParkingSpot):
    def __init__(self, spot_number: int):
        self.spot_number = spot_number
        self.vehicle = None

    def get_spot_number(self) -> int:
        return self.spot_number

    def is_available(self) -> bool:
        return self.vehicle is None

    def occupy(self, vehicle: Vehicle) -> None:
        if self.is_available():
            self.vehicle = vehicle
        else:
            raise ValueError("Spot is already occupied.")

    def vacate(self) -> None:
        self.vehicle = None

    def get_size(self) -> VehicleSize:
        return VehicleSize.MEDIUM


class OversizedSpot(ParkingSpot):
    def __init__(self, spot_number: int):
        self.spot_number = spot_number
        self.vehicle = None

    def get_spot_number(self) -> int:
        return self.spot_number

    def is_available(self) -> bool:
        return self.vehicle is None

    def occupy(self, vehicle: Vehicle) -> None:
        if self.is_available():
            self.vehicle = vehicle
        else:
            raise ValueError("Spot is already occupied.")

    def vacate(self) -> None:
        self.vehicle = None

    def get_size(self) -> VehicleSize:
        return VehicleSize.LARGE


class HandicappedSpot(ParkingSpot):
    def __init__(self, spot_number: int):
        self.spot_number = spot_number
        self.vehicle = None

    def get_spot_number(self) -> int:
        return self.spot_number

    def is_available(self) -> bool:
        return self.vehicle is None

    def occupy(self, vehicle: Vehicle) -> None:
        if self.is_available():
            self.vehicle = vehicle
        else:
            raise ValueError("Spot is already occupied.")

    def vacate(self) -> None:
        self.vehicle = None

    def get_size(self) -> VehicleSize:
        return VehicleSize.MEDIUM


class ParkingManager:
    def __init__(self, available_spots: dict[VehicleSize, list[ParkingSpot]]):
        self.available_spots = available_spots
        self.vehicle_to_spot_map: dict[Vehicle, ParkingSpot] = {}
        self.spot_to_vehicle_map: dict[ParkingSpot, Vehicle] = {}

    def find_spot_for_vehicle(self, vehicle: Vehicle):
        vehicle_size = vehicle.get_size()

        # Exact match only
        spots = self.available_spots.get(vehicle_size, [])
        for spot in spots:
            if spot.is_available():
                return spot

        return None

    def park_vehicle(self, vehicle: Vehicle):
        spot = self.find_spot_for_vehicle(vehicle)
        if spot is not None:
            spot.occupy(vehicle)
            self.vehicle_to_spot_map[vehicle] = spot
            self.spot_to_vehicle_map[spot] = vehicle
            self.available_spots[spot.get_size()].remove(spot)
            return spot
        return None

    def unpark_vehicle(self, vehicle: Vehicle) -> None:
        spot = self.vehicle_to_spot_map.pop(vehicle, None)
        if spot is not None:
            self.spot_to_vehicle_map.pop(spot, None)
            spot.vacate()
            self.available_spots[spot.get_size()].append(spot)

    def find_vehicle_spot(self, vehicle: Vehicle):
        return self.vehicle_to_spot_map.get(vehicle)

    def find_spot_vehicle(self, spot: ParkingSpot):
        return self.spot_to_vehicle_map.get(spot)


class Ticket:
    def __init__(
        self,
        ticket_id: str,
        vehicle: Vehicle,
        parking_spot: ParkingSpot,
        entry_time: datetime,
    ):
        self.ticket_id = ticket_id
        self.vehicle = vehicle
        self.parking_spot = parking_spot
        self.entry_time = entry_time
        self.exit_time = None

    def calculate_parking_duration(self) -> Decimal:
        end_time = self.exit_time or datetime.now()
        minutes = int((end_time - self.entry_time).total_seconds() // 60)
        return Decimal(minutes)

    def get_vehicle(self) -> Vehicle:
        return self.vehicle

    def get_entry_time(self) -> datetime:
        return self.entry_time

    def get_exit_time(self):
        return self.exit_time

    def set_exit_time(self, exit_time: datetime) -> None:
        self.exit_time = exit_time


class FareStrategy(ABC):
    @abstractmethod
    def calculate_fare(self, ticket: Ticket, input_fare: Decimal) -> Decimal:
        pass


class BaseFareStrategy(FareStrategy):
    SMALL_VEHICLE_RATE = Decimal("1.0")
    MEDIUM_VEHICLE_RATE = Decimal("2.0")
    LARGE_VEHICLE_RATE = Decimal("3.0")

    def calculate_fare(self, ticket: Ticket, input_fare: Decimal) -> Decimal:
        fare = input_fare
        vehicle_size = ticket.get_vehicle().get_size()

        if vehicle_size == VehicleSize.MEDIUM:
            rate = self.MEDIUM_VEHICLE_RATE
        elif vehicle_size == VehicleSize.LARGE:
            rate = self.LARGE_VEHICLE_RATE
        else:
            rate = self.SMALL_VEHICLE_RATE

        fare = fare + (rate * ticket.calculate_parking_duration())
        return fare


class PeakHoursFareStrategy(FareStrategy):
    PEAK_HOURS_MULTIPLIER = Decimal("1.5")

    def calculate_fare(self, ticket: Ticket, input_fare: Decimal) -> Decimal:
        fare = input_fare
        if self.is_peak_hours(ticket.get_entry_time()):
            fare = fare * self.PEAK_HOURS_MULTIPLIER
        return fare

    def is_peak_hours(self, time: datetime) -> bool:
        hour = time.hour
        return (7 <= hour <= 10) or (16 <= hour <= 19)


class FareCalculator:
    def __init__(self, fare_strategies: list[FareStrategy]):
        self.fare_strategies = fare_strategies

    def calculate_fare(self, ticket: Ticket) -> Decimal:
        fare = Decimal("0")
        for strategy in self.fare_strategies:
            fare = strategy.calculate_fare(ticket, fare)
        return fare


class ParkingLot:
    def __init__(self, parking_manager: ParkingManager, fare_calculator: FareCalculator):
        self.parking_manager = parking_manager
        self.fare_calculator = fare_calculator

    def generate_ticket_id(self) -> str:
        return str(uuid.uuid4())

    def enter_vehicle(self, vehicle: Vehicle):
        spot = self.parking_manager.park_vehicle(vehicle)

        if spot is not None:
            ticket = Ticket(
                ticket_id=self.generate_ticket_id(),
                vehicle=vehicle,
                parking_spot=spot,
                entry_time=datetime.now(),
            )
            return ticket

        return None

    def leave_vehicle(self, ticket: Ticket):
        if ticket is not None and ticket.get_exit_time() is None:
            ticket.set_exit_time(datetime.now())
            self.parking_manager.unpark_vehicle(ticket.get_vehicle())
            fare = self.fare_calculator.calculate_fare(ticket)
            return fare

        return None


if __name__ == "__main__":
    available_spots = {
        VehicleSize.SMALL: [CompactSpot(1), CompactSpot(2)],
        VehicleSize.MEDIUM: [RegularSpot(3), HandicappedSpot(4)],
        VehicleSize.LARGE: [OversizedSpot(5)],
    }

    parking_manager = ParkingManager(available_spots)

    fare_calculator = FareCalculator([
        BaseFareStrategy(),
        PeakHoursFareStrategy(),
    ])

    parking_lot = ParkingLot(parking_manager, fare_calculator)

    vehicle = Car("ABC123")
    ticket = parking_lot.enter_vehicle(vehicle)

    if ticket:
        print(f"Vehicle entered. Ticket ID: {ticket.ticket_id}")
        print(f"Spot number: {ticket.parking_spot.get_spot_number()}")

        # simulate parked time for demo
        ticket.entry_time = datetime.now().replace(hour=8, minute=0, second=0, microsecond=0)

        # fake exit after some minutes for demo
        from datetime import timedelta
        ticket.exit_time = ticket.entry_time + timedelta(minutes=120)

        fare = fare_calculator.calculate_fare(ticket)
        parking_manager.unpark_vehicle(vehicle)

        print(f"Parking duration in minutes: {ticket.calculate_parking_duration()}")
        print(f"Fare: {fare}")
    else:
        print("No parking spot available.")


```


**Advanced LLD Parking Lot**

This version adds:

multiple floors

entry and exit gates

display board

payment support

payment methods and status

ticket and spot states

duplicate vehicle protection

parking spot assignment strategy

extensible fare strategy

reserved and maintenance ready structure

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from datetime import datetime
from decimal import Decimal
from enum import Enum
import uuid


# =========================
# Enums
# =========================

class VehicleSize(str, Enum):
    SMALL = "SMALL"
    MEDIUM = "MEDIUM"
    LARGE = "LARGE"


class SpotStatus(str, Enum):
    AVAILABLE = "AVAILABLE"
    OCCUPIED = "OCCUPIED"
    RESERVED = "RESERVED"
    MAINTENANCE = "MAINTENANCE"


class TicketStatus(str, Enum):
    ACTIVE = "ACTIVE"
    PAID = "PAID"
    LOST = "LOST"
    CLOSED = "CLOSED"


class PaymentStatus(str, Enum):
    PENDING = "PENDING"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"


class PaymentMethod(str, Enum):
    CASH = "CASH"
    CARD = "CARD"
    UPI = "UPI"


# =========================
# Vehicle
# =========================

class Vehicle(ABC):
    def __init__(self, license_plate: str, size: VehicleSize):
        self.license_plate = license_plate
        self.size = size

    def get_size(self) -> VehicleSize:
        return self.size

    def __hash__(self) -> int:
        return hash(self.license_plate)

    def __eq__(self, other: object) -> bool:
        return isinstance(other, Vehicle) and self.license_plate == other.license_plate

    def __repr__(self) -> str:
        return f"{self.__class__.__name__}({self.license_plate})"


class Bike(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleSize.SMALL)


class Car(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleSize.MEDIUM)


class Truck(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleSize.LARGE)


# =========================
# Parking Spots
# =========================

class ParkingSpot(ABC):
    def __init__(self, spot_number: int, floor_id: str):
        self.spot_number = spot_number
        self.floor_id = floor_id
        self.vehicle: Vehicle | None = None
        self.status = SpotStatus.AVAILABLE

    def is_available(self) -> bool:
        return self.status == SpotStatus.AVAILABLE and self.vehicle is None

    def occupy(self, vehicle: Vehicle) -> None:
        if not self.is_available():
            raise ValueError(f"Spot {self.spot_number} is not available.")
        self.vehicle = vehicle
        self.status = SpotStatus.OCCUPIED

    def vacate(self) -> None:
        self.vehicle = None
        self.status = SpotStatus.AVAILABLE

    def mark_maintenance(self) -> None:
        if self.status == SpotStatus.OCCUPIED:
            raise ValueError("Cannot mark occupied spot as maintenance.")
        self.vehicle = None
        self.status = SpotStatus.MAINTENANCE

    def reserve(self) -> None:
        if self.status != SpotStatus.AVAILABLE:
            raise ValueError("Only available spots can be reserved.")
        self.status = SpotStatus.RESERVED

    def release_reservation(self) -> None:
        if self.status == SpotStatus.RESERVED:
            self.status = SpotStatus.AVAILABLE

    def get_spot_number(self) -> int:
        return self.spot_number

    @abstractmethod
    def get_size(self) -> VehicleSize:
        pass

    def __repr__(self) -> str:
        return f"{self.__class__.__name__}(floor={self.floor_id}, spot={self.spot_number}, status={self.status})"


class CompactSpot(ParkingSpot):
    def get_size(self) -> VehicleSize:
        return VehicleSize.SMALL


class RegularSpot(ParkingSpot):
    def get_size(self) -> VehicleSize:
        return VehicleSize.MEDIUM


class OversizedSpot(ParkingSpot):
    def get_size(self) -> VehicleSize:
        return VehicleSize.LARGE


class HandicappedSpot(ParkingSpot):
    def get_size(self) -> VehicleSize:
        return VehicleSize.MEDIUM


# =========================
# Display Board
# =========================

class DisplayBoard:
    def show_floor_availability(self, floor: "ParkingFloor") -> dict[VehicleSize, int]:
        return floor.get_available_spot_count_by_size()

    def show_lot_availability(self, parking_lot: "ParkingLot") -> dict[VehicleSize, int]:
        result = {
            VehicleSize.SMALL: 0,
            VehicleSize.MEDIUM: 0,
            VehicleSize.LARGE: 0,
        }
        for floor in parking_lot.floors:
            floor_counts = floor.get_available_spot_count_by_size()
            for size, count in floor_counts.items():
                result[size] += count
        return result


# =========================
# Parking Floor
# =========================

class ParkingFloor:
    def __init__(self, floor_id: str, spots: list[ParkingSpot]):
        self.floor_id = floor_id
        self.spots = spots

    def get_available_spots_by_size(self, size: VehicleSize) -> list[ParkingSpot]:
        return [spot for spot in self.spots if spot.get_size() == size and spot.is_available()]

    def get_available_spot_count_by_size(self) -> dict[VehicleSize, int]:
        counts = {
            VehicleSize.SMALL: 0,
            VehicleSize.MEDIUM: 0,
            VehicleSize.LARGE: 0,
        }
        for spot in self.spots:
            if spot.is_available():
                counts[spot.get_size()] += 1
        return counts

    def add_spot(self, spot: ParkingSpot) -> None:
        self.spots.append(spot)

    def __repr__(self) -> str:
        return f"ParkingFloor({self.floor_id})"


# =========================
# Spot Assignment Strategy
# =========================

class SpotAssignmentStrategy(ABC):
    @abstractmethod
    def find_spot(self, vehicle: Vehicle, floors: list[ParkingFloor]) -> ParkingSpot | None:
        pass


class ExactMatchSpotAssignmentStrategy(SpotAssignmentStrategy):
    def find_spot(self, vehicle: Vehicle, floors: list[ParkingFloor]) -> ParkingSpot | None:
        vehicle_size = vehicle.get_size()
        for floor in floors:
            spots = floor.get_available_spots_by_size(vehicle_size)
            if spots:
                return spots[0]
        return None


# =========================
# Ticket
# =========================

class Ticket:
    def __init__(
        self,
        ticket_id: str,
        vehicle: Vehicle,
        parking_spot: ParkingSpot,
        entry_time: datetime,
    ):
        self.ticket_id = ticket_id
        self.vehicle = vehicle
        self.parking_spot = parking_spot
        self.entry_time = entry_time
        self.exit_time: datetime | None = None
        self.status = TicketStatus.ACTIVE

    def calculate_parking_duration_minutes(self) -> Decimal:
        end_time = self.exit_time or datetime.now()
        minutes = int((end_time - self.entry_time).total_seconds() // 60)
        return Decimal(minutes)

    def close(self) -> None:
        self.status = TicketStatus.CLOSED

    def mark_paid(self) -> None:
        self.status = TicketStatus.PAID

    def __repr__(self) -> str:
        return f"Ticket(id={self.ticket_id}, vehicle={self.vehicle}, status={self.status})"


# =========================
# Fare Strategies
# =========================

class FareStrategy(ABC):
    @abstractmethod
    def calculate_fare(self, ticket: Ticket, input_fare: Decimal) -> Decimal:
        pass


class BaseFareStrategy(FareStrategy):
    SMALL_RATE = Decimal("1.0")
    MEDIUM_RATE = Decimal("2.0")
    LARGE_RATE = Decimal("3.0")

    def calculate_fare(self, ticket: Ticket, input_fare: Decimal) -> Decimal:
        size = ticket.vehicle.get_size()

        if size == VehicleSize.SMALL:
            rate = self.SMALL_RATE
        elif size == VehicleSize.MEDIUM:
            rate = self.MEDIUM_RATE
        else:
            rate = self.LARGE_RATE

        return input_fare + (rate * ticket.calculate_parking_duration_minutes())


class PeakHoursFareStrategy(FareStrategy):
    PEAK_HOURS_MULTIPLIER = Decimal("1.5")

    def calculate_fare(self, ticket: Ticket, input_fare: Decimal) -> Decimal:
        if self.is_peak_hours(ticket.entry_time):
            return input_fare * self.PEAK_HOURS_MULTIPLIER
        return input_fare

    def is_peak_hours(self, time: datetime) -> bool:
        hour = time.hour
        return (7 <= hour <= 10) or (16 <= hour <= 19)


class DailyCapFareStrategy(FareStrategy):
    DAILY_CAP = Decimal("500.0")

    def calculate_fare(self, ticket: Ticket, input_fare: Decimal) -> Decimal:
        return min(input_fare, self.DAILY_CAP)


class FareCalculator:
    def __init__(self, fare_strategies: list[FareStrategy]):
        self.fare_strategies = fare_strategies

    def calculate_fare(self, ticket: Ticket) -> Decimal:
        fare = Decimal("0")
        for strategy in self.fare_strategies:
            fare = strategy.calculate_fare(ticket, fare)
        return fare


# =========================
# Payment
# =========================

class Payment:
    def __init__(self, payment_id: str, amount: Decimal, method: PaymentMethod):
        self.payment_id = payment_id
        self.amount = amount
        self.method = method
        self.status = PaymentStatus.PENDING
        self.created_at = datetime.now()

    def mark_completed(self) -> None:
        self.status = PaymentStatus.COMPLETED

    def mark_failed(self) -> None:
        self.status = PaymentStatus.FAILED

    def __repr__(self) -> str:
        return f"Payment(id={self.payment_id}, amount={self.amount}, method={self.method}, status={self.status})"


class PaymentProcessor:
    def process(self, amount: Decimal, method: PaymentMethod) -> Payment:
        payment = Payment(
            payment_id=str(uuid.uuid4()),
            amount=amount,
            method=method,
        )
        payment.mark_completed()
        return payment


# =========================
# Parking Manager
# =========================

class ParkingManager:
    def __init__(self, floors: list[ParkingFloor], assignment_strategy: SpotAssignmentStrategy):
        self.floors = floors
        self.assignment_strategy = assignment_strategy
        self.vehicle_to_spot_map: dict[Vehicle, ParkingSpot] = {}
        self.spot_to_vehicle_map: dict[ParkingSpot, Vehicle] = {}

    def find_spot_for_vehicle(self, vehicle: Vehicle) -> ParkingSpot | None:
        return self.assignment_strategy.find_spot(vehicle, self.floors)

    def park_vehicle(self, vehicle: Vehicle) -> ParkingSpot | None:
        if vehicle in self.vehicle_to_spot_map:
            raise ValueError(f"Vehicle {vehicle.license_plate} is already parked.")

        spot = self.find_spot_for_vehicle(vehicle)
        if spot is None:
            return None

        spot.occupy(vehicle)
        self.vehicle_to_spot_map[vehicle] = spot
        self.spot_to_vehicle_map[spot] = vehicle
        return spot

    def unpark_vehicle(self, vehicle: Vehicle) -> ParkingSpot | None:
        spot = self.vehicle_to_spot_map.pop(vehicle, None)
        if spot is None:
            return None

        self.spot_to_vehicle_map.pop(spot, None)
        spot.vacate()
        return spot

    def find_vehicle_spot(self, vehicle: Vehicle) -> ParkingSpot | None:
        return self.vehicle_to_spot_map.get(vehicle)

    def find_spot_vehicle(self, spot: ParkingSpot) -> Vehicle | None:
        return self.spot_to_vehicle_map.get(spot)


# =========================
# Gates
# =========================

class EntryGate:
    def __init__(self, gate_id: str):
        self.gate_id = gate_id

    def issue_ticket(self, parking_lot: "ParkingLot", vehicle: Vehicle) -> Ticket | None:
        return parking_lot.enter_vehicle(vehicle)


class ExitGate:
    def __init__(self, gate_id: str):
        self.gate_id = gate_id

    def process_exit(
        self,
        parking_lot: "ParkingLot",
        ticket: Ticket,
        method: PaymentMethod,
    ) -> tuple[Decimal, Payment] | None:
        return parking_lot.leave_vehicle(ticket, method)


# =========================
# Parking Lot Facade
# =========================

class ParkingLot:
    def __init__(
        self,
        lot_id: str,
        floors: list[ParkingFloor],
        parking_manager: ParkingManager,
        fare_calculator: FareCalculator,
        payment_processor: PaymentProcessor,
        display_board: DisplayBoard,
    ):
        self.lot_id = lot_id
        self.floors = floors
        self.parking_manager = parking_manager
        self.fare_calculator = fare_calculator
        self.payment_processor = payment_processor
        self.display_board = display_board

    def generate_ticket_id(self) -> str:
        return str(uuid.uuid4())

    def enter_vehicle(self, vehicle: Vehicle) -> Ticket | None:
        spot = self.parking_manager.park_vehicle(vehicle)
        if spot is None:
            return None

        return Ticket(
            ticket_id=self.generate_ticket_id(),
            vehicle=vehicle,
            parking_spot=spot,
            entry_time=datetime.now(),
        )

    def leave_vehicle(
        self,
        ticket: Ticket,
        method: PaymentMethod,
    ) -> tuple[Decimal, Payment] | None:
        if ticket is None or ticket.status != TicketStatus.ACTIVE:
            return None

        ticket.exit_time = datetime.now()

        fare = self.fare_calculator.calculate_fare(ticket)
        payment = self.payment_processor.process(fare, method)

        if payment.status != PaymentStatus.COMPLETED:
            return None

        self.parking_manager.unpark_vehicle(ticket.vehicle)
        ticket.mark_paid()
        ticket.close()

        return fare, payment

    def get_availability(self) -> dict[VehicleSize, int]:
        return self.display_board.show_lot_availability(self)


# =========================
# Demo
# =========================

if __name__ == "__main__":
    floor_1 = ParkingFloor(
        floor_id="F1",
        spots=[
            CompactSpot(1, "F1"),
            CompactSpot(2, "F1"),
            RegularSpot(3, "F1"),
            HandicappedSpot(4, "F1"),
            OversizedSpot(5, "F1"),
        ],
    )

    floor_2 = ParkingFloor(
        floor_id="F2",
        spots=[
            CompactSpot(1, "F2"),
            RegularSpot(2, "F2"),
            RegularSpot(3, "F2"),
            OversizedSpot(4, "F2"),
        ],
    )

    floors = [floor_1, floor_2]

    strategy = ExactMatchSpotAssignmentStrategy()
    parking_manager = ParkingManager(floors, strategy)

    fare_calculator = FareCalculator([
        BaseFareStrategy(),
        PeakHoursFareStrategy(),
        DailyCapFareStrategy(),
    ])

    payment_processor = PaymentProcessor()
    display_board = DisplayBoard()

    parking_lot = ParkingLot(
        lot_id="LOT-101",
        floors=floors,
        parking_manager=parking_manager,
        fare_calculator=fare_calculator,
        payment_processor=payment_processor,
        display_board=display_board,
    )

    entry_gate = EntryGate("ENTRY-1")
    exit_gate = ExitGate("EXIT-1")

    car = Car("ABC123")
    truck = Truck("TRK999")

    print("Initial availability:", parking_lot.get_availability())

    ticket1 = entry_gate.issue_ticket(parking_lot, car)
    if ticket1:
        print("Car entered:", ticket1.ticket_id, ticket1.parking_spot)

    ticket2 = entry_gate.issue_ticket(parking_lot, truck)
    if ticket2:
        print("Truck entered:", ticket2.ticket_id, ticket2.parking_spot)

    print("Availability after entry:", parking_lot.get_availability())

    if ticket1:
        ticket1.entry_time = ticket1.entry_time.replace(hour=8, minute=0, second=0, microsecond=0)
        ticket1.exit_time = ticket1.entry_time.replace(hour=10, minute=0, second=0, microsecond=0)

        fare = fare_calculator.calculate_fare(ticket1)
        payment = payment_processor.process(fare, PaymentMethod.CARD)
        parking_manager.unpark_vehicle(ticket1.vehicle)
        ticket1.mark_paid()
        ticket1.close()

        print("Car fare:", fare)
        print("Car payment:", payment)

    print("Final availability:", parking_lot.get_availability())
```



**Updated advanced version with shared capacity support**

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from datetime import datetime
from decimal import Decimal
from enum import Enum
import uuid


class VehicleSize(str, Enum):
    SMALL = "SMALL"
    MEDIUM = "MEDIUM"
    LARGE = "LARGE"


class SpotStatus(str, Enum):
    AVAILABLE = "AVAILABLE"
    OCCUPIED = "OCCUPIED"
    RESERVED = "RESERVED"
    MAINTENANCE = "MAINTENANCE"


class TicketStatus(str, Enum):
    ACTIVE = "ACTIVE"
    PAID = "PAID"
    LOST = "LOST"
    CLOSED = "CLOSED"


class PaymentStatus(str, Enum):
    PENDING = "PENDING"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"


class PaymentMethod(str, Enum):
    CASH = "CASH"
    CARD = "CARD"
    UPI = "UPI"


class Vehicle(ABC):
    def __init__(self, license_plate: str, size: VehicleSize):
        self.license_plate = license_plate
        self.size = size

    def get_size(self) -> VehicleSize:
        return self.size

    def __hash__(self) -> int:
        return hash(self.license_plate)

    def __eq__(self, other: object) -> bool:
        return isinstance(other, Vehicle) and self.license_plate == other.license_plate

    def __repr__(self) -> str:
        return f"{self.__class__.__name__}({self.license_plate})"


class Bike(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleSize.SMALL)


class Car(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleSize.MEDIUM)


class Truck(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleSize.LARGE)


class ParkingSpot(ABC):
    def __init__(self, spot_number: int, floor_id: str):
        self.spot_number = spot_number
        self.floor_id = floor_id
        self.parked_vehicles: list[Vehicle] = []
        self.status = SpotStatus.AVAILABLE

    @abstractmethod
    def get_size(self) -> VehicleSize:
        pass

    @abstractmethod
    def total_capacity_units(self) -> int:
        pass

    @abstractmethod
    def capacity_units_for_vehicle(self, vehicle: Vehicle) -> int | None:
        pass

    def used_capacity_units(self) -> int:
        total = 0
        for vehicle in self.parked_vehicles:
            units = self.capacity_units_for_vehicle(vehicle)
            if units is None:
                raise ValueError("Invalid parked vehicle for this spot.")
            total += units
        return total

    def remaining_capacity_units(self) -> int:
        return self.total_capacity_units() - self.used_capacity_units()

    def can_fit_vehicle(self, vehicle: Vehicle) -> bool:
        if self.status in {SpotStatus.MAINTENANCE, SpotStatus.RESERVED}:
            return False

        needed = self.capacity_units_for_vehicle(vehicle)
        if needed is None:
            return False

        return self.remaining_capacity_units() >= needed

    def is_available(self) -> bool:
        return self.status == SpotStatus.AVAILABLE and self.remaining_capacity_units() > 0

    def occupy(self, vehicle: Vehicle) -> None:
        if not self.can_fit_vehicle(vehicle):
            raise ValueError(f"Vehicle {vehicle.license_plate} cannot fit in spot {self.spot_number}.")
        self.parked_vehicles.append(vehicle)
        self.status = SpotStatus.OCCUPIED if self.remaining_capacity_units() == 0 else SpotStatus.AVAILABLE

    def vacate(self, vehicle: Vehicle) -> None:
        if vehicle not in self.parked_vehicles:
            raise ValueError(f"Vehicle {vehicle.license_plate} is not parked in spot {self.spot_number}.")
        self.parked_vehicles.remove(vehicle)
        self.status = SpotStatus.AVAILABLE

    def mark_maintenance(self) -> None:
        if self.parked_vehicles:
            raise ValueError("Cannot mark non-empty spot as maintenance.")
        self.status = SpotStatus.MAINTENANCE

    def reserve(self) -> None:
        if self.parked_vehicles:
            raise ValueError("Cannot reserve occupied spot.")
        self.status = SpotStatus.RESERVED

    def release_reservation(self) -> None:
        if self.status == SpotStatus.RESERVED:
            self.status = SpotStatus.AVAILABLE

    def get_spot_number(self) -> int:
        return self.spot_number

    def __repr__(self) -> str:
        return (
            f"{self.__class__.__name__}("
            f"floor={self.floor_id}, spot={self.spot_number}, "
            f"used={self.used_capacity_units()}/{self.total_capacity_units()}, "
            f"vehicles={self.parked_vehicles})"
        )


class CompactSpot(ParkingSpot):
    def get_size(self) -> VehicleSize:
        return VehicleSize.SMALL

    def total_capacity_units(self) -> int:
        return 1

    def capacity_units_for_vehicle(self, vehicle: Vehicle) -> int | None:
        if vehicle.get_size() == VehicleSize.SMALL:
            return 1
        return None


class RegularSpot(ParkingSpot):
    def get_size(self) -> VehicleSize:
        return VehicleSize.MEDIUM

    def total_capacity_units(self) -> int:
        return 2

    def capacity_units_for_vehicle(self, vehicle: Vehicle) -> int | None:
        if vehicle.get_size() == VehicleSize.SMALL:
            return 1
        if vehicle.get_size() == VehicleSize.MEDIUM:
            return 2
        return None


class OversizedSpot(ParkingSpot):
    def get_size(self) -> VehicleSize:
        return VehicleSize.LARGE

    def total_capacity_units(self) -> int:
        return 4

    def capacity_units_for_vehicle(self, vehicle: Vehicle) -> int | None:
        if vehicle.get_size() == VehicleSize.SMALL:
            return 1
        if vehicle.get_size() == VehicleSize.MEDIUM:
            return 2
        if vehicle.get_size() == VehicleSize.LARGE:
            return 4
        return None


class HandicappedSpot(ParkingSpot):
    def get_size(self) -> VehicleSize:
        return VehicleSize.MEDIUM

    def total_capacity_units(self) -> int:
        return 2

    def capacity_units_for_vehicle(self, vehicle: Vehicle) -> int | None:
        if vehicle.get_size() == VehicleSize.SMALL:
            return 1
        if vehicle.get_size() == VehicleSize.MEDIUM:
            return 2
        return None


class DisplayBoard:
    def show_floor_availability(self, floor: "ParkingFloor") -> dict[VehicleSize, int]:
        return floor.get_available_vehicle_slots_by_size()

    def show_lot_availability(self, parking_lot: "ParkingLot") -> dict[VehicleSize, int]:
        result = {
            VehicleSize.SMALL: 0,
            VehicleSize.MEDIUM: 0,
            VehicleSize.LARGE: 0,
        }
        for floor in parking_lot.floors:
            floor_counts = floor.get_available_vehicle_slots_by_size()
            for size, count in floor_counts.items():
                result[size] += count
        return result


class ParkingFloor:
    def __init__(self, floor_id: str, spots: list[ParkingSpot]):
        self.floor_id = floor_id
        self.spots = spots

    def get_candidate_spots(self, vehicle: Vehicle) -> list[ParkingSpot]:
        return [spot for spot in self.spots if spot.can_fit_vehicle(vehicle)]

    def get_available_vehicle_slots_by_size(self) -> dict[VehicleSize, int]:
        counts = {
            VehicleSize.SMALL: 0,
            VehicleSize.MEDIUM: 0,
            VehicleSize.LARGE: 0,
        }

        for spot in self.spots:
            if spot.status in {SpotStatus.MAINTENANCE, SpotStatus.RESERVED}:
                continue

            if spot.get_size() == VehicleSize.SMALL:
                counts[VehicleSize.SMALL] += spot.remaining_capacity_units()

            elif spot.get_size() == VehicleSize.MEDIUM:
                counts[VehicleSize.SMALL] += spot.remaining_capacity_units()
                counts[VehicleSize.MEDIUM] += spot.remaining_capacity_units() // 2

            elif spot.get_size() == VehicleSize.LARGE:
                counts[VehicleSize.SMALL] += spot.remaining_capacity_units()
                counts[VehicleSize.MEDIUM] += spot.remaining_capacity_units() // 2
                counts[VehicleSize.LARGE] += spot.remaining_capacity_units() // 4

        return counts


class SpotAssignmentStrategy(ABC):
    @abstractmethod
    def find_spot(self, vehicle: Vehicle, floors: list[ParkingFloor]) -> ParkingSpot | None:
        pass


class BestFitSpotAssignmentStrategy(SpotAssignmentStrategy):
    def find_spot(self, vehicle: Vehicle, floors: list[ParkingFloor]) -> ParkingSpot | None:
        for preferred_size in self._preferred_spot_order(vehicle):
            for floor in floors:
                for spot in floor.spots:
                    if spot.get_size() == preferred_size and spot.can_fit_vehicle(vehicle):
                        return spot
        return None

    def _preferred_spot_order(self, vehicle: Vehicle) -> list[VehicleSize]:
        size = vehicle.get_size()
        if size == VehicleSize.SMALL:
            return [VehicleSize.SMALL, VehicleSize.MEDIUM, VehicleSize.LARGE]
        if size == VehicleSize.MEDIUM:
            return [VehicleSize.MEDIUM, VehicleSize.LARGE]
        return [VehicleSize.LARGE]


class Ticket:
    def __init__(
        self,
        ticket_id: str,
        vehicle: Vehicle,
        parking_spot: ParkingSpot,
        entry_time: datetime,
    ):
        self.ticket_id = ticket_id
        self.vehicle = vehicle
        self.parking_spot = parking_spot
        self.entry_time = entry_time
        self.exit_time: datetime | None = None
        self.status = TicketStatus.ACTIVE

    def calculate_parking_duration_minutes(self) -> Decimal:
        end_time = self.exit_time or datetime.now()
        minutes = int((end_time - self.entry_time).total_seconds() // 60)
        return Decimal(minutes)

    def close(self) -> None:
        self.status = TicketStatus.CLOSED

    def mark_paid(self) -> None:
        self.status = TicketStatus.PAID


class FareStrategy(ABC):
    @abstractmethod
    def calculate_fare(self, ticket: Ticket, input_fare: Decimal) -> Decimal:
        pass


class BaseFareStrategy(FareStrategy):
    SMALL_RATE = Decimal("1.0")
    MEDIUM_RATE = Decimal("2.0")
    LARGE_RATE = Decimal("3.0")

    def calculate_fare(self, ticket: Ticket, input_fare: Decimal) -> Decimal:
        size = ticket.vehicle.get_size()
        if size == VehicleSize.SMALL:
            rate = self.SMALL_RATE
        elif size == VehicleSize.MEDIUM:
            rate = self.MEDIUM_RATE
        else:
            rate = self.LARGE_RATE
        return input_fare + (rate * ticket.calculate_parking_duration_minutes())


class PeakHoursFareStrategy(FareStrategy):
    PEAK_HOURS_MULTIPLIER = Decimal("1.5")

    def calculate_fare(self, ticket: Ticket, input_fare: Decimal) -> Decimal:
        if self.is_peak_hours(ticket.entry_time):
            return input_fare * self.PEAK_HOURS_MULTIPLIER
        return input_fare

    def is_peak_hours(self, time: datetime) -> bool:
        hour = time.hour
        return (7 <= hour <= 10) or (16 <= hour <= 19)


class FareCalculator:
    def __init__(self, fare_strategies: list[FareStrategy]):
        self.fare_strategies = fare_strategies

    def calculate_fare(self, ticket: Ticket) -> Decimal:
        fare = Decimal("0")
        for strategy in self.fare_strategies:
            fare = strategy.calculate_fare(ticket, fare)
        return fare


class Payment:
    def __init__(self, payment_id: str, amount: Decimal, method: PaymentMethod):
        self.payment_id = payment_id
        self.amount = amount
        self.method = method
        self.status = PaymentStatus.PENDING
        self.created_at = datetime.now()

    def mark_completed(self) -> None:
        self.status = PaymentStatus.COMPLETED

    def mark_failed(self) -> None:
        self.status = PaymentStatus.FAILED


class PaymentProcessor:
    def process(self, amount: Decimal, method: PaymentMethod) -> Payment:
        payment = Payment(str(uuid.uuid4()), amount, method)
        payment.mark_completed()
        return payment


class ParkingManager:
    def __init__(self, floors: list[ParkingFloor], assignment_strategy: SpotAssignmentStrategy):
        self.floors = floors
        self.assignment_strategy = assignment_strategy
        self.vehicle_to_spot_map: dict[Vehicle, ParkingSpot] = {}
        self.spot_to_vehicles_map: dict[ParkingSpot, list[Vehicle]] = {}

    def find_spot_for_vehicle(self, vehicle: Vehicle) -> ParkingSpot | None:
        return self.assignment_strategy.find_spot(vehicle, self.floors)

    def park_vehicle(self, vehicle: Vehicle) -> ParkingSpot | None:
        if vehicle in self.vehicle_to_spot_map:
            raise ValueError(f"Vehicle {vehicle.license_plate} is already parked.")

        spot = self.find_spot_for_vehicle(vehicle)
        if spot is None:
            return None

        spot.occupy(vehicle)
        self.vehicle_to_spot_map[vehicle] = spot

        if spot not in self.spot_to_vehicles_map:
            self.spot_to_vehicles_map[spot] = []
        self.spot_to_vehicles_map[spot].append(vehicle)

        return spot

    def unpark_vehicle(self, vehicle: Vehicle) -> ParkingSpot | None:
        spot = self.vehicle_to_spot_map.pop(vehicle, None)
        if spot is None:
            return None

        spot.vacate(vehicle)

        vehicles = self.spot_to_vehicles_map.get(spot, [])
        if vehicle in vehicles:
            vehicles.remove(vehicle)
        if not vehicles:
            self.spot_to_vehicles_map.pop(spot, None)

        return spot

    def find_vehicle_spot(self, vehicle: Vehicle) -> ParkingSpot | None:
        return self.vehicle_to_spot_map.get(vehicle)

    def find_spot_vehicles(self, spot: ParkingSpot) -> list[Vehicle]:
        return self.spot_to_vehicles_map.get(spot, [])


class EntryGate:
    def __init__(self, gate_id: str):
        self.gate_id = gate_id

    def issue_ticket(self, parking_lot: "ParkingLot", vehicle: Vehicle) -> Ticket | None:
        return parking_lot.enter_vehicle(vehicle)


class ExitGate:
    def __init__(self, gate_id: str):
        self.gate_id = gate_id

    def process_exit(
        self,
        parking_lot: "ParkingLot",
        ticket: Ticket,
        method: PaymentMethod,
    ) -> tuple[Decimal, Payment] | None:
        return parking_lot.leave_vehicle(ticket, method)


class ParkingLot:
    def __init__(
        self,
        lot_id: str,
        floors: list[ParkingFloor],
        parking_manager: ParkingManager,
        fare_calculator: FareCalculator,
        payment_processor: PaymentProcessor,
        display_board: DisplayBoard,
    ):
        self.lot_id = lot_id
        self.floors = floors
        self.parking_manager = parking_manager
        self.fare_calculator = fare_calculator
        self.payment_processor = payment_processor
        self.display_board = display_board

    def generate_ticket_id(self) -> str:
        return str(uuid.uuid4())

    def enter_vehicle(self, vehicle: Vehicle) -> Ticket | None:
        spot = self.parking_manager.park_vehicle(vehicle)
        if spot is None:
            return None

        return Ticket(
            ticket_id=self.generate_ticket_id(),
            vehicle=vehicle,
            parking_spot=spot,
            entry_time=datetime.now(),
        )

    def leave_vehicle(
        self,
        ticket: Ticket,
        method: PaymentMethod,
    ) -> tuple[Decimal, Payment] | None:
        if ticket is None or ticket.status != TicketStatus.ACTIVE:
            return None

        ticket.exit_time = datetime.now()
        fare = self.fare_calculator.calculate_fare(ticket)
        payment = self.payment_processor.process(fare, method)

        if payment.status != PaymentStatus.COMPLETED:
            return None

        self.parking_manager.unpark_vehicle(ticket.vehicle)
        ticket.mark_paid()
        ticket.close()
        return fare, payment

    def get_availability(self) -> dict[VehicleSize, int]:
        return self.display_board.show_lot_availability(self)


if __name__ == "__main__":
    floor_1 = ParkingFloor(
        floor_id="F1",
        spots=[
            CompactSpot(1, "F1"),
            CompactSpot(2, "F1"),
            RegularSpot(3, "F1"),
            OversizedSpot(4, "F1"),
        ],
    )

    floors = [floor_1]

    parking_manager = ParkingManager(floors, BestFitSpotAssignmentStrategy())
    fare_calculator = FareCalculator([BaseFareStrategy(), PeakHoursFareStrategy()])
    payment_processor = PaymentProcessor()
    display_board = DisplayBoard()

    parking_lot = ParkingLot(
        lot_id="LOT-1",
        floors=floors,
        parking_manager=parking_manager,
        fare_calculator=fare_calculator,
        payment_processor=payment_processor,
        display_board=display_board,
    )

    entry_gate = EntryGate("E1")

    b1 = Bike("B1")
    b2 = Bike("B2")
    b3 = Bike("B3")
    b4 = Bike("B4")
    c1 = Car("C1")
    c2 = Car("C2")

    t1 = entry_gate.issue_ticket(parking_lot, b1)
    t2 = entry_gate.issue_ticket(parking_lot, b2)
    t3 = entry_gate.issue_ticket(parking_lot, b3)
    t4 = entry_gate.issue_ticket(parking_lot, b4)
    t5 = entry_gate.issue_ticket(parking_lot, c1)
    t6 = entry_gate.issue_ticket(parking_lot, c2)

    print("Availability:", parking_lot.get_availability())
    for floor in floors:
        for spot in floor.spots:
            print(spot)

```

## Design Patterns Used in the Parking Lot LLD

### 1. Strategy Pattern

**Where used**
- `FareStrategy`
- `BaseFareStrategy`
- `PeakHoursFareStrategy`
- `FareCalculator`
- `SpotAssignmentStrategy`
- `BestFitSpotAssignmentStrategy`

**Why we used it**
The system has behaviors that can change independently.

For fare calculation, we may have:
- base fare
- peak hour pricing
- weekend discount
- daily cap

For spot assignment, we may have:
- exact match allocation
- best fit allocation
- nearest gate allocation
- handicapped priority allocation

Instead of putting all this logic into one large class with many `if else` statements, we separate each rule into its own strategy class.

**Simple reason**  
When behavior can vary, Strategy helps us swap logic easily without changing the core system.

---

### 2. Facade Pattern

**Where used**
- `ParkingLot`

**Why we used it**
The parking lot system has many internal components:
- `ParkingManager`
- `FareCalculator`
- `PaymentProcessor`
- `DisplayBoard`
- `Ticket`

A client should not need to directly interact with all of them.  
So `ParkingLot` acts as a simple entry point and provides methods like:
- `enter_vehicle()`
- `leave_vehicle()`
- `get_availability()`

It hides the internal complexity and coordinates the flow.

**Simple reason**  
Facade makes a complex system easier to use.

---

### 3. Abstraction and Polymorphism

**Where used**
- `Vehicle`
- `ParkingSpot`
- `FareStrategy`
- `SpotAssignmentStrategy`

**Why we used it**
The code depends on abstractions instead of concrete classes.

For example:
- `ParkingManager` works with `ParkingSpot`, not only `CompactSpot`
- `FareCalculator` works with `FareStrategy`, not only `BaseFareStrategy`

This allows us to add new classes later without changing existing logic.

Examples:
- `HandicappedSpot`
- `ElectricSpot`
- `WeekendDiscountStrategy`
- `NearestGateStrategy`

**Simple reason**  
Abstraction makes the system extensible and easier to maintain.

---

### 4. Composition Over Inheritance

**Where used**
- `ParkingLot` contains `ParkingManager`, `FareCalculator`, `PaymentProcessor`, and `DisplayBoard`
- `FareCalculator` contains a list of `FareStrategy`
- `ParkingManager` contains a `SpotAssignmentStrategy`

**Why we used it**
Instead of building one large inheritance hierarchy, we build the system by combining smaller objects.

For example:
- `FareCalculator` does not inherit pricing logic
- it contains pricing strategies and applies them one by one

This makes the design more flexible and easier to test.

**Simple reason**  
Composition allows behavior to be assembled dynamically.

---

### 5. State Modeling

**Where used**
- `SpotStatus`
- `TicketStatus`
- `PaymentStatus`

**Why we used it**
Parking spots, tickets, and payments all move through different states.

Examples:
- Spot: `AVAILABLE`, `OCCUPIED`, `RESERVED`, `MAINTENANCE`
- Ticket: `ACTIVE`, `PAID`, `CLOSED`
- Payment: `PENDING`, `COMPLETED`, `FAILED`

This makes the system realistic and prevents invalid operations.

**Important note**  
This is state representation using enums. It is not the full GoF State pattern with separate state classes.

**Simple reason**  
State modeling keeps the lifecycle of objects clear and safe.

---

### 6. Open Closed Principle Through Pattern Usage

**Where visible**
- Adding new fare strategies
- Adding new spot types
- Adding new vehicle types
- Adding new assignment strategies

**Why we used it**
The system should be open for extension but closed for modification.

Examples:
- add `WeekendDiscountStrategy`
- add `ElectricVehicleSpot`
- add `VIPSpot`
- add `NearestGateSpotAssignmentStrategy`

This is possible because the system is built around abstractions and strategies.

**Simple reason**  
We can extend the system without rewriting existing working code.

---

### 7. Bidirectional Mapping as a Performance Design Choice

**Where used**
- `vehicle_to_spot_map`
- `spot_to_vehicles_map`

**Why we used it**
We want fast lookup in both directions:
- given a vehicle, find its spot
- given a spot, find which vehicles are parked there

This avoids scanning all parked vehicles or all spots.

**Simple reason**  
It improves lookup performance and makes the system more scalable.

---

### 8. Capacity Based Spot Modeling

**Where used**
- `total_capacity_units()`
- `capacity_units_for_vehicle()`
- `remaining_capacity_units()`
- `parked_vehicles`

**Why we used it**
The advanced requirement says:
- if compact spots are full, a medium spot can hold 2 small vehicles
- an oversized spot can hold 4 small vehicles
- an oversized spot can also hold 2 medium vehicles

So one parking spot can no longer be modeled as one vehicle only.  
We changed the spot design to be capacity based.

This keeps the logic inside the spot classes and avoids spreading parking rules everywhere.

**Simple reason**  
Capacity based modeling supports flexible parking allocation cleanly.

---

### 9. Optional Pattern That Can Be Added Later: Observer Pattern

**Where it can be added**
- `DisplayBoard`

**Why it can be useful**
Right now, the display board is updated when queried.

If we want real time updates, we can use Observer pattern:
- parking spots or parking manager become the subject
- display board becomes an observer
- whenever a spot changes state, the display board gets notified automatically

**Simple reason**  
Observer would help for real time availability updates.

---

### 10. Optional Pattern That Can Be Added Later: Factory Pattern

**Where it can be added**
- vehicle creation
- parking spot creation
- ticket creation

**Why it can be useful**
Right now objects are created directly using constructors.

If creation logic becomes more complex, a factory can centralize object creation.

Examples:
- `VehicleFactory`
- `ParkingSpotFactory`

**Simple reason**  
Factory can simplify object creation when rules become more complex.

---

## Final Summary

### Patterns definitely used
- **Strategy Pattern**
- **Facade Pattern**

### Strong supporting design ideas
- **Abstraction and Polymorphism**
- **Composition Over Inheritance**
- **State Modeling**
- **Open Closed Principle**
- **Bidirectional Mapping for performance**
- **Capacity Based Domain Modeling**

### Patterns that can be added later
- **Observer Pattern**
- **Factory Pattern**
- **Full State Pattern**

---

## Interview Ready Answer

In this parking lot design, the main patterns used are **Strategy** and **Facade**.

- **Strategy** is used for fare calculation and parking spot assignment because those rules can change independently.
- **Facade** is used through the `ParkingLot` class, which provides a simple interface to clients and hides internal complexity.

I also used **abstraction and polymorphism** through base classes like `ParkingSpot`, `Vehicle`, `FareStrategy`, and `SpotAssignmentStrategy`, which keeps the system extensible.

In addition, I used **composition** to assemble the system from smaller reusable components, **state modeling** for ticket, payment, and spot lifecycles, and **capacity based modeling** to support shared parking in larger spots.


