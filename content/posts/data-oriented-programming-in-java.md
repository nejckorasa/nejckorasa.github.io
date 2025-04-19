---
title: "Data Oriented Programming (DOP) in Java"
date: 2025-04-18
tags: ["Java", "OOO", "DOP", "Object Oriented Programming", "Design Patterns", "Pattern Matching"]
categories: Software Engineering
ShowToc: true
TocOpen: true
---

## What is Data Oriented Programming?

Data Oriented Programming (DOP) is gaining momentum in the Java ecosystem due to recent language features streamlining its adoption. While conceptually straightforward, DOP offers significant advantages. But what is it?

How do we build our objects? Where does the state go? Where does the behavior go? Within OOO you could just bundle state and behavior together. But what if we separated this? What if data became the primary focus, with logic completely separated? This is the central idea of Data Oriented Programming (DOP), simple.

So instead of emphasizing objects with bundled state and methods, **DOP centers around simple data structures**. The application's logic and behavior are implemented as independent functions that operate on this data. The data itself is passive; the intelligence lies in the functions.

## Why Consider DOP? The Benefits

Why might you choose this approach? Here are a few compelling reasons:

* **Simpler and More Readable Code:** Separating data from behavior leads to clearer data structures and focused functions, making the code easier to understand and follow.
* **Improved Maintainability:** With simple data structures and distinct logic, modifications are less likely to create ripple effects across the codebase.
* **Enhanced Code Optionality and Reduced Coupling:** Adding new functionality often involves creating new functions rather than modifying existing data structures, leading to less invasive changes and reduced coupling between different parts of the system.
* **Easier Testing:** Functions operating on plain data are often easier to test in isolation.

## Java's Embrace of Data

Modern Java provides excellent tools that make DOP a viable option:

* **Records:** Simple, immutable data carriers. Less boilerplate, letting you focus on the data itself.

    ```java
    record Point(double x, double y) {}
    ```

* **Sealed Classes:** These allow you to restrict the possible subtypes of a class or interface. This is crucial for ensuring you can have exhaustive knowledge of the data you're dealing with.

    ```java
    sealed interface Shape permits Circle, Rectangle {}
    record Circle(Point center, double radius) implements Shape {}
    record Rectangle(Point topLeft, Point bottomRight) implements Shape {}
    record Triangle(Point p1, Point p2, Point p3) implements Shape {}
    ```

* **Switch and Pattern Matching with Exhaustiveness Checks:** This is main one, the one that closes the loop and brings the main advantage. The enhanced `switch` statement in Java, with its support for pattern matching, works hand in hand with sealed classes. The compiler helps you with exhaustiveness checks, shifting runtime errors to compile time. This is not limited to just sealed classes, pure enums also work.

    ```java
    int numOfEdgesCircle = switch (shape) {
        case Circle c -> 0;
        case Rectangle r -> 4;
        case Triangle t -> 3;
    };
    ```

## Textbook Example

### Introducing New Behavior

Consider the `Shape` example. In a traditional OOP approach, you might add a `getCenter()` method to the `Shape` interface and implement it in each concrete shape class. If you later needed to perform a new operation or modify an existing one, you'd likely need to update the `Shape` interface and all its implementations, which can lead to tightly coupled code.

With DOP, we define the data structures and then create separate functions to operate on them. This separation of concerns makes adding new functionality cleaner and less coupled. Here's how the `getCenter` function looks in a DOP style:

```java
public Point getCenter(Shape shape) {
    return switch (shape) {
        case Circle(Point center, double _) -> center;
        case Rectangle(Point topLeft, Point bottomRight) ->
            new Point(
                (topLeft.x + bottomRight.x) / 2,
                (topLeft.y + bottomRight.y) / 2
            );
        case Triangle(Point p1, Point p2, Point p3) ->
            new Point(
                (p1.x + p2.x + p3.x) / 3,
                (p1.y + p2.y + p3.y) / 3
            );
    };
}
```

The enhanced `switch` statement, combined with sealed classes, ensures that all possible cases are handled at compile time. If you are familiar with programming design patterns, this makes the [Visitor Pattern](https://refactoring.guru/design-patterns/visitor) redundant. The new features simplify similar scenarios dramatically by allowing you to handle all cases directly in a type-safe and concise manner.

### Introducing New Data

While DOP simplifies introducing new behavior, it also ensures consistency when introducing new data types. The compiler enforces the implementation of all missing operations, ensuring your code remains consistent and complete. This is one of the most powerful advantages of these new Java features.

For instance, if you add a new `Pentagon` shape, the compiler will flag the switch statement in the `getCenter()` method as incomplete, requiring you to implement the logic for the new shape. This compile-time enforcement not only prevents runtime errors but also ensures that your codebase evolves safely and predictably as new data types are added.

However, it's important to avoid using a default branch in your switch statements. A default branch bypasses the exhaustiveness checks provided by the compiler, which can lead to missed cases and potential bugs.

## Handling Outcomes with Clarity

Data Oriented Programming also lends itself well in scenarios where clear and explicit handling of outcomes is required, such as processing different types of results or managing errors/failures. Consider this example for handling the result of a `process` function:

``` java
sealed interface Result<T> {
    record Ok<T>(T value) implements Result<T> {}
    record Error<T>(String message) implements Result<T> {}
}

public static Result<String> process() {
    // Does some actual processing…
    if (operationSuccessful) {
        return new Ok("Success return value");
    } else {
        return new Error("Processing error");
    }
}

String result = switch (process()) {
    case Ok(var value) -> value;
    case Error(var message) -> throw new IllegalStateException("Processing error: " + message);
};
```

This approach allows callers to easily handle all possible results using a `switch`expression, promoting explicit and type-safe result processing. This eliminates ambiguity about potential return values and encourages explicit error handling.

## In Conclusion: Clear Benefits of DOP and Modern Java

By focusing on data and keeping it separate from business logic and processing, Data Oriented Programming together with modern Java offer some great advantages:

- **Simpler and More Readable Code:** Easier to understand and follow due to the separation of concerns.
- **Improved Maintainability:** Modifications are less likely to have widespread impact.
- **Enhanced Code Optionality and Reduced Coupling:** Adding new features is less invasive and reduces dependencies.
- **Easier Testing:** Functions operating on plain data are more straightforward to test.
- **Keeps Data Clean and Decoupled from Business Logic.**
- **Safer (and Cheaper) to Refactor and Change:** Minimizing coupling reduces the cost of future changes, as explained in [Tidy First? By Kent Beck](https://www.oreilly.com/library/view/tidy-first/9781098151232/), cost of software is approximately the same as the cost of changing it.

## References and Further Reading

- [Data Oriented Programming InfoQ Article](https://www.infoq.com/articles/data-oriented-programming-java/)
- [Data-Oriented Programming in Java YouTube Video](https://www.youtube.com/watch?v=UQAw3pvZPCY)
- [Data Oriented Programming in Java 21 by Nicolai Parlog](https://www.youtube.com/watch?v=8FRU_aGY4mY)

## What Do You Think?

Have you tried Data Oriented Programming in your Java projects? What challenges or benefits have you experienced? Share your thoughts in the comments or reach out to discuss further!
