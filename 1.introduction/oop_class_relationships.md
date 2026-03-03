# 3. Class Relationships in OOP

1. **Association:** A relationship where one class uses another class. It can be one-to-one, one-to-many, or many-to-many. The classes are independent of each other.

**Types of association:**

2. **one-to-one:** A single instance of one class is associated with a single instance of another class.
```python
class Person:
    def __init__(self, name):
        self.name = name
        self.passport = None  # A person can have one passport
class Passport:
    def __init__(self, number):
        self.number = number
# Association: A person has a passport
person = Person("Alice")
passport = Passport("123456789")
person.passport = passport  # person is associated with passport

print(f"{person.name} has passport number {person.passport.number}")
```
In this example, a `Person` is associated with a `Passport`. Each person has one passport, and each passport is associated with one person. but how are they related? They are independent classes, and the association is established through an attribute in the `Person` class that references a `Passport` instance. how it is person.passport = passport. This is a one-to-one association because each person has one passport, and each passport is associated with one person.



2. **one-to-many:** A single instance of one class is associated with multiple instances of another

```python
class Author:
    def __init__(self, name):
        self.name = name
        self.books = []  # An author can have multiple books
class Book:
    def __init__(self, title):
        self.title = title
# Association: An author has multiple books
author = Author("J.K. Rowling")
book1 = Book("Harry Potter and the Sorcerer's Stone")
book2 = Book("Harry Potter and the Chamber of Secrets")
author.books.append(book1)  # author is associated with book1
author.books.append(book2)  # author is associated with book2
print(f"{author.name} has written the following books:")
for book in author.books:
    print(f"- {book.title}")
```
In this example, an `Author` is associated with multiple `Book` instances. Each author can have multiple books, but each book is associated with only one author. This is a one-to-many association because one author can have many books, but each book is associated with only one author.

3. **many-to-many:** Multiple instances of one class are associated with multiple instances of another class.

```python
class Student:
    def __init__(self, name):
        self.name = name
        self.courses = []  # A student can enroll in multiple courses
class Course:
    def __init__(self, title):
        self.title = title
        self.students = []  # A course can have multiple students
# Association: Students enroll in courses
student1 = Student("Alice")
student2 = Student("Bob")
course1 = Course("Math 101")
course2 = Course("History 101")
student1.courses.append(course1)  # student1 is associated with course1
student1.courses.append(course2)  # student1 is associated with course2
student2.courses.append(course1)  # student2 is associated with course1
course1.students.append(student1)  # course1 is associated with student1
course1.students.append(student2)  # course1 is associated with student2
course2.students.append(student1)  # course2 is associated with student1
print(f"{student1.name} is enrolled in the following courses:")
for course in student1.courses:
    print(f"- {course.title}")
print(f"{student2.name} is enrolled in the following courses:")
for course in student2.courses:
    print(f"- {course.title}")
```
In this example, `Student` instances are associated with multiple `Course` instances, and each `Course` instance is associated with multiple `Student` instances. This is a many-to-many association because multiple students can enroll in multiple courses, and each course can have multiple students. The association is established through attributes in both classes that reference each other, allowing for a bidirectional relationship between students and courses.


**Practical Example**
Practical Example: Hospital Appointment System

Let's build a system that combines multiple association types in a realistic domain. A hospital manages doctors, patients, rooms, and appointments. The relationships between these entities demonstrate unidirectional, bidirectional, one-to-many, and many-to-many associations working together.

Here's how the classes connect:

