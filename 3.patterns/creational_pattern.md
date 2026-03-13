#   1. Creational Patterns

## 1. Singleton Pattern

```text
Singleton Pattern is a creational design pattern that guarantees a class 
has only one instance and provides a global point of access to it.
```

Two requirements define the pattern:

- Single instance: No matter how many times any part of the code requests it, the same object is returned.
- Global access: Any component can reach the instance without needing it passed through constructors or method parameters.

**Singleton is useful in scenarios like:**

- Managing Shared Resources (database connections, thread pools, caches, configuration settings)
- Coordinating System-Wide Actions (logging, print spoolers, file managers)
- Managing State (user session, application state)

**Specific Examples:**

- Logger Classes: Many logging frameworks use the Singleton pattern to provide a global logging object. This ensures that log messages are consistently handled and written to the same output stream.
- Database Connection Pools: Connection pools help manage and reuse database connections efficiently. A Singleton can ensure that only one pool is created and used throughout the application.
- Cache Objects: In-memory caches are often implemented as Singletons to provide a single point of access for cached data across the application.
- Thread Pools: Thread pools manage a collection of worker threads. A Singleton ensures that the same pool is used throughout the application, preventing resource overuse.
File System: File systems often use Singleton objects to represent the file system and provide a unified interface for file operations.

**Class Diagram**

To implement the singleton pattern, we must prevent external objects from creating instances of the singleton class. Only the singleton class should be permitted to create its own objects.

Additionally, we need to provide a method for external objects to access the singleton object.

```
An instance field stores the one and only Singleton object.
The constructor is private or otherwise restricted, so other code cannot create new instances directly.
A getInstance() (or similar) class-level method returns the shared instance and is accessible from anywhere.
```

**How It Works**
Step 1: First Request
A client calls Singleton.getInstance(). The method checks if an instance already exists.

Step 2: Instance Creation
If no instance exists, the method creates one using the private constructor and stores it in the static field.

Step 3: Return Instance
The method returns the newly created instance.

Step 4: Subsequent Requests
Later calls to getInstance() find the instance already exists and return it immediately, skipping creation entirely.

The sequence diagram above shows two clients requesting the instance. The first triggers creation; the second returns the existing one. Both end up with references to the same object.


**Implementation**

Singleton implementation varies across languages. The central challenge is thread safety: if two threads call getInstance() simultaneously when the instance has not been created yet, both might create separate instances.

We will start with the simplest (but broken) approach and progressively improve it. After the shared implementations, we cover language-specific idioms that are recommended for production use.

1. Lazy Initialization (Not Thread-Safe)

```python
class LazySingleton:
    # Holds the single shared instance (initially not created)
    _instance = None

    # Constructor prevents direct creation if instance already exists
    def __init__(self):
        if LazySingleton._instance is not None:
            raise Exception("Use get_instance() instead.")

    # Global access point to get the Singleton instance
    @staticmethod
    def get_instance():

        # Create the instance only when first requested (lazy initialization)
        if LazySingleton._instance is None:
            LazySingleton._instance = LazySingleton()

        # Return the shared instance
        return LazySingleton._instance
```

How it works

- getInstance() method checks if an instance already exists.
- If not, it creates a new instance.
- If an instance already exists, it skips the creation step.

2. Thread-Safe with Locking

This approach extends lazy initialization by ensuring the Singleton is safe to use in multi-threaded environments.

When multiple threads try to access the instance at the same time, synchronization (or locking) ensures that only one thread can create the object, while others wait.

```python
class ThreadSafeSingleton:
    _instance = None
    _lock = threading.Lock()

    def __init__(self):
        if ThreadSafeSingleton._instance is not None:
            raise Exception("Use get_instance() instead.")

    @staticmethod
    def get_instance():
        with ThreadSafeSingleton._lock:
            if ThreadSafeSingleton._instance is None:
                ThreadSafeSingleton._instance = ThreadSafeSingleton()
        return ThreadSafeSingleton._instance
```

