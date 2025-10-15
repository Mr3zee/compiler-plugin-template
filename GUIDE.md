# Guide — Build this Kotlin Compiler Plugin from Scratch

This is a step‑by‑step walkthrough to build, from an empty Gradle project, exactly what this repository contains: a Kotlin compiler plugin targeting K2 (FIR) with an IR backend step, a Gradle subplugin, a tiny annotations library, and tests based on the Kotlin Compiler Test Framework.

End result:
- A plugin that generates a top‑level class `foo.bar.MyClass` with a function `foo(): String`.
- IR codegen supplies the method body so `MyClass().foo()` returns `"Hello world"`.

Assumed toolchain:
- JDK 17+
- Gradle Wrapper (provided by this repo)
- Kotlin 2.2.0

Repository layout we are aiming for:
- `:compiler-plugin` — the compiler plugin (K2 FIR + IR), service registration files, test infra wiring.
- `:plugin-annotations` — annotations library (Kotlin Multiplatform) to use from consumer code.
- `:gradle-plugin` — Gradle subplugin that wires the compiler plugin into Kotlin compilations and adds the annotations dependency.

If you prefer a quick overview instead of a full tutorial, see QUICK_START.md.

---

## 0) Initialize the Gradle multi‑module build

Create the root files.

- `settings.gradle.kts`:
  ```kotlin
  pluginManagement {
      repositories {
          mavenCentral()
          gradlePluginPortal()
      }
  }

  dependencyResolutionManagement {
      repositories {
          mavenCentral()
      }
  }

  rootProject.name = "compiler-plugin-template"

  include("compiler-plugin")
  include("gradle-plugin")
  include("plugin-annotations")
  ```

- `build.gradle.kts`:
  ```kotlin
  plugins {
      kotlin("multiplatform") version "2.2.0" apply false
      kotlin("jvm") version "2.2.0" apply false
      id("com.github.gmazzo.buildconfig") version "5.6.5"
      id("org.jetbrains.kotlinx.binary-compatibility-validator") version "0.16.3" apply false
  }

  allprojects {
      group = "org.jetbrains.kotlin.compiler.plugin.template"
      version = "0.1.0-SNAPSHOT"
  }
  ```

  Note on plugins in the root build script
  - com.github.gmazzo.buildconfig: generates a `BuildConfig` source in each module that applies it. We use it to expose constants such as `KOTLIN_PLUGIN_ID`, the plugin group/name/version, and the annotations library coordinates so both the Gradle subplugin and the compiler plugin share the same values without duplication.
  - org.jetbrains.kotlinx.binary-compatibility-validator: provides `apiDump` and `apiCheck` tasks to track binary API compatibility. In this repo it is applied to `:plugin-annotations` to keep its public API stable. Useful commands:
    - `./gradlew :plugin-annotations:apiDump` — record/update the API under `plugin-annotations/api/`
    - `./gradlew :plugin-annotations:apiCheck` — verify changes against the dump, failing the build on incompatible changes.

---

## 1) Create the annotations module (`:plugin-annotations`)

Purpose: Types available to end users; the Gradle subplugin will automatically add this dependency to their compilations.

- `plugin-annotations/build.gradle.kts` (Kotlin Multiplatform minimal setup):
  ```kotlin
  @file:OptIn(org.jetbrains.kotlin.gradle.ExperimentalWasmDsl::class)

  plugins {
      kotlin("multiplatform")
      id("org.jetbrains.kotlinx.binary-compatibility-validator")
  }

  kotlin {
      explicitApi()

      androidNativeArm32()
      androidNativeArm64()
      androidNativeX64()
      androidNativeX86()

      iosArm64(); iosSimulatorArm64(); iosX64()
      js().nodejs()
      jvm()
      linuxArm64(); linuxX64()
      macosArm64(); macosX64()
      mingwX64()
      tvosArm64(); tvosSimulatorArm64(); tvosX64()
      wasmJs().nodejs(); wasmWasi().nodejs()
      watchosArm32(); watchosArm64(); watchosDeviceArm64(); watchosSimulatorArm64(); watchosX64()

      applyDefaultHierarchyTemplate()
  }
  ```

  Note: the `binary-compatibility-validator` plugin is applied here to track the public API of this library. The current API dump lives in `plugin-annotations/api/`. Run `./gradlew :plugin-annotations:apiCheck` on CI and `./gradlew :plugin-annotations:apiDump` when you intentionally change the public API.

