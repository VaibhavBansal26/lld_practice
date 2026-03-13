# 2. Movie Ticket Booking System


```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from decimal import Decimal
from typing import Dict, List, Optional


# =========================
# Movie
# =========================

@dataclass(frozen=True)
class Movie:
    title: str
    genre: str
    duration_in_minutes: int

    def get_duration(self) -> timedelta:
        return timedelta(minutes=self.duration_in_minutes)


# =========================
# Cinema
# =========================

class Cinema:
    def __init__(self, name: str, location: str):
        self.name = name
        self.location = location
        self.rooms: List[Room] = []

    def add_room(self, room: "Room") -> None:
        self.rooms.append(room)


# =========================
# Room
# =========================

@dataclass(frozen=True)
class Room:
    room_number: str
    layout: "Layout"


# =========================
# Pricing Strategy
# =========================

class PricingStrategy(ABC):
    @abstractmethod
    def get_price(self) -> Decimal:
        pass


@dataclass(frozen=True)
class NormalRate(PricingStrategy):
    price: Decimal

    def get_price(self) -> Decimal:
        return self.price


@dataclass(frozen=True)
class PremiumRate(PricingStrategy):
    price: Decimal

    def get_price(self) -> Decimal:
        return self.price


@dataclass(frozen=True)
class VIPRate(PricingStrategy):
    price: Decimal

    def get_price(self) -> Decimal:
        return self.price


# =========================
# Seat
# =========================

@dataclass(frozen=True)
class Seat:
    seat_number: str
    pricing_strategy: Optional[PricingStrategy] = None


# =========================
# Layout
# =========================

class Layout:
    def __init__(self, rows: int, columns: int):
        self.rows = rows
        self.columns = columns
        self.seats_by_number: Dict[str, Seat] = {}
        self.seats_by_position: Dict[int, Dict[int, Seat]] = {}
        self._initialize_layout()

    def _initialize_layout(self) -> None:
        for i in range(self.rows):
            for j in range(self.columns):
                seat_number = f"{i}-{j}"
                self.add_seat(seat_number, i, j, Seat(seat_number, None))

    def add_seat(self, seat_number: str, row: int, column: int, seat: Seat) -> None:
        self.seats_by_number[seat_number] = seat

        if row not in self.seats_by_position:
            self.seats_by_position[row] = {}

        self.seats_by_position[row][column] = seat

    def get_seat_by_number(self, seat_number: str) -> Optional[Seat]:
        return self.seats_by_number.get(seat_number)

    def get_seat_by_position(self, row: int, column: int) -> Optional[Seat]:
        row_seats = self.seats_by_position.get(row)
        return row_seats.get(column) if row_seats else None

    def get_all_seats(self) -> List[Seat]:
        return list(self.seats_by_number.values())


# =========================
# Screening
# =========================

@dataclass(frozen=True)
class Screening:
    movie: Movie
    room: Room
    start_time: datetime
    end_time: datetime

    def get_duration(self) -> timedelta:
        return self.end_time - self.start_time


# =========================
# Ticket
# =========================

@dataclass(frozen=True)
class Ticket:
    screening: Screening
    seat: Seat
    price: Decimal


# =========================
# Order
# =========================

class Order:
    def __init__(self, order_date: datetime):
        self.tickets: List[Ticket] = []
        self.order_date = order_date

    def add_ticket(self, ticket: Ticket) -> None:
        self.tickets.append(ticket)

    def calculate_total_price(self) -> Decimal:
        total = Decimal("0")
        for ticket in self.tickets:
            total += ticket.price
        return total


# =========================
# Screening Manager
# =========================

class ScreeningManager:
    def __init__(self):
        self.screenings_by_movie: Dict[Movie, List[Screening]] = {}
        self.tickets_by_screening: Dict[Screening, List[Ticket]] = {}

    def add_screening(self, movie: Movie, screening: Screening) -> None:
        if movie not in self.screenings_by_movie:
            self.screenings_by_movie[movie] = []
        self.screenings_by_movie[movie].append(screening)

    def get_screenings_for_movie(self, movie: Movie) -> List[Screening]:
        return list(self.screenings_by_movie.get(movie, []))

    def add_ticket(self, screening: Screening, ticket: Ticket) -> None:
        if screening not in self.tickets_by_screening:
            self.tickets_by_screening[screening] = []
        self.tickets_by_screening[screening].append(ticket)

    def get_tickets_for_screening(self, screening: Screening) -> List[Ticket]:
        return list(self.tickets_by_screening.get(screening, []))

    def get_available_seats(self, screening: Screening) -> List[Seat]:
        all_seats = screening.room.layout.get_all_seats()
        booked_tickets = self.get_tickets_for_screening(screening)

        booked_seats = {ticket.seat for ticket in booked_tickets}
        return [seat for seat in all_seats if seat not in booked_seats]


# =========================
# Movie Booking System
# =========================

class MovieBookingSystem:
    def __init__(self):
        self.movies: List[Movie] = []
        self.cinemas: List[Cinema] = []
        self.screening_manager = ScreeningManager()

    def add_movie(self, movie: Movie) -> None:
        self.movies.append(movie)

    def add_cinema(self, cinema: Cinema) -> None:
        self.cinemas.append(cinema)

    def add_screening(self, movie: Movie, screening: Screening) -> None:
        self.screening_manager.add_screening(movie, screening)

    def book_ticket(self, screening: Screening, seat: Seat) -> Ticket:
        if seat not in self.get_available_seats(screening):
            raise ValueError(f"Seat {seat.seat_number} is already booked for this screening.")

        if seat.pricing_strategy is None:
            raise ValueError(f"Seat {seat.seat_number} does not have a pricing strategy.")

        price = seat.pricing_strategy.get_price()
        ticket = Ticket(screening, seat, price)
        self.screening_manager.add_ticket(screening, ticket)
        return ticket

    def get_screenings_for_movie(self, movie: Movie) -> List[Screening]:
        return self.screening_manager.get_screenings_for_movie(movie)

    def get_available_seats(self, screening: Screening) -> List[Seat]:
        return self.screening_manager.get_available_seats(screening)

    def get_ticket_count(self, screening: Screening) -> int:
        return len(self.screening_manager.get_tickets_for_screening(screening))

    def get_tickets_for_screening(self, screening: Screening) -> List[Ticket]:
        return self.screening_manager.get_tickets_for_screening(screening)


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    # Create movie
    movie = Movie("Inception", "Sci-Fi", 148)

    # Create layout
    layout = Layout(rows=3, columns=3)

    # Override seat pricing
    layout.add_seat("0-0", 0, 0, Seat("0-0", VIPRate(Decimal("25.00"))))
    layout.add_seat("0-1", 0, 1, Seat("0-1", PremiumRate(Decimal("18.00"))))
    layout.add_seat("0-2", 0, 2, Seat("0-2", PremiumRate(Decimal("18.00"))))
    layout.add_seat("1-0", 1, 0, Seat("1-0", NormalRate(Decimal("12.00"))))
    layout.add_seat("1-1", 1, 1, Seat("1-1", NormalRate(Decimal("12.00"))))
    layout.add_seat("1-2", 1, 2, Seat("1-2", NormalRate(Decimal("12.00"))))
    layout.add_seat("2-0", 2, 0, Seat("2-0", NormalRate(Decimal("10.00"))))
    layout.add_seat("2-1", 2, 1, Seat("2-1", NormalRate(Decimal("10.00"))))
    layout.add_seat("2-2", 2, 2, Seat("2-2", NormalRate(Decimal("10.00"))))

    # Create room and cinema
    room = Room("R1", layout)
    cinema = Cinema("PVR", "Downtown")
    cinema.add_room(room)

    # Create screening
    start = datetime(2026, 3, 13, 19, 0, 0)
    end = datetime(2026, 3, 13, 21, 28, 0)
    screening = Screening(movie, room, start, end)

    # Create booking system
    booking_system = MovieBookingSystem()
    booking_system.add_movie(movie)
    booking_system.add_cinema(cinema)
    booking_system.add_screening(movie, screening)

    # Book tickets
    seat1 = layout.get_seat_by_number("0-0")
    seat2 = layout.get_seat_by_position(1, 1)

    ticket1 = booking_system.book_ticket(screening, seat1)
    ticket2 = booking_system.book_ticket(screening, seat2)

    # Create order
    order = Order(datetime.now())
    order.add_ticket(ticket1)
    order.add_ticket(ticket2)

    print("Movie duration:", movie.get_duration())
    print("Screening duration:", screening.get_duration())
    print("Available seats:", [seat.seat_number for seat in booking_system.get_available_seats(screening)])
    print("Tickets sold:", booking_system.get_ticket_count(screening))
    print("Order total:", order.calculate_total_price())
```