How it works?

- The instance is created only when first requested (lazy initialization).
- The method that returns the instance uses a lock / mutex / synchronization mechanism.
- When a thread enters the protected section, it acquires the lock. Other threads must wait until the lock is released.
- This guarantees that only one instance is created, even under concurrent access.

3. Double-Checked Locking

Double-checked locking reduces the performance overhead by only synchronizing during the first object creation. After the instance exists, threads skip the lock entirely.

```python
class DoubleCheckedSingleton:
    # Holds the single shared instance
    _instance = None
    # Lock used only during first-time creation
    _lock = threading.Lock()

    # Constructor prevents accidental direct creation
    def __init__(self):
        if DoubleCheckedSingleton._instance is not None:
            raise Exception("Use get_instance() instead.")

    # Global access point to get the Singleton instance
    @staticmethod
    def get_instance():
        
		# Fast path: first check without locking
        if DoubleCheckedSingleton._instance is None:
            # Lock only when the instance might need to be created
            with DoubleCheckedSingleton._lock:
                # Second check inside the lock (prevents double creation)
                if DoubleCheckedSingleton._instance is None:
                    DoubleCheckedSingleton._instance = DoubleCheckedSingleton()

        # Return the shared instance
        return DoubleCheckedSingleton._instance
```

- If the first check passes, we synchronize/lock and check the same condition one more time because multiple threads may have passed the first check.
- The instance is created only if both checks pass.
  
4. Eager Initialization

In eager initialization, the Singleton instance is created as soon as the class/module is loaded, before any thread can access it. That makes it inherently thread-safe without explicit locks, because initialization happens once during load/initialization.

This approach is suitable if your application always creates and uses the singleton instance, or the overhead of creating it is minimal.

```python
class EagerSingleton:
    # Holds the single shared instance (created immediately at module load time)
    _instance = None

    # Private-like constructor guard to prevent direct instantiation
    def __init__(self):
        if EagerSingleton._instance is not None:
            raise Exception("Use get_instance() instead.")

    # Global access point to get the Singleton instance
    @staticmethod
    def get_instance():
        # Return the already-created shared instance
        return EagerSingleton._instance
```

- A class-level/static variable holds the single shared instance.
- The instance is created during class/module initialization, not on first use.
- No locks are needed because the runtime initializes static/class state once.


**Language Specific Implementations**

Module-Level Singleton

In Python, modules are imported once. The interpreter caches modules in sys.modules after the first import, so subsequent imports get the same module object. This makes module-level instances natural singletons:

```python
# config_manager.py

class _ConfigManager:
    def __init__(self):
        self._settings = {}

    def set(self, key, value):
        self._settings[key] = value

    def get(self, key, default=None):
        return self._settings.get(key, default)


# Created once when module is first imported
config = _ConfigManager()


# Usage (from other files):
# from config_manager import config
# config.set("debug", True)
```
The underscore prefix on _ConfigManager signals that the class is an implementation detail. Users import config, not the class.

This is the most Pythonic singleton approach: simple, clear, and naturally thread-safe at the module loading level. The Python community generally prefers this over class-based singleton tricks.


__new__ Override

For cases where you want Singleton() to always return the same instance (class-based interface):

```python
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        # Warning: __init__ is called every time Singleton() is called
        pass

# Usage
s1 = Singleton()
s2 = Singleton()
assert s1 is s2  # True
```

One gotcha: __init__ runs every time you call Singleton(), even though __new__ returns the same object. If your __init__ resets state, you will lose data. Guard against this with a flag or move initialization into __new__.

Metaclass Singleton

A metaclass-based approach separates the singleton logic from the class itself. Any class using SingletonMeta as its metaclass becomes a singleton automatically:

