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