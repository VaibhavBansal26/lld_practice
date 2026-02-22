# OOP Fundamentals

Cornerstones of Object-Oriented Programming

1. **Encapsulation:** Bundling data and methods that operate on that data within a single unit (class) and restricting access to some of the object's components.
2. **Abstraction:** Hiding complex implementation details and showing only the necessary features of an object.
3. **Inheritance:** A mechanism where a new class (child) inherits properties and behaviors (attributes and methods) from an existing class (parent).
4. **Polymorphism:** The ability of different classes to be treated as instances of the same class through a common interface, often achieved through method overriding or method overloading.

Other Important OOP Concepts:

1. **Composition:** Building complex objects by combining simpler ones, often using "has-a" relationships instead of "is-a" relationships (inheritance).
2. **SOLID Principles:** A set of design principles that promote maintainable and scalable software design, including Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion principles.
3. **Design Patterns:** Reusable solutions to common software design problems, such as Singleton, Factory, Observer, and Strategy patterns.


**Encapsulation**

``python
class Car:
    def __init__(self, make, model):
        self.make = make
        self.model = model
        self._engine_status = 'off'  # Private attribute    
    def start_engine(self):
        self._engine_status = 'on'
    def stop_engine(self):
        self._engine_status = 'off'
    def get_engine_status(self):
        return self._engine_status
    def __str__(self):
        return f"{self.make} {self.model}: Engine is {self._engine_status}"
```