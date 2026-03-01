# OOP Fundamentals

Cornerstones of Object-Oriented Programming

1. **Encapsulation:** Bundling data and methods that operate on that data within a single unit (class) and restricting access to some of the object's components.
2. **Abstraction:** Hiding complex implementation details and showing only the necessary features of an object.
3. **Inheritance:** A mechanism where a new class (child) inherits properties and behaviors (attributes and methods) from an existing class (parent).
4. **Polymorphism:** The ability of different classes to be treated as instances of the same class through a common interface, often achieved through method overriding or method overloading.

Key Notes:
```text
- Encapsulation:
Protected data + controlled access.
- Abstraction:
Hiding complexity + exposing essential features.
- Inheritance:
Code reuse + hierarchical relationships.
- Polymorphism:
Same action, different behavior.
```

Other Important OOP Concepts:

1. **Composition:** Building complex objects by combining simpler ones, often using "has-a" relationships instead of "is-a" relationships (inheritance).
2. **SOLID Principles:** A set of design principles that promote maintainable and scalable software design, including Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion principles.
3. **Design Patterns:** Reusable solutions to common software design problems, such as Singleton, Factory, Observer, and Strategy patterns.


**1. Encapsulation**

Encapsulation means hiding internal details and allowing access only through controlled methods.

You bundle data + behavior together and prevent direct misuse.

🧠 Encapsulation = data hiding + controlled access

```text
Encapsulation is an object oriented principle where data and methods are bundled together and internal state is hidden from direct access. Users interact with the object through controlled methods, which protects data integrity, improves maintainability, and reduces unintended modifications.
```

```python
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

Why Do We Use Encapsulation?
1. Protect data
    - Prevent invalid values.
    - Control access to sensitive data.
2. Reduce bugs
    - Encapsulated code is easier to debug and maintain.
3. Easy maintenance
    - Changes to internal implementation do not affect external code.
4. Security
   - Hiding internal implementation details prevents unauthorized access and manipulation of data.

```python
class BankAccount:
    def __init__(self):
        self._account_type = "Savings"   # protected
        self.__balance = 1000            # private

    def show_balance(self):
        return self.__balance


class ChildAccount(BankAccount):
    def show(self):
        print(self._account_type)  # ✅ works
        # print(self.__balance)    # ❌ fails
```

**Where Encapsulation EXISTS**

Encapsulation exists when:

    - ✔ Data is hidden
    - ✔ Access happens via methods
    - ✔ Internal logic is protected

Examples:
- Classes with private variables
- APIs hiding implementation
- Database access layers
- Service layers in backend
- React components managing internal state

Encapsulation is not just private variables.

It also means:

- ✅ hiding complexity
- ✅ exposing simple interfaces
- ✅ separating WHAT from HOW

This is why:

- APIs
- Microservices
- SDKs
- React hooks
- all rely heavily on encapsulation.


**Common pitfalls**

While encapsulation is powerful, avoid these common mistakes:

- Over-encapsulation: Creating excessive getter and setter methods for every attribute can make code verbose and harder to maintain.
- Under-encapsulation: Failing to hide internal details can lead to tight coupling and reduced modularity. For instance, if name and age in the Person class were public, other parts of the code could modify them directly, leading to potential inconsistencies.

**Private vs Protected in Python**

```text
_protected  → "Please don't touch"
__private   → "Harder to touch accidentally"
```

```text
Protected variables use a single underscore and are accessible both outside the class and in subclasses by convention. Private variables use double underscores, triggering name mangling, which prevents direct access outside the class and hides them from subclasses unless accessed using the mangled name.
```

**2. Abstraction**

Abstraction is the process of hiding complex implementation details and showing only essential features of an object.

Abstraction can simplify complex systems by hiding unnecessary details. It separates the "what" an object does from the "how" it does it, enabling users to interact with objects through simplified interface without needing to understand the underlying complexity.

🧠 Abstraction = hiding complexity + exposing essential features

Abstraction means showing WHAT an object can do, but hiding HOW it does it.
In Abstraction, you define required behavior, not implementation.

```text
Abstraction is an object oriented principle where complex implementation details are hidden and only essential features are exposed to the user. It allows developers to focus on what an object does rather than how it does it, improving code readability and maintainability.
```

```
Abstraction is an object oriented concept where a base class defines a common interface or behavior without implementing full logic. In this example, the Shape class defines an area method but does not implement it, forcing subclasses like Rectangle and Circle to provide their own implementations. This allows different objects to be used uniformly while hiding implementation details.
```

```python
class Shape:
    def __init__(self, name): 
        # Every shape MUST have an area() method. 
        # But Shape itself does not know how to calculate area.
        self.name = name

    def area(self):
        # Why raise NotImplementedError instead of pass? This method must be overridden.”
        raise NotImplementedError("Subclass must implement abstract method")

