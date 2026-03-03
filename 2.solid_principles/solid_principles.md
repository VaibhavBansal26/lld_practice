# 1. SOLID Principles

The SOLID principles are a set of five design principles that `help developers write maintainable and scalable software`. They were introduced by Robert C. Martin (Uncle Bob) and are widely used in object-oriented programming.

1. **Single Responsibility Principle (SRP):** A class should have only one reason to change, meaning it should have only one job or responsibility.
2. **Open/Closed Principle (OCP):** Software entities (classes, modules, functions) should be open for extension but closed for modification. This promotes the idea of writing code that can be easily extended without modifying existing code, which can help prevent bugs and improve maintainability.
3. **Liskov Substitution Principle (LSP):** Objects of a superclass should be replaceable with objects of its subclasses without breaking the application.
4. **Interface Segregation Principle (ISP):** Clients should not be forced to depend on interfaces they do not use.
5. **Dependency Inversion Principle (DIP):** High-level modules should not depend on low-level modules. Both should depend on abstractions. This promotes decoupling between components and allows for more flexible and maintainable code.

These principles help in creating robust and flexible software systems that are easier to maintain and extend over time.

**Solid Principles Discussion**

**1. Single Responsibility Principle (SRP)**
   
```text
A class should have a single, well-defined responsibility or task within a software system.
```

```python
class Employee:
    def __init__(self, name, salary):
        self._name = name        # protected (Python convention)
        self._salary = salary

    def calculate_salary(self):
        return self._salary * 12   # annual salary

    def generate_payroll_report(self):
        print(f"Payroll Report for {self._name}: ${self._salary * 12}")


# Example usage
emp = Employee("John", 5000)

print(emp.calculate_salary())
emp.generate_payroll_report()

```
The employee has two responsibilities: calculating salary and generating payroll reports. 
This means class can change for two reasons: if the salary calculation logic changes or if the report generation format changes.

**Fixing the violations**

To address the violation, let's refactor the code to separate concerns and ensure that each class has a single, well-defined responsibility. We'll create distinct classes for calculating an employee's salary and generating a payroll report:

- The Employee class manages employee data, such as name and salary, and calculates the annual salary based on the monthly salary.
- The PayrollReportGenerator class takes an employee’s data and produces payroll reports.

This separation ensures that each class has a single task, and changes to salary calculations won’t affect reporting, and updates to report formats won’t impact employee data, making the system easier to maintain.

Best practices
- To adhere to the SRP effectively, consider the following guidelines:
- Aim to define a clear role for each class, focusing on one specific task.
- If a class handles multiple tasks, refactor it into smaller, focused classes with single responsibilities.
- Design classes so that changes to one task don’t impact others.

```python
class Employee: 
    def __init__(self, name, salary): 
        self._name = name 
        self._salary = salary 

    def calculate_annual_salary(self): 
        return self._salary * 12

class PayrollReportGenerator: 
    def generate_report(self, employee): 
        annual_salary = employee.calculate_annual_salary() 
        print(f"Payroll Report for {employee._name}: ${annual_salary}")
```
This refactored design adheres to the Single Responsibility Principle by ensuring that the Employee class is solely responsible for managing employee data and calculating the annual salary, while the PayrollReportGenerator class is responsible for generating payroll reports. This separation of concerns makes the code more maintainable and easier to extend in the future.

**2. Open/Closed Principle (OCP)**

The Open/Closed Principle states that software entities (classes, modules, functions) should be open for extension but closed for modification. This means that you should be able to add new functionality to a system without changing existing code, which helps prevent bugs and maintain the integrity of the system.

**Violation of OCP**

```python
class Rectangle:
    def __init__(self, width, height):
        self._width = width     # protected convention
        self._height = height

    def calculate_area(self):
        return self._width * self._height


class AreaCalculator:
    def calculate_area(self, rectangle):
        return rectangle.calculate_area()


# Example usage
rect = Rectangle(5, 10)
calculator = AreaCalculator()

print(calculator.calculate_area(rect))
```
In the above example, the AreaCalculator class works only with Rectangle objects. Adding support for new shapes, like circles or triangles, would require modifying its code. This makes the system harder to maintain and prone to errors, as each new shape requires altering the core logic.

```python
class Rectangle:
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def calculate_area(self):
        return self.width * self.height

class AreaCalculator:
    def calculate_total_area(self, shapes):
        total = 0
        for shape in shapes:
            if isinstance(shape, Rectangle):
                total += shape.calculate_area()
        return total
```

In this example, the `AreaCalculator` class is closed for modification but open for extension. If we want to add a new shape like a circle, we would create a new class `Circle` and modify the `AreaCalculator` to handle it. However, this violates the OCP because we are modifying the existing code to support new shapes.

To fix this violation, we can use polymorphism and make the `AreaCalculator` open for extension:

```python
class Shape:
    def calculate_area(self):
        raise NotImplementedError("Subclasses must implement this method")

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def calculate_area(self):
        return self.width * self.height

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius

    def calculate_area(self):
        import math
        return math.pi * self.radius ** 2

class AreaCalculator:
    def calculate_total_area(self, shapes):
        total = 0
        for shape in shapes:
            total += shape.calculate_area()
        return total
```

**Fixing the violation**

- We can refactor this design to support new shapes without changes by introducing an abstract Shape class that defines a common behavior for all shapes: calculating their area.
- Specific shapes, like Rectangle and Circle, inherit from Shape and provide their area calculations.
  
This design allows new shapes, such as triangles, to be added by creating new classes that inherit from Shape, without modifying AreaCalculator or existing shape classes.


**Best practices**
With this extensible design in place, here are guidelines to ensure OCP compliance:

- Consider introducing abstract classes or interfaces to create flexible blueprints that classes can extend with new functionality.
- Allow subclasses to override methods to provide specific behaviors, such as unique area calculations for different shapes.
- Use polymorphism to treat objects of different classes, like various shapes, uniformly through a common interface or base class.

**3. Liskov Substitution Principle (LSP)**

The Liskov Substitution Principle (LSP) states that objects of a derived class should be able to replace objects of the base class without affecting the correctness of the program. In other words, if class A is a subtype of class B, then instances of class B should be replaceable with instances of class A without causing issues.

Violation of LSP
```python
class Bird:
    def fly(self):
        print("Flying")
class Sparrow(Bird):
    def fly(self):
        print("Sparrow flying")
class Ostrich(Bird):
    def fly(self):
        raise NotImplementedError("Ostriches can't fly")
```
In this example, the `Ostrich` class violates the Liskov Substitution Principle because it cannot be substituted for the `Bird` class without causing a runtime error. If we have code that expects a `Bird` object and we pass an `Ostrich` object, it will raise an exception when trying to call the `fly` method.

In this code, the Ostrich class inherits from Bird but throws an exception for fly, as ostriches cannot fly. This breaks the expectation that any Bird can fly when a program tests this behavior. This violates the principle that derived classes should behave as expected when replacing their base class, making the design unreliable.

fixing the violation

To align with LSP, we can refactor this hierarchy to ensure substitutability:

- Instead of assuming all birds can fly, we redefine the Bird class as an abstract class with a more general behavior, such as moving, that all birds can perform.
- The Bird class defines a move method, which each bird implements according to its abilities. For example, a Sparrow might implement move by describing flying in the sky, while an Ostrich implements move by describing running on land.
- This design ensures that any derived class, like Ostrich, can replace Bird seamlessly, maintaining the program’s correctness.

```python
class Bird:
    def move(self):
        raise NotImplementedError("Subclasses must implement this method")
class Sparrow(Bird):
    def move(self):
        print("Sparrow flying")
class Ostrich(Bird):
    def move(self):
        print("Ostrich running")
```

Best practices

With this flexible design in place, here are the guidelines to ensure LSP compliance:

- Ensure that derived classes maintain the behavioral compatibility of their base classes. Methods in derived classes should follow the same contracts as the base class methods.
- When overriding methods, derived classes must respect the base class’s method contracts.      
  - Specifically:
    - Preconditions: Derived class methods should require the same or weaker preconditions (e.g., input constraints) than the base class method. This ensures that the derived class does not impose stricter requirements that could break client expectations.
    - Postconditions: Derived class methods must provide the same or stronger postconditions (e.g., output guarantees) as the base class method, ensuring that the method’s results meet or exceed the base class’s promises.
    - Invariants: Derived classes must maintain all invariants defined by the base class, ensuring that the object’s state remains valid according to the base class’s rules.
- Use polymorphism to allow derived class objects to replace base class objects, often by overriding methods to provide specialized behavior while maintaining the core functionality of the base class.

**4. Interface Segregation Principle (ISP)**

The “I” in the SOLID acronym stands for the Interface Segregation Principle (ISP), which emphasizes that clients (classes or components that use interfaces) should not be forced to depend on interfaces they don't use. In other words, an interface should have a specific and focused set of methods that are relevant to the implementing classes.

Violation of ISP

```python
class Worker:
    def work(self):
        print("Working")
    def eat(self):
        print("Eating")
    def sleep(self):
        print("Sleeping")

class Robot(Worker):
    def work(self):
        print("Robot working")
    def eat(self):
        raise NotImplementedError("Robots don't eat")
    def sleep(self):
        raise NotImplementedError("Robots don't sleep")

class Human(Worker):
    def work(self):
        print("Human working")
    def eat(self):
        print("Human eating")
    def sleep(self):
        print("Human sleeping")
```
In this code, the Worker interface forces Robot to implement eat and sleep, which are irrelevant, leading to unsupported operations. This makes the code harder to maintain and prone to errors, as classes must handle methods that don’t apply to them.


Fixing the violation

```python
class Workable:
    def work(self):
        raise NotImplementedError("Subclasses must implement this method")
class Eatable:
    def eat(self):
        raise NotImplementedError("Subclasses must implement this method")
class Sleepable:
    def sleep(self):
        raise NotImplementedError("Subclasses must implement this method")

class Robot(Workable):
    def work(self):
        print("Robot working")

class Human(Workable, Eatable, Sleepable):
    def work(self):
        print("Human working")
    def eat(self):
        print("Human eating")
    def sleep(self):
        print("Human sleeping")
```

To align with ISP, we can refactor this interface to be more focused:

- Instead of a single Worker interface with unrelated methods, we split it into three smaller, tailored interfaces: Workable, Eatable, and Sleepable.
- The Workable interface includes only the work method, which all workers, like robots and humans, can implement.
- The Eatable interface includes the eat method, relevant to humans but not robots.
- The Sleepable interface includes the sleep method, which is also specific to humans.

This design ensures classes implement only the methods they need, making the code cleaner and more maintainable.

Best Practices

Here are guidelines to ensure ISP compliance:

- Aim to design interfaces with a specific purpose, including only methods directly related to that purpose.
- Consider creating multiple smaller interfaces that classes can choose to implement, rather than a single large interface with unrelated methods.
- Think from the perspective of the classes implementing the interface, providing only the methods they require.



**5. Dependency Inversion Principle (DIP)**

The Dependency Inversion Principle (DIP) states that high-level modules (or classes) should not depend on low-level modules; both should depend on abstractions, such as interfaces. In simpler terms, the principle encourages the use of abstract interfaces to decouple higher-level components from lower-level details.

Violation of DIP

```python
class LightBulb:
    def turn_on(self):
        print("LightBulb is on")
    def turn_off(self):
        print("LightBulb is off")
class Switch:
    def __init__(self, bulb):
        self._bulb = bulb
    def operate(self, action):
        if action == "on":
            self._bulb.turn_on()
        elif action == "off":
            self._bulb.turn_off()
```

In this code, the Switch class directly depends on the LightBulb class. If we want
to use a different type of bulb, like an LED or a fluorescent bulb, we would need to modify the Switch class, which violates the Dependency Inversion Principle. This tight coupling makes the code less flexible and harder to maintain.

Fixing the violation

```python
class Switchable:
    def turn_on(self):
        raise NotImplementedError("Subclasses must implement this method")
    def turn_off(self):
        raise NotImplementedError("Subclasses must implement this method")
class LightBulb(Switchable):
    def turn_on(self):
        print("LightBulb is on")
    def turn_off(self):
        print("LightBulb is off")
class Fan(Switchable):
    def turn_on(self):
        print("Fan is on")
    def turn_off(self):
        print("Fan is off")
class Switch:
    def __init__(self, device):
        self._device = device
    def operate(self, action):
        if action == "on":
            self._device.turn_on()
        elif action == "off":
            self._device.turn_off()
```

Fixing the violation
To align with DIP, we can refactor this design to use abstractions:

- We introduce a Switchable interface that defines standard methods, such as turnOn and turnOff, which any switchable device can implement.
- The Switch class is modified to depend only on the Switchable interface, not on specific devices.
- The LightBulb class implements the Switchable interface, providing its turnOn and turnOff behavior.
- We can also add new devices, like a Fan class that implements Switchable, without modifying the Switch class.
  
Now, Switch can work with any device that implements Switchable, like a Fan or a Heater, without needing changes to its code. This design decouples Switch from low-level details, making the system more flexible and easier to extend

Best Practices
With this decoupled design in place, here are guidelines to ensure DIP compliance:

- Introduce interfaces or abstract classes to represent dependencies, allowing high-level modules to depend on these abstractions.
- Use dependency injection to inject concrete implementations into high-level modules through their abstractions. This promotes loose coupling.