```python
import threading

class SingletonMeta(type):
    _instances = {}
    _lock = threading.Lock()

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            with cls._lock:
                if cls not in cls._instances:
                    instance = super().__call__(*args, **kwargs)
                    cls._instances[cls] = instance
        return cls._instances[cls]


class Database(metaclass=SingletonMeta):
    def __init__(self):
        self.connection = "connected"


# Usage
db1 = Database()
db2 = Database()
assert db1 is db2  # True
```

The metaclass intercepts the class call (Database()), checks if an instance already exists, and returns the existing one or creates a new one. This is thread-safe (uses double-checked locking with a lock) and reusable across multiple classes. The downside is that metaclasses can be confusing to developers unfamiliar with Python's object model.


Decorator Singleton

A function decorator that caches the first instance and returns it on subsequent calls:

```python
import threading

def singleton(cls):
    instances = {}
    lock = threading.Lock()

    def get_instance(*args, **kwargs):
        if cls not in instances:
            with lock:
                if cls not in instances:
                    instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance


@singleton
class AppConfig:
    def __init__(self):
        self.settings = {}


# Usage
c1 = AppConfig()
c2 = AppConfig()
assert c1 is c2  # True
```
The decorator replaces the class with a wrapper function. Calling AppConfig() actually calls get_instance(), which returns the cached instance. This is clean and readable, but be aware that AppConfig is no longer a class reference after decoration, which means isinstance(c1, AppConfig) will not work as expected.
   

**Practical Example: In-Memory Cache Manager**

Building an application where multiple components (HTTP handlers, database layer, background jobs) all need to cache expensive data like user profiles, configuration, and query results.

We want one shared cache so that any component's writes are immediately visible to all others, without duplicate maps, stale reads, or wasted memory.

Without Singleton:

```python
CacheManager cacheA = new CacheManager();
cacheA.put("user:42", userData);

CacheManager cacheB = new CacheManager();
cacheB.get("user:42"); // null! Different instance, different map

// Problems:
// - Duplicate HashMaps wasting memory
// - Writes in one component invisible to others
// - TTL cleanup duplicated across instances
```

With Singleton:

```text
HTTP Reader--------------
                        |
DB Layer-------------- Cache Manager (Singleton)  ----------> In-Memory Cache
                        |
Background Job-----------
```

All components access the single CacheManager instance, which manages one shared map, handles TTL expiry on reads, and synchronizes access internally.

```python
# cache_manager.py
import threading
import time


class _CacheManager:
    def __init__(self):
        self._lock = threading.Lock()
        self._cache: dict[str, tuple[str, float | None]] = {}

    def put(self, key: str, value: str, ttl_seconds: int = 0):
        expiry = time.time() + ttl_seconds if ttl_seconds > 0 else None
        with self._lock:
            self._cache[key] = (value, expiry)

    def get(self, key: str) -> str | None:
        with self._lock:
            entry = self._cache.get(key)
            if entry is None:
                return None
            value, expiry = entry
            if expiry is not None and time.time() > expiry:
                del self._cache[key]
                return None
            return value

    def remove(self, key: str):
        with self._lock:
            self._cache.pop(key, None)

    def size(self) -> int:
        now = time.time()
        with self._lock:
            self._cache = {
                k: (v, exp) for k, (v, exp) in self._cache.items()
                if exp is None or now <= exp
            }
            return len(self._cache)


# Module-level singleton
cache_manager = _CacheManager()


# --- Main ---
if __name__ == "__main__":
    # Both references point to the same CacheManager instance
    cache1 = cache_manager
    cache2 = cache_manager

    print(f"Same instance? {cache1 is cache2}")  # True

    # Component A caches data
    cache1.put("user:42", "{name: 'Alice'}", 5)  # 5-second TTL
    cache1.put("config:theme", "dark")            # no expiry

    # Component B reads from the same cache
    print(f"user:42 = {cache2.get('user:42')}")         # {name: 'Alice'}
    print(f"config:theme = {cache2.get('config:theme')}") # dark
    print(f"Cache size: {cache2.size()}")                 # 2
```