**Code with both Pessimistic and Optimistic Locking**

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime, timedelta
from decimal import Decimal
from threading import Lock
from typing import Dict, List, Optional


# =========================
# Movie
# =========================

@dataclass(frozen=True)
class Movie:
    title: str
    genre: str
    duration_in_minutes: int

    def get_duration(self) -> timedelta:
        return timedelta(minutes=self.duration_in_minutes)


# =========================
# Cinema
# =========================

class Cinema:
    def __init__(self, name: str, location: str):
        self.name = name
        self.location = location
        self.rooms: List[Room] = []

    def add_room(self, room: "Room") -> None:
        self.rooms.append(room)


# =========================
# Room
# =========================

@dataclass(frozen=True)
class Room:
    room_number: str
    layout: "Layout"


# =========================
# Pricing Strategy
# =========================

class PricingStrategy(ABC):
    @abstractmethod
    def get_price(self) -> Decimal:
        pass


@dataclass(frozen=True)
class NormalRate(PricingStrategy):
    price: Decimal

    def get_price(self) -> Decimal:
        return self.price


@dataclass(frozen=True)
class PremiumRate(PricingStrategy):
    price: Decimal

    def get_price(self) -> Decimal:
        return self.price


@dataclass(frozen=True)
class VIPRate(PricingStrategy):
    price: Decimal

    def get_price(self) -> Decimal:
        return self.price


# =========================
# Seat
# =========================

@dataclass(frozen=True)
class Seat:
    seat_number: str
    pricing_strategy: Optional[PricingStrategy] = None


# =========================
# Layout
# =========================

class Layout:
    def __init__(self, rows: int, columns: int):
        self.rows = rows
        self.columns = columns
        self.seats_by_number: Dict[str, Seat] = {}
        self.seats_by_position: Dict[int, Dict[int, Seat]] = {}
        self._initialize_layout()

    def _initialize_layout(self) -> None:
        for i in range(self.rows):
            for j in range(self.columns):
                seat_number = f"{i}-{j}"
                self.add_seat(seat_number, i, j, Seat(seat_number, None))

    def add_seat(self, seat_number: str, row: int, column: int, seat: Seat) -> None:
        self.seats_by_number[seat_number] = seat
        if row not in self.seats_by_position:
            self.seats_by_position[row] = {}
        self.seats_by_position[row][column] = seat

    def get_seat_by_number(self, seat_number: str) -> Optional[Seat]:
        return self.seats_by_number.get(seat_number)

    def get_seat_by_position(self, row: int, column: int) -> Optional[Seat]:
        row_seats = self.seats_by_position.get(row)
        return row_seats.get(column) if row_seats else None

    def get_all_seats(self) -> List[Seat]:
        return list(self.seats_by_number.values())


# =========================
# Screening
# =========================

@dataclass(frozen=True)
class Screening:
    screening_id: str
    movie: Movie
    room: Room
    start_time: datetime
    end_time: datetime

    def get_duration(self) -> timedelta:
        return self.end_time - self.start_time


# =========================
# Ticket
# =========================

@dataclass(frozen=True)
class Ticket:
    screening: Screening
    seat: Seat
    price: Decimal


# =========================
# Order
# =========================

