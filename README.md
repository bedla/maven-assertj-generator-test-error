This repository helps me reproduce and correctly understand classloader problem with generation of AssertJ POJO
assertion classes.

Github issue [has #57](https://github.com/assertj/assertj-assertions-generator-maven-plugin/issues/57).

# Test compile problem with AssertJ Generator Maven plugin

Current version of assertj-assertions-generator-maven-plugin has an unpleasant bug, when user runs `mvn clean install`
and later just `mvn test` generator fails on following error:

```
[ERROR] Failed to execute goal org.assertj:assertj-assertions-generator-maven-plugin:2.2.0:generate-assertions (default) on project maven-test-classpath: Execution default of goal org.assertj:assertj-assertions-generator-maven-plugin:2.2.0:generate-assertions failed: A required class was missing while executing org.assertj:assertj-assertions-generator-maven-plugin:2.2.0:generate-assertions: org/junit/rules/TestRule
[ERROR] -----------------------------------------------------
[ERROR] realm =    plugin>org.assertj:assertj-assertions-generator-maven-plugin:2.2.0
[ERROR] strategy = org.codehaus.plexus.classworlds.strategy.SelfFirstStrategy
[ERROR] urls[0] = file:/C:/Users/ivo.smid/.m2/repository/org/assertj/assertj-assertions-generator-maven-plugin/2.2.0/assertj-assertions-generator-maven-plugin-2.2.0.jar
[ERROR] urls[1] = file:/C:/Users/ivo.smid/.m2/repository/org/assertj/assertj-assertions-generator/2.2.0/assertj-assertions-generator-2.2.0.jar
[ERROR] urls[2] = file:/C:/Users/ivo.smid/.m2/repository/org/apache/commons/commons-lang3/3.5/commons-lang3-3.5.jar
[ERROR] urls[3] = file:/C:/Users/ivo.smid/.m2/repository/commons-cli/commons-cli/1.2/commons-cli-1.2.jar
[ERROR] urls[4] = file:/C:/Users/ivo.smid/.m2/repository/com/google/guava/guava/20.0/guava-20.0.jar
[ERROR] urls[5] = file:/C:/Users/ivo.smid/.m2/repository/ch/qos/logback/logback-classic/1.0.13/logback-classic-1.0.13.jar
[ERROR] urls[6] = file:/C:/Users/ivo.smid/.m2/repository/ch/qos/logback/logback-core/1.0.13/logback-core-1.0.13.jar
[ERROR] urls[7] = file:/C:/Users/ivo.smid/.m2/repository/org/assertj/assertj-core/2.9.1/assertj-core-2.9.1.jar
[ERROR] urls[8] = file:/C:/Users/ivo.smid/.m2/repository/commons-collections/commons-collections/3.2.1/commons-collections-3.2.1.jar
[ERROR] urls[9] = file:/C:/Users/ivo.smid/.m2/repository/commons-io/commons-io/2.5/commons-io-2.5.jar
[ERROR] urls[10] = file:/C:/Users/ivo.smid/.m2/repository/backport-util-concurrent/backport-util-concurrent/3.1/backport-util-concurrent-3.1.jar
[ERROR] urls[11] = file:/C:/Users/ivo.smid/.m2/repository/org/codehaus/plexus/plexus-interpolation/1.11/plexus-interpolation-1.11.jar
[ERROR] urls[12] = file:/C:/Users/ivo.smid/.m2/repository/org/codehaus/plexus/plexus-utils/1.5.15/plexus-utils-1.5.15.jar
[ERROR] Number of foreign imports: 1
[ERROR] import: Entry[import  from realm ClassRealm[maven.api, parent: null]]
[ERROR] 
[ERROR] -----------------------------------------------------
[ERROR] : org.junit.rules.TestRule
```

It basically says that class `org.junit.rules.TestRule` cannot be found, but we have this class on classpath!

Running `mvn test -X` to see stack trace and debug output of Maven execution. Output reveals that class real cannot
be found. Again, why?

```
...
Caused by: java.lang.ClassNotFoundException: org.junit.rules.TestRule
    at org.codehaus.plexus.classworlds.strategy.SelfFirstStrategy.loadClass (SelfFirstStrategy.java:50)
    at org.codehaus.plexus.classworlds.realm.ClassRealm.unsynchronizedLoadClass (ClassRealm.java:271)
    at org.codehaus.plexus.classworlds.realm.ClassRealm.loadClass (ClassRealm.java:247)
    at org.codehaus.plexus.classworlds.realm.ClassRealm.loadClass (ClassRealm.java:239)
    at java.lang.ClassLoader.defineClass1 (Native Method)
    at java.lang.ClassLoader.defineClass (ClassLoader.java:1016)
...
``` 

## Problem description

AssertJ generator mojo works as follows.

- First it looks at project classloader and detects configured classes. In our case `cz.bedla.dto` package.
- Second it generates assertion classes for those detected classes

When we have default plugin configuration as follows:

```xml
            <plugin>
                <groupId>org.assertj</groupId>
                <artifactId>assertj-assertions-generator-maven-plugin</artifactId>
                <version>2.2.0</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>generate-assertions</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <packages>
                        <param>cz.bedla.dto</param>
                    </packages>
                </configuration>
            </plugin>
```   

After run, we expect few generated classes in directory `target/generated-test-sources/assertj-assertions`

```
cz/bedla/dto/AbstractFooAssert.java
cz/bedla/dto/Assertions.java
cz/bedla/dto/BddAssertions.java
cz/bedla/dto/FooAssert.java
cz/bedla/dto/JUnitSoftAssertions.java
```

Those classes are compiled without error with `mvn clean install`. With `mvn test` command we got error as
described at beginning. 

First run of `mvn clean install` means that package `cz.bedla.dto` contains only one class `cz.bedla.dto.Foo`
as input for the generator.

Second run of `mvn test` means that package `cz.bedla.dto` contains that DTO class `cz.bedla.dto.Foo` and 
 generated classes from first run:

```
cz.bedla.dto.AbstractFooAssert
cz.bedla.dto.Assertions
cz.bedla.dto.BddAssertions
cz.bedla.dto.FooAssert
cz.bedla.dto.JUnitSoftAssertions
```

From plugin point of view those classes have to be analysed and that's the problem. For details see 
"Technical description of problem" chapter.

### Workaround 1 - you need JUnit4 soft assertions generated

When you need JUnit4 soft assertions to be generated you have to repeat JUnit4 dependency on plugin's classpath.

```xml
            <plugin>
                <groupId>org.assertj</groupId>
                <artifactId>assertj-assertions-generator-maven-plugin</artifactId>
                <version>2.2.0</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>generate-assertions</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <generateJUnitSoftAssertions>true</generateJUnitSoftAssertions>
                    <packages>
                        <param>cz.bedla.dto</param>
                    </packages>
                </configuration>
                <dependencies>
                    <dependency>
                        <groupId>junit</groupId>
                        <artifactId>junit</artifactId>
                        <version>4.12</version>
                    </dependency>
                </dependencies>
            </plugin>
```

Build dependency of JUnit4 is only used to detect if it is possible to generate JUnit4 soft-assertions.
See `org.assertj.maven.AssertJAssertionsGeneratorMojo#junitFoundBy` method, and it's usage.

### Workaround 2 - you don't need JUnit4 soft assertions generated

When you don't need JUnit4 soft assertion (for example when you use JUnit5, etc.) it is possible disable their 
generation.

```xml
            <plugin>
                <groupId>org.assertj</groupId>
                <artifactId>assertj-assertions-generator-maven-plugin</artifactId>
                <version>2.2.0</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>generate-assertions</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <generateJUnitSoftAssertions>false</generateJUnitSoftAssertions>
                    <packages>
                        <param>cz.bedla.dto</param>
                    </packages>
                </configuration>
            </plugin>
```

Note: in this case you don't need JUnit4 plugin's dependency.

### Workaround 3 (Not working) - you need JUnit4 soft assertions generated

When you need JUnit4 soft assertion, but don't want to add artificial plugin dependency of JUnit4 on the plugin's 
class path.

Flag `cleanTargetDir` seems reasonable but, it clears only generates `.java` files, not compiled `.class` files from 
previous run. This means that error occurs even in this case.

```xml
            <plugin>
                <groupId>org.assertj</groupId>
                <artifactId>assertj-assertions-generator-maven-plugin</artifactId>
                <version>2.2.0</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>generate-assertions</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <generateJUnitSoftAssertions>true</generateJUnitSoftAssertions>
                    <cleanTargetDir>true</cleanTargetDir>
                    <packages>
                        <param>cz.bedla.dto</param>
                    </packages>
                </configuration>
            </plugin>
```

Note: in this case you don't need JUnit4 plugin's dependency.

## Technical description of problem

As described above second run of generator with already generated assertion classes fails badly with classloader error.

Problem is with plugin flag `generateJUnitSoftAssertions=true` and generated class `cz.bedla.dto.JUnitSoftAssertions`
 from `target/generated-test-sources/assertj-assertions` directory.

First let's look at class in question.

Generated class extends class from `assertj-core-2.9.1.jar`.

```java
package cz.bedla.dto;

@javax.annotation.Generated(value="assertj-assertions-generator")
public class JUnitSoftAssertions extends org.assertj.core.api.JUnitSoftAssertions {
 // class body here 
}
```

AssertJ Core class implements interface from `junit-4.12.jar`.

```java
package org.assertj.core.api;
public class JUnitSoftAssertions extends AbstractStandardSoftAssertions implements org.junit.rules.TestRule {
 // class body here 
}
```

AssertJ plugin generator loads all the classes from package `cz.bedla.dto` and process its metadata to decide if it
is possible to generate assertions for them.

Steps are following:

### 1) Execute AssertJ generator mojo 

```
org.assertj.maven.AssertJAssertionsGeneratorMojo#execute(...)
```

### 2) Create `projectClassloader` as new instance of `URLClassLoader` with urls from current project's
compile and test classpath.

Method `org.assertj.maven.AssertJAssertionsGeneratorMojo#getProjectClassLoader(...)`

``` 
project.getCompileClasspathElements()
|
+- 0 = C:\Users\ivo.smid\IdeaProjects\maven-assertj-generator-test-error\target\classes
+- ...
project.getTestClasspathElements()
|
+- 0 = "C:\Users\ivo.smid\IdeaProjects\maven-assertj-generator-test-error\target\test-classes"
+- 1 = "C:\Users\ivo.smid\IdeaProjects\maven-assertj-generator-test-error\target\classes"
+- 9 = "C:\Users\ivo.smid\.m2\repository\org\assertj\assertj-core\2.9.1\assertj-core-2.9.1.jar"
+- 30 = "C:\Users\ivo.smid\.m2\repository\junit\junit\4.12\junit-4.12.jar"
+- ...
```

As you can see test classpath correctly contains `assertj-core-2.9.1.jar` and `junit-4.12.jar`.

This `projectClassLoader` has a parent classloader from current `Thread.currentThread().getContextClassLoader()`. 

```
projectClassLoader = new URLClassLoader(..., Thread.currentThread().getContextClassLoader());
 .URLClassPath
  |
  +- 37 = {URL@4444} "file:/C:/Users/ivo.smid/.m2/repository/org/assertj/assertj-core/2.9.1/assertj-core-2.9.1.jar"
  +- 58 = {URL@4465} "file:/C:/Users/ivo.smid/.m2/repository/junit/junit/4.12/junit-4.12.jar"
  +- ... 
```

The `contextClassLoader` is ClassWorlds Realm of current AssertJ generator plugin.

```
contextClassLoader = Thread.currentThread().getContextClassLoader() =
 ClassRealm[plugin>org.assertj:assertj-assertions-generator-maven-plugin:2.2.0, parent: jdk.internal.loader.ClassLoaders$AppClassLoader@6ed3ef1]
  .URLClassPath
   |
   +- 7 = {URL@4348} "file:/C:/Users/ivo.smid/.m2/repository/org/assertj/assertj-core/2.9.1/assertj-core-2.9.1.jar"
   +- ...
  .importRealms
   |
   +- ClassRealm[maven.api, parent: null] ]
   +- ...
```

Note 1: there is not `junit-4.12.jar` on `contextClassLoader`'s classpath.

Note 2: `importRealm` is only one and points to main Maven's realm. 

### 3) Find classes to process for package `cz.bedla.dto`

```
org.assertj.assertions.generator.util.ClassUtil#getPackageClassesFromClasspathFiles
0 = {URL@3728} "file:/C:/Users/ivo.smid/IdeaProjects/maven-assertj-generator-test-error/target/classes/cz%5cbedla%5cdto"
1 = {URL@3729} "file:/C:/Users/ivo.smid/IdeaProjects/maven-assertj-generator-test-error/target/test-classes/cz%5cbedla%5cdto"
```

This will also find `cz/bedla/dto/JUnitSoftAssertions.class` class from compiled from generated test sources. And
will try to load metadata from it.

### 4) Load class metadata from detected `.class` files 

```java
  private static TypeToken<?> loadClass(String className, ClassLoader classLoader) throws ClassNotFoundException {
    return TypeToken.of(Class.forName(className, false, classLoader));
  }
```

This will invoke `Class#forName(...)` with `projectClassLoader` as parameter. As we know from previous information
`projectClassLoader` contains all the classes/libraries necessary to load `JUnitSoftAssertions` classes.

```java
projectClassLoader.loadClass("cz.bedla.dto.JUnitSoftAssertions")
```

### 5) Load `cz.bedla.dto.JUnitSoftAssertions` class

The default strategy to load class is to ask parent classloader, if it already contains 
desired class.

This means that when JVM sees that `cz.bedla.dto.JUnitSoftAssertions` extends 
`org.assertj.core.api.JUnitSoftAssertions`, it will try to load class by asking parent classloader for that class.

```java
 projectClassLoader.loadClass("org.assertj.core.api.JUnitSoftAssertions");
 parent = contextClassLoader;

  if (parent != null) {
   c = parent.loadClass(name, false);
  } else {
   // ...
  }
```

### 6) Load `org.assertj.core.api.JUnitSoftAssertions` from `contextClassLoader`

Context class loader is ClassWorlds realm 
`ClassRealm[plugin>org.assertj:assertj-assertions-generator-maven-plugin:2.2.0, parent: ...]` with 
`assertj-core-2.9.1.jar` library, but without `junit-4.12.jar`. 

```java
 contextClassLoader.findClass("org.assertj.core.api.JUnitSoftAssertions")

 Resource res = urlClassPath.getResource("org/assertj/core/api/JUnitSoftAssertions.class", false);
 if (res != null) {
  return defineClass(name, res);
 } else {
  return null;
 }
```

`contextClassLoader` asks it's `urlClassPath` if it contains `org.assertj.core.api.JUnitSoftAssertions` and it 
answers yes by returning non-null `res` variable.

### 7) Load `org.junit.rules.TestRule` class from `contextClassLoader`

Because realm of `contextClassLoader` returns positive answer about having `org.assertj.core.api.JUnitSoftAssertions`,
class loader is going to try to define this class.

Now it sees that `org.assertj.core.api.JUnitSoftAssertions` class implements `org.junit.rules.TestRule` interface.
Correctly `contextClassLoader` tries to load that interface.
But it does not have one, it will ask it's `importRealm` to load it, but without success and 
with `ClassNotFoundException` thrown.

```java
contextClassLoader.loadClass("org.junit.rules.TestRule") == null

contextClassLoader.loadClassFromImport("org.junit.rules.TestRule")
 
importRealm.loadClass(""org.junit.rules.TestRule"") = throw new ClassNotFoundException(...)
``` 

### 8) Why does this happen

Interface `org.junit.rules.TestRule` currently lives in `projectClassLoader` and `contextClassLoader` tries to load it.
This two class loaders are in different hierarchy and that's why it is not possible to load that interface.

When you look at `Workaround 1 - you need JUnit4 soft assertions generated`, it adds artificial dependency to
`junit-4.12.jar` library to satisfy this load. It does not matter that JUnit4 (as AssertJ Core) loaded two times 
inside `projectClassLoader` and `contextClassLoader` because it is only used to load metadata and generate assertions.