class Rectangle(Shape): # Rectangle inherits the abstraction.
    def __init__(self, name, width, height):
        super().__init__(name)
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

class Circle(Shape):
    def __init__(self, name, radius):
        super().__init__(name)
        self.radius = radius

    def area(self):
        return 3.14 * self.radius * self.radius
```

`Class Shape` which represents a general concept of a shape.

All shapes:

- ✅ have a name
- ✅ can calculate area

But:

❌ area calculation is different for each shape.

So you force subclasses to implement it.

| Part                  | Why it's abstraction    |
| --------------------- | ----------------------- |
| `Shape` class         | Defines common behavior |
| `area()` method       | Contract/interface      |
| `NotImplementedError` | Forces implementation   |
| Subclasses            | Provide hidden logic    |




Why Do We Use Abstraction?
1. Simplify complexity
   - Hide implementation details from users.
2. Improve maintainability
   - Changes in internal logic don't affect external code.
3. Enhance reusability
   - Abstract interfaces can be reused across different implementations.

```python
class Animal:
    def __init__(self, name):
        self.name = name

    def make_sound(self):
        raise NotImplementedError("Subclass must implement abstract method")

class Dog(Animal):
    def make_sound(self):
        return f"{self.name} barks"

class Cat(Animal):
    def make_sound(self):
        return f"{self.name} meows"
```

**Where Abstraction EXISTS**

Abstraction exists when:

    - ✔ Complex logic is hidden
    - ✔ Simple interfaces are exposed
    - ✔ Implementation details are separated from usage

Examples:
- Abstract base classes in programming languages like Java and C#
- APIs that hide internal complexity (e.g., database connections)
- Frameworks that provide simplified interfaces for complex operations (e.g., Django ORM)

Abstraction is not just about hiding code.

It also means:

- ✅ simplifying complex systems
- ✅ providing clean interfaces
- ✅ separating concerns between design and implementation

This is why:

- APIs
- Frameworks
- SDKs
- all rely heavily on abstraction.

**Interview answer**

```
The Shape class acts as an abstract class that defines common behavior and method signatures but does not implement the logic. It raises an error to force subclasses to provide their own implementation. The methods defined in the abstract class serve as an interface or contract that subclasses must follow. 

This allows different shapes to be used interchangeably while hiding the specific implementation details of how each shape calculates its area. The abstraction provided by the Shape class simplifies the design and promotes code reuse, as new shapes can be added without modifying existing code.
```

Python example uses manual abstraction, not a formal abstract class yet.
Class `Shape` is called a pseudo abstract class. The official Python abstraction uses ABC.

```python
from abc import ABC, abstractmethod

class Shape(ABC):

    @abstractmethod
    def area(self):
        pass

# Now Python ENFORCES abstraction:
Shape()   # ❌ Cannot instantiate

```

Mental Note

```
Abstract Class
    ↓
Defines RULES

Subclass
    ↓
Implements RULES
```

Duck Typing (VERY Pythonic): Python cares about behavior, not inheritance.

Decision

```
Is this a framework or shared API?
        YES → use ABC

Is this internal logic or small system?
        YES → manual abstraction