class Order:
    def __init__(self, order_date: datetime):
        self.tickets: List[Ticket] = []
        self.order_date = order_date

    def add_ticket(self, ticket: Ticket) -> None:
        self.tickets.append(ticket)

    def calculate_total_price(self) -> Decimal:
        total = Decimal("0")
        for ticket in self.tickets:
            total += ticket.price
        return total


# =========================
# Seat Lock Manager
# Pessimistic Locking
# =========================

@dataclass
class SeatLock:
    user_id: str
    expiration_time: datetime

    def is_expired(self) -> bool:
        return datetime.now() > self.expiration_time


class SeatLockManager:
    def __init__(self, lock_duration: timedelta):
        self.locked_seats: Dict[str, SeatLock] = {}
        self.lock_duration = lock_duration
        self._lock = Lock()

    def lock_seat(self, screening: Screening, seat: Seat, user_id: str) -> bool:
        with self._lock:
            lock_key = self._generate_lock_key(screening, seat)
            self._cleanup_lock_if_expired(lock_key)

            if self.is_locked(screening, seat):
                return False

            lock = SeatLock(
                user_id=user_id,
                expiration_time=datetime.now() + self.lock_duration,
            )
            self.locked_seats[lock_key] = lock
            return True

    def unlock_seat(self, screening: Screening, seat: Seat, user_id: str) -> bool:
        with self._lock:
            lock_key = self._generate_lock_key(screening, seat)
            lock = self.locked_seats.get(lock_key)

            if lock is None:
                return False

            if lock.user_id != user_id:
                return False

            del self.locked_seats[lock_key]
            return True

    def is_locked(self, screening: Screening, seat: Seat) -> bool:
        lock_key = self._generate_lock_key(screening, seat)
        self._cleanup_lock_if_expired(lock_key)
        return lock_key in self.locked_seats

    def _cleanup_lock_if_expired(self, lock_key: str) -> None:
        lock = self.locked_seats.get(lock_key)
        if lock is not None and lock.is_expired():
            del self.locked_seats[lock_key]

    def _generate_lock_key(self, screening: Screening, seat: Seat) -> str:
        return f"{screening.screening_id}-{seat.seat_number}"


# =========================
# Screening Manager
# Includes optimistic and pessimistic booking methods
# =========================

class ScreeningManager:
    def __init__(self):
        self.screenings_by_movie: Dict[Movie, List[Screening]] = {}
        self.tickets_by_screening: Dict[Screening, List[Ticket]] = {}
        self._booking_lock = Lock()

    def add_screening(self, movie: Movie, screening: Screening) -> None:
        if movie not in self.screenings_by_movie:
            self.screenings_by_movie[movie] = []
        self.screenings_by_movie[movie].append(screening)

    def get_screenings_for_movie(self, movie: Movie) -> List[Screening]:
        return list(self.screenings_by_movie.get(movie, []))

    def add_ticket(self, screening: Screening, ticket: Ticket) -> None:
        if screening not in self.tickets_by_screening:
            self.tickets_by_screening[screening] = []
        self.tickets_by_screening[screening].append(ticket)

    def get_tickets_for_screening(self, screening: Screening) -> List[Ticket]:
        return list(self.tickets_by_screening.get(screening, []))

    def is_seat_booked(self, screening: Screening, seat: Seat) -> bool:
        tickets = self.get_tickets_for_screening(screening)
        return any(ticket.seat == seat for ticket in tickets)

    def get_available_seats(self, screening: Screening) -> List[Seat]:
        all_seats = screening.room.layout.get_all_seats()
        booked_tickets = self.get_tickets_for_screening(screening)
        booked_seats = {ticket.seat for ticket in booked_tickets}
        return [seat for seat in all_seats if seat not in booked_seats]

    # Optimistic locking
    def book_seat_optimistically(self, screening: Screening, seat: Seat) -> Ticket:
        with self._booking_lock:
            if self.is_seat_booked(screening, seat):
                raise ValueError("Seat is already booked")

            if seat.pricing_strategy is None:
                raise ValueError(f"Seat {seat.seat_number} does not have a pricing strategy")

            price = seat.pricing_strategy.get_price()
            ticket = Ticket(screening, seat, price)

            if screening not in self.tickets_by_screening:
                self.tickets_by_screening[screening] = []
            self.tickets_by_screening[screening].append(ticket)

            return ticket

    # Pessimistic locking
    def book_seat_with_lock(
        self,
        screening: Screening,
        seat: Seat,
        user_id: str,
        seat_lock_manager: SeatLockManager,
    ) -> Ticket:
        with self._booking_lock:
            if not seat_lock_manager.is_locked(screening, seat):
                raise ValueError("Seat is not locked by the user")

            if self.is_seat_booked(screening, seat):
                seat_lock_manager.unlock_seat(screening, seat, user_id)
                raise ValueError("Seat is already booked")

            if seat.pricing_strategy is None:
                seat_lock_manager.unlock_seat(screening, seat, user_id)
                raise ValueError(f"Seat {seat.seat_number} does not have a pricing strategy")

            price = seat.pricing_strategy.get_price()
            ticket = Ticket(screening, seat, price)

            if screening not in self.tickets_by_screening:
                self.tickets_by_screening[screening] = []
            self.tickets_by_screening[screening].append(ticket)

            seat_lock_manager.unlock_seat(screening, seat, user_id)
            return ticket


# =========================
# Movie Booking System
# Facade
# =========================