**Benefits Achieved:**

- Single shared cache, no duplicate data or wasted memory
- Any component's put() is immediately visible to all others
- Thread-safe with internal synchronization
- TTL expiry handled in one place with lazy cleanup
- No need to pass cache references through constructors

**Pros and Cons**

Pros:
- Ensures a single instance of a class and provides a global point of access to it.
- Only one object is created, which can be particularly beneficial for resource-heavy classes.
- Provides a way to maintain global state within an application.
- Supports lazy loading, where the instance is only created when it's first needed.
- Guarantees that every object in the application uses the same global resource.

Cons:

- Violates the Single Responsibility Principle: The pattern solves two problems at the same time.
- In multithreaded environments, special care must be taken to implement Singletons correctly to avoid race conditions.
- Introduces global state into an application, which might be difficult to manage.
- Classes using the singleton can become tightly coupled to the singleton class.
- Singleton patterns can make unit testing difficult due to the global state it introduces.

```text
It's important to note that the Singleton pattern should be used judiciously, as it introduces global state and can make testing and maintenance more challenging.
```


- **2. Factory Method Pattern**
   

- **3. Abstract Factory Pattern**


- **4. Builder Pattern**

```text
The Builder Design Pattern is a creational pattern that lets you construct complex objects step-by-step, separating the construction logic from the final representation.
```

It’s particularly useful in situations where:

- An object has many optional fields, and most callers only need a subset.
- You want to avoid telescoping constructors or long parameter lists.
- The object must be assembled through multiple steps, possibly in a specific order.
  
Without a builder, developers often end up with overloaded constructors or a wide set of setters. For example, a User object might include fields like name, email, phone, address, and preferences. As the number of fields grows, the API becomes harder to use correctly, easier to misuse, and more difficult to maintain.

The Builder Pattern addresses this by introducing a dedicated builder that owns the creation logic. Clients configure the builder step by step and then build the final object, which can remain immutable, validated, and consistently constructed.


1. The Problem: Building Complex HttpRequest Objects

Imagine you're building a system that needs to configure and create HTTP requests.

Each HttpRequest can contain a mix of required and optional fields:

URL (required)
HTTP Method (e.g., GET, POST, PUT, defaults to GET)
Headers (optional, multiple key-value pairs)
Query Parameters (optional, multiple key-value pairs)
Request Body (optional, typically for POST/PUT)
Timeout (optional, default to 30 seconds)
At first glance, it seems manageable. But as the number of optional fields increases, so does the complexity of object construction.

The Naive Approach: Telescoping Constructors
A common approach is constructor overloading, often called the telescoping constructor anti-pattern. You define multiple constructors with increasing numbers of parameters:

```python
class HttpRequestTelescoping:
    def __init__(self, url, method="GET", headers=None, query_params=None, body=None, timeout=30000):
        self.url = url
        self.method = method
        self.headers = headers if headers is not None else {}
        self.query_params = query_params if query_params is not None else {}
        self.body = body
        self.timeout = timeout

        print(f"HttpRequest Created: URL={url}, "
              f"Method={method}, "
              f"Headers={len(self.headers)}, "
              f"Params={len(self.query_params)}, "
              f"Body={body is not None}, "
              f"Timeout={timeout}")

    # Optional: add getter methods if needed
```
Example Client Code

```python
if __name__ == "__main__":
    req1 = HttpRequestTelescoping("https://api.example.com/data")

    req2 = HttpRequestTelescoping(
        "https://api.example.com/submit",
        "POST",
        None,
        None,
        '{"key":"value"}'
    )

    req3 = HttpRequestTelescoping(
        "https://api.example.com/config",
        "PUT",
        {"X-API-Key": "secret"},
        None,
        "config_data",
        5000
    )
```

What’s Wrong with This Approach?