- Appointment holds a reference to a Room (unidirectional, the room doesn't know about its appointments).
- Doctor has a list of Appointment objects, and each Appointment points back to its Doctor (bidirectional one-to-many).
- Patient has a list of Appointment objects, and each Appointment points back to its Patient (bidirectional one-to-many).
- Doctor and Patient are connected many-to-many through Appointment as an intermediary. A doctor sees many patients, and a patient can visit many doctors, but they don't reference each other directly.

```python
class Room:
    def __init__(self, number: str, floor: int):
        self.number = number
        self.floor = floor

class Appointment:
    def __init__(self, doctor, patient, room: Room, time: str):
        self.doctor = doctor
        self.patient = patient
        self.room = room
        self.time = time
        doctor.add_appointment(self)
        patient.add_appointment(self)

class Doctor:
    def __init__(self, name: str, specialization: str):
        self.name = name
        self.specialization = specialization
        self.appointments = []

    def add_appointment(self, appt: Appointment):
        self.appointments.append(appt)

    def get_patients(self):
        seen = set()
        result = []
        for appt in self.appointments:
            if id(appt.patient) not in seen:
                seen.add(id(appt.patient))
                result.append(appt.patient)
        return result

class Patient:
    def __init__(self, name: str):
        self.name = name
        self.appointments = []

    def add_appointment(self, appt: Appointment):
        self.appointments.append(appt)

    def get_doctors(self):
        seen = set()
        result = []
        for appt in self.appointments:
            if id(appt.doctor) not in seen:
                seen.add(id(appt.doctor))
                result.append(appt.doctor)
        return result

# Usage
dr_smith = Doctor("Dr. Smith", "Cardiology")
dr_patel = Doctor("Dr. Patel", "Neurology")

alice = Patient("Alice")
bob = Patient("Bob")

room_101 = Room("101", 1)
room_205 = Room("205", 2)

Appointment(dr_smith, alice, room_101, "9:00 AM")
Appointment(dr_smith, bob, room_101, "10:00 AM")
Appointment(dr_patel, alice, room_205, "2:00 PM")

print(f"{dr_smith.name}'s patients:")
for p in dr_smith.get_patients():
    print(f"  - {p.name}")

print(f"{alice.name}'s doctors:")
for d in alice.get_doctors():
    print(f"  - {d.name} ({d.specialization})")

print(f"{dr_smith.name}'s schedule:")
for a in dr_smith.appointments:
    print(f"  - {a.time} with {a.patient.name} in Room {a.room.number}")
```

Why This Design Works
- The Appointment class is the intermediary. Instead of Doctor and Patient holding direct references to each other (which would create a tangled many-to-many), they connect through Appointment. This is a common pattern for modeling many-to-many relationships in code, analogous to a join table in a relational database.
- Navigation works both ways. A doctor can find all their patients by walking their appointments. A patient can find all their doctors the same way. Neither class needs to maintain a separate list of the other.
- Room stays simple. The room doesn't need to know about appointments. It's just a location. This keeps the relationship unidirectional and avoids unnecessary coupling.
- Adding data to the relationship is natural. Because Appointment is a full object, you can add fields like time, status, notes, or diagnosis without modifying Doctor or Patient. Try doing that with a direct many-to-many reference.


1. **Aggregation:** A special form of association where one class is a part of another class. The part can exist independently of the whole. It represents a "has-a" relationship.

```python
class Department:
    def __init__(self, name):
        self.name = name
        self.employees = []  # A department can have multiple employees
class Employee:
    def __init__(self, name):
        self.name = name
# Aggregation: A department has employees
department = Department("HR")
employee1 = Employee("Alice")
employee2 = Employee("Bob")
department.employees.append(employee1)  # department has employee1
department.employees.append(employee2)  # department has employee2
print(f"{department.name} department has the following employees:")
for employee in department.employees:
    print(f"- {employee.name}")
```
In this example, a `Department` is composed of multiple `Employee` instances. The employees can exist independently of the department, meaning they can be created and exist without being part of a department. This is an aggregation relationship because the department "has" employees, but the employees can exist on their own. The association is established through an attribute in the `Department` class that references a list of `Employee` instances, allowing the department to manage its employees while still allowing the employees to exist independently.

**Practical Example: Music Library System**

Let's build a system that demonstrates aggregation across multiple classes. A music library manages artists, songs, playlists, and users. The relationships between these entities show how parts (songs) can be shared across multiple wholes (playlists), and how deleting a whole leaves its parts intact.

Here's how the classes connect:

- Artist is an independent entity that creates songs.
- Song belongs to an Artist but exists independently of any playlist.
- Playlist aggregates multiple Song objects. The same song can appear in different playlists.
- User aggregates multiple Playlist objects. Deleting a user's playlist doesn't destroy the songs.
- Library holds the master collection of all songs, independent of any playlist or user.

```python
class Artist:
    def __init__(self, name: str):
        self.name = name

class Song:
    def __init__(self, title: str, artist: Artist, duration: int):
        self.title = title
        self.artist = artist
        self.duration = duration

    def __str__(self):
        return f"{self.title} by {self.artist.name} ({self.duration}s)"

class Playlist:
    def __init__(self, name: str):
        self.name = name
        self.songs = []

    def add_song(self, song: Song):
        self.songs.append(song)

    def remove_song(self, song: Song):
        self.songs.remove(song)

    def get_song_count(self):
        return len(self.songs)

    def get_total_duration(self):
        return sum(song.duration for song in self.songs)

class User:
    def __init__(self, name: str):
        self.name = name
        self.playlists = []

    def create_playlist(self, playlist_name: str):
        playlist = Playlist(playlist_name)
        self.playlists.append(playlist)
        return playlist

    def delete_playlist(self, playlist: Playlist):
        self.playlists.remove(playlist)

class Library:
    def __init__(self):
        self.songs = []

    def add_song(self, song: Song):
        self.songs.append(song)

    def get_song_count(self):
        return len(self.songs)

# Usage
if __name__ == "__main__":
    coldplay = Artist("Coldplay")
    adele = Artist("Adele")

    yellow = Song("Yellow", coldplay, 269)
    clocks = Song("Clocks", coldplay, 307)
    hello = Song("Hello", adele, 295)
    someone = Song("Someone Like You", adele, 285)

    library = Library()
    library.add_song(yellow)
    library.add_song(clocks)
    library.add_song(hello)
    library.add_song(someone)

    alice = User("Alice")
    workout = alice.create_playlist("Workout Mix")
    chill = alice.create_playlist("Chill Vibes")

    workout.add_song(yellow)
    workout.add_song(clocks)
    workout.add_song(hello)

    chill.add_song(hello)
    chill.add_song(someone)

    print(f"Library has {library.get_song_count()} songs")
    print()

    print(f"{workout.name} ({workout.get_song_count()} songs, {workout.get_total_duration()}s):")
    for s in workout.songs:
        print(f"  - {s}")
    print()

    print(f"{chill.name} ({chill.get_song_count()} songs, {chill.get_total_duration()}s):")
    for s in chill.songs:
        print(f"  - {s}")
    print()

    alice.delete_playlist(workout)
    print(f"After deleting '{workout.name}':")
    print(f"  Library still has {library.get_song_count()} songs")
    print(f"  '{chill.name}' still has {chill.get_song_count()} songs")
    print(f"  'Yellow' still exists: {yellow.title}")
```

Why This Design Works
- Songs are shared across playlists. "Hello" appears in both "Workout Mix" and "Chill Vibes." - Both playlists reference the same Song object. If Adele updates the song metadata, both playlists see the change automatically.
- Deleting a playlist doesn't destroy songs. When "Workout Mix" is deleted, the library still has all 4 songs and "Chill Vibes" is unaffected. The playlist was just a container of references.
- The library is the authoritative source. Songs exist in the Library independently of any playlist. Playlists are views over the library's data, not owners of it.
- Users own playlists, but playlists don't own songs. This is aggregation at two levels: User aggregates Playlist, and Playlist aggregates Song. At each level, the parts survive the deletion of the whole.

3. **Composition:** A stronger form of aggregation where the part cannot exist independently of the whole. If the whole is destroyed, the part is also destroyed. It also represents a "has-a" relationship but with stronger ownership.

```python
class Car:
    def __init__(self, make: str, model: str):
        self.make = make
        self.model = model
        self.engine = Engine()  # A car has an engine (composition)
class Engine:
    def __init__(self):
        self.status = 'off'
    def start(self):
        self.status = 'on'
    def stop(self):
        self.status = 'off'
# Composition: A car has an engine, and the engine cannot exist without the car
car = Car("Toyota", "Camry")
print(f"{car.make} {car.model} has an engine that is currently {car.engine.status}")
car.engine.start()
print(f"After starting the engine, it is now {car.engine.status}")
```
In this example, a `Car` is composed of an `Engine`. The engine cannot exist independently of the car. If the car is destroyed, the engine is also destroyed. This is a composition relationship because the car "has" an engine, and the engine is an integral part of the car's existence. The association is established through an attribute in the `Car` class that creates an instance of `Engine`, indicating that the engine is a part of the car and cannot exist without it.

Multiple Composition:
```python
class Car:
    def __init__(self, make: str, model: str):
        self.make = make
        self.model = model
        self.engine = Engine()  # A car has an engine (composition)
        self.transmission = Transmission()  # A car has a transmission (composition)
        self.chassis = Chassis()  # A car has a chassis (composition)
class Engine:
    def __init__(self):
        self.status = 'off'
    def start(self):
        self.status = 'on'
    def stop(self):
        self.status = 'off'
class Transmission:
    def __init__(self):
        self.gear = 'neutral'
    def shift(self, new_gear):
        self.gear = new_gear
class Chassis:
    def __init__(self):
        self.type = 'sedan'
# Composition: A car has multiple parts that cannot exist without the car
car = Car("Honda", "Civic")
print(f"{car.make} {car.model} has an engine that is currently {car.engine.status}, a transmission in {car.transmission.gear} gear, and a {car.chassis.type} chassis.")
car.engine.start()
car.transmission.shift('drive')
print(f"After starting the engine and shifting the transmission, the engine is now {car.engine.status} and the transmission is in {car.transmission.gear} gear.")
```
In this example, the `Car` class is composed of multiple parts: `Engine`, `Transmission`, and `Chassis`. Each of these parts cannot exist independently of the car. If the car is destroyed, all of its parts are also destroyed. This demonstrates multiple composition relationships within a single class, where the car is the whole and the engine, transmission, and chassis are integral parts that define the car's functionality and existence. The association is established through attributes in the `Car` class that create instances of each part, indicating that they are all essential components of the car.


Practical Example:
et's model the ordering scenario. An Order composes multiple LineItem objects. The order creates line items internally when items are added, and destroys them when the order is destroyed.

```python
class LineItem:
    def __init__(self, product_name, quantity, unit_price):
        self.product_name = product_name
        self.quantity = quantity
        self.unit_price = unit_price

    def get_subtotal(self):
        return self.quantity * self.unit_price

    def describe(self):
        print(f"{self.product_name} x{self.quantity} "
              f"@ ${self.unit_price:.2f} = ${self.get_subtotal():.2f}")

class Order:
    def __init__(self, order_id):
        self.order_id = order_id
        self.line_items = []

    def add_item(self, product, quantity, unit_price):
        self.line_items.append(LineItem(product, quantity, unit_price))

    def remove_item(self, product):
        self.line_items = [
            item for item in self.line_items
            if item.product_name != product
        ]

    def get_total(self):
        return sum(item.get_subtotal() for item in self.line_items)

    def print_receipt(self):
        print(f"Order: {self.order_id}")
        for item in self.line_items:
            item.describe()
        print(f"Total: ${self.get_total():.2f}")

if __name__ == "__main__":
    order = Order("ORD-1001")
    order.add_item("Wireless Mouse", 2, 29.99)
    order.add_item("USB-C Cable", 3, 9.99)
    order.add_item("Laptop Stand", 1, 49.99)

    order.print_receipt()

    # When order is deleted, all LineItems are destroyed with it.
    # No LineItem exists outside of an Order.
    del order
```
In this example, the `Order` class composes multiple `LineItem` objects. The `Order` is responsible for creating and managing its line items. When an order is created, it can add line items through the `add_item` method, which creates new `LineItem` instances. If the order is deleted, all of its line items are also destroyed, demonstrating a composition relationship where the parts (line items) cannot exist independently of the whole (order). The association is established through an attribute in the `Order` class that holds a list of `LineItem` instances, indicating that the line items are integral parts of the order's existence.

4. **Dependency:** A relationship where one class depends on another class to function. It is a temporary relationship where one class uses another class for a specific purpose, such as calling a method or accessing an attribute.

```python
class Document:
    def __init__(self, content):
        self.content = content
class Printer:
    def print_document(self, document: Document):
        print(f"Printing document with content: {document.content}")
# Dependency: Printer depends on Document to print
printer = Printer()
doc = Document("Hello, World!")
printer.print_document(doc)  # printer depends on doc to print
```
In this example, the `Printer` class depends on the `Document` class to function. The `Printer` uses the `Document` instance to access its content and print it. This is a dependency relationship because the `Printer` relies on the `Document` to perform its task, but the `Document` can exist independently of the `Printer`. The association is established through a method parameter in the `print_document` method, where the `Printer` class takes a `Document` instance as an argument to perform its function, indicating that the `Printer` depends on the `Document` for its operation.

Pay attention to what makes this a dependency and not an association:

- Printer has no fields referencing Document. The Printer class has zero instance variables pointing to Document. The document exists only as a parameter inside the print() method.
- The relationship is scoped to one method call. Once print() returns, the Printer object has no knowledge that a Document was ever involved.
- The Document is created externally and passed in. The Printer doesn't construct, own, or manage the document's lifecycle.
- If the Printer stored a Document reference as a field (private Document lastPrinted), it would become an association. The structural link would persist beyond the method call.


**Practical Example: Event Ticketing System**

Let's put dependency into practice with a realistic scenario. A TicketBookingService handles the complete flow of booking an event ticket. During the bookTicket() method, it needs to validate that seats are available, process a payment, generate a QR code for the ticket, and send a confirmation email. Each of these responsibilities belongs to a separate class, and the booking service depends on all four of them, but only during the booking method.

```python
class SeatValidator:
    def is_available(self, event_id, seat_number):
        print(f"Checking seat {seat_number} for event {event_id}")
        return True  # Simulated: seat is available

class PaymentProcessor:
    def charge(self, email, amount):
        print(f"Charging ${amount} to {email}")
        return True  # Simulated: payment succeeds

class QRCodeGenerator:
    def generate(self, event_id, seat_number):
        qr_code = f"QR-{event_id}-{seat_number}"
        print(f"Generated QR code: {qr_code}")
        return qr_code

class EmailService:
    def send_confirmation(self, email, qr_code):
        print(f"Sending confirmation to {email} with code {qr_code}")

class TicketBookingService:
    def book_ticket(self, event_id, seat_number, email, amount,
                    validator, payment, qr_generator, email_service):
        if not validator.is_available(event_id, seat_number):
            print("Seat not available.")
            return False

        if not payment.charge(email, amount):
            print("Payment failed.")
            return False

        qr_code = qr_generator.generate(event_id, seat_number)
        email_service.send_confirmation(email, qr_code)

        print("Booking confirmed!")
        return True
		
if __name__ == "__main__":
    booking_service = TicketBookingService()

    # All dependencies are created externally and passed in
    validator = SeatValidator()
    payment = PaymentProcessor()
    qr_generator = QRCodeGenerator()
    email_service = EmailService()

    booking_service.book_ticket("CONF-2025", "A12", "alice@example.com",
        99.99, validator, payment, qr_generator, email_service)		
```
In this example, the `TicketBookingService` class depends on four other classes: `SeatValidator`, `PaymentProcessor`, `QRCodeGenerator`, and `EmailService`. The booking service relies on these classes to perform its function of booking a ticket, but it only interacts with them during the execution of the `book_ticket` method. Each of these classes is responsible for a specific aspect of the booking process, and the `TicketBookingService` uses them as needed without maintaining any long-term association with them. This demonstrates a dependency relationship where the `TicketBookingService` depends on other classes to complete its task, but those classes can exist independently and are not owned or managed by the booking service.

Why This Design Works
- All dependencies are method parameters. TicketBookingService has zero fields. Every collaborator comes in through bookTicket() and disappears when the method returns. This is pure dependency with no structural coupling.
- Each class has a single responsibility. SeatValidator validates seats. PaymentProcessor handles payments. QRCodeGenerator generates codes. EmailService sends emails. The booking service just coordinates the flow.
- Testing is straightforward. You can pass mock implementations of any dependency without touching the booking service. Want to test what happens when payment fails? Pass a mock - - PaymentProcessor that returns false.
- Swapping implementations is trivial. Need to switch from email to SMS notifications? Pass an SmsService instead of EmailService. The booking service doesn't care, it just calls the method on whatever it receives.



5. **Realization:** A relationship where one class implements an interface defined by another class. It is a relationship between an interface and a class that implements that interface.

```python
from abc import ABC, abstractmethod
class Shape(ABC):
    @abstractmethod
    def area(self):
        pass
class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius
    def area(self):
        return 3.14 * self.radius ** 2
# Realization: Circle realizes the Shape interface by implementing the area() method
circle = Circle(5)
print(f"The area of the circle is: {circle.area()}")
```
In this example, the `Circle` class realizes the `Shape` interface by implementing the `area()` method defined in the `Shape` abstract base class. The `Shape` class defines a contract that any class that realizes it must implement the `area()` method. The `Circle` class fulfills this contract by providing its own implementation of the `area()` method, allowing it to be treated as a `Shape` while still providing specific behavior for calculating the area of a circle. This is a realization relationship because the `Circle` class is realizing the interface defined by the `Shape` class

**Practical Example:**

Let's implement the Flyable scenario. Three completely unrelated classes, Bird, Airplane, and Drone, all realize the same Flyable interface. Each has different internal state and different behavior, but they all fulfill the same contract.

```python
from abc import ABC, abstractmethod

class Flyable(ABC):
    @abstractmethod
    def fly(self):
        pass

    @abstractmethod
    def get_flight_info(self):
        pass
		
class Bird(Flyable):
    def __init__(self, species, wing_span):
        self.species = species
        self.wing_span = wing_span

    def fly(self):
        print(f"{self.species} flaps its wings and takes off.")

    def get_flight_info(self):
        return f"{self.species} (wingspan: {self.wing_span}m, powered by muscle)"


class Airplane(Flyable):
    def __init__(self, model, max_altitude):
        self.model = model
        self.max_altitude = max_altitude

    def fly(self):
        print(f"{self.model} engines roar as it accelerates down the runway.")

    def get_flight_info(self):
        return f"{self.model} (max altitude: {self.max_altitude}ft, powered by jet engines)"


class Drone(Flyable):
    def __init__(self, battery_level, max_range):
        self.battery_level = battery_level
        self.max_range = max_range

    def fly(self):
        print(f"Drone propellers spin up. Battery at {self.battery_level}%.")

    def get_flight_info(self):
        return f"Drone (range: {self.max_range}km, battery: {self.battery_level}%)"
		
if __name__ == "__main__":
    flying_things = [
        Bird("Eagle", 2.3),
        Airplane("Boeing 737", 41000),
        Drone(85, 10.0),
    ]

    for flyer in flying_things:
        print(flyer.get_flight_info())
        flyer.fly()
        print()		
```
In this example, the `Bird`, `Airplane`, and `Drone` classes all realize the `Flyable` interface by implementing the `fly()` and `get_flight_info()` methods. Each class has its own unique implementation of these methods, reflecting their different internal states and behaviors. However, they all fulfill the same contract defined by the `Flyable` interface, allowing them to be treated polymorphically as flyable entities. This demonstrates a realization relationship where multiple classes can implement the same interface while providing their own specific behavior.

Pay attention to four things that make this realization:

- The interface defines the contract, not the implementation. Flyable declares fly() and getFlightInfo(), but provides zero code. Each class writes its own version from scratch. This is fundamentally different from inheritance, where the child gets the parent's code for free.
- The three classes are completely unrelated. Bird, Airplane, and Drone share no parent class, no common fields, no shared behavior. A bird has a wingspan and species. An airplane has a model and altitude. A drone has a battery and range. They have nothing in common except the Flyable contract.
- Calling code depends only on the interface. The main method works with a List<Flyable>. It doesn't know or care what's in the list. It could be three birds, three airplanes, or a mix. The code is the same either way.
- Adding a new flying thing requires zero changes to existing code. Want to add a Helicopter? Write one new class that implements Flyable, add it to the list, and everything works. No existing class needs to be modified.
   
 