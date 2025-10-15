# Quick Start — Kotlin Compiler Plugin Template

This repository is a ready‑to‑use template for building Kotlin compiler plugins on K2 (FIR) with an IR backend phase, plus a Gradle subplugin for effortless consumption.

What the sample plugin does:
- Registers a K2 FIR declaration generator that creates a top‑level class `foo.bar.MyClass`.
- Supplies IR bodies so that `MyClass().foo()` returns `"Hello world"`.

Key pieces:
- `:compiler-plugin` — the compiler plugin (FIR + IR) and service registrations.
- `:plugin-annotations` — annotations library you can depend on from user code.
- `:gradle-plugin` — Gradle subplugin wiring the compiler plugin into Kotlin compilations.

Note: the template doesn’t use `SomeAnnotation` yet; it’s there as a starting point so you can mark declarations to transform later. See GUIDE.md for how to evolve in that direction.

Minimum requirements:
- JDK 17+
- Gradle Wrapper (provided)
- Kotlin 2.2.0

## 1) Build and test

- Build everything:
  ```bash
  ./gradlew build
  ```
- Run compiler plugin tests (Kotlin Compiler Test Framework):
  ```bash
  ./gradlew :compiler-plugin:test
  ```
  Notes:
  - Test sources live in `compiler-plugin/testData` with expected FIR/IR outputs.
  - Generated JUnit suites are written to `compiler-plugin/test-gen` via the `generateTests` task (wired automatically).

## 2) Try the plugin in a consumer project quickly

You have two convenient options.

### Option A — Composite build (no publishing)
Point your consumer project to this repo so Gradle can resolve the subplugin by ID.

- In the consumer project’s `settings.gradle.kts`:
  ```kotlin
  pluginManagement {
      repositories {
          gradlePluginPortal()
          mavenCentral()
      }
      // Make the Gradle subplugin from this repo available by its ID
      includeBuild("/absolute/path/to/compiler-plugin-template")
  }
  ```
- In the consumer project’s `build.gradle.kts`:
  ```kotlin
  plugins {
      kotlin("jvm") version "2.2.0"
      // Plugin ID equals this repo’s group (see root build.gradle.kts)
      id("org.jetbrains.kotlin.compiler.plugin.template")
  }

  repositories { mavenCentral() }

  // Optional: subplugin extension (empty in the template)
  simplePlugin { }
  ```

Run a build; the subplugin will attach the compiler plugin and add the annotations dependency to the relevant source sets automatically.

### Option B — Attach the plugin JAR via -Xplugin
If you prefer raw CLI wiring:

1) Build the compiler plugin JAR:
```bash
./gradlew :compiler-plugin:jar
```

2) In the consumer project, point Kotlin to that JAR (example for JVM):
```kotlin
// build.gradle.kts in the consumer project
plugins {
    kotlin("jvm") version "2.2.0"
}

repositories { mavenCentral() }

dependencies {
    // If your plugin exposes annotations you use in the consumer code
    implementation("org.jetbrains.kotlin.compiler.plugin.template:plugin-annotations:0.1.0-SNAPSHOT")
}

tasks.withType<org.jetbrains.kotlin.gradle.tasks.KotlinCompile>().configureEach {
    kotlinOptions.freeCompilerArgs += listOf(
        "-Xplugin=/absolute/path/to/compiler-plugin-template/compiler-plugin/build/libs/compiler-plugin-0.1.0-SNAPSHOT.jar"
    )
}
```

> Tip: adjust the path to your cloned template’s JAR.

## 3) Validate it works
In your consumer project, call the generated code:
```kotlin
fun main() {
    println(foo.bar.MyClass().foo()) // -> Hello world
}
```
If you used Option A (composite build), just run `./gradlew run` or `./gradlew build` depending on your setup.

## 4) Where to tweak next
- FIR generator entry point: `compiler-plugin/src/.../fir/SimpleClassGenerator.kt`.
- IR body generator: `compiler-plugin/src/.../ir/SimpleIrBodyGenerator.kt`.
- Extension registration: `SimplePluginComponentRegistrar` and `SimplePluginRegistrar`.
- Gradle subplugin: `gradle-plugin/src/.../SimpleGradlePlugin.kt` (+ `SimpleGradleExtension`).
- Annotations: `plugin-annotations/src/.../SomeAnnotation.kt`.
- Service registration files: `compiler-plugin/resources/META-INF/services/...`.
- CLI options hook: `SimpleCommandLineProcessor` (currently no options).

## 5) Useful commands
- Build all: `./gradlew build`
- Tests only: `./gradlew :compiler-plugin:test`
- Regenerate JUnit suites: `./gradlew :compiler-plugin:generateTests`

For a complete walkthrough, see GUIDE.md.