While it works functionally, this design quickly becomes unwieldy and error-prone as the object becomes more complex.

1. Hard to Read and Write
Multiple parameters of the same type (e.g., String, Map) make it easy to accidentally swap arguments. Code is difficult to understand at a glance, especially when most parameters are null.

2. Error-Prone
Clients must pass null for optional parameters they do not want to set, increasing the risk of bugs. One wrong position and you silently assign a value to the wrong field.

3. Inflexible and Fragile
If you want to set parameter 5 but not 3 and 4, you are forced to pass null for 3 and 4. You must follow the exact parameter order, which hurts both readability and usability.

4. Poor Scalability
Adding a new optional parameter requires adding or changing constructors, which may break existing code. Testing and documentation become increasingly difficult to maintain.

We need a more flexible, readable, and maintainable way to construct HttpRequest objects. This is exactly where the Builder pattern comes in.

2. What is the Builder Pattern

The Builder pattern separates the construction of a complex object from its representation, allowing the same construction process to create different configurations.

Two ideas define the pattern:

Step-by-step construction: Instead of passing everything to a constructor at once, you set each field through individual method calls. You only call the methods for the fields you need.
Fluent interface: Each setter method returns the builder itself, allowing you to chain calls into a single readable expression that ends with build().
Before: Telescoping Constructor

```python
req = HttpRequest(
    url,                 # url
    method,              # method
    headers,             # headers
    None,                # body
    None,                # query_params
    30000                # timeout_ms
)
```
After: Builder Pattern

```python
req = (HttpRequest.Builder(url)
       .method("POST")
       .add_header("key", "val")
       .build())
```



Class Diagram

The Builder pattern involves four participants. In many real-world implementations, the Director is optional and is often skipped when using fluent builders.

Builder (e.g., HttpRequestBuilder)

- Exposes methods to configure the product step by step.
- Typically returns the builder itself from each method to enable fluent chaining.
- Often implemented as a static nested class inside the product class.
  
ConcreteBuilder (e.g., StandardHttpRequestBuilder)

- Implements the builder API (either via an interface or directly through fluent methods).
- Stores intermediate state for the object being constructed.
- Implements build() to validate inputs and produce the final product instance.

Product (e.g., HttpRequest)

- The complex object being constructed.
- Often immutable and created only through the builder.
- Commonly has a private constructor that copies state from the builder.

Director (Optional) (e.g., HttpRequestDirector)

- Coordinates the construction process by calling builder steps in a specific sequence.
- Useful when you want to encapsulate standard configurations or reusable construction sequences.
- Often omitted in fluent builder style, where the client effectively plays this role by chaining builder calls.

How It Works

Step 1: Create the Builder
The client creates a Builder, passing any required parameters to its constructor.

Step 2: Configure Optional Fields
The client calls setter methods on the Builder for each optional field it needs. Each method returns the Builder itself, enabling chaining. The order of these calls does not matter.

Step 3: Build the Product
The client calls build(). The Builder passes itself to the Product's private constructor, which copies the configured state into immutable fields.

Step 4: Use the Product
The client receives a fully constructed, immutable Product. The Builder can be discarded or reused to create a different configuration.


Implementing Builder

Now let's implement the Builder pattern for our HttpRequest example. We create the Product class with a private constructor and a static nested Builder class.

1. Create the Product (HttpRequest) and Builder
We start by creating the HttpRequest class, the product we want to build. It has multiple fields (some required, some optional), and its constructor will be private, forcing clients to construct it via the builder.

The builder class will be defined as a static nested class within HttpRequest, and the constructor will accept an instance of that builder to initialize the fields.