class MovieBookingSystem:
    def __init__(self, lock_duration_seconds: int = 30):
        self.movies: List[Movie] = []
        self.cinemas: List[Cinema] = []
        self.screening_manager = ScreeningManager()
        self.seat_lock_manager = SeatLockManager(timedelta(seconds=lock_duration_seconds))

    def add_movie(self, movie: Movie) -> None:
        self.movies.append(movie)

    def add_cinema(self, cinema: Cinema) -> None:
        self.cinemas.append(cinema)

    def add_screening(self, movie: Movie, screening: Screening) -> None:
        self.screening_manager.add_screening(movie, screening)

    # Basic booking
    def book_ticket(self, screening: Screening, seat: Seat) -> Ticket:
        return self.screening_manager.book_seat_optimistically(screening, seat)

    # Optimistic locking booking
    def book_ticket_optimistically(self, screening: Screening, seat: Seat) -> Ticket:
        return self.screening_manager.book_seat_optimistically(screening, seat)

    # Pessimistic locking helpers
    def lock_seat(self, screening: Screening, seat: Seat, user_id: str) -> bool:
        return self.seat_lock_manager.lock_seat(screening, seat, user_id)

    def unlock_seat(self, screening: Screening, seat: Seat, user_id: str) -> bool:
        return self.seat_lock_manager.unlock_seat(screening, seat, user_id)

    def is_seat_locked(self, screening: Screening, seat: Seat) -> bool:
        return self.seat_lock_manager.is_locked(screening, seat)

    def book_ticket_with_lock(self, screening: Screening, seat: Seat, user_id: str) -> Ticket:
        return self.screening_manager.book_seat_with_lock(
            screening=screening,
            seat=seat,
            user_id=user_id,
            seat_lock_manager=self.seat_lock_manager,
        )

    def get_screenings_for_movie(self, movie: Movie) -> List[Screening]:
        return self.screening_manager.get_screenings_for_movie(movie)

    def get_available_seats(self, screening: Screening) -> List[Seat]:
        return self.screening_manager.get_available_seats(screening)

    def get_ticket_count(self, screening: Screening) -> int:
        return len(self.screening_manager.get_tickets_for_screening(screening))

    def get_tickets_for_screening(self, screening: Screening) -> List[Ticket]:
        return self.screening_manager.get_tickets_for_screening(screening)


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    # Create movie
    movie = Movie("Inception", "Sci-Fi", 148)

    # Create layout
    layout = Layout(rows=2, columns=3)

    # Assign pricing
    layout.add_seat("0-0", 0, 0, Seat("0-0", VIPRate(Decimal("25.00"))))
    layout.add_seat("0-1", 0, 1, Seat("0-1", PremiumRate(Decimal("18.00"))))
    layout.add_seat("0-2", 0, 2, Seat("0-2", PremiumRate(Decimal("18.00"))))
    layout.add_seat("1-0", 1, 0, Seat("1-0", NormalRate(Decimal("12.00"))))
    layout.add_seat("1-1", 1, 1, Seat("1-1", NormalRate(Decimal("12.00"))))
    layout.add_seat("1-2", 1, 2, Seat("1-2", NormalRate(Decimal("12.00"))))

    # Create room and cinema
    room = Room("R1", layout)
    cinema = Cinema("PVR", "Downtown")
    cinema.add_room(room)

    # Create screening
    start = datetime(2026, 3, 13, 19, 0, 0)
    end = datetime(2026, 3, 13, 21, 28, 0)
    screening = Screening("SCR-101", movie, room, start, end)

    # Create system
    booking_system = MovieBookingSystem(lock_duration_seconds=30)
    booking_system.add_movie(movie)
    booking_system.add_cinema(cinema)
    booking_system.add_screening(movie, screening)

    # Optimistic locking example
    seat_a = layout.get_seat_by_number("0-0")
    ticket_a = booking_system.book_ticket_optimistically(screening, seat_a)
    print("Optimistic booking successful:", ticket_a)

    # Pessimistic locking example
    seat_b = layout.get_seat_by_number("0-1")
    user_id = "user-123"

    locked = booking_system.lock_seat(screening, seat_b, user_id)
    print("Seat locked:", locked)

    if locked:
        ticket_b = booking_system.book_ticket_with_lock(screening, seat_b, user_id)
        print("Pessimistic booking successful:", ticket_b)

    print("Available seats:", [seat.seat_number for seat in booking_system.get_available_seats(screening)])
    print("Total tickets sold:", booking_system.get_ticket_count(screening))

