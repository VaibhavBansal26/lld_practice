# 4. Unix File System

# Code

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Generic, List, Optional, Set, TypeVar
import re


# =========================
# File Attribute
# =========================

class FileAttribute(Enum):
    IS_DIRECTORY = "IS_DIRECTORY"
    SIZE = "SIZE"
    OWNER = "OWNER"
    FILENAME = "FILENAME"


# =========================
# File
# =========================

@dataclass(eq=False)
class File:
    is_directory: bool
    size: int
    owner: str
    filename: str
    entries: Set["File"] = field(default_factory=set)

    def extract(self, attribute_name: FileAttribute) -> Any:
        if attribute_name == FileAttribute.SIZE:
            return self.size
        if attribute_name == FileAttribute.OWNER:
            return self.owner
        if attribute_name == FileAttribute.IS_DIRECTORY:
            return self.is_directory
        if attribute_name == FileAttribute.FILENAME:
            return self.filename
        raise ValueError("invalid filter criteria type")

    def add_entry(self, entry: "File") -> None:
        self.entries.add(entry)

    def get_entries(self) -> Set["File"]:
        return self.entries


# =========================
# Predicate
# =========================

class Predicate(ABC):
    @abstractmethod
    def is_match(self, input_file: File) -> bool:
        pass


# =========================
# Comparison Operators
# =========================

T = TypeVar("T")


class ComparisonOperator(ABC, Generic[T]):
    @abstractmethod
    def is_match(self, attribute_value: T, expected_value: T) -> bool:
        pass


class EqualsOperator(ComparisonOperator[T]):
    def is_match(self, attribute_value: T, expected_value: T) -> bool:
        return attribute_value == expected_value


class GreaterThanOperator(ComparisonOperator[float]):
    def is_match(self, attribute_value: float, expected_value: float) -> bool:
        return float(attribute_value) > float(expected_value)


class LessThanOperator(ComparisonOperator[float]):
    def is_match(self, attribute_value: float, expected_value: float) -> bool:
        return float(attribute_value) < float(expected_value)


class RegexMatchOperator(ComparisonOperator[str]):
    def is_match(self, attribute_value: str, expected_value: str) -> bool:
        return re.fullmatch(expected_value, attribute_value) is not None


# =========================
# Simple Predicate
# =========================

class SimplePredicate(Predicate, Generic[T]):
    def __init__(
        self,
        attribute_name: FileAttribute,
        operator: ComparisonOperator[T],
        expected_value: T,
    ):
        self.attribute_name = attribute_name
        self.operator = operator
        self.expected_value = expected_value

    def is_match(self, input_file: File) -> bool:
        actual_value = input_file.extract(self.attribute_name)

        if isinstance(actual_value, type(self.expected_value)):
            return self.operator.is_match(actual_value, self.expected_value)

        # optional support for int/float cross comparison
        if isinstance(actual_value, (int, float)) and isinstance(self.expected_value, (int, float)):
            return self.operator.is_match(actual_value, self.expected_value)

        return False


# =========================
# Composite Predicate
# =========================

class CompositePredicate(Predicate):
    pass


class AndPredicate(CompositePredicate):
    def __init__(self, operands: List[Predicate]):
        self.operands = operands

    def is_match(self, input_file: File) -> bool:
        return all(predicate.is_match(input_file) for predicate in self.operands)


class OrPredicate(CompositePredicate):
    def __init__(self, operands: List[Predicate]):
        self.operands = operands

    def is_match(self, input_file: File) -> bool:
        return any(predicate.is_match(input_file) for predicate in self.operands)


class NotPredicate(CompositePredicate):
    def __init__(self, operand: Predicate):
        self.operand = operand

    def is_match(self, input_file: File) -> bool:
        return not self.operand.is_match(input_file)


# =========================
# File Search Criteria
# =========================

class FileSearchCriteria:
    def __init__(self, predicate: Predicate):
        self.predicate = predicate

    def is_match(self, input_file: File) -> bool:
        return self.predicate.is_match(input_file)


# =========================
# File Search
# =========================