```python
class HttpRequest:
    def __init__(self, builder):
        self.url = builder._url
        self.method = builder._method
        self.headers = dict(builder._headers)  # defensive copy
        self.query_params = dict(builder._query_params)
        self.body = builder._body
        self.timeout = builder._timeout

    def __str__(self):
        return (f"HttpRequest(url='{self.url}', method='{self.method}', "
                f"headers={self.headers}, query_params={self.query_params}, "
                f"body='{self.body}', timeout={self.timeout})")

    class Builder:
        def __init__(self, url):
            self._url = url  # required
            self._method = "GET"
            self._headers = {}
            self._query_params = {}
            self._body = None
            self._timeout = 30000

        def method(self, method):
            self._method = method
            return self

        def add_header(self, key, value):
            self._headers[key] = value
            return self

        def add_query_param(self, key, value):
            self._query_params[key] = value
            return self

        def body(self, body):
            self._body = body
            return self

        def timeout(self, timeout):
            self._timeout = timeout
            return self

        def build(self):
            return HttpRequest(self)
```

2. Using the Builder from Client Code
Here is how clients construct different types of HTTP requests:


```python
if __name__ == "__main__":
    # Simple GET request
    get = HttpRequest.Builder("https://api.example.com/users") \
        .build()

    # POST with body and custom timeout
    post = HttpRequest.Builder("https://api.example.com/users") \
        .method("POST") \
        .add_header("Content-Type", "application/json") \
        .body('{"name":"Alice","email":"alice@example.com"}') \
        .timeout(5000) \
        .build()

    # Authenticated PUT with query parameters
    put = HttpRequest.Builder("https://api.example.com/config") \
        .method("PUT") \
        .add_header("Authorization", "Bearer token123") \
        .add_header("Content-Type", "application/json") \
        .add_query_param("env", "production") \
        .add_query_param("version", "2") \
        .body('{"feature_flag":true}') \
        .timeout(10000) \
        .build()

    print(get)
    print(post)
    print(put)
```

Compare this to the telescoping constructor version. Every field is named. No nulls. No positional guessing. You can set fields in any order, and it is immediately obvious what each request looks like.

What We Achieved

No telescoping constructors or null arguments. Each field is set by name through a dedicated method.
Readable, self-documenting code. The chain of method calls reads like a specification of the request.
Immutable products. Once built, the HttpRequest cannot be modified. Thread-safe by design.
Easy to extend. Adding a new optional field means adding one method to the Builder. No existing code breaks.
Flexible ordering. Clients can call builder methods in any order. No positional coupling.

The Director Pattern

So far, the client has been calling builder methods directly. But what happens when multiple parts of your codebase need to create the same type of request?

For example, every API call to your payment service needs the same authorization header, content type, and timeout. Duplicating that configuration across 20 call sites is a maintenance problem waiting to happen.

The Director solves this by encapsulating common construction sequences into named methods. Instead of every client knowing how to configure a builder, the Director provides pre-built recipes.

```python
import uuid

class HttpRequestDirector:

    def build_simple_get(self, url):
        return HttpRequest.Builder(url) \
            .method("GET") \
            .timeout(30000) \
            .build()

    def build_authenticated_post(self, url, token, body):
        return HttpRequest.Builder(url) \
            .method("POST") \
            .add_header("Authorization", f"Bearer {token}") \
            .add_header("Content-Type", "application/json") \
            .body(body) \
            .timeout(10000) \
            .build()

    def build_internal_service_call(self, url):
        return HttpRequest.Builder(url) \
            .method("GET") \
            .add_header("X-Internal-Service", "true") \
            .add_header("X-Trace-Id", str(uuid.uuid4())) \
            .timeout(5000) \
            .build()

# Usage
if __name__ == "__main__":
    director = HttpRequestDirector()

    get = director.build_simple_get("https://api.example.com/users")
    post = director.build_authenticated_post(
        "https://api.example.com/orders", "token123", '{"item":"book"}')
    internal = director.build_internal_service_call(
        "https://internal.service/health")

    print(get)
    print(post)
    print(internal)
```