```

**Encapsulation vs Abstraction**

```text
Abstraction answers: WHAT operations exist?
Encapsulation answers: HOW data is protected?
```


| Characteristics | Abstraction | Encapsulation |
|---|---|---|
| Focus | Hiding complexity by exposing only **what** an object does through simplified interfaces.<br><br>Does not reveal **how** it performs the operation internally. | Bundling data and methods into a single unit (a class).<br><br>Protects internal data by restricting direct access. |
| Purpose | Simplifies user interaction.<br><br>Promotes flexibility and extensibility by defining high level behaviors. | Ensures data integrity.<br><br>Improves maintainability by controlling how data is accessed and modified. |
| Implementation | Uses abstract classes and interfaces.<br><br>Example: `Shape` abstract class with `area()`.<br>Example: `Drawable` interface with `draw()`. | Uses access modifiers and controlled methods.<br><br>Example: private `radius` in `Circle` with public `getRadius()` and `setRadius()`. |



**3. Inheritance**

Inheritance allows a class (subclass or derived class) to inherit properties and behaviors from another class (superclass or base class). It promotes code reuse and creates a hierarchical relationship between classes. Think of inheritance like a family tree, where children inherit traits from their parents, and grandchildren inherit traits from both their parents and grandparents. The subclass can extend and specialize the functionality of its superclass, reducing code duplication.

Common patterns of class hierarchy

1. Single Inheritance: A subclass inherits from one superclass.

```text
The Dog, Cat class inherits the eat() method from Animal

Animal {eat(), sleep()}
   |
   ├── Dog
   └── Cat
```

2. Multilevel Inheritance: A subclass inherits from a superclass, which in turn inherits from another superclass.

```text
The Dog class inherits behavior from the Animal and Mammal classes. Multilevel inheritance is useful when you have classes that exhibit a hierarchical relationship, with each subclass specializing and adding new behavior to the existing hierarchy.

Animals {eat(), sleep()}
   |
   | extends
   ├── Mammal {run()}
   |     |
   |     └── Dog {bark()}
   |
   | extends
   └── Bird {fly()}
         |
         └── Sparrow
```

3. Hierarchical Inheritance: Multiple subclasses inherit from a single superclass.

```text

In hierarchical inheritance, multiple subclasses inherit from the same superclass, forming a hierarchical structure. In the example below, you can see two subclasses, Car and Motorcycle, both inherit from the Vehicle class, forming a hierarchical inheritance relationship.

Hierarchical inheritance is beneficial when multiple classes share common attributes or behaviors from a single superclass.

Vehicle {start(), stop()}
   |
   | extends
   ├── Car {drive()}
   |
   | extends
   ├── Motorcycle {ride()}
   |
   | extends
   └── Truck {haul()}
```

When to use inheritance?
Inheritance is particularly useful in the following scenarios:

- Whenever we encounter an 'is-a' relationship between objects, we can use inheritance.
- When multiple classes share common attributes or methods, a superclass can define them once, allowing all subclasses to inherit them and avoid duplication.
- When classes form a natural hierarchy, such as Animal being a parent to Dog and Cat, inheritance organizes the structure clearly.

Drawbacks of inheritance

While Inheritance promotes code reuse, its overuse can complicate designs. Here are the key drawbacks to consider:

- Tight coupling: Subclasses depend heavily on their superclass. Changes to the superclass, such as modifying the Animal class’s eat() method, can break subclasses like Dog or Cat, making the code harder to maintain.
- Inappropriate behavior inheritance: Inheritance can force subclasses to inherit behaviors that don’t apply. For example, adding a fly() method to the Animal superclass assumes all subclasses (e.g., Penguin) can fly, leading to errors or awkward workarounds like throwing exceptions.
- Limited flexibility: Inheritance `locks in relationships at design time`. If you later need a RobotDog that barks but doesn’t eat, it can’t inherit from Animal without inheriting irrelevant methods.

**Inheritance vs. Composition**

Given inheritance’s limitations, it’s helpful to compare it with composition, another way to structure classes. Inheritance creates an `“is-a” relationship`, where a subclass is a type of its superclass (e.g., Dog is an Animal). Composition creates a `“has-a” relationship`, where a class contains other objects to provide its behavior, such as installing apps on a phone for specific tasks.

Consider the Dog and RobotDog scenario. Using inheritance, RobotDog extends Animal to inherit bark(), but it also gets eat(), which doesn’t apply, causing issues like exceptions. Using composition, you define a BarkBehavior interface with a bark() method. Dog and RobotDog each have a BarkBehavior object, implemented differently (e.g., DogBark for “Woof!” and RobotBark for “Beep!”). This lets RobotDog bark without inheriting eat().

```python
class BarkBehavior:
    def bark(self):
        pass

class DogBark(BarkBehavior):
    def bark(self):
        return "Woof!"

class RobotBark(BarkBehavior):
    def bark(self):
        return "Beep!"