- Add an example annotation at `plugin-annotations/src/commonMain/kotlin/org/jetbrains/kotlin/compiler/plugin/template/SomeAnnotation.kt`:
  ```kotlin
  public annotation class SomeAnnotation
  ```

  Note about `SomeAnnotation`
  - The template does not consume `@SomeAnnotation` anywhere yet. It is published as a small annotations library so your users can depend on it from day one.
  - Typical next step: mark the classes/functions you want to process with `@SomeAnnotation`, and have your FIR/IR extensions discover those declarations to generate code or transform bodies.
    - FIR idea: extend your FIR extension(s) to look up declarations annotated with `@SomeAnnotation` and generate related symbols or diagnostics.
    - IR idea: in an IR generation/transform step, check `declaration.annotations` for `@SomeAnnotation` to decide whether to modify a body or synthesize members.
  - The Gradle subplugin in this template already adds the annotations dependency to all Kotlin compilations automatically, so consumers can use `@SomeAnnotation` without extra setup.

---

## 2) Create the compiler plugin module (`:compiler-plugin`)

This houses the K2 FIR and IR logic, registers extensions, and wires tests.

- `compiler-plugin/build.gradle.kts` essentials:
  ```kotlin
  plugins {
      kotlin("jvm")
      `java-test-fixtures`
      id("com.github.gmazzo.buildconfig")
      idea
  }

  sourceSets {
      main {
          java.setSrcDirs(listOf("src"))
          resources.setSrcDirs(listOf("resources"))
      }
      testFixtures { java.setSrcDirs(listOf("test-fixtures")) }
      test {
          java.setSrcDirs(listOf("test", "test-gen"))
          resources.setSrcDirs(listOf("testData"))
      }
  }

  val annotationsRuntimeClasspath: Configuration by configurations.creating { isTransitive = false }

  dependencies {
      compileOnly(kotlin("compiler"))

      testFixturesApi(kotlin("test-junit5"))
      testFixturesApi(kotlin("compiler-internal-test-framework"))
      testFixturesApi(kotlin("compiler"))

      annotationsRuntimeClasspath(project(":plugin-annotations"))

      testRuntimeOnly("junit:junit:4.13.2")
      testRuntimeOnly(kotlin("reflect"))
      testRuntimeOnly(kotlin("test"))
      testRuntimeOnly(kotlin("script-runtime"))
      testRuntimeOnly(kotlin("annotations-jvm"))
  }

  buildConfig {
      useKotlinOutput { internalVisibility = true }
      packageName(group.toString())
      buildConfigField("String", "KOTLIN_PLUGIN_ID", "\"${rootProject.group}\"")
  }

  tasks.test {
      dependsOn(annotationsRuntimeClasspath)
      useJUnitPlatform()
      workingDir = rootDir
      systemProperty("annotationsRuntime.classpath", annotationsRuntimeClasspath.asPath)
      // Additional properties for the internal test framework are set in the real file
  }

  kotlin { compilerOptions { optIn.add("org.jetbrains.kotlin.compiler.plugin.ExperimentalCompilerApi") } }

  // Test generation — see the real file for details
  ```

  Note: the `com.github.gmazzo.buildconfig` plugin is used here to generate `BuildConfig` with `KOTLIN_PLUGIN_ID`. The compiler plugin reads it in `SimpleCommandLineProcessor` and the Gradle subplugin reads the same value to ensure both sides agree on the plugin ID without hardcoding it twice.

### 2.1 Service registrations

Create the following files so Kotlin locates your plugin at runtime:
- `compiler-plugin/resources/META-INF/services/org.jetbrains.kotlin.compiler.plugin.CommandLineProcessor`
- `compiler-plugin/resources/META-INF/services/org.jetbrains.kotlin.compiler.plugin.CompilerPluginRegistrar`

Each file contains a single FQCN line pointing to your implementation classes (they are present in this template under the same package as the sources).

### 2.2 Register compiler extensions

- `compiler-plugin/src/.../SimplePluginComponentRegistrar.kt` registers K2 FIR and IR extensions:
  ```kotlin
  class SimplePluginComponentRegistrar : CompilerPluginRegistrar() {
      override val supportsK2: Boolean get() = true
      override fun ExtensionStorage.registerExtensions(configuration: CompilerConfiguration) {
          FirExtensionRegistrarAdapter.registerExtension(SimplePluginRegistrar())
          IrGenerationExtension.registerExtension(SimpleIrGenerationExtension())
      }
  }
  ```

