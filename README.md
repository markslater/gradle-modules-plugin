[![Build Status](https://img.shields.io/github/workflow/status/java9-modularity/gradle-modules-plugin/Build)](https://github.com/java9-modularity/gradle-modules-plugin/actions?query=workflow%3A%22Build%22)


Introduction
===

This Gradle plugin helps working with the Java Platform Module System.
The plugin is published in the [Gradle plugin repository](https://plugins.gradle.org/plugin/org.javamodularity.moduleplugin).
It makes building, testing and running modules seamless from the Gradle perspective.
It sets up compiler and jvm settings with flags such as `--module-path`, so that you can build, test and run JPMS modules without manually setting up your build files.

:bulb: When using this plugin, you should not set the `--module-path` compiler option explicitly.
Also, you should disable the `modularity.inferModulePath` option introduced in Gradle 6.4:
`modularity.inferModulePath.set(false)`

The plugin is designed to work in repositories that contain multiple modules.
The plugin currently supports:

* Compiling modules
* Testing module code with whitebox tests (traditional unit tests)
* Testing modules blackbox (testing module boundaries and services)
* Running/packaging modular applications using the application plugin

The plugin supports the following test engines:

* JUnit 5
* JUnit 4
* TestNG
* ![Since 1.7.0](https://img.shields.io/badge/since-1.7.0-brightgreen) Spock 2 with Groovy 3
* ![Since 1.7.0](https://img.shields.io/badge/since-1.7.0-brightgreen) AssertJ
* ![Since 1.7.0](https://img.shields.io/badge/since-1.7.0-brightgreen) Mockito
* ![Since 1.7.0](https://img.shields.io/badge/since-1.7.0-brightgreen) EasyMock

An example application using this plugin is available [here](https://github.com/java9-modularity/gradle-modules-plugin-example).

Setup
===

For this guide we assume the following directory structure:

```
.
├── build.gradle
├── gradle
├── greeter.api
├── greeter.javaexec
├── greeter.provider
├── greeter.provider.test
├── greeter.provider.testfixture
├── greeter.runner
├── greeter.startscripts
└── settings.gradle
```

* greeter.api: Exports an interface
* greeter.javaexec: Applications that can be started with `ModularJavaExec` tasks
* greeter.provider: Provides a service implementation for the interface provided by `greeter.api`
* greeter.provider.test: Blackbox module test for `greeter.provider`
* greeter.provider.testfixture: Blackbox module test with test fixtures for `greeter.provider`
* greeter.runner: Main class that uses the `Greeter` service, that can be started/packaged with the `application plugin`
* greeter.startscripts: Applications with start scripts generated by `ModularCreateStartScripts` tasks

The main build file should look as follows:

```groovy
plugins {
    id 'org.javamodularity.moduleplugin' version '1.8.13' apply false
}

subprojects {
    apply plugin: 'java'
    apply plugin: "org.javamodularity.moduleplugin"

    version "1.0-SNAPSHOT"

    sourceCompatibility = 11
    targetCompatibility = 11

    repositories {
        mavenCentral()
    }

    test {
        useJUnitPlatform()

        testLogging {
            events 'PASSED', 'FAILED', 'SKIPPED'
        }
    }

    dependencies {
        testImplementation 'org.junit.jupiter:junit-jupiter-api:5.3.1'
        testImplementation 'org.junit.jupiter:junit-jupiter-params:5.3.1'
        testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.3.1'
    }
}
```

The most important line in this build file is: `apply plugin: "org.javamodularity.moduleplugin"` which enables the module plugin for sub projects.
Because this is an external plugin, we need to tell Gradle where to find it, which is done in the `buildscript` section.
The subprojects typically don't need extra configuration in their build files related to modules.

To build the project simply run `./gradlew build` like you're used to.

Creating a module
===

The only thing that makes a module a module is the existence of `module-info.java`.
In a `module-info.java` you'll need to define the module name, and possibly declare your exports, dependencies on other modules, and service uses/provides.
A simple example is the following.
The module name is `greeter.api` and the module exports a package `examples.greeter.api`.

```java
module greeter.api {
    exports examples.greeter.api;
}
```

For Gradle, just make sure the plugin is applied.
This will make sure the correct compiler flags are used such as `--module-path` instead of `-cp`.

Setting the module version
---

![Since 1.7.0](https://img.shields.io/badge/since-1.7.0-brightgreen)
By default, the plugin uses the value of the project version to set the version of the module. You can configure a different module version using the `modularity.moduleVersion` method:
<details open>
<summary>Groovy DSL</summary>

```groovy
modularity.moduleVersion '1.2.3'
```

</details>
<details>
<summary>Kotlin DSL</summary>

```kotlin
modularity.moduleVersion("1.2.3")
```

</details>

If no project version is specified and you don't call the `modularity.moduleVersion` method, the module is created without a version.

Module dependencies
===

When a module depends on another module, the dependency needs to be declared in two different places.
First it needs to be declared in the `dependencies` section of the gradle build file.

```groovy
dependencies {
    implementation project(':greeter.api') //Example of dependency on another module in the project
    implementation "com.fasterxml.jackson.core:jackson-databind:2.9.5" //Example of an external dependency
}
```

Next, it needs to be defined in `module-info.java`.

```java
module greeter.provider {
    requires greeter.api; //This is another module provided by our project
    requires java.net.http; //This is a module provided by the JDK
    requires com.fasterxml.jackson.databind; //This is an external module
}
```

Note that the coordinates for the Gradle dependency are not necessarily the same as the module name!

Why do we need to define dependencies in two places!?
---
We need the Gradle definition so that during build time, Gradle knows how to locate the modules.
The plugin puts these modules on the `--module-path`.
Next, when Gradle invokes the Java compiler, the compiler is set up with the correct `--module-path` so that the compiler has access to them.
When using the module system the compiler checks dependencies and encapsulation based on the `requires`, `exports` and `opens` keywords in `module-info-java`.
These are related, but clearly two different steps.

MonkeyPatching the module
===
There are times when explicit modular settings may be needed on compile, test, and run tasks. You have the option to specify these settings using
a `moduleOptions` extension on the target task, for example

```
compileJava {
    moduleOptions {
        addModules = ['com.acme.foo']
    }
}
```

The following options are supported by the `moduleOptions` extension:

* `addModules`: Maps to `--add-modules`. Value is of type `List<String>`, e.g, `['com.acme.foo']`.
* `addReads`: Maps to `--add-reads`. Value is of type `Map<String, String>`, e.g, `['module1': 'module2']`.
* `addExports`: Maps to `--add-exports`. Value is of type `Map<String, String>`, e.g, `['module1/package': 'module2']`.
* `addOpens`: Maps to `--add-opens`. Value is of type `Map<String, String>`, e.g, `['module1/package': 'module2']`
  (available only for `test` and `run` tasks).

Note that multiple entries matching the same left hand side may be added to `addReads`, `addOpens`, and `addExports` but
no value accumulation is performed, the last entry overrides the previous one. If you need to combine multiple values then
you must do so explicitly. The following block resolves to `--add-reads module1=module3`

```
compileJava {
    moduleOptions {
        addReads = [
            'module1': 'module2',
            'module1': 'module3'
        ]
    }
}
```

Whereas the following block resolves to `--add-reads module1=module2,module3`

```
compileJava {
    moduleOptions {
        addReads = [
            'module1': 'module2,module3'
        ]
    }
}
```

Whitebox testing
===

Whitebox testing is your traditional unit test, where an implementation class is tested in isolation.

Typically we would have a structure as follows:

```
.
├── build.gradle
└── src
    ├── main
    │   ├── java
    │   │   ├── examples
    │   │   │   └── greeter
    │   │   │       └── Friendly.java
    │   │   └── module-info.java
    │   └── resources
    └── test
        ├── java
        │   └── examples
        │       └── greeter
        │           └── FriendlyTest.java
        └── resources

```

This poses a challenge for the module system, because the whole point of encapsulation is to hide implementation classes!
A class that is not exported can't be accessed from outside the module.
In the example above we have another problem, the main and test code uses the same package structure (which is very common).
The module system does not allow split packages however.

We have two different options to work around this:

* Run whitebox tests on the classpath (ignore the fact that we're in the module world)
* *Patch* the module so that it contains the code from both the main and test sources.

Either option is fine.
By default, the plugin will automatically setup the compiler and test runtime to run on the module path, and patch the module to avoid split packages.

How does it work?
----

Essentially, the plugin enables the following compiler flags:

* `--module-path` containing all dependencies
* `--patch-module` to merge the test classes into the modules
* `--add-modules` to add the test runtime (JUnit 5, JUnit 4, TestNG, Spock, AssertJ, Mockito, and EasyMock are supported)
* `--add-reads` for the test runtime. This way we don't have to `require` the test engine in our module.
* `--add-opens` so that the test engine can access the tests without having to export/open them in `--module-info.java`.

The plugin also integrates additional compiler flags specified in a `module-info.test` file.
For example, if your tests need to access types from a module shipping with the JDK (here: `java.scripting`).
Note that each non-comment line represents a single argument that is passed to the compiler as an option.

```text
// Make module visible.
--add-modules
  java.scripting

// Same "requires java.scripting" in a regular module descriptor.
--add-reads
  greeter.provider=java.scripting
```

See `src/test/java/module-info.test` and `src/test/java/greeter/ScriptingTest.java` in `test-project/greeter.provider` for details.

Fall-back to classpath mode
----

If for whatever reason this is unwanted or introduces problems, you can enable classpath mode, which essentially turns off the plugin while running tests.

<details open>
<summary>Groovy DSL</summary>

```groovy
test {
    moduleOptions {
        runOnClasspath = true
    }
}
```

</details>
<details>
<summary>Kotlin DSL</summary>

```kotlin
tasks {
    test {
        extensions.configure(TestModuleOptions::class) {
            runOnClasspath = true
        }
    }
}
```

</details>

You can also enable classpath mode, which essentially turns off the plugin while running tests.

<details open>
<summary>Groovy DSL</summary>

```groovy
compileTestJava {
    moduleOptions {
        compileOnClasspath = true
    }
}
```

</details>
<details>
<summary>Kotlin DSL</summary>

```kotlin
tasks {
  compileTestJava {
        extensions.configure(CompileTestModuleOptions::class) {
            compileOnClasspath = true
        }
    }
}
```

</details>

Blackbox testing
===

It can be very useful to test modules as a blackbox.
Are packages exported correctly, and are services provided correctly?
This allows you to test your module as if you were a user of the module.
To do this, we create a separate module that contains the test.
This module `requires` and/or `uses` the module under test, and tests it's externally visible behaviour.
In the following example we test a module `greeter.provider`, which provides a service implementation of type `Greeter`.
The `Greeter` type is provided by yet another module `greeter.api`.

The test module would typically be named something similar to the module it's testing, e.g. `greeter.provider.test`.
In `src/main/java` it has some code that looks like code that you would normally write to use the module that's being tested.
For example, we do a service lookup.

```java
package tests;

import examples.greeter.api.Greeter;

import java.util.ServiceLoader;

public class GreeterLocator {
    public Greeter findGreeter() {
        return ServiceLoader.load(Greeter.class).findFirst().orElseThrow(() -> new RuntimeException("No Greeter found"));
    }
}
```

In `src/test/java` we have our actual tests.

```java
package tests;

import examples.greeter.api.Greeter;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertFalse;

class GreeterTest {
    @Test
    void testLocate() {
        Greeter greeter = new GreeterLocator().findGreeter();
        assertFalse(greeter.hello().isBlank());
    }
}
```

Because we clearly want to run this code as a module, we also need to have a `module-info.java`.

```java
import examples.greeter.api.Greeter;

module greeter.provider.test {
    requires greeter.api;

    uses Greeter;
}
```

As we've discussed before, we also need to configure the `--module-path` so that the compiler knows about the `greeter.api` module, and the JVM also starts with the `greeter.provider` module available.
In the `build.gradle` we should add dependencies to do this.

```gradle
dependencies {
    implementation project(':greeter.api')
    runtimeOnly project(':greeter.provider')
}

```  

Using the Application plugin
===
Typically you use the `application` plugin in Gradle to run the application from Gradle and, more importantly, package it in a distributable zip/tar.
To work with modules correctly JVM needs to be configured with the correct arguments such as `--module-path` to use the module path instead of the classpath.
The plugin takes care of all that automatically.

When starting a main class from a module, the module name needs to be provided.
To make this easier, the plugin injects a variable `$moduleName` in the build script.

With Gradle 6.4 or newer, you should use the new syntax introduced by the Application plugin:

```gradle
apply plugin: 'application'
application {
    mainClass = "examples.Runner"
    mainModule = moduleName
}
```

When using older versions of Gradle (before 6.4) you can include the module name in the `mainClassName` property:

```gradle
apply plugin: 'application'
mainClassName = "$moduleName/examples.Runner"
```

As usual, you can still set extra JVM arguments using the `run` configuration.

```gradle
run {
    jvmArgs = [
            "-XX:+PrintGCDetails"
    ]

    applicationDefaultJvmArgs = [
            "-XX:+PrintGCDetails"
    ]
}
```

Using the ModularJavaExec task
===
The `application` plugin can handle only one executable application.
To start multiple applications, you typically need to create a `JavaExec` task for each executable application.
The module plugin offers a similar task named `ModularJavaExec`, which helps executing modular applications.
This task automatically configures the JVM with the correct arguments such as `--module-path`.
It exposes the same properties and methods as the [`JavaExec`](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.JavaExec.html) task,
the only difference being that the module name should also be provided when setting the `main` property.  

```gradle
task runDemo1(type: ModularJavaExec) {
    group = "Demo"
    description = "Run the Demo1 program"
    main = "greeter.javaexec/demo.Demo1"
    jvmArgs = ["-Xmx128m"]
}
```

Using the ModularCreateStartScripts task
===
If you have several application classes in the same Gradle project, you may want to create a distribution that provides separate start scripts for each of them.
To help you with this, the plugin offers a `ModularCreateStartScripts` task, which automatically configures the start scripts with the correct JVM arguments for modular applications.

Each `ModularCreateStartScripts` task needs an associated `ModularJavaExec` task that provides information about the application to be started. The typical way to create a distribution containing multiple start scripts is:
* designate one of the application classes as primary and assign its name to the `mainClassName` property of the `application` plugin
* for each of the remaining application classes:
    * configure a `ModularJavaExec` task for running the application class
    * configure a `ModularCreateStartScripts` task associated with the above `ModularJavaExec` task.

Suppose we have a project with two application classes: `MainDemo` and `Demo1`. If we designate `MainDemo` as primary, a possible build script will look like this:

```gradle
import org.javamodularity.moduleplugin.tasks.ModularJavaExec
import org.javamodularity.moduleplugin.tasks.ModularCreateStartScripts

plugins {
    id 'application'
    id 'org.javamodularity.moduleplugin'
}

dependencies {
    implementation project(':greeter.api')
    runtimeOnly project(':greeter.provider')
}

application {
    mainClassName = "greeter.startscripts/startscripts.MainDemo"
    applicationName = "demo"
    applicationDefaultJvmArgs = ["-Xmx128m"]
}

task runDemo1(type: ModularJavaExec) {
    group = "Demo"
    description = "Run the Demo1 program"
    main = "greeter.startscripts/startscripts.Demo1"
    jvmArgs = ["-Xmx128m"]
}

task createStartScriptsDemo1(type: ModularCreateStartScripts) {
    runTask = tasks.runDemo1
    applicationName = 'demo1'
}

installDist.finalizedBy tasks.createStartScriptsDemo1
```

If you have more than two application classes, it's advisable to  create programmatically the `ModularJavaExec` and `ModularCreateStartScripts` tasks, as shown in the [greeter.startscripts](https://github.com/java9-modularity/gradle-modules-plugin/blob/master/test-project/greeter.startscripts/build.gradle) project.

The `ModularCreateStartScripts` task introduces the mandatory property `runTask`, which indicates the associated `ModularJavaExec` task.
Additionally, it exposes the same properties and methods as the [`CreateStartScripts`](https://docs.gradle.org/current/dsl/org.gradle.jvm.application.tasks.CreateStartScripts.html) task.
However, you don't need to set the properties `mainClassName`, `outputDir`, `classpath`, or `defaultJvmOpts`, because they are automatically set by the plugin, based on the configuration of the associated `runTask`.

Patching modules to prevent split packages
===

The Java Platform Module System doesn't allow split packages.
A split package means that the same package exists in multiple modules.
While this is a good thing, it can be a roadblock to use the module system, because split packages are very common in (older) libraries, specially libraries related to Java EE.
The module system has a solution for this problem by allowing to "patch" modules.
The contents of a JAR file can be added to a module, by patching that module, so that it contains classes from both JARs.
This way we can drop the second JAR file, which removes the split package.

Patching a module can be done with the `--patch-module module=somelib.jar` syntax for the different Java commands (javac, java, javadoc, ...).
The plugin helps making patching easy by providing DSL syntax.
Because patching typically needs to happen on all tasks the patch config is set in the build.gradle file directly.

In this example, the `java.annotation` module is patched with the `jsr305-3.0.2.jar` JAR file.
The plugin takes care of the following:

* Adding the `--patch-module` to all Java commands
* Removing the JAR from the module path
* Moving the JAR to a `patchlibs` folder for distribution tasks

**Recommended approach**

![Since 1.7.0](https://img.shields.io/badge/since-1.7.0-brightgreen)
The recommended way to patch modules is by means of the [`modularity.patchModule`][ModularityExtension] function:

```groovy
modularity.patchModule("java.annotation", "jsr305-3.0.2.jar")
```

The `patchModule` method can be called more than once for the same module, if the module needs to be patched with the content of several JARs.

**Legacy approach**

![Legacy 1.x](https://img.shields.io/badge/legacy-1.x-darkgray)
Before 1.7.0, patching modules was possible only by setting `patchModules.config`:

```groovy
patchModules.config = [
        "java.annotation=jsr305-3.0.2.jar"
]
```

For compatibility reasons, this way of patching modules is still supported in newer releases, but it is discouraged. Note that this method does not allow patching a module with more than one JAR.

Compilation
===

Compilation to a specific Java release
----

You might want to run your builds on a recent JDK (e.g. JDK 12), but target an older version of Java, e.g.:
- Java 11, which is the current [Long-Term Support (LTS) release](https://www.oracle.com/technetwork/java/java-se-support-roadmap.html),
- Java 8, whose production use in 2018 was almost 85%, according to [this survey](https://www.baeldung.com/java-in-2018).

You can do that by setting the Java compiler [`--release`][javacRelease] option
(e.g. to `6` for Java 6, etc.). Note that when you build using:
- JDK 11: you can only target Java 6-11 using its
[`--release`](https://docs.oracle.com/en/java/javase/11/tools/javac.html) option,
- JDK 12: you can only target Java 7-12 using its
[`--release`](https://docs.oracle.com/en/java/javase/12/tools/javac.html) option,
- etc.

Finally, note that JPMS was introduced in Java 9, so you can't compile `module-info.java` to Java release 6-8
(this plugin provides a workaround for that, though &mdash; see below).

Concluding, to configure your project to support JPMS and target:
- Java **6-8**: call the [`modularity.mixedJavaRelease`][ModularityExtension] function
(see [Separate compilation of `module-info.java`](#separate-compilation-of-module-infojava) for details),
- Java **9+**: call the [`modularity.standardJavaRelease`][ModularityExtension] function,

and the plugin will take care of setting the [`--release`][javacRelease] option(s) appropriately.


Separate compilation of `module-info.java`
----

If you need to compile the main `module-info.java` separately from the rest of `src/main/java`
files, you can enable `moduleOptions.compileModuleInfoSeparately` option on `compileJava` task. It will exclude `module-info.java`
from `compileJava` and introduce a dedicated `compileModuleInfoJava` task.

Typically, this feature would be used by libraries which target JDK 6-8 but want to make the most of JPMS by:
- providing `module-info.class` for consumers who put the library on module path,
- compiling `module-info.java` against the remaining classes of this module and against other modules
(which provides better encapsulation and prevents introducing split packages).

This plugin provides an easy way to do just that by means of its
[`modularity.mixedJavaRelease`][ModularityExtension] function, which implicitly sets
`compileJava.moduleOptions.compileModuleInfoSeparately = true` and configures the [`--release`][javacRelease] compiler options.

For example, if your library targets JDK 8, and you want your `module-info.class` to target JDK 9
(default), put the following line in your `build.gradle(.kts)`:

<details open>
<summary>Groovy DSL</summary>

```groovy
modularity.mixedJavaRelease 8
```

</details>
<details>
<summary>Kotlin DSL</summary>

```kotlin
modularity.mixedJavaRelease(8)
```

</details>

Note that `modularity.mixedJavaRelease` does *not* configure a
[multi-release JAR](https://docs.oracle.com/javase/9/docs/specs/jar/jar.html#Multi-release)
(in other words, `module-info.class` remains in the root directory of the JAR).

Improve Eclipse `.classpath`-file
===
When applying the [eclipse-plugin](https://docs.gradle.org/current/userguide/eclipse_plugin.html)
(among others) a task called "eclipseClasspath" is added by that plugin creating a `.classpath`-file.
The `.classpath`-file created by that task doesn't take into account modularity. As described by the
documentation it is possible to configure the `.classpath`-file via configuration hooks.

![Since 1.7.0](https://img.shields.io/badge/since-1.7.0-brightgreen)
The gradle-modules-plugin provides the `modularity.improveEclipseClasspathFile()` method,
which configures the `.classpath`-file via configuration hooks:


<details open>
<summary>Groovy DSL</summary>

```groovy
modularity.improveEclipseClasspathFile()
```

</details>
<details>
<summary>Kotlin DSL</summary>

```kotlin
modularity.improveEclipseClasspathFile()
```

</details>

Examples on how and where Gradle's eclipse-plugin could (and should) be improved and how a
`.classpath`-file is affected if the feature is enabled are available on
[GitHub](https://github.com/Alfred-65/gradle-modules-plugin.investigation).


Limitations
===

Please file issues if you run into any problems or have additional requirements!

Requirements
===

This plugin requires JDK 11 or newer to be used when running Gradle.

The minimum Gradle version supported by this plugin is 5.1.
However, we strongly recommend to use at least Gradle 6.0, because there are a few special cases that cannot be handled correctly when using older versions.

Contributing
===

Please tell us if you're using the plugin on [@javamodularity](https://twitter.com/javamodularity)!
We would also like to hear about any issues or limitations you run into.
Please file issues in the Github project.
Bonus points for providing a test case that illustrates the issue.

Contributions are very much welcome.
Please open a Pull Request with your changes.
Make sure to rebase before creating the PR so that the PR only contains your changes, this makes the review process much easier.
Again, bonus points for providing tests for your changes.


[javacRelease]: http://openjdk.java.net/jeps/247
[ModularityExtension]: src/main/java/org/javamodularity/moduleplugin/extensions/ModularityExtension.java