class Dog:
    def __init__(self):
        self.bark_behavior = DogBark()
    def perform_bark(self):
        return self.bark_behavior.bark()

class RobotDog:
    def __init__(self):
        self.bark_behavior = RobotBark()
    def perform_bark(self):
        return self.bark_behavior.bark()

dog = Dog(DogBark())
robot_dog = RobotDog(RobotBark())

dog.perform_bark()      # Woof!
robot_dog.perform_bark()  # Beep!
```


When to use composition?
To choose between inheritance and composition, follow these guidelines:

- Design Choice: Use inheritance for clear “is-a” relationships with stable, shared behaviors. Choose composition for “has-a” relationships or when you need flexible, swappable behaviors, as it’s easier to modify and maintain.
- Interview Strategy: In OOD interviews, favor composition when flexibility or loose coupling is key, as it’s preferred in modern design. For example, explain how RobotDog uses BarkBehavior to avoid inheritance’s tight coupling.


**4. Polymorphism**

Polymorphism is the concept of implementing objects that can take on multiple forms or behave differently depending on their context, all within a common interface. It provides the flexibility to add new behaviors without modifying existing code.

```
Polymorphism means different objects can use the same method name but perform different actions depending on the object.
```

```
👉 Same action, different behavior.

Same method name ✅
Different behavior ✅
```

Consider a media player as a real-world example. Different types of media, such as audio, video, and streaming content, would be played on the same rendering widget and controlled by the same “play” button. But they require different internal processing and rendering logic. The user only interacts with a uniform interface, while polymorphic behavior manages the varying objects.


Types of Polymorphism
Polymorphism in object-oriented programming is typically categorized into two main types: 

1. Compile-Time polymorphism via method overloading

Python:
- is dynamically typed
- has no compilation phase deciding types
- types are known only when program runs

```python

```

Can Python Simulate Overloading?

Yes — using type checking.

```python
class MathOperations:

    def add(self, a, b):
        if isinstance(a, str):
            return a + b
        elif isinstance(a, float):
            return a + b
        else:
            return a + b
```

Operator overloading + duck typing replaces compile time polymorphism.


```
In Java, method overloading is compile time polymorphism because the compiler determines which method to call based on parameter types before execution. Python does not support true compile time polymorphism since it is dynamically typed; instead, the same behavior is achieved through runtime polymorphism where method behavior is decided during execution.
```

2. Runtime polymorphism via method overriding

```python
class Animal:
    def make_sound(self):
        raise NotImplementedError("Subclass must implement abstract method")
class Dog(Animal):
    def make_sound(self):
        return "Woof!"
class Cat(Animal):
    def make_sound(self):
        return "Meow!"

# Java requires type declaration.
# Python does not.
a1 = Dog()
a2 = Cat()
print(a1.make_sound())  # Woof!
print(a2.make_sound())  # Meow!
```


When to use polymorphism?
Polymorphism is particularly valuable in the following scenarios:

- Shared Interface: When multiple classes need to perform the same action in different ways, such as a play method for various media types (e.g., audio, video). Interfaces or superclasses ensure a consistent contract across implementations.
- Extensibility: When designing systems that need to accommodate new classes without modifying existing code. For example, adding a new media type to a player only requires implementing the existing play interface, preserving system stability.
- Customization: When subclasses need to tailor the behavior of inherited methods. For instance, a Dog barking differently from a Cat uses method overriding to provide specific implementations while adhering to the Animal interface.


**Abstraction vs Polymorphism** 

```text
Abstraction focuses on hiding complexity and exposing essential features, while polymorphism allows objects of different classes to be treated as instances of the same class through a common interface. Abstraction defines what an object does, while polymorphism defines how different objects can perform the same action in their own way.

Abstraction: It defines what you can do.
```

Abstraction → defines common method
Polymorphism → allows different implementations

        Shape
        area()        ← Abstraction (rule)
           |
    -------------------
    |                 |
 Rectangle          Circle
 area()             area()
 (different logic)  (different logic)

          ↑
     Polymorphism

```
Abstraction defines a common method like area() that all shapes must implement, while polymorphism allows different shapes to implement that method differently. Abstraction creates the interface, and polymorphism provides multiple behaviors using that interface.
```

```
Abstraction → WHAT is required
Polymorphism → HOW it happens differently
Encapsulation → WHO can access data
Inheritance → WHO gets features
```