- `compiler-plugin/src/.../SimplePluginRegistrar.kt` registers the FIR generator:
  ```kotlin
  class SimplePluginRegistrar : FirExtensionRegistrar() {
      override fun ExtensionRegistrarContext.configurePlugin() {
          +::SimpleClassGenerator
      }
  }
  ```

### 2.3 FIR: generate declarations

- `compiler-plugin/src/.../fir/SimpleClassGenerator.kt` declares a top‑level class and its members:
  ```kotlin
  class SimpleClassGenerator(session: FirSession) : FirDeclarationGenerationExtension(session) {
      companion object {
          val MY_CLASS_ID = ClassId(FqName.fromSegments(listOf("foo", "bar")), Name.identifier("MyClass"))
          val FOO_ID = CallableId(MY_CLASS_ID, Name.identifier("foo"))
      }

      @ExperimentalTopLevelDeclarationsGenerationApi
      override fun generateTopLevelClassLikeDeclaration(classId: ClassId): FirClassLikeSymbol<*>? {
          if (classId != MY_CLASS_ID) return null
          val klass = createTopLevelClass(MY_CLASS_ID, Key)
          return klass.symbol
      }

      override fun generateConstructors(context: MemberGenerationContext): List<FirConstructorSymbol> {
          val classId = context.owner.classId
          require(classId == MY_CLASS_ID)
          val constructor = createConstructor(context.owner, Key, /*generateDelegatedNoArgConstructorCall = true*/)
          return listOf(constructor.symbol)
      }

      override fun generateFunctions(
          callableId: CallableId,
          context: MemberGenerationContext?
      ): List<FirNamedFunctionSymbol> {
          val owner = context?.owner ?: return emptyList()
          val function = createMemberFunction(
              owner,
              Key,
              callableId.callableName,
              returnType = session.builtinTypes.stringType.coneType
          )
          return listOf(function.symbol)
      }

      override fun getCallableNamesForClass(
          classSymbol: FirClassSymbol<*>,
          context: MemberGenerationContext
      ): Set<Name> {
          return if (classSymbol.classId == MY_CLASS_ID) {
              setOf(FOO_ID.callableName, SpecialNames.INIT)
          } else {
              emptySet()
          }
      }

      @ExperimentalTopLevelDeclarationsGenerationApi
      override fun getTopLevelClassIds(): Set<ClassId> = setOf(MY_CLASS_ID)

      override fun hasPackage(packageFqName: FqName): Boolean =
          packageFqName == MY_CLASS_ID.packageFqName

      object Key : GeneratedDeclarationKey()
  }
  ```

  Explanation
  - `MY_CLASS_ID` identifies the generated top-level class `foo.bar.MyClass`; `FOO_ID` identifies its `foo()` member.
  - `generateTopLevelClassLikeDeclaration` creates the class symbol when the frontend asks for `MY_CLASS_ID`.
  - `generateConstructors` contributes a default constructor for `MyClass`.
  - `generateFunctions` contributes `fun foo(): String` and declares its return type as `kotlin.String`.
  - `getTopLevelClassIds`/`hasPackage` advertise the presence of the generated class to the frontend.
  - `getCallableNamesForClass` announces which member names may be generated for this class (constructor and `foo`).
  - `Key` tags generated declarations so the IR pass can recognize them and provide bodies.

  More about the FIR plugins here: https://github.com/JetBrains/kotlin/blob/master/docs/fir/fir-plugins.md

### 2.4 IR: generate bodies

- `compiler-plugin/src/.../ir/SimpleIrBodyGenerator.kt` assigns the bodies for generated declarations:
  ```kotlin
  class SimpleIrBodyGenerator(pluginContext: IrPluginContext) : AbstractTransformerForGenerator(pluginContext) {
      override fun interestedIn(key: GeneratedDeclarationKey?): Boolean {
          return key == SimpleClassGenerator.Key
      }

      override fun generateBodyForFunction(function: IrSimpleFunction, key: GeneratedDeclarationKey?): IrBody {
          require(function.name == SimpleClassGenerator.FOO_ID.callableName)
          // '-1' offsets are for synthetic nodes
          val const = IrConstImpl(-1, -1, irBuiltIns.stringType, IrConstKind.String, value = "Hello world")
          val returnStatement = IrReturnImpl(-1, -1, irBuiltIns.nothingType, function.symbol, const)
          return irFactory.createBlockBody(-1, -1, listOf(returnStatement))
      }

      override fun generateBodyForConstructor(constructor: IrConstructor, key: GeneratedDeclarationKey?): IrBody? {
          return generateBodyForDefaultConstructor(constructor)
      }
  }
  ```

  Explanation
  - `interestedIn` ensures we only touch declarations generated by our FIR extension (tagged with `SimpleClassGenerator.Key`).
  - `generateBodyForFunction` returns a block body with a single `return "Hello world"`.
  - `generateBodyForConstructor` delegates to a helper that emits a default constructor body.
  - Synthetic IR nodes use `-1` offsets; real PSI positions are not required for these generated bodies.

