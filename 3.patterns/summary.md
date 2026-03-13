# 5.Summary

**Singleton**

Singleton ensures only one instance of a class exists. Use it when you need exactly one shared resource like a configuration manager, connection pool, or logger.
Most of the time you don't actually need a Singleton. You can just pass shared objects through constructors instead — it's clearer and easier to test. Singletons hide dependencies and make testing harder.

```python
# NOTE: Singletons are not idiomatic in Python. In Python, modules
# themselves are singletons - they're only imported once. For shared
# resources, create a module-level instance instead of this pattern.

class DatabaseConnection:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def query(self, sql: str) -> None:
        # Database operations
        pass

# Usage
db = DatabaseConnection()
db.query("SELECT * FROM users")


```

```text
In Python, module-level variables are natural singletons since modules are only loaded once. The class-based approach shown here uses get_instance for consistency with other languages, but you could also just use a module-level object directly.

In interviews, know what Singleton is and when not to use it. If an interviewer asks "should this be a Singleton?", the answer is usually no unless they explicitly want a single shared instance across the entire system. There are thread-safe versions of Singleton, but interviewers don't expect you to implement them in LLD interviews.
```

2. **Builder**


A builder is a helper that lets you create a complex object step by step without worrying about the order or messy construction details. It's used when an object has many optional parts or configuration choices.
This shows up when designing things like HTTP requests, database queries, or configuration objects. Instead of a constructor with ten parameters where half are null, you build the object incrementally.

```python
from typing import Optional

# NOTE: Builder is less common in Python. Python has better alternatives like
# dataclasses with default values, keyword arguments, or simple dictionaries.
# This pattern adds unnecessary complexity for most Python use cases.

class HttpRequest:
    def __init__(self):
        self.url: Optional[str] = None
        self.method: Optional[str] = None
        self.headers: dict[str, str] = {}
        self.body: Optional[str] = None

    class Builder:
        def __init__(self):
            self._request = HttpRequest()

        def url(self, url: str) -> 'HttpRequest.Builder':
            self._request.url = url
            return self

        def method(self, method: str) -> 'HttpRequest.Builder':
            self._request.method = method
            return self

        def header(self, key: str, value: str) -> 'HttpRequest.Builder':
            self._request.headers[key] = value
            return self

        def body(self, body: str) -> 'HttpRequest.Builder':
            self._request.body = body
            return self

        def build(self) -> 'HttpRequest':
            # Validate required fields
            if self._request.url is None:
                raise ValueError("URL is required")
            return self._request

# Usage
request = (HttpRequest.Builder()
    .url("https://api.example.com")
    .method("POST")
    .header("Content-Type", "application/json")
    .body('{"key": "value"}')
    .build())


```

```text
Builder makes construction readable and handles optional fields cleanly. 
It most commonly shows up in LLD interviews when you're designing API 
clients or complex configurations, but is very rarely used in other contexts.
```

3. **Factory Method**

A factory is a helper that makes the right kind of object for you so you don't have to decide which one to create. They're used to hide creation logic and keep your code flexible when the exact type you need can change.

Factory shows up regularly in interviews, usually when requirements say "support different notification types" or "handle multiple payment methods." Instead of writing new EmailNotification() throughout your code, you call notificationFactory.create(type). Now when you add SMS notifications, you update the factory. The rest of your code never changes.

```python
from abc import ABC, abstractmethod

class Notification(ABC):
    @abstractmethod
    def send(self, message: str) -> None:
        pass

class EmailNotification(Notification):
    def send(self, message: str) -> None:
        # Email sending logic
        pass

class SMSNotification(Notification):
    def send(self, message: str) -> None:
        # SMS sending logic
        pass

class NotificationFactory:
    @staticmethod
    def create(notification_type: str) -> Notification:
        if notification_type == "email":
            return EmailNotification()
        elif notification_type == "sms":
            return SMSNotification()
        raise ValueError("Unknown type")

# Usage
notif = NotificationFactory.create("email")
notif.send("Hello")


```

```text
The factory centralizes creation logic. When you add push notifications, you modify one place. Factory controls which object gets instantiated. It makes the decision once and returns the right type.

```