```

**Advanced interview-ready version with:**

seat hold state

order status

payment flow

booking expiration

retry support

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from decimal import Decimal
from enum import Enum
from threading import Lock
from typing import Dict, List, Optional
import uuid


# =========================
# Enums
# =========================

class SeatState(str, Enum):
    AVAILABLE = "AVAILABLE"
    HELD = "HELD"
    BOOKED = "BOOKED"


class OrderStatus(str, Enum):
    PENDING = "PENDING"
    HOLDING = "HOLDING"
    PAYMENT_PENDING = "PAYMENT_PENDING"
    CONFIRMED = "CONFIRMED"
    CANCELLED = "CANCELLED"
    EXPIRED = "EXPIRED"
    FAILED = "FAILED"


class PaymentStatus(str, Enum):
    INITIATED = "INITIATED"
    SUCCESS = "SUCCESS"
    FAILED = "FAILED"


class PaymentMethod(str, Enum):
    CARD = "CARD"
    UPI = "UPI"
    CASH = "CASH"


# =========================
# Movie
# =========================

@dataclass(frozen=True)
class Movie:
    title: str
    genre: str
    duration_in_minutes: int

    def get_duration(self) -> timedelta:
        return timedelta(minutes=self.duration_in_minutes)


# =========================
# Pricing Strategy
# =========================

class PricingStrategy(ABC):
    @abstractmethod
    def get_price(self) -> Decimal:
        pass


@dataclass(frozen=True)
class NormalRate(PricingStrategy):
    price: Decimal

    def get_price(self) -> Decimal:
        return self.price


@dataclass(frozen=True)
class PremiumRate(PricingStrategy):
    price: Decimal

    def get_price(self) -> Decimal:
        return self.price


@dataclass(frozen=True)
class VIPRate(PricingStrategy):
    price: Decimal

    def get_price(self) -> Decimal:
        return self.price


# =========================
# Seat / Layout / Room / Cinema
# =========================

@dataclass(frozen=True)
class Seat:
    seat_number: str
    pricing_strategy: Optional[PricingStrategy] = None


class Layout:
    def __init__(self, rows: int, columns: int):
        self.rows = rows
        self.columns = columns
        self.seats_by_number: Dict[str, Seat] = {}
        self.seats_by_position: Dict[int, Dict[int, Seat]] = {}
        self._initialize_layout()

    def _initialize_layout(self) -> None:
        for i in range(self.rows):
            for j in range(self.columns):
                seat_number = f"{i}-{j}"
                self.add_seat(seat_number, i, j, Seat(seat_number, None))

    def add_seat(self, seat_number: str, row: int, column: int, seat: Seat) -> None:
        self.seats_by_number[seat_number] = seat
        if row not in self.seats_by_position:
            self.seats_by_position[row] = {}
        self.seats_by_position[row][column] = seat

    def get_seat_by_number(self, seat_number: str) -> Optional[Seat]:
        return self.seats_by_number.get(seat_number)

    def get_seat_by_position(self, row: int, column: int) -> Optional[Seat]:
        row_seats = self.seats_by_position.get(row)
        return row_seats.get(column) if row_seats else None

    def get_all_seats(self) -> List[Seat]:
        return list(self.seats_by_number.values())


@dataclass(frozen=True)
class Room:
    room_number: str
    layout: Layout


class Cinema:
    def __init__(self, name: str, location: str):
        self.name = name
        self.location = location
        self.rooms: List[Room] = []

    def add_room(self, room: Room) -> None:
        self.rooms.append(room)


# =========================
# Screening
# =========================

@dataclass(frozen=True)
class Screening:
    screening_id: str
    movie: Movie
    room: Room
    start_time: datetime
    end_time: datetime

    def get_duration(self) -> timedelta:
        return self.end_time - self.start_time


# =========================
# Ticket / Order / Payment
# =========================

@dataclass(frozen=True)
class Ticket:
    screening: Screening
    seat: Seat
    price: Decimal


@dataclass
class Payment:
    payment_id: str
    order_id: str
    amount: Decimal
    method: PaymentMethod
    status: PaymentStatus
    created_at: datetime = field(default_factory=datetime.now)


@dataclass
class Order:
    order_id: str
    user_id: str
    screening: Screening
    seats: List[Seat]
    tickets: List[Ticket] = field(default_factory=list)
    status: OrderStatus = OrderStatus.PENDING
    created_at: datetime = field(default_factory=datetime.now)
    expires_at: Optional[datetime] = None
    payment: Optional[Payment] = None
    retry_count: int = 0

    def calculate_total_price(self) -> Decimal:
        total = Decimal("0")
        for ticket in self.tickets:
            total += ticket.price
        return total


# =========================
# Seat Hold
# =========================

@dataclass
class SeatHold:
    hold_id: str
    screening_id: str
    seat_number: str
    user_id: str
    order_id: str
    expires_at: datetime

    def is_expired(self) -> bool:
        return datetime.now() > self.expires_at


# =========================
# Payment Processor
# =========================

class PaymentProcessor:
    def process(self, order: Order, method: PaymentMethod, should_fail: bool = False) -> Payment:
        payment = Payment(
            payment_id=str(uuid.uuid4()),
            order_id=order.order_id,
            amount=order.calculate_total_price(),
            method=method,
            status=PaymentStatus.FAILED if should_fail else PaymentStatus.SUCCESS,
        )
        return payment


# =========================
# Screening Manager
# Seat states, holds, booking, expiration
# =========================

class ScreeningManager:
    def __init__(self, hold_duration_seconds: int = 30):
        self.screenings_by_movie: Dict[Movie, List[Screening]] = {}
        self.tickets_by_screening: Dict[Screening, List[Ticket]] = {}

        self.seat_holds: Dict[str, SeatHold] = {}
        self.booked_seats: Dict[str, Ticket] = {}

        self.hold_duration = timedelta(seconds=hold_duration_seconds)
        self._lock = Lock()

    def add_screening(self, movie: Movie, screening: Screening) -> None:
        if movie not in self.screenings_by_movie:
            self.screenings_by_movie[movie] = []
        self.screenings_by_movie[movie].append(screening)

    def get_screenings_for_movie(self, movie: Movie) -> List[Screening]:
        return list(self.screenings_by_movie.get(movie, []))

    def get_tickets_for_screening(self, screening: Screening) -> List[Ticket]:
        return list(self.tickets_by_screening.get(screening, []))

    def _seat_key(self, screening: Screening, seat: Seat) -> str:
        return f"{screening.screening_id}:{seat.seat_number}"

    def _cleanup_expired_hold_by_key(self, seat_key: str) -> None:
        hold = self.seat_holds.get(seat_key)
        if hold and hold.is_expired():
            del self.seat_holds[seat_key]

    def cleanup_all_expired_holds(self) -> None:
        with self._lock:
            expired_keys = [key for key, hold in self.seat_holds.items() if hold.is_expired()]
            for key in expired_keys:
                del self.seat_holds[key]

    def get_seat_state(self, screening: Screening, seat: Seat) -> SeatState:
        seat_key = self._seat_key(screening, seat)
        self._cleanup_expired_hold_by_key(seat_key)

        if seat_key in self.booked_seats:
            return SeatState.BOOKED
        if seat_key in self.seat_holds:
            return SeatState.HELD
        return SeatState.AVAILABLE

    def hold_seats(self, screening: Screening, seats: List[Seat], user_id: str, order_id: str) -> List[SeatHold]:
        with self._lock:
            created_holds: List[SeatHold] = []

            for seat in seats:
                seat_key = self._seat_key(screening, seat)
                self._cleanup_expired_hold_by_key(seat_key)

                if seat_key in self.booked_seats:
                    raise ValueError(f"Seat {seat.seat_number} is already booked.")

                existing_hold = self.seat_holds.get(seat_key)
                if existing_hold and existing_hold.user_id != user_id:
                    raise ValueError(f"Seat {seat.seat_number} is currently held by another user.")

            for seat in seats:
                seat_key = self._seat_key(screening, seat)
                hold = SeatHold(
                    hold_id=str(uuid.uuid4()),
                    screening_id=screening.screening_id,
                    seat_number=seat.seat_number,
                    user_id=user_id,
                    order_id=order_id,
                    expires_at=datetime.now() + self.hold_duration,
                )
                self.seat_holds[seat_key] = hold
                created_holds.append(hold)

            return created_holds

    def release_order_holds(self, screening: Screening, order_id: str) -> None:
        with self._lock:
            to_remove = []
            for seat_key, hold in self.seat_holds.items():
                if hold.screening_id == screening.screening_id and hold.order_id == order_id:
                    to_remove.append(seat_key)

            for seat_key in to_remove:
                del self.seat_holds[seat_key]

    def confirm_booking(self, screening: Screening, seats: List[Seat], user_id: str, order_id: str) -> List[Ticket]:
        with self._lock:
            tickets: List[Ticket] = []

            for seat in seats:
                seat_key = self._seat_key(screening, seat)
                self._cleanup_expired_hold_by_key(seat_key)

                if seat_key in self.booked_seats:
                    raise ValueError(f"Seat {seat.seat_number} is already booked.")

                hold = self.seat_holds.get(seat_key)
                if hold is None:
                    raise ValueError(f"Seat {seat.seat_number} is not held.")
                if hold.user_id != user_id or hold.order_id != order_id:
                    raise ValueError(f"Seat {seat.seat_number} is held by another user or order.")

            for seat in seats:
                if seat.pricing_strategy is None:
                    raise ValueError(f"Seat {seat.seat_number} has no pricing strategy.")

                ticket = Ticket(
                    screening=screening,
                    seat=seat,
                    price=seat.pricing_strategy.get_price(),
                )
                seat_key = self._seat_key(screening, seat)
                self.booked_seats[seat_key] = ticket

                if screening not in self.tickets_by_screening:
                    self.tickets_by_screening[screening] = []
                self.tickets_by_screening[screening].append(ticket)

                if seat_key in self.seat_holds:
                    del self.seat_holds[seat_key]

                tickets.append(ticket)

            return tickets

    def get_available_seats(self, screening: Screening) -> List[Seat]:
        self.cleanup_all_expired_holds()
        all_seats = screening.room.layout.get_all_seats()
        available = []
        for seat in all_seats:
            if self.get_seat_state(screening, seat) == SeatState.AVAILABLE:
                available.append(seat)
        return available


# =========================
# Order Manager
# =========================

class OrderManager:
    def __init__(
        self,
        screening_manager: ScreeningManager,
        payment_processor: PaymentProcessor,
        max_retries: int = 3,
    ):
        self.screening_manager = screening_manager
        self.payment_processor = payment_processor
        self.orders_by_id: Dict[str, Order] = {}
        self.max_retries = max_retries
        self._lock = Lock()

    def create_order(self, user_id: str, screening: Screening, seats: List[Seat]) -> Order:
        with self._lock:
            order = Order(
                order_id=str(uuid.uuid4()),
                user_id=user_id,
                screening=screening,
                seats=seats,
                status=OrderStatus.PENDING,
            )
            self.orders_by_id[order.order_id] = order
            return order

    def hold_order_seats(self, order: Order) -> None:
        with self._lock:
            if order.status not in {OrderStatus.PENDING, OrderStatus.FAILED}:
                raise ValueError(f"Order {order.order_id} cannot move to hold from {order.status}.")

            self.screening_manager.hold_seats(
                screening=order.screening,
                seats=order.seats,
                user_id=order.user_id,
                order_id=order.order_id,
            )
            order.status = OrderStatus.HOLDING
            order.expires_at = datetime.now() + self.screening_manager.hold_duration

    def expire_order_if_needed(self, order: Order) -> None:
        with self._lock:
            if order.expires_at and datetime.now() > order.expires_at and order.status in {
                OrderStatus.HOLDING,
                OrderStatus.PAYMENT_PENDING,
            }:
                self.screening_manager.release_order_holds(order.screening, order.order_id)
                order.status = OrderStatus.EXPIRED

    def checkout(self, order: Order, method: PaymentMethod, should_fail: bool = False) -> Order:
        with self._lock:
            self.expire_order_if_needed(order)

            if order.status == OrderStatus.EXPIRED:
                raise ValueError("Order has expired.")

            if order.status != OrderStatus.HOLDING:
                raise ValueError(f"Order {order.order_id} is not ready for checkout.")

            order.status = OrderStatus.PAYMENT_PENDING

            try:
                tickets = self.screening_manager.confirm_booking(
                    screening=order.screening,
                    seats=order.seats,
                    user_id=order.user_id,
                    order_id=order.order_id,
                )
                order.tickets = tickets

                payment = self.payment_processor.process(order, method, should_fail=should_fail)
                order.payment = payment

                if payment.status == PaymentStatus.SUCCESS:
                    order.status = OrderStatus.CONFIRMED
                else:
                    order.status = OrderStatus.FAILED
                    order.retry_count += 1

            except Exception:
                order.status = OrderStatus.FAILED
                order.retry_count += 1
                raise

            return order

    def retry_payment(self, order: Order, method: PaymentMethod, should_fail: bool = False) -> Order:
        with self._lock:
            self.expire_order_if_needed(order)

            if order.status == OrderStatus.EXPIRED:
                raise ValueError("Order has expired.")

            if order.retry_count >= self.max_retries:
                raise ValueError("Max retry limit reached.")

            if order.status not in {OrderStatus.FAILED, OrderStatus.PAYMENT_PENDING}:
                raise ValueError(f"Order {order.order_id} is not eligible for retry.")

            if not order.tickets:
                raise ValueError("No tickets found for retry. Booking was not confirmed.")

            payment = self.payment_processor.process(order, method, should_fail=should_fail)
            order.payment = payment

            if payment.status == PaymentStatus.SUCCESS:
                order.status = OrderStatus.CONFIRMED
            else:
                order.status = OrderStatus.FAILED
                order.retry_count += 1

            return order

    def cancel_order(self, order: Order) -> None:
        with self._lock:
            if order.status in {OrderStatus.CONFIRMED, OrderStatus.CANCELLED, OrderStatus.EXPIRED}:
                raise ValueError(f"Order {order.order_id} cannot be cancelled from {order.status}.")

            self.screening_manager.release_order_holds(order.screening, order.order_id)
            order.status = OrderStatus.CANCELLED

    def get_order(self, order_id: str) -> Optional[Order]:
        return self.orders_by_id.get(order_id)


# =========================
# Facade
# =========================

class MovieBookingSystem:
    def __init__(self, hold_duration_seconds: int = 30, max_payment_retries: int = 3):
        self.movies: List[Movie] = []
        self.cinemas: List[Cinema] = []
        self.screening_manager = ScreeningManager(hold_duration_seconds=hold_duration_seconds)
        self.payment_processor = PaymentProcessor()
        self.order_manager = OrderManager(
            screening_manager=self.screening_manager,
            payment_processor=self.payment_processor,
            max_retries=max_payment_retries,
        )

    def add_movie(self, movie: Movie) -> None:
        self.movies.append(movie)

    def add_cinema(self, cinema: Cinema) -> None:
        self.cinemas.append(cinema)

    def add_screening(self, movie: Movie, screening: Screening) -> None:
        self.screening_manager.add_screening(movie, screening)

    def get_screenings_for_movie(self, movie: Movie) -> List[Screening]:
        return self.screening_manager.get_screenings_for_movie(movie)

    def get_available_seats(self, screening: Screening) -> List[Seat]:
        return self.screening_manager.get_available_seats(screening)

    def get_seat_state(self, screening: Screening, seat: Seat) -> SeatState:
        return self.screening_manager.get_seat_state(screening, seat)

    def create_order(self, user_id: str, screening: Screening, seats: List[Seat]) -> Order:
        return self.order_manager.create_order(user_id, screening, seats)

    def hold_order_seats(self, order: Order) -> None:
        self.order_manager.hold_order_seats(order)

    def checkout(self, order: Order, method: PaymentMethod, should_fail: bool = False) -> Order:
        return self.order_manager.checkout(order, method, should_fail=should_fail)

    def retry_payment(self, order: Order, method: PaymentMethod, should_fail: bool = False) -> Order:
        return self.order_manager.retry_payment(order, method, should_fail=should_fail)

    def cancel_order(self, order: Order) -> None:
        self.order_manager.cancel_order(order)

    def expire_order_if_needed(self, order: Order) -> None:
        self.order_manager.expire_order_if_needed(order)


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    movie = Movie("Interstellar", "Sci-Fi", 169)

    layout = Layout(rows=2, columns=3)
    layout.add_seat("0-0", 0, 0, Seat("0-0", VIPRate(Decimal("25.00"))))
    layout.add_seat("0-1", 0, 1, Seat("0-1", PremiumRate(Decimal("18.00"))))
    layout.add_seat("0-2", 0, 2, Seat("0-2", PremiumRate(Decimal("18.00"))))
    layout.add_seat("1-0", 1, 0, Seat("1-0", NormalRate(Decimal("12.00"))))
    layout.add_seat("1-1", 1, 1, Seat("1-1", NormalRate(Decimal("12.00"))))
    layout.add_seat("1-2", 1, 2, Seat("1-2", NormalRate(Decimal("12.00"))))

    room = Room("R1", layout)
    cinema = Cinema("AMC", "Downtown")
    cinema.add_room(room)

    screening = Screening(
        screening_id="SCR-001",
        movie=movie,
        room=room,
        start_time=datetime(2026, 3, 13, 19, 0, 0),
        end_time=datetime(2026, 3, 13, 21, 49, 0),
    )

    system = MovieBookingSystem(hold_duration_seconds=10, max_payment_retries=2)
    system.add_movie(movie)
    system.add_cinema(cinema)
    system.add_screening(movie, screening)

    seat_a = layout.get_seat_by_number("0-0")
    seat_b = layout.get_seat_by_number("1-1")

    # Step 1: create order
    order = system.create_order("user-123", screening, [seat_a, seat_b])
    print("Order created:", order.order_id, order.status)

    # Step 2: hold seats
    system.hold_order_seats(order)
    print("Order after hold:", order.status, order.expires_at)
    print("Seat A state:", system.get_seat_state(screening, seat_a).value)

    # Step 3: checkout with forced failure
    try:
        system.checkout(order, PaymentMethod.CARD, should_fail=True)
    except Exception as exc:
        print("Checkout exception:", exc)

    print("Order after failed payment:", order.status, "retries:", order.retry_count)

    # Step 4: retry payment successfully
    updated_order = system.retry_payment(order, PaymentMethod.UPI, should_fail=False)
    print("Order after retry:", updated_order.status)
    print("Payment status:", updated_order.payment.status if updated_order.payment else None)
    print("Total price:", updated_order.calculate_total_price())

    # availability after booking
    print("Available seats:", [s.seat_number for s in system.get_available_seats(screening)])
```

