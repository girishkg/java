
## Java Enumerations – The “Enum” Fundamentals

Java 5 introduced **enumerations** (enums) as a first‑class language feature.  
They give you a *type‑safe* way to work with a fixed set of constants instead of the classic
`public static final int` pattern.

Below is a self‑contained, reference‑style walk‑through that covers:

| Topic | Why it matters |
|-------|----------------|
| What an enum *is* | A special kind of class that extends `java.lang.Enum`. |
| Syntax & definition | How to write a declaration. |
| Enum constants | The immutable instances you’ll use. |
| Constructors & fields | Adding data to each constant. |
| Methods | Adding behaviour, overriding, abstract methods. |
| Built‑in methods | `name()`, `ordinal()`, `values()`, `valueOf()`. |
| `switch` and `EnumSet/EnumMap` | Common idioms. |
| Serialization & `enum` constants | Singleton guarantees. |
| Best practices & pitfalls | Things that look “ok” but are wrong. |
| Advanced tricks | Enum‑specific class bodies, sealed interfaces, pattern matching, generics. |

Feel free to skim or deep‑dive – all the core concepts are in one place.

---

## 1. What is an Enum?

* A **final** class (cannot be subclassed).
* Extends `java.lang.Enum<E>` automatically.
* Each enum constant is a **singleton** instance of that class.
* Cannot be instantiated via `new` (private constructor).
* Implements `Comparable<E>`, `Serializable`, and `Enum<E>`.

Because it is a *class*, an enum can have:

```java
public enum Day {
    MONDAY, TUESDAY, ...;   // constants
    // fields, constructors, methods
}
```

But because it is *static* and *final*, it behaves like a group of named constants.

---

## 2. Basic Syntax

```java
public enum TrafficLight {
    RED,
    YELLOW,
    GREEN
}
```

