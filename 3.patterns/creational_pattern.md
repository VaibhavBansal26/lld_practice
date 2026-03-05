#   1. Creational Patterns

## 1. Singleton Pattern**

```text
Singleton Pattern is a creational design pattern that guarantees a class has only one instance and provides a global point of access to it.
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


- **5. Prototype Pattern**