class FileSearch:
    def search(self, root: File, criteria: FileSearchCriteria) -> List[File]:
        result: List[File] = []
        recursion_stack: List[File] = [root]

        while recursion_stack:
            next_file = recursion_stack.pop()

            if criteria.is_match(next_file):
                result.append(next_file)

            for entry in next_file.get_entries():
                recursion_stack.append(entry)

        return result


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    # create files and directories
    root = File(is_directory=True, size=0, owner="root", filename="root")
    logs = File(is_directory=True, size=0, owner="admin", filename="logs")
    src = File(is_directory=True, size=0, owner="alice", filename="src")

    file1 = File(is_directory=False, size=5, owner="bob", filename="notes.txt")
    file2 = File(is_directory=False, size=20, owner="bob", filename="server.log")
    file3 = File(is_directory=False, size=50, owner="alice", filename="app.py")
    file4 = File(is_directory=False, size=100, owner="bob", filename="error.log")

    root.add_entry(logs)
    root.add_entry(src)
    root.add_entry(file1)
    logs.add_entry(file2)
    logs.add_entry(file4)
    src.add_entry(file3)

    # condition: size > 10 AND owner == "bob"
    size_predicate = SimplePredicate(FileAttribute.SIZE, GreaterThanOperator(), 10)
    owner_predicate = SimplePredicate(FileAttribute.OWNER, EqualsOperator(), "bob")

    combined = AndPredicate([size_predicate, owner_predicate])
    criteria = FileSearchCriteria(combined)

    search_engine = FileSearch()
    matches = search_engine.search(root, criteria)

    print("Files matching size > 10 AND owner == 'bob':")
    for f in matches:
        print(f.filename)

    # condition: filename matches .*\.log
    regex_predicate = SimplePredicate(
        FileAttribute.FILENAME,
        RegexMatchOperator(),
        r".*\.log"
    )
    regex_criteria = FileSearchCriteria(regex_predicate)
    log_matches = search_engine.search(root, regex_criteria)

    print("\nFiles matching regex .*\\.log:")
    for f in log_matches:
        print(f.filename)
