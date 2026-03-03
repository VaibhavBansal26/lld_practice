# 3. Class Relationships in OOP

1. **Association:** A relationship where one class uses another class. It can be one-to-one, one-to-many, or many-to-many. The classes are independent of each other.

1. one-to-one: A single instance of one class is associated with a single instance of another class.
```python
class Person:
    def __init__(self, name):
        self.name = name
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
how will it show output? It will not show any output unless we print the attributes. but why we have not reference the passport inside 


2. one-to-many: A single instance of one class is associated with multiple instances of another
3. many-to-many: Multiple instances of one class are associated with multiple instances of another class.


4. **Aggregation:** A special form of association where one class is a part of another class. The part can exist independently of the whole. It represents a "has-a" relationship.
5. **Composition:** A stronger form of aggregation where the part cannot exist independently of the whole. If the whole is destroyed, the part is also destroyed. It also represents a "has-a" relationship but with stronger ownership.
6. **Dependency:** A relationship where one class depends on another class to function. It is a temporary relationship where one class uses another class for a specific purpose, such as calling a method or accessing an attribute.
 