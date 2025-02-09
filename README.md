
- [Java 9 Modules](#java-9-modules)
  - [strong encapsulation](#strong-encapsulation)
  - [better performance](#better-performance)
  - [Reducing Application Size](#reducing-application-size)
  - [Practical example of working with modules](#practical-example-of-working-with-modules)
- [Java 10 feature: JVM container awarness](#java-10-feature-jvm-container-awarness)


# Java 9 Modules

A module is a self-contained unit of code that groups related **packages**, **resources**, and configuration. It is defined by a `module-info.java` file, which specifies:

- **Dependencies** (`requires`): Other modules it needs.
- **Exported Packages** (`exports`): Packages accessible to other modules.

If there is 2 packages (1 & 2) inside module A, and it exports only package 1, then, if moduleB requires moduleA, it will only be able to use package 1 (because its the only one exported by module A).

So This Modules system has benefits: 

## strong encapsulation

before Java 9, all `public` classes in a JAR were accessible to any other JAR in the classpath. This led to **accidental dependencies** on internal APIs that were never meant to be used directly.

**Example problem before Java 9:**

Imagine you create a library with two packages:

- `com.example.api` → Intended to be used by other developers.
- `com.example.internal` → Internal utilities **not** meant for public use.

If you package everything in a JAR, **nothing stops** another developer from using `com.example.internal`. They might use it in production, and if you later remove or change it, their application **breaks**. (A concrete example of the weak encapsulation of prior jaav 9 is the internal API sun.misc.Unsafe that was accessible though it should not)

✅ **How modules solve it:**

With Java modules, you can **explicitly control** which packages are exposed.

```java
module com.example.library {
    exports com.example.api; // Only API is accessible
}
```

Now, `com.example.internal` remains **completely hidden**, even though it's inside the same JAR.

With Java 8, you may think of making the classes inside of com.package.internal private so that the outside world cant use them, but then, even the api package com.example.api which is inside the same jar, cant use them.

So, this is a key limitation of java 8,  It doesn't allow **internal sharing** within a JAR **without also exposing it to everyone**.

## better performance

before java 9:

- Every Java program, whether small or large, had to rely on the **entire Java Standard Library** (all `rt.jar` and `tools.jar`).
- Even if your application only needed **a few features** (e.g., collections, IO, networking), it still had to carry the **entire JDK** at runtime.
- There was **no way to remove unnecessary parts**, leading to **large memory usage and slow startup times**.

`rt.jar` (short for **runtime JAR**) was a **monolithic JAR file** that contained **all** the core Java classes used by the Java Runtime Environment (JRE).

It was located in: <JAVA_HOME>/jre/lib/rt.jar

And was automatically loaded by the JVM whenever a Java program was executed.

To understand why having this huge jar file that contained all java classes (even the ones the app does not need) leads to bad performance and slow startup time we need to understand how a java program is executed:

1. you write your program:

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}

```

This program uses:

- `System.out.println()`, which comes from **`java.lang.System`**
- `String[] args`, which is a **Java core class** (`java.lang.String`)

These classes **are not inside your project**—they come from **the JDK**, specifically from `rt.jar`.

2. you compile the program:

```powershell
javac Main.java
```

- The compiler **looks for standard Java classes (e.g., `System`, `String`) inside `rt.jar`** to resolve dependencies
- This generates `Main.class`, which contains **Java bytecode**.

3. You Run the Program

```bash
java Main
```

At this point, the **JVM (Java Virtual Machine) starts up** and follows these steps:

JVM Starts and Loads the Bootstrap Class Loader

- The JVM **needs to execute `System.out.println()`**, but `System` is **not inside your project**.
- The **Bootstrap ClassLoader** tries to find `System.class` inside `rt.jar` in order to load it into memory
- *This is the crucial point:* Because rt.jar was so large, this search process involved scanning a massive file containing thousands of classes, even though your program only needed a tiny fraction of them. This linear search (or even indexed search) through a large JAR file introduced significant overhead, slowing down startup time.

so performance impact of the monolithic `rt.jar` came from startup time spent indexing the entire rt.jar to build the class lookup tables 

Java 9 introduced the `Java Platform Module System (JPMS)`, which broke down this monolithic `rt.jar` into about 70 distinct modules.

Each module, focused on a specific set of related functionality. For example:

- **java.base:** This is the *fundamental* module. It contains the core classes essential for almost every Java application, such as java.lang.* (e.g., String, Object, System), java.util.* (collections, etc.), and java.io.*. *Every module implicitly depends on java.base.*
- **java.sql:** Contains classes for database connectivity (JDBC).
- **java.xml:** Provides tools for XML processing.
- **java.desktop:** Contains AWT and Swing, used for building graphical user interfaces (GUIs).
- **java.logging:** Contains the Java Logging API.

Now, when you run a Java application, it only loads the modules it actually needs. If your application just does some basic string processing and math, it might only need the java.base module, leaving all the other modules unloaded.

With fewer classes to load and initialize, the JVM starts up much faster. The class loader has a much smaller set of modules to search, drastically reducing the time spent resolving class dependencies.

Another benefit of modules is **Strong Encapsulation:** Modules have *explicit dependencies*. A module must declare which other modules it relies on. Furthermore, a module only exposes specific packages for use by other modules. This strong encapsulation:

- **Prevents Accidental Dependencies:** You can't accidentally use internal JDK classes that were never intended for public use. This improves the long-term maintainability of your code.
- **Improves Security:** By limiting the exposed surface area of the JDK, the attack surface is reduced.

## Reducing Application Size

One of the most compelling advantages of adopting Java modules (introduced in Java 9) is the ability to **radically shrink the size of your deployed application runtime.** This is especially critical for developers distributing self-contained applications to end-users who may not have Java installed

If you want to create a java app that will be used without needing java to be installed on that machine, you need to bundle the JRE  ( monolithic ~200MB package) with the app (even if the app is a simple CLI tool requiring just java.base)

Java 9 modules address this problem head-on by enabling **custom runtime images** via the `jlink` tool.

By declaring explicit dependencies in `module-info.java`, you define exactly which modules your app requires (e.g., `java.sql`, `jdk.httpserver`

The `jlink` tool analyzes your app’s module dependencies and creates a **minimal runtime image** that includes:

- Only the modules your app actually uses.

For example:

- A simple app requiring only `java.base` (the core module) could produce a runtime as small as **~40MB**.
- Even apps with moderate dependencies (e.g., a REST server using `java.net`, `java.sql`, and `jdk.httpserver`) often stay under **60–80MB**—less than half the size of the full JRE.

## Practical example of working with modules

Create this project structure:

```bash
java-modules-example/
│── modules/
│   ├── com.example.library/
│   │   ├── src/
│   │   │   ├── com/example/library/api/LibraryService.java
│   │   │   ├── com/example/library/internal/InternalUtil.java
│   │   ├── module-info.java
│   ├── com.example.app/
│   │   ├── src/
│   │   │   ├── com/example/app/MainApp.java
│   │   ├── module-info.java

```

Define the `com.example.library` Module in `com.example.library/module-info.java`

```java
module com.example.library {
    exports com.example.library.api;  // Only API is accessible
}
```

Exposed API (`com.example.library.api.LibraryService`)

```java
package com.example.library.api;

public class LibraryService {
    public String getMessage() {
        return "Hello from Library API!";
    }
}
```

Internal Class (`com.example.library.internal.InternalUtil`)

```java
package com.example.library.internal;

public class InternalUtil {
    public static String secret() {
        return "This is an internal secret!";
    }
}
```

- `InternalUtil` is **not accessible** outside the module.

Define the `com.example.app` Module`com.example.app/module-info.java`

```java

module com.example.app {
    requires com.example.library;
}
```

Main Application (`com.example.app.MainApp`)

```java
package com.example.app;

import com.example.library.api.LibraryService;

public class MainApp {
    public static void main(String[] args) {
        LibraryService service = new LibraryService();
        System.out.println(service.getMessage());
    }
}
```

- This app **can use** `LibraryService` but **cannot access** `InternalUtil`.

Compile the Modules:

compile module `com.example.library`

```bash
javac -d target/com.example.library $(find modules/com.example.library/src -name "*.java") modules/com.example.library/module-info.java
```

- **`javac`** → Java compiler command.
- **-`d target/com.example.library`** → Specifies the output directory for compiled `.class` files.
- **`$(find modules/com.example.library/src -name "*.java")`** → Finds all `.java` files in the `src` folder of `com.example.library`.
- **`modules/com.example.library/module-info.java`** → Ensures that the `module-info.java` file is also compiled.

compile module `com.example.app`

```bash
javac --module-path target -d target/com.example.app $(find modules/com.example.app/src -name "*.java") modules/com.example.app/module-info.java

```

- **`javac`** → Java compiler.
- **`--module-path` target** → Specifies where to find required modules (`com.example.library`).
- **`-d modules/com.example.app`** → Outputs compiled class files into the `com.example.app` directory.
- **`$(find modules/com.example.app/src -name "*.java")`** → Finds all `.java` files in the `src` folder of `com.example.app`.
- **`modules/com.example.app/module-info.java`** → Ensures that `module-info.java` is also compiled.

🔹 Here, `--module-path` target is crucial because it tells the compiler **where to find required modules** (`com.example.library`).

After compiling both modules, here is the file structure:

```bash
PS C:\Users\NCDD0815\work\tests\java-modules-example> tree /F
Folder PATH listing for volume System
Volume serial number is FE20-6F30
C:.
├───modules
│   ├───com.example.app
│   │   │   module-info.java
│   │   │
│   │   └───src
│   │       └───com
│   │           └───example
│   │               └───app
│   │                       MainApp.java
│   │
│   └───com.example.library
│       │   module-info.java
│       │
│       ├───com
│       │   └───example
│       └───src
│           └───com
│               └───example
│                   └───library
│                       ├───api
│                       │       LibraryService.java
│                       │
│                       └───internal
│                               InternalUtil.java
│
└───target
    ├───com.example.app
    │   │   module-info.class
    │   │
    │   └───com
    │       └───example
    │           └───app
    │                   MainApp.class
    │
    └───com.example.library
        │   module-info.class
        │
        └───com
            └───example
                └───library
                    ├───api
                    │       LibraryService.class
                    │
                    └───internal
                            InternalUtil.class
```

Run the main class from module com.example.app

```bash
java --module-path target -m com.example.app/com.example.app.MainApp
```

- **`java`** → Runs the Java application.
- **`--module-path` target** → Specifies where to find the compiled modules.
- **`-m com.example.app/com.example.app.MainApp`** → Runs the `MainApp` class from the `com.example.app` module (notice the format **-m module/class**)
    - **`com.example.app`** → The module name.
    - **`com.example.app.MainApp`** → The fully qualified class name with `main` method.



# Java 10 feature: JVM container awarness

Java 10 introduced container awarness in the JVM, allowing it to detect that it being run inside a container and by that knowing the CPU and memory limits inside the container.

Before Java 10, the JVM **ignored container limits** and assumed full access to the **host machine’s resources**. This led to:

- **Excessive memory usage** → OOM (Out of Memory) errors.
- **Inefficient CPU allocation** → JVM using all available cores, affecting performance.

With **Java 10+,** the JVM correctly respects **container constraints** when running in a container.

We can test this using java 9 and java 10.

Lets run a container for jdk 9

```bash
docker container run -it --cpus=2 openjdk:9-jdk
```

we’ll have a jshell prompt because openjdk:9-jdk is configued to lanuch it by default when using interactive mode -it

Now, run 

```java
jshell> Runtime.getRuntime().availableProcessors();
$1 ==> 8
```

You see, 8, because the host machine has 8 cores, so JVM incorretly sees the host CPUs and not the container CPUs

Now lets run an container instance of a jdk 10 image

```bash
docker container run -it --cpus=2 openjdk:10-jdk
```

Now run:

```java
shell> Runtime.getRuntime().availableProcessors();
$1 ==> 2
```

JVM now correctly detects only 2 CPUs allocated for the container 🙂