### 2.5 Command line processor (plugin ID and options)

- `compiler-plugin/src/.../SimpleCommandLineProcessor.kt`:
  ```kotlin
  class SimpleCommandLineProcessor : CommandLineProcessor {
      override val pluginId: String = BuildConfig.KOTLIN_PLUGIN_ID
      override val pluginOptions: Collection<CliOption> = emptyList()
      override fun processOption(
          option: AbstractCliOption,
          value: String,
          configuration: CompilerConfiguration
      ) { error("Unexpected config option: '${option.optionName}'") }
  }
  ```

> At this point the compiler plugin already generates `foo.bar.MyClass` and `foo()`; K2 FIR creates the declarations, IR provides bodies.

---

## 3) Create the Gradle subplugin module (`:gradle-plugin`)

The subplugin makes consumption easy: users add one Gradle plugin ID and get your compiler plugin attached to all Kotlin compilations.

- `gradle-plugin/build.gradle.kts` essentials:
  ```kotlin
  plugins {
      kotlin("jvm")
      id("com.github.gmazzo.buildconfig")
      id("java-gradle-plugin")
  }

  dependencies { implementation(kotlin("gradle-plugin-api")) }

  buildConfig {
      packageName(project.group.toString())
      buildConfigField("String", "KOTLIN_PLUGIN_ID", "\"${rootProject.group}\"")
      val pluginProject = project(":compiler-plugin")
      buildConfigField("String", "KOTLIN_PLUGIN_GROUP", "\"${pluginProject.group}\"")
      buildConfigField("String", "KOTLIN_PLUGIN_NAME", "\"${pluginProject.name}\"")
      buildConfigField("String", "KOTLIN_PLUGIN_VERSION", "\"${pluginProject.version}\"")
      val annotationsProject = project(":plugin-annotations")
      buildConfigField(
          type = "String",
          name = "ANNOTATIONS_LIBRARY_COORDINATES",
          expression = "\"${annotationsProject.group}:${annotationsProject.name}:${annotationsProject.version}\""
      )
  }

  gradlePlugin {
      plugins {
          create("SimplePlugin") {
              id = rootProject.group.toString()
              displayName = "SimplePlugin"
              description = "SimplePlugin"
              implementationClass = "org.jetbrains.kotlin.compiler.plugin.template.SimpleGradlePlugin"
          }
      }
  }
  ```

- Subplugin implementation at `gradle-plugin/src/.../SimpleGradlePlugin.kt`:
  ```kotlin
  class SimpleGradlePlugin : KotlinCompilerPluginSupportPlugin {
      override fun apply(target: Project) {
          target.extensions.create("simplePlugin", SimpleGradleExtension::class.java)
      }
      override fun isApplicable(kotlinCompilation: KotlinCompilation<*>) = true
      override fun getCompilerPluginId(): String = BuildConfig.KOTLIN_PLUGIN_ID
      override fun getPluginArtifact() = SubpluginArtifact(
          groupId = BuildConfig.KOTLIN_PLUGIN_GROUP,
          artifactId = BuildConfig.KOTLIN_PLUGIN_NAME,
          version = BuildConfig.KOTLIN_PLUGIN_VERSION,
      )
      override fun applyToCompilation(kotlinCompilation: KotlinCompilation<*>) =
          kotlinCompilation.target.project.provider {
              // Ensure user code sees the annotations
              kotlinCompilation.dependencies { implementation(BuildConfig.ANNOTATIONS_LIBRARY_COORDINATES) }
              emptyList<SubpluginOption>()
          }
  }
  ```

- (Optional) Subplugin extension at `gradle-plugin/src/.../SimpleGradleExtension.kt` for user‑configurable options.

---

## 4) Wire tests with the Kotlin Compiler Test Framework

