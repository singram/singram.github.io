---
layout: post
title: Separating Unit from Integration tests in Java using Gradle
date: '2016-09-15 02:07:32'
tags:
- java
- gradle
- unit-testing
- integration-testing
---

Having spent some significant time in the Ruby community and finding a new found appreciation for clean unit and integration tests, it often befuddles me why there isn't such a clean separation of test responsibility and scope in other languages.

Java has learned a lot from other test frameworks over the last decade.  The venerable [Junit](http://junit.org/) test framework has matured significantly but with the addition of [fluent libraries](https://en.wikipedia.org/wiki/Fluent_interface) such as [RestAssured](https://github.com/rest-assured/rest-assured), simple mocking frameworks such as [Mockito](http://mockito.org/), and verbose matching capabilities such as [Hamcrest](http://hamcrest.org/JavaHamcrest/), to name a few, it's now possible to write tests with properties matching many other languages touted for readability and focused test intent.

So having established that Java has some really great testing libraries, that doesn't address how to use them and this is part of the problem.  There is no simple way to separate unit test from integration test in Java so why should there be a clean understanding of what a unit or integration test is?  Often times I see purported unit tests in Junit that are truely integration tests requiring a database and a full application stack to be stood up with no sign of a true unit test in sight.

Very simply stated;

- If your unit tests require a database you're doing it wrong.
- If your unit tests require an external service or dependency you're doing it wrong.
- If your unit tests start the spring framework or an application container, you're doing it wrong.
- If your unit tests spend a lot of time setting up preconditions in other classes you're doing it wrong.
- If your integration tests are not using publicly exposed interfaces you're doing it wrong.
- If your integration tests are stubbing or mocking parts of the system, you're doing it wrong.  (External service stubbing however makes sense)
- If your integration tests are not hitting a running application you're doing it wrong.

The design, purpose and intent of unit and integration tests is the subject of a much larger discussion and outside the scope of this post.

So back to the problem at hand.  Having a desire to separate fast running unit tests from  integration tests I have struggled for an answer until I came across the [gradle-testsets-plugin](https://github.com/unbroken-dome/gradle-testsets-plugin) and these posts from [Petri kainulainen](https://www.petrikainulainen.net/), [here](https://www.petrikainulainen.net/programming/gradle/getting-started-with-gradle-integration-testing/) and [here](https://www.petrikainulainen.net/programming/gradle/getting-started-with-gradle-integration-testing-with-the-testsets-plugin/)

I would strongly recommend reading both posts, but for brevity, here are the main mechanics and some further tips beyond.

**Step 1.**

Include `jcenter` as a source for your build script dependencies and pull in the [gradle-testsets-plugin](https://github.com/unbroken-dome/gradle-testsets-plugin) dependency

```language-groovy
buildscript {
  repositories {
    jcenter()
  }
  dependencies {
    classpath 'org.unbroken-dome.gradle-plugins:gradle-testsets-plugin:1.0.2'
  }
}
```

**Step 2.**

Apply the plugin to the build.  Be sure to activate this after the `java` plugin and before any plugins which may build off the gradle tasks automatically created by the plugin.
```language-groovy
apply plugin: 'org.unbroken-dome.test-sets'
```

**Step 3.**

Create the new test set definition and configuration.  Here we want to add an integration test suite but this could be any category of tests you wish to scope together.
```language-groovy
testSets {
  integrationTest
}
```
Ensure that the `check` step executes the new test definition and that the new `integrationTest` step runs after the normal `test` (unit) step.
```language-groovy
check.dependsOn integrationTest
integrationTest.mustRunAfter test
```
Ensure that integration tests are always run regardless if they passed on previous runs
```language-groovy
project.integrationTest {
  outputs.upToDateWhen { false }
}
```
Finally ensure that the output for tasks of type `Test` are namespaced appropriately so reports are separated for the `test` (unit) and `integrationTest` tasks
```language-groovy
tasks.withType(Test) {
  reports.html.destination = file("${reporting.baseDir}/${name}")
}
```

**Step 4.**

Test compile dependencies should be reviewed and the new `integrationTestCompile` dependencies declared appropriately
*e.g.*
```language-groovy
testCompile("junit:junit")
integrationTestCompile("org.springframework.boot:spring-boot-starter-test",
                       "com.jayway.restassured:json-path:2.8.0",
                       "com.jayway.restassured:rest-assured:2.8.0",
                       "com.jayway.restassured:spring-mock-mvc:2.8.0",
                       "com.jayway.restassured:xml-path:2.8.0")
```
**Step 5.**

Restructure your test file layout.  Your directory structure should look something like the following.
```
src/
  main/
    java/...
    resources/...
  integrationTest/
    java/...
    resources/...
  test/
    java/...
    resources/...

```

At this point you should be able to run `gradle clean build` and see your separate `test` and `integrationTest` related tasks execute.

**Real time test reporting**

To see a visual report of test execution and outcome as it happens in the console, add the following
```language-groovy
test {
  afterTest { desc, result ->
    println "Executing test [${desc.className}].${desc.name} with result: ${result.resultType}"
    }
}
integrationTest {
  afterTest { desc, result ->
    println "Executing test [${desc.className}].${desc.name} with result: ${result.resultType}"
    }
}
```

**Test Coverage**

I use [Jacoco](http://www.eclemma.org/jacoco/) for test coverage with the help of the [Jacoco gradle plugin](https://docs.gradle.org/current/userguide/jacoco_plugin.html).  While it would be ideal to have separate test coverage for integration and unit test suites in separate reports I was unable to find a simple method to generate them independently.  However you can combine the coverage from both suites with the following;
```language-groovy

apply plugin: 'jacoco'
.....
jacoco {
    toolVersion = "0.7.5.201505241946"
}

jacocoTestReport {
    reports {
        xml.enabled false
        csv.enabled false
        html{
            enabled true
            destination "${buildDir}/reports/jacoco"
        }
    }
    executionData(test, integrationTest)
}

tasks.build.dependsOn(jacocoTestReport)

```

I hope this post has proved useful.  The separation of test types has many benefits including;

- forcing developers to think about test types & purpose
- enforcing unit test conventions.  If you need anything beyond java or are firing up an application server it's not a unit test.
- separating fail fast unit tests from potentially costly integration tests
- allowing finer control over CI builds and development process. 