Practical Example: SQL QueryBuilder
A query builder that constructs SQL SELECT statements step-by-step. This is a common pattern in ORMs and database libraries.

```python
class SqlQuery:
    def __init__(self, builder):
        self.table = builder._table
        self.columns = list(builder._columns)
        self.conditions = list(builder._conditions)
        self.order_by = builder._order_by
        self.order_direction = builder._order_direction
        self.limit_val = builder._limit
        self.offset_val = builder._offset

    def to_sql(self):
        cols = ", ".join(self.columns) if self.columns else "*"
        sql = f"SELECT {cols} FROM {self.table}"
        if self.conditions:
            sql += " WHERE " + " AND ".join(self.conditions)
        if self.order_by:
            sql += f" ORDER BY {self.order_by} {self.order_direction}"
        if self.limit_val > 0:
            sql += f" LIMIT {self.limit_val}"
        if self.offset_val > 0:
            sql += f" OFFSET {self.offset_val}"
        return sql

    class Builder:
        def __init__(self, table):
            self._table = table
            self._columns = []
            self._conditions = []
            self._order_by = None
            self._order_direction = "ASC"
            self._limit = 0
            self._offset = 0

        def select(self, *cols):
            self._columns.extend(cols)
            return self

        def where(self, condition):
            self._conditions.append(condition)
            return self

        def order_by(self, column, direction="ASC"):
            self._order_by = column
            self._order_direction = direction
            return self

        def limit(self, limit):
            self._limit = limit
            return self

        def offset(self, offset):
            self._offset = offset
            return self

        def build(self):
            return SqlQuery(self)

if __name__ == "__main__":
    query1 = SqlQuery.Builder("users") \
        .select("name", "email") \
        .where("age > 18") \
        .where("active = true") \
        .order_by("name", "ASC") \
        .limit(10) \
        .build()

    query2 = SqlQuery.Builder("orders") \
        .select("id", "total", "created_at") \
        .where("status = 'completed'") \
        .where("total > 100") \
        .order_by("created_at", "DESC") \
        .limit(20) \
        .offset(40) \
        .build()

    print(query1.to_sql())
    print(query2.to_sql())
```

The SQL QueryBuilder shows how Builder naturally fits domain-specific languages. Each method call adds a clause, and build() (or toSql()) assembles everything into a valid SQL string.




- **5. Prototype Pattern**

```
The Prototype Design Pattern is a creational design pattern that lets you create new objects by cloning existing ones, instead of instantiating them from scratch.
```

t’s particularly useful in situations where:

Creating a new object is expensive, time-consuming, or resource-intensive.
You want to avoid duplicating complex initialization logic.
You need many similar objects with only slight differences.
The Prototype Pattern allows you to create new instances by cloning a pre-configured prototype object, ensuring consistency while reducing boilerplate and complexity.

1. The Challenge of Cloning Objects
Imagine you have an object in your system, and you want to create an exact copy of it. How would you do it?

Your first instinct might be to:

Create a new object of the same class.
Manually copy each field from the original object to the new one.
Simple enough, right?

Well, not quite.

Problem 1: Encapsulation Gets in the Way
This approach assumes that all fields of the object are publicly accessible. But in a well-designed system, many fields are private and hidden behind encapsulation. That means your cloning logic can’t access them directly.

Unless you break encapsulation (which defeats the purpose of object-oriented design), you can’t reliably copy the object this way.

Problem 2: Class-Level Dependency
Even if you could access all the fields, you'd still need to know the concrete class of the object to instantiate a copy.

This tightly couples your cloning logic to the object's class, which introduces problems:

It violates the Open/Closed Principle.
It reduces flexibility if the object's implementation changes.
It becomes harder to scale when you work with polymorphism.
Problem 3: Interface-Only Contexts
In many cases, your code doesn’t work with concrete classes at all, it works with interfaces.

For example:

```python
def process_shape(shape: Shape):
    # Do something with shape
    pass
```