## Design Patterns Used in the Movie Ticket Booking System

### 1. Strategy Pattern

**Where used**
- `PricingStrategy`
- `NormalRate`
- `PremiumRate`
- `VIPRate`

**Why we used it**
Different seats can have different pricing rules.

Examples:
- normal seat pricing
- premium seat pricing
- VIP seat pricing

Instead of hardcoding pricing logic inside the `Seat` class, we delegate pricing to strategy classes.

This makes the system flexible and extensible.  
If we later want to add:
- weekend pricing
- festival pricing
- dynamic surge pricing

we can do so by adding a new strategy without changing the existing seat model.

**Simple reason**  
When behavior can vary independently, Strategy is a good fit.

---

### 2. Facade Pattern

**Where used**
- `MovieBookingSystem`

**Why we used it**
The movie booking system contains many internal components:
- `ScreeningManager`
- `OrderManager`
- `PaymentProcessor`
- `Movie`
- `Cinema`
- `Screening`
- `Seat`

A client should not have to directly coordinate all these classes.

So `MovieBookingSystem` acts as a facade and provides a simplified interface such as:
- `add_movie()`
- `add_cinema()`
- `add_screening()`
- `create_order()`
- `hold_order_seats()`
- `checkout()`
- `retry_payment()`
- `get_available_seats()`