This repository wires the Kotlin internal compiler test framework so you can run FIR (frontend) and IR (codegen) tests locally and on CI.

What lives where
- Test data under `compiler-plugin/testData`:
  - `box/` — IR/codegen tests (assert runtime behavior or resulting IR)
  - `diagnostics/` — FIR/frontend tests (assert diagnostics and symbol generation)
- Generated JUnit 5 suites under `compiler-plugin/test-gen` (created from the test data). Do not edit these files manually.
- Test fixtures under `compiler-plugin/test-fixtures` provide runners and services:
  - `.../runners/AbstractJvmBoxTest.kt`
  - `.../runners/AbstractJvmDiagnosticTest.kt`
  - `.../services/ExtensionRegistrarConfigurator.kt` — registers your FIR and IR extensions for tests
  - `.../services/PluginAnnotationsProvider.kt` — supplies the annotations classpath to the test framework

Build and test configuration (see `compiler-plugin/build.gradle.kts`)
- A detached configuration `annotationsRuntimeClasspath` contains `:plugin-annotations`; tests receive it via `systemProperty("annotationsRuntime.classpath", annotationsRuntimeClasspath.asPath)`.
- JUnit Platform is enabled; `workingDir = rootDir` for predictable relative paths.
- The Kotlin internal test framework needs explicit paths to core Kotlin jars. The build sets them via helper `setLibraryProperty(...)` calls for:
  - `org.jetbrains.kotlin.test.kotlin-stdlib`
  - `org.jetbrains.kotlin.test.kotlin-stdlib-jdk8`
  - `org.jetbrains.kotlin.test.kotlin-reflect`
  - `org.jetbrains.kotlin.test.kotlin-test`
  - `org.jetbrains.kotlin.test.kotlin-script-runtime`
  - `org.jetbrains.kotlin.test.kotlin-annotations-jvm`
- Test generation: task `:compiler-plugin:generateTests` scans `testData/` and writes sources to `test-gen/`. Task `compileTestKotlin` depends on it, so tests are always (re)generated before compilation.

Common commands
```bash
./gradlew :compiler-plugin:generateTests
./gradlew :compiler-plugin:test
```

Auto-update expected test data dumps 
You can instruct the Kotlin compiler test framework to rewrite expected FIR/IR dumps under `compiler-plugin/testData/**` by running tests with the system property `kotlin.test.update.test.data=true`:
```bash
./gradlew :compiler-plugin:test -Pkotlin.test.update.test.data=true
```

Tips
- Inspect the generated suites under `compiler-plugin/test-gen/...` if you wonder how a given test is wired.
- Add new `.kt` files and expected outputs under `testData/box` and `testData/diagnostics` to grow coverage.

> Test Framework is updated from time to time. If you see test failures, try updating the test framework from the latest version of this repository. 

---

## 5) Build and smoke test the plugin

Build everything:
```bash
./gradlew build
```

Optionally, try it in a separate consumer project.

### Option A — Composite build
Add this to the consumer’s `settings.gradle.kts` to expose the subplugin by ID:
```kotlin
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
    includeBuild("/absolute/path/to/compiler-plugin-template")
}
```
Then in the consumer’s `build.gradle.kts`:
```kotlin
plugins {
    kotlin("jvm") version "2.2.0"
    id("org.jetbrains.kotlin.compiler.plugin.template")
}

repositories { mavenCentral() }

simplePlugin { }
```

### Option B — Attach the plugin JAR via -Xplugin
```bash
./gradlew :compiler-plugin:jar
```
Then in the consumer build script, add a free compiler arg pointing to that JAR and, if needed, the annotations dependency (see QUICK_START.md for an example snippet).

A minimal code check:
```kotlin
fun main() {
    println(foo.bar.MyClass().foo()) // -> Hello world
}
```

---

## 6) Next steps: customize the plugin

Places to change:
- Change what is generated in FIR: `compiler-plugin/src/.../fir/SimpleClassGenerator.kt`.
- Change bodies or add IR passes: `compiler-plugin/src/.../ir/`.
- Expose user options: add fields to `SimpleGradleExtension`, propagate them in `SimpleGradlePlugin.applyToCompilation`, declare matching `CliOption`s in `SimpleCommandLineProcessor`, and read them in your extensions.
- Add/modify annotations in `:plugin-annotations`.
- Expand tests by adding `.kt` files and expected outputs in `testData/box` and `testData/diagnostics`.

That’s it — you now have a full baseline to evolve your own Kotlin compiler plugin.