* **Public** by default (if no modifier, it's `public` in the file it resides).
* Constant names are usually all caps, following Java constant naming conventions.

You can put a **semicolon** after the last constant if you want to add fields/methods:

```java
public enum Color {
    RED("#FF0000"),
    GREEN("#00FF00"),
    BLUE("#0000FF");

    private final String hex;
    Color(String hex) { this.hex = hex; }
    public String getHex() { return hex; }
}
```

---

## 3. Enum Constants

* **Immutable**: you cannot change the identity of a constant.
* **Singleton**: each constant is instantiated exactly once by the JVM.
* **Public static final**: automatically `public static final Color RED;` etc.
* **Ordinal**: the position in the declaration list (0‑based).
* **Name**: the string you wrote (`"RED"`).

```java
System.out.println(Color.RED.ordinal()); // 0
System.out.println(Color.RED.name());    // "RED"
```

**Tip**: Don’t rely on `ordinal()` for business logic. It is implementation‑detail and can change if you reorder the list.

---

## 4. Adding Data & Behaviour

### Fields

```java
public enum Planet {
    MERCURY (3.303e+23, 2.4397e6),
    VENUS   (4.869e+24, 6.0518e6),
    // …

    private final double mass;   // in kilograms
    private final double radius; // in meters

    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }

    public double surfaceGravity() {
        final double G = 6.67300E-11;
        return G * mass / (radius * radius);
    }
}
```

### Methods

* Instance methods (called on a constant): `Planet.MARS.surfaceGravity()`.
* Static methods: `Planet.allNames()` that returns an array of names.
* **Override `toString()`** to provide a nicer string.

### Abstract Methods & Constant‑Specific Class Bodies

```java
public enum Operation {
    PLUS {
        double apply(double x, double y) { return x + y; }
    },
    MINUS {
        double apply(double x, double y) { return x - y; }
    },
    TIMES {
        double apply(double x, double y) { return x * y; }
    },
    DIVIDE {
        double apply(double x, double y) { return x / y; }
    };

    abstract double apply(double x, double y);
}
```

Each constant supplies its own implementation of `apply`.  
The enum itself declares the method as **abstract**.

---

## 5. Built‑in Enum Methods

| Method | What it does | Typical usage |
|--------|--------------|---------------|
| `values()` | `public static E[] values()` | iterate over all constants (`for (Planet p : Planet.values())`) |
| `valueOf(String name)` | `public static E valueOf(String)` | parse a string back into a constant |
| `name()` | Returns the literal name (`"RED"`). | debugging, logs |
| `ordinal()` | Returns declaration order. | rarely used in code, mainly for serialization |
| `compareTo(E)` | Natural ordering by ordinal. | sorting |
| `equals(Object)` | Same as `==`. | identity comparison |
| `hashCode()` | Usually the same as `ordinal()`. | hash‑based collections |

> **Remember**: `valueOf()` throws `IllegalArgumentException` if the string does not match any constant. Use `try‑catch` or `EnumUtils` from Apache Commons.

---

## 6. Switching on Enums

```java
switch (day) {
    case MONDAY:
    case FRIDAY:
    case SUNDAY:
        System.out.println("Weekend");
        break;
    default:
        System.out.println("Weekday");
}
```

Switching is **type‑safe** and the compiler can warn you if you miss a constant (especially with `@SuppressWarnings("switch")` removed).

Since Java 14 you can use **pattern matching for switch** (preview) or **switch expressions**:

```java
String result = switch (day) {
    case MONDAY -> "Start of week";
    case FRIDAY -> "End of week";
    default -> "Mid‑week";
};
```

---

## 7. EnumSet & EnumMap

Java’s collections framework has **special‑purpose** implementations for enums:

| Type | Use case | Example |
|------|----------|---------|
| `EnumSet<E>` | Efficient bit‑set of enum constants | `EnumSet<Day> weekend = EnumSet.of(SATURDAY, SUNDAY);` |
| `EnumMap<E, V>` | Map from enum constant to a value | `EnumMap<Color, String> map = new EnumMap<>(Color.class);` |

Both are *faster* and *memory‑efficient* than `HashSet`/`HashMap` for enums.

---

## 8. Serialization

* Enums are serialized by their **name** (`EnumSet` and `EnumMap` are also supported).
* Guarantees **singleton** property after deserialization – no duplicate instances.
* No `serialVersionUID` needed; if you change the enum (add/remove constants), you may need to handle backward compatibility.

```java
ObjectOutputStream out = new ObjectOutputStream(...);
out.writeObject(Color.RED);
...
ObjectInputStream in = new ObjectInputStream(...);
Color c = (Color) in.readObject(); // c == Color.RED
```

---

## 9. Common Pitfalls

| Pitfall | Why it’s a problem | Fix |
|---------|-------------------|-----|
| Using `ordinal()` for logic | `ordinal()` is implementation detail. | Use a dedicated field or method. |
| Relying on the order of `values()` array | Reordering constants changes the array. | Explicitly use `EnumSet` or map. |
| Storing mutable state in enums | Enums are singletons; shared mutable state = thread‑safety problems. | Make fields immutable (`final`), or use thread‑safe wrappers. |
| Overusing enums for large configurations | Enums become too large and hard to maintain. | Split into multiple enums or external config. |
| Ignoring `valueOf()`’s exception | Can crash on bad input. | Validate input or catch `IllegalArgumentException`. |
| Using `Enum.valueOf()` in performance‑critical loops | `valueOf()` is `O(n)` over the number of constants. | Pre‑cache a `Map<String, E>` if you need many lookups. |

---

## 10. Advanced Tricks

### 10.1 Sealed Interfaces

Since Java 17 you can pair enums with sealed interfaces:

```java
public sealed interface Shape permits Circle, Square { /* ... */ }

public enum Circle implements Shape {
    ONE_UNIT, TWO_UNIT;
    // ...
}

public enum Square implements Shape {
    ONE_UNIT, TWO_UNIT;
    // ...
}
```

### 10.2 Pattern Matching on Enums (Java 21)

You can now use `instanceof` with enums in pattern matching:

```java
switch (color) {
    case RED r -> System.out.println("Red: " + r.getHex());
    case BLUE -> System.out.println("Blue");
}
```

### 10.3 Generic Enums

```java
public enum Response<T> {
    OK(200, null),
    NOT_FOUND(404, "Not Found"),
    BAD_REQUEST(400, "Bad Request");

    private final int code;
    private final T message;
    Response(int code, T message) { this.code = code; this.message = message; }
    public int getCode() { return code; }
    public T getMessage() { return message; }
}
```

### 10.4 Enums in Annotations

```java
@Retention(RetentionPolicy.RUNTIME)
public @interface HttpMethod {
    HttpVerb value();
}
```

### 10.5 Using `java.util.concurrent.atomic.AtomicReferenceArray` with EnumSet

For lock‑free structures where you need a map from enum to a value, you can use an array whose indices correspond to the ordinal of the enum constant.

```java
AtomicReferenceArray<Integer> counter = new AtomicReferenceArray<>(Color.values().length);
counter.set(Color.RED.ordinal(), 5);
```

---

## 11. Quick Reference Cheat‑Sheet

| Feature | Code Snippet |
|---------|--------------|
| Basic enum | `enum State { ON, OFF }` |
| Data + ctor | `enum Size { SMALL("S"), MEDIUM("M") }` |
| Abstract method | `enum Op { PLUS { double apply(...) } }` |
| EnumSet | `EnumSet<Day> weekdays = EnumSet.range(MONDAY, FRIDAY);` |
| EnumMap | `EnumMap<Day, String> map = new EnumMap<>(Day.class);` |
| Switch expression | `String s = switch(state) { case ON -> "yes"; default -> "no"; };` |
| valueOf | `Color c = Color.valueOf("RED");` |
| values | `for (Color c : Color.values()) {}` |

---

## 12. When to Use an Enum

| Scenario | Enum is a good fit |
|----------|--------------------|
| A fixed set of constants that never change at runtime. | ✅ |
| You need to associate data or behaviour with each constant. | ✅ |
| You want compile‑time type safety (no magic numbers). | ✅ |
| You need to use them in `switch` statements. | ✅ |
| You’ll use them as keys in a map/set. | ✅ |
| They can’t be represented by simple `int` flags. | ✅ |
| You want built‑in serialization guarantees. | ✅ |

---

## 13. Final Thought

Enums in Java are *more than just named constants*.  
They are fully fledged classes that:

* guarantee immutability and singleton behaviour,
* let you attach data and custom behaviour per constant,
* integrate tightly with the Java collections framework,
* provide compile‑time safety and easy-to‑read code.

Use them whenever you have a *closed* set of values, and you’ll rarely look back.

Happy coding!