It hides internal complexity and provides a clean entry point to the system.

**Simple reason**  
Facade makes a complex system easier to use.

---

### 3. Abstraction and Polymorphism

**Where used**
- `PricingStrategy`

**Why we used it**
The system depends on abstractions rather than concrete implementations.

For example:
- `Seat` depends on `PricingStrategy`
- it does not care whether the actual implementation is `NormalRate`, `PremiumRate`, or `VIPRate`

This makes the pricing logic easily replaceable and extensible.

**Simple reason**  
Abstraction allows the code to remain flexible and open for extension.

---

### 4. Composition Over Inheritance

**Where used**
- `Seat` contains a `PricingStrategy`
- `Room` contains a `Layout`
- `Cinema` contains `Room` objects
- `Screening` contains a `Movie` and a `Room`
- `Order` contains `Ticket` objects
- `MovieBookingSystem` contains `ScreeningManager`, `OrderManager`, and `PaymentProcessor`

**Why we used it**
Instead of building a deep inheritance hierarchy, the system is created by combining smaller objects with clear responsibilities.

Examples:
- a `Seat` does not inherit pricing behavior, it contains a pricing strategy
- a `Cinema` does not inherit room behavior, it contains rooms

This makes the design easier to modify and test.

**Simple reason**  
Composition allows behavior and structure to be built from reusable parts.