```

**DEEP DIVE**

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
from typing import Any, Generic, List, Optional, Set, TypeVar
import re


# =========================
# File Attribute
# =========================

class FileAttribute(Enum):
    IS_DIRECTORY = "IS_DIRECTORY"
    SIZE = "SIZE"
    OWNER = "OWNER"
    FILENAME = "FILENAME"
    EXTENSION = "EXTENSION"
    MODIFIED_AT = "MODIFIED_AT"


# =========================
# File
# =========================

@dataclass(eq=False)
class File:
    is_directory: bool
    size: int
    owner: str
    filename: str
    modified_at: datetime
    entries: Set["File"] = field(default_factory=set)

    def extract(self, attribute_name: FileAttribute) -> Any:
        if attribute_name == FileAttribute.IS_DIRECTORY:
            return self.is_directory
        if attribute_name == FileAttribute.SIZE:
            return self.size
        if attribute_name == FileAttribute.OWNER:
            return self.owner
        if attribute_name == FileAttribute.FILENAME:
            return self.filename
        if attribute_name == FileAttribute.EXTENSION:
            return self.get_extension()
        if attribute_name == FileAttribute.MODIFIED_AT:
            return self.modified_at
        raise ValueError("invalid filter criteria type")

    def add_entry(self, entry: "File") -> None:
        if not self.is_directory:
            raise ValueError("Cannot add entries to a non-directory file.")
        self.entries.add(entry)

    def get_entries(self) -> Set["File"]:
        return self.entries

    def get_extension(self) -> str:
        if self.is_directory:
            return ""
        if "." not in self.filename:
            return ""
        return self.filename.rsplit(".", 1)[1]


# =========================
# Predicate
# =========================

class Predicate(ABC):
    @abstractmethod
    def is_match(self, input_file: File) -> bool:
        pass


# =========================
# Comparison Operators
# =========================

T = TypeVar("T")


class ComparisonOperator(ABC, Generic[T]):
    @abstractmethod
    def is_match(self, attribute_value: T, expected_value: T) -> bool:
        pass


class EqualsOperator(ComparisonOperator[T]):
    def is_match(self, attribute_value: T, expected_value: T) -> bool:
        return attribute_value == expected_value


class GreaterThanOperator(ComparisonOperator[T]):
    def is_match(self, attribute_value: T, expected_value: T) -> bool:
        return attribute_value > expected_value


class LessThanOperator(ComparisonOperator[T]):
    def is_match(self, attribute_value: T, expected_value: T) -> bool:
        return attribute_value < expected_value


class RegexMatchOperator(ComparisonOperator[str]):
    def is_match(self, attribute_value: str, expected_value: str) -> bool:
        return re.fullmatch(expected_value, attribute_value) is not None


class ContainsOperator(ComparisonOperator[str]):
    def is_match(self, attribute_value: str, expected_value: str) -> bool:
        return expected_value in attribute_value


class InOperator(ComparisonOperator[T]):
    def is_match(self, attribute_value: T, expected_value: set[T]) -> bool:
        return attribute_value in expected_value


# =========================
# Simple Predicate
# =========================

class SimplePredicate(Predicate, Generic[T]):
    def __init__(
        self,
        attribute_name: FileAttribute,
        operator: ComparisonOperator,
        expected_value: T,
    ):
        self.attribute_name = attribute_name
        self.operator = operator
        self.expected_value = expected_value

    def is_match(self, input_file: File) -> bool:
        actual_value = input_file.extract(self.attribute_name)

        if actual_value is None:
            return False

        if isinstance(actual_value, (int, float)) and isinstance(self.expected_value, (int, float)):
            return self.operator.is_match(actual_value, self.expected_value)

        if isinstance(actual_value, datetime) and isinstance(self.expected_value, datetime):
            return self.operator.is_match(actual_value, self.expected_value)

        if isinstance(self.expected_value, set):
            return self.operator.is_match(actual_value, self.expected_value)

        if isinstance(actual_value, type(self.expected_value)):
            return self.operator.is_match(actual_value, self.expected_value)

        return False


# =========================
# Composite Predicate
# =========================

class CompositePredicate(Predicate):
    pass


class AndPredicate(CompositePredicate):
    def __init__(self, operands: List[Predicate]):
        self.operands = operands

    def is_match(self, input_file: File) -> bool:
        return all(predicate.is_match(input_file) for predicate in self.operands)


class OrPredicate(CompositePredicate):
    def __init__(self, operands: List[Predicate]):
        self.operands = operands

    def is_match(self, input_file: File) -> bool:
        return any(predicate.is_match(input_file) for predicate in self.operands)


class NotPredicate(CompositePredicate):
    def __init__(self, operand: Predicate):
        self.operand = operand

    def is_match(self, input_file: File) -> bool:
        return not self.operand.is_match(input_file)


# =========================
# File Search Criteria
# =========================

class FileSearchCriteria:
    def __init__(self, predicate: Predicate):
        self.predicate = predicate

    def is_match(self, input_file: File) -> bool:
        return self.predicate.is_match(input_file)


# =========================
# File Search
# =========================

class FileSearch:
    def search(self, root: File, criteria: FileSearchCriteria) -> List[File]:
        result: List[File] = []
        stack: List[File] = [root]

        while stack:
            current = stack.pop()

            if criteria.is_match(current):
                result.append(current)

            if current.is_directory:
                for entry in current.get_entries():
                    stack.append(entry)

        return result


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    now = datetime.now()

    root = File(True, 0, "root", "root", now - timedelta(days=10))
    src = File(True, 0, "alice", "src", now - timedelta(days=5))
    logs = File(True, 0, "ops", "logs", now - timedelta(days=2))

    app_py = File(False, 120, "alice", "app.py", now - timedelta(hours=3))
    util_py = File(False, 90, "alice", "util.py", now - timedelta(days=1))
    notes_txt = File(False, 20, "bob", "notes.txt", now - timedelta(days=15))
    server_log = File(False, 500, "ops", "server.log", now - timedelta(hours=1))
    error_log = File(False, 300, "george", "error.log", now - timedelta(days=4))
    config_json = File(False, 60, "admin", "config.json", now - timedelta(days=7))

    root.add_entry(src)
    root.add_entry(logs)
    root.add_entry(notes_txt)

    src.add_entry(app_py)
    src.add_entry(util_py)
    src.add_entry(config_json)

    logs.add_entry(server_log)
    logs.add_entry(error_log)

    file_search = FileSearch()

    # Example 1:
    # Non-directories with extension py
    criteria_1 = FileSearchCriteria(
        AndPredicate([
            SimplePredicate(FileAttribute.IS_DIRECTORY, EqualsOperator(), False),
            SimplePredicate(FileAttribute.EXTENSION, EqualsOperator(), "py"),
        ])
    )

    result_1 = file_search.search(root, criteria_1)
    print("Python files:")
    for f in result_1:
        print(f.filename)

    # Example 2:
    # Files modified within last 2 days
    criteria_2 = FileSearchCriteria(
        AndPredicate([
            SimplePredicate(FileAttribute.IS_DIRECTORY, EqualsOperator(), False),
            SimplePredicate(FileAttribute.MODIFIED_AT, GreaterThanOperator(), now - timedelta(days=2)),
        ])
    )

    result_2 = file_search.search(root, criteria_2)
    print("\nModified within last 2 days:")
    for f in result_2:
        print(f.filename)

    # Example 3:
    # Non-directories owned by george and filename matching .*\.log
    criteria_3 = FileSearchCriteria(
        AndPredicate([
            SimplePredicate(FileAttribute.IS_DIRECTORY, EqualsOperator(), False),
            SimplePredicate(FileAttribute.OWNER, RegexMatchOperator(), r"ge.*"),
            SimplePredicate(FileAttribute.FILENAME, RegexMatchOperator(), r".*\.log"),
        ])
    )

    result_3 = file_search.search(root, criteria_3)
    print("\nGeorge log files:")
    for f in result_3:
        print(f.filename)

    # Example 4:
    # Files with extension in {py, json} and size > 80
    criteria_4 = FileSearchCriteria(
        AndPredicate([
            SimplePredicate(FileAttribute.IS_DIRECTORY, EqualsOperator(), False),
            SimplePredicate(FileAttribute.EXTENSION, InOperator(), {"py", "json"}),
            SimplePredicate(FileAttribute.SIZE, GreaterThanOperator(), 80),
        ])
    )

    result_4 = file_search.search(root, criteria_4)
    print("\nLarge code/config files:")
    for f in result_4:
        print(f.filename)

    # Example 5:
    # NOT txt files
    criteria_5 = FileSearchCriteria(
        AndPredicate([
            SimplePredicate(FileAttribute.IS_DIRECTORY, EqualsOperator(), False),
            NotPredicate(
                SimplePredicate(FileAttribute.EXTENSION, EqualsOperator(), "txt")
            ),
        ])
    )

    result_5 = file_search.search(root, criteria_5)
    print("\nNon-txt files:")
    for f in result_5:
        print(f.filename)
```

