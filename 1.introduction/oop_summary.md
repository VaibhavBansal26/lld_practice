# Summary


```
Abstraction → WHAT is required
Polymorphism → HOW it happens differently
Encapsulation → WHO can access data
Inheritance → WHO gets features
```


**Encapsulation**

Encapsulation means hiding internal data and allowing access only through controlled methods.

Internal variables should not be accessed directly. Instead, public methods are used to read or modify the data safely.

Purpose:
- Protect data
- Maintain data integrity
- Control how data changes

Example idea:

```
private data → accessed using getters/setters

```

**Inheritance**
Inheritance allows a subclass to reuse properties and behaviors of a parent class.

The child class automatically gets attributes and methods from the parent class.

Common types of inheritance:
- Single inheritance
- Multilevel inheritance
- Hierarchical inheritance

Purpose:

- Code reuse
- Logical relationship between classes
- Easier maintenance

**3. Abstraction**

Abstraction means defining the structure or behavior of a class without implementing all the details.

An abstract class defines methods (interfaces) that subclasses must implement.

It focuses on what an object should do, not how it does it.

Different subclasses can provide different implementations for the same abstract method.

**4. Polymorphism**

Polymorphism means “many forms”.

A single method name can behave differently depending on the object using it.

Two styles (general OOP concept):

- Compile time polymorphism (method overloading)

- Runtime polymorphism (method overriding)

In Python:

- Python is dynamically typed.
- True compile time polymorphism does not exist.
- Polymorphism mainly happens through runtime method overriding and duck typing.
- Different subclasses implement the same method differently based on context.

**Relationship Between Abstraction and Polymorphism**

Abstraction and polymorphism work together:

- Abstraction defines a common method that all subclasses must implement.
- Polymorphism allows each subclass to implement that method differently.
- Method behavior is decided at runtime.

**⭐ Cleaner Interview Version (Short)**

1. Encapsulation: hides data and controls access using methods.
2. Inheritance: allows subclasses to reuse parent class properties.
3. Abstraction: defines required behaviors without implementation details.
4. Polymorphism: allows the same method to behave differently for different objects.