---

### 5. State Modeling

**Where used**
- `SeatState`
- `OrderStatus`
- `PaymentStatus`

**Why we used it**
Seats, orders, and payments move through different states during the booking process.

Examples:
- Seat: `AVAILABLE`, `HELD`, `BOOKED`
- Order: `PENDING`, `HOLDING`, `PAYMENT_PENDING`, `CONFIRMED`, `FAILED`, `EXPIRED`
- Payment: `INITIATED`, `SUCCESS`, `FAILED`

This makes the booking lifecycle explicit and prevents invalid actions.

**Important note**  
This is state representation using enums. It is not the full GoF State pattern with separate state classes.

**Simple reason**  
State modeling makes workflows easier to manage and reason about.

---

### 6. Manager / Coordinator Pattern

**Where used**
- `ScreeningManager`
- `OrderManager`

**Why we used it**
Some classes are responsible for coordinating specific workflows.

#### `ScreeningManager`
Handles:
- screening registration
- seat holds
- booking confirmation
- available seat calculation
- seat state tracking

#### `OrderManager`
Handles:
- order creation
- hold lifecycle
- expiration checks
- checkout
- payment retries
- cancellation

This separates business coordination logic from entity objects like `Movie`, `Seat`, or `Order`.

**Simple reason**  
Managers keep workflow logic organized and prevent entity classes from becoming too large.

---

### 7. Open Closed Principle Through Pattern Usage

**Where visible**
- adding new pricing strategies
- adding new payment methods
- adding new seat states
- adding new order states

**Why we used it**
The system is designed so that new functionality can be added without modifying stable existing logic.

Examples:
- add `DynamicRate`
- add `StudentDiscountRate`
- add `Wallet` as payment method
- add new order states if flow becomes more advanced

**Simple reason**  
The design is open for extension and avoids unnecessary modification of existing code.

---

### 8. Concurrency Control Design

**Where used**
- `SeatHold`
- `ScreeningManager`
- `OrderManager`
- internal thread locks

**Why we used it**
Movie booking systems face race conditions when multiple users try to reserve the same seat at the same time.

We use:
- temporary seat holds
- expiration timestamps
- atomic critical sections using locks

This ensures:
- no double booking
- consistent seat state
- reliable booking confirmation

**Simple reason**  
Concurrency control is necessary in systems where many users may act on the same resource simultaneously.

---

### 9. Optional Observer Pattern Extension

**Where it can be added**
- real time UI updates
- seat map updates
- notification system

**Why it can be useful**
Right now seat availability is queried when needed.

If we want real time updates, Observer can be added so that:
- seat state changes trigger notifications
- booking confirmations update client apps
- expired holds notify listeners

Examples:
- UI seat map refresh
- push notification on payment success
- event stream for booking analytics

**Simple reason**  
Observer is useful for real time updates and notifications.

---

### 10. Optional Factory Pattern Extension

**Where it can be added**
- ticket creation
- order creation
- payment creation
- seat creation

**Why it can be useful**
Right now objects are created directly.

If object creation becomes more complex, a factory can centralize and standardize creation logic.

Examples:
- `TicketFactory`
- `OrderFactory`
- `PaymentFactory`

**Simple reason**  
Factory is useful when object creation rules become more complex.

---

## Final Summary

### Patterns definitely used
- **Strategy Pattern**
- **Facade Pattern**

### Strong supporting design ideas
- **Abstraction and Polymorphism**
- **Composition Over Inheritance**
- **State Modeling**
- **Manager / Coordinator Pattern**
- **Concurrency Control with seat holds**
- **Open Closed Principle**

### Patterns that can be added later
- **Observer Pattern**
- **Factory Pattern**
- **Full State Pattern**

---

## Interview Ready Answer

In this movie ticket booking system, the main patterns used are **Strategy** and **Facade**.

- **Strategy** is used for seat pricing through `PricingStrategy`, allowing different pricing rules like normal, premium, and VIP pricing without changing the `Seat` class.
- **Facade** is used through `MovieBookingSystem`, which provides a simple interface for clients while hiding the internal complexity of screening management, seat holds, order management, and payment flow.

I also used **abstraction and polymorphism** so the system depends on interfaces rather than concrete implementations, **composition** to build the system from smaller reusable objects, and **state modeling** for seat, order, and payment lifecycles.

For concurrency, I added **seat hold management with expiration and atomic locking**, which is critical in booking systems to prevent double booking.