## Design Patterns Used in the Unix File Search System

### 1. Strategy Pattern

**Where used**
- `ComparisonOperator`
- `EqualsOperator`
- `GreaterThanOperator`
- `LessThanOperator`
- `RegexMatchOperator`
- `ContainsOperator`
- `InOperator`

**Why we used it**
Different attributes need different comparison logic.

Examples:
- `size > 100`
- `owner == "alice"`
- `filename matches .*\.log`
- `extension in {"py", "json"}`

Instead of hardcoding all comparisons inside one class, we encapsulate each comparison rule in its own operator class.

This makes the system flexible and easy to extend.  
If we later want to add:
- starts with
- ends with
- case insensitive match
- date range operator

we can do so by adding a new operator without changing the existing search flow.

**Simple reason**  
When comparison behavior can vary independently, Strategy is a good fit.

---

### 2. Composite Pattern

**Where used**
- `Predicate`
- `SimplePredicate`
- `CompositePredicate`
- `AndPredicate`
- `OrPredicate`
- `NotPredicate`

**Why we used it**
Search queries are often made of multiple smaller conditions.

Examples:
- `size > 10 AND owner == "bob"`
- `extension == "py" OR extension == "json"`
- `NOT is_directory`

The Composite pattern allows us to treat:
- one condition
- or a combination of conditions

in the same way because they all implement `Predicate`.

This lets us build tree-like conditions recursively.

**Simple reason**  
Composite helps us represent and evaluate nested logical conditions cleanly.

---

### 3. Facade-like Wrapper / Criteria Object

**Where used**
- `FileSearchCriteria`

**Why we used it**
`FileSearchCriteria` wraps a `Predicate` and acts as a simple interface between:
- `FileSearch`
- predicate based filtering logic

This keeps `FileSearch` focused only on traversal and avoids coupling it directly to the predicate tree structure.

`FileSearch` only needs to call:

- `criteria.is_match(file)`

and does not need to know whether the underlying condition is:
- a simple predicate
- an AND of many predicates
- a nested OR and NOT tree

**Simple reason**  
The criteria wrapper keeps traversal logic clean and hides filtering complexity.

---

### 4. Abstraction and Polymorphism

**Where used**
- `Predicate`
- `ComparisonOperator`

**Why we used it**
The design depends on abstractions rather than concrete implementations.

