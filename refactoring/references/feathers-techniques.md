# Feathers Dependency-Breaking Techniques

Detailed reference for the 25 dependency-breaking techniques from "Working Effectively with Legacy Code."

## The Core Problem

Legacy code resists change because:
1. You can't test it (dependencies prevent isolation)
2. You can't understand it (too large, too tangled)
3. You can't change it safely (no safety net)

The solution: **break dependencies to get code under test, then refactor freely.**

---

## Techniques (Alphabetical)

### Adapt Parameter
**When:** Method takes a parameter whose type is hard to instantiate in tests.
**How:** Create a simpler interface that wraps just what the method needs.

### Break Out Method Object
**When:** A long method uses many instance variables making extraction hard.
**How:** Move the entire method to a new class; the old class's fields become constructor params.

### Definition Completion
**When:** Can't instantiate a class because base class has unimplemented methods.
**How:** Create a test subclass that provides trivial implementations.

### Encapsulate Global References
**When:** Functions depend on global state (singletons, module-level variables).
**How:** Wrap globals in a class/object; pass as dependency.

### Expose Static Method
**When:** Method doesn't use instance state but is trapped in a class.
**How:** Make it static → now testable without instantiation.

### Extract and Override Call
**When:** One problematic dependency call is buried in an otherwise good method.
**How:** Extract that call to a virtual method; override in test subclass.

### Extract and Override Factory Method
**When:** Constructor creates dependencies that make testing hard.
**How:** Extract creation to a factory method; override in test subclass.

### Extract and Override Getter
**When:** Instance variable initialization is hard to control in tests.
**How:** Access through a getter; override getter in test subclass.

### Extract Implementer
**When:** A class mixes interface and implementation (making it hard to substitute).
**How:** Extract an interface, then the original becomes "the implementation."

### Extract Interface
**When:** You need to substitute a dependency but it has no interface.
**How:** Create an interface from the methods you actually call.

### Introduce Instance Delegator
**When:** Important behavior is in static methods (untestable with mocking).
**How:** Create instance methods that delegate to the static methods.

### Introduce Static Setter
**When:** Singleton or global prevents test isolation.
**How:** Add a static setter for tests to inject a fake (use sparingly!).

### Link Substitution
**When:** Need to replace a dependency at the module/import level.
**How:** Swap the import path (DI via module system).

### Parameterize Constructor
**When:** Constructor hard-codes dependencies.
**How:** Accept dependencies as constructor parameters (classic DI).

### Parameterize Method
**When:** Method hard-codes a dependency inside its body.
**How:** Accept the dependency as a parameter.

### Primitivize Parameter
**When:** Method takes a complex object but only uses simple data from it.
**How:** Pass the primitives directly.

### Pull Up Feature
**When:** Subclass has behavior that should be shared/testable independently.
**How:** Move to superclass or extract to a utility.

### Push Down Dependency
**When:** Base class has a dependency that only some subclasses need.
**How:** Move dependency to the subclass that uses it.

### Replace Function with Function Pointer
**When:** Need to substitute a free function for testing.
**How:** Accept the function as a parameter (higher-order function / callback).

### Replace Global Reference with Getter
**When:** Code reads a global variable directly.
**How:** Access through a getter method that can be overridden in tests.

### Subclass and Override Method
**When:** A method does something untestable (I/O, network, time).
**How:** Create test subclass that overrides just that method.

### Supersede Instance Variable
**When:** Can't control an instance variable's initialization.
**How:** Add a setter or re-assignment point after construction (for tests only).

### Template Redefinition
**When:** Behavior variation is needed but inheritance is impractical.
**How:** Use template method pattern; test via different template implementations.

### Text Redefinition
**When:** In dynamic languages, redefine methods at runtime for tests.
**How:** Monkey-patch in test setup (use sparingly!).

---

## Choosing a Technique

| Situation | Best Technique |
|-----------|---------------|
| Can't instantiate class | Parameterize Constructor, Extract Interface |
| One bad line in a good method | Extract and Override Call |
| Static method dependency | Introduce Instance Delegator |
| Global state | Encapsulate Global References |
| Complex parameter type | Adapt Parameter, Primitivize Parameter |
| Hard-coded file/network I/O | Parameterize Method (inject callback) |
| Module-level coupling | Link Substitution (re-export from wrapper) |

## The Safety Net Progression

1. **No tests, no understanding** → Write characterization tests first
2. **Have tests, tangled code** → Break dependencies using techniques above
3. **Isolated code** → Apply Fowler's refactoring catalog freely
4. **Clean code** → Maintain with review thresholds