For example:
- `FileSearchCriteria` depends on `Predicate`
- `SimplePredicate` depends on `ComparisonOperator`

This allows different implementations to be plugged in without changing the consumers.

Examples:
- any new predicate can be used wherever `Predicate` is expected
- any new operator can be used wherever `ComparisonOperator` is expected

**Simple reason**  
Abstraction keeps the design flexible and extensible.

---

### 5. Composition Over Inheritance

**Where used**
- `SimplePredicate` contains a `ComparisonOperator`
- `AndPredicate` contains a list of `Predicate`
- `OrPredicate` contains a list of `Predicate`
- `NotPredicate` contains a `Predicate`
- `FileSearchCriteria` contains a `Predicate`

**Why we used it**
Instead of creating many subclasses for every possible condition combination, we compose behavior by combining smaller objects.

Examples:
- `SimplePredicate` does not inherit comparison behavior, it contains an operator
- `AndPredicate` does not hardcode any file condition, it combines other predicates

This makes the design easier to extend and maintain.

**Simple reason**  
Composition lets us build complex search logic from small reusable pieces.

---

### 6. Open Closed Principle Through Pattern Usage

**Where visible**
- adding new file attributes
- adding new comparison operators
- adding new predicate combinations

**Why we used it**
The system should be open for extension but closed for modification.

Examples:
- add `CREATED_AT`
- add `ACCESSED_AT`
- add `StartsWithOperator`
- add `BetweenOperator`

These can be added without rewriting the core search traversal logic.

**Simple reason**  
The system is designed so new search capabilities can be added safely.

---

### 7. Hierarchical Tree Modeling

**Where used**
- `File`
- `entries`

**Why we used it**
A file system is naturally hierarchical:
- directories contain files
- directories contain subdirectories
- subdirectories contain more files

The `File` class models this tree using child `entries`.

This enables recursive or stack based traversal through the file system.

**Simple reason**  
The data model reflects the real tree structure of a file system.

---

### 8. Separation of Concerns

**Where used**
- `File` handles file metadata
- `ComparisonOperator` handles comparison rules
- `Predicate` handles condition evaluation
- `FileSearchCriteria` wraps matching logic
- `FileSearch` handles traversal only

**Why we used it**
Each class has one clear responsibility.

Examples:
- `FileSearch` does not know how a condition is evaluated
- `Predicate` does not know how traversal works
- `ComparisonOperator` does not know anything about directories

This makes the system easier to understand, test, and extend.

**Simple reason**  
Keeping responsibilities separate makes the design cleaner.

---

### 9. Optional Visitor Pattern Extension

**Where it can be added**
- file traversal with multiple operations

**Why it can be useful**
Right now traversal is only used for searching.

If we later want traversal to support:
- size aggregation
- duplicate detection
- reporting
- permission audits

a Visitor pattern could separate traversal from the operation being performed.

**Simple reason**  
Visitor could help if many different operations must run on the file tree.

---

### 10. Optional Iterator Pattern Extension

**Where it can be added**
- `FileSearch`
- traversal output

**Why it can be useful**
Right now search returns a full list of matching files.

If we want:
- lazy evaluation
- streaming results
- early stopping

we could expose an iterator or generator based traversal.

This would be useful for very large directory trees.

**Simple reason**  
Iterator would improve memory efficiency and streaming behavior.

---

## Final Summary

### Patterns definitely used
- **Strategy Pattern**
- **Composite Pattern**

### Strong supporting design ideas
- **Abstraction and Polymorphism**
- **Composition Over Inheritance**
- **Facade-like Criteria Wrapper**
- **Open Closed Principle**
- **Separation of Concerns**
- **Hierarchical Tree Modeling**

### Patterns that can be added later
- **Visitor Pattern**
- **Iterator Pattern**

---

## Interview Ready Answer

In this Unix file search system, the main patterns used are **Strategy** and **Composite**.

- **Strategy** is used through `ComparisonOperator`, where different operator implementations like equals, greater than, less than, and regex matching can be plugged in without changing predicate logic.
- **Composite** is used through `Predicate`, `AndPredicate`, `OrPredicate`, and `NotPredicate`, which allows simple and complex search conditions to be treated uniformly.

I also used **abstraction and polymorphism** so the system depends on interfaces rather than concrete implementations, **composition** to build complex predicates from smaller reusable parts, and a **criteria wrapper** to keep traversal logic separate from filtering logic.

This makes the design extensible, testable, and easy to explain in an interview.