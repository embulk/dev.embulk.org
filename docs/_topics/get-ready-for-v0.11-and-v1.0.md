---
layout: posts
title: "For Embulk plugin developers: Get ready for v0.11 and v1.0!"
date: 2021-04-26 17:49:30 +0900
last_modified_at: 2021-04-26 18:02:15 +0900
authors:
- "dmikurube"
---

We have worked hard toward Embulk v0.11 and v1.0 since the last announcements on Embulk:

* [Embulk v0.10 series, which is a milestone to v1.0](https://www.embulk.org/articles/2020/03/13/embulk-v0.10.html)
* [More detailed plan of Embulk v0.10, v0.11, and v1 -- Meetup!](https://www.embulk.org/articles/2020/07/01/meetup-20200709.html)
* [Java plugins to catch up with Embulk v0.10 from v0.9](https://dev.embulk.org/topics/catchup-with-v0.10.html)

Embulk is approaching Embulk v0.11.0 now! The new Embulk plugin API/SPI have been almost fixed.

This article re-summarizes how an Embulk (Java) plugin can catch up with Embulk v0.11, and v1.0.

### First, when you can declare "This plugin is ready for Embulk v0.11 and v1.0!"

A plugin should be ready for Embulk v0.11 and v1.0 when :

* It does not depend on an Maven artifact, `org.embulk:embulk-core`.
* It depends on `org.embulk:embulk-api` and `org.embulk:embulk-spi` with `compileOnly`.
* It is built with the Gradle plugin `org.embulk.embulk-plugins`.

The `build.gradle` would be like below:

```
plugins {
    id "java"
    id "maven-publish"
    id "org.embulk.embulk-plugins" version "0.4.2"  // 0.4.2 is the latest as of April 26, 2021.
}

group = "your.maven.group"  // Choose your own Maven groupId, not "org.embulk", ex. "io.github.your_github_username"
version = "X.Y.Z"

dependencies {
    // From 0.11, "embulk-api" and "embulk-spi" will have a different version scheme from other artifacts.
    compileOnly "org.embulk:embulk-api:0.10.??"
    compileOnly "org.embulk:embulk-spi:0.10.??"
    // NO! compileOnly "org.embulk:embulk-core:0.10.??"
    // NO! compileOnly "org.embulk:embulk-deps:0.10.??"

    // The Gradle plugin is not ready for Gradle's "implementation". It still needs to be "compile".
    // Your Gradle needs to be 6, not 7, then. (We'll be working on it to support Gradle 7.)

    // Some libraries "org.embulk:embulk-util-*" are available. Use them instead of depending on "embulk-core" directly.
    compile "org.embulk:embulk-util-config:0.2.1"  // 0.2.1 is the latest as of April 26, 2021.

    // ...

    testCompile "org.embulk:embulk-api:0.10.??"
    testCompile "org.embulk:embulk-spi:0.10.??"
    testCompile "org.embulk:embulk-junit4:0.10.??"
    testCompile "org.embulk:embulk-core:0.10.??"  // "embulk-core" can be used for "testCompile".
    testCompile "org.embulk:embulk-deps:0.10.??"  // "embulk-deps" can be used for "testCompile".
}
```

In other words, if your plugin still needs `org.embulk:embulk-core` to build, your plugin is not ready for Embulk v0.11 yet.

### How to deal with Ruby plugins

JRuby is no longer built-in with Embulk v0.11+. The JRuby integration will not be maintained actively from Embulk v1.0. In other words, Ruby (plugins) will not be with the top-tier support in Embulk.

Please expect that some existing Ruby plugins may suddenly stop working at some point after Embulk v1.0.0. We no longer recommend to build a new Ruby plugin. We'd even suggest rewriting Ruby plugins in Java. (You are still welcome to join maintaining Embulk's JRuby integration, though!)

### How your Embulk plugin can catch up with Embulk v0.11

#### Upgrade your Gradle wrapper to Gradle 6

```
./gradlew wrapper --gradle-version=6.8.3
```

(6.8.3 is the latest Gradle 6 as of April 26, 2021.)

#### Apply the Gradle plugin `org.embulk.embulk-plugins`

Follow the guidance at: [https://github.com/embulk/gradle-embulk-plugins](https://github.com/embulk/gradle-embulk-plugins)

#### Choose your own Maven `groupId`

JRuby is no longer built-in with Embulk v0.11+ as mentioned above. Ruby gems will not be the primary format for Embulk plugins. Maven artifacts will be the primary format for Embulk plugins, instead. They will be released in a Maven repository.

We would suggest [Maven Central](https://search.maven.org/) as the first choice for open-sourced Embulk plugins in the JAR format rather than other public Maven repositories. (You saw [the sunset of Bintray and JCenter](https://jfrog.com/blog/into-the-sunset-bintray-jcenter-gocenter-and-chartcenter/), right?)

To release your plugin in Maven Central, you have to set your Maven `groupId` to a name that you have a previledge to release with. Choose your own `groupId` that follows the guidelines below.

* Maven's [Guide to naming conventions on groupId, artifactId, and version](http://maven.apache.org/guides/mini/guide-naming-conventions.html)
* Java's [Unique Package Names](https://docs.oracle.com/javase/specs/jls/se6/html/packages.html#7.7)

A good example is your GitHub domain name, such as `io.github.your_github_username`. We'd suggest to [set up your own GitHub Pages](https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages#types-of-github-pages-sites) before you start with your GitHub domain name, though.

Do not use `org.embulk` for your plugin's `groupId`. It is for plugins maintained under [https://github.com/embulk](https://github.com/embulk).

#### Configure to release your plugin in a Maven repository

We would suggest [Maven Central](https://search.maven.org/) as the first choice for open-sourced Embulk plugins, as mentioned above, even though others are not prohibited. For any Maven repository, you will have to configure your `build.gradle`.

The configuration would be like below. For Maven Central, check out [their guide](https://central.sonatype.org/publish/publish-guide/) and [their requirements](https://central.sonatype.org/pages/requirements.html).

```
// Maven Central needs "-javadoc" and "-sources" JARs.
// This configuration generates those JARs automatically in Gradle 6+.
java {
    withJavadocJar()
    withSourcesJar()
}

// "./gradlew publishMavenPublicationToMavenCentralRepository" will release your plugin in Maven Central with the configuration.
// You have to complete your initial Sonatype OSSRH JIRA setup, a New Project ticket in the OSSRH JIRA, and your PGP signature.
publishing {
    publications {
        maven(MavenPublication) {
            groupId = "${project.group}"
            artifactId = "${project.name}"

            from components.java

            // Maven Central needs "-javadoc" and "-sources" JARs.
            // "javadocJar" and "sourcesJar" are added automatically by "java.withJavadocJar()" and "java.withSourcesJar()" above.
            // See: https://docs.gradle.org/current/javadoc/org/gradle/api/plugins/JavaPluginExtension.html

            pom {  // https://central.sonatype.org/pages/requirements.html
                packaging "jar"

                name = "${project.name}"
                description = "${project.description}"
                // url = "https://your-github-username.github.io"

                licenses {
                    // Maven Central needs an explicit license declaration.
                    // This is an example of MIT License.
                    license {
                        // http://central.sonatype.org/pages/requirements.html#license-information
                        name = "MIT License"
                        url = "http://www.opensource.org/licenses/mit-license.php"
                    }
                }

                developers {
                    developer {
                        name = "Your Name"
                        email = "you@example.com"
                    }
                }

                scm {
                    connection = "scm:git:git://github.com/your_github_username/your_repository.git"
                    developerConnection = "scm:git:git@github.com:your_github_username/your_repository.git"
                    url = "https://github.com/your_github_username/your_repository"
                }
            }
        }
    }

    repositories {
        maven {
            name = "mavenCentral"
            if (project.version.endsWith("-SNAPSHOT")) {
                url "https://oss.sonatype.org/content/repositories/snapshots"
            } else {
                url "https://oss.sonatype.org/service/local/staging/deploy/maven2"
            }

            // This configuration expects that your "~/.gradle/gradle.properties" has "ossrhUsername" and "ossrhPassword"
            // configured with your OSSRH username and password.
            // https://central.sonatype.org/publish/publish-gradle/#credentials
            credentials {
                username = project.hasProperty("ossrhUsername") ? ossrhUsername : ""
                password = project.hasProperty("ossrhPassword") ? ossrhPassword : ""
            }
        }
    }
}

// Maven Central needs signing.
// https://central.sonatype.org/publish/requirements/gpg/
// https://central.sonatype.org/publish/publish-gradle/#credentials
signing {
    sign publishing.publications.maven
}
```

#### Cleanup dependencies

As described first, a "modern" Embulk plugin for Embulk v0.11+ should no longer depend on `org.embulk:embulk-core`. It means that a modern plugin can no longer use a class that is only in `org.embulk:embulk-core`, not in `org.embulk:embulk-api` nor `org.embulk:embulk-spi`.

We know it is a big impact on plugin compatibility. We however went ahead with it so that future Embulk plugins will be free from complicated tangled dependencies.

`org.embulk:embulk-core` had dependencies on Jackson 2.6.7, for example. Plugins are loaded under `org.embulk:embulk-core`, then they had to use Jackson 2.6.7 even though Jackson 2.6.7 was so old. On the other hand, upgrading Jackson in `org.embulk:embulk-core` was not easy as well because the upgrade could easily affect existing plugins which still expected Jackson 2.6.7. It has been a double bind.

`org.embulk:embulk-core` also had many utility classes in it. When the Embulk core team made a change there for a new feature, many plugins could have been impacted unexpectedly by the change. It has caused us to hesitate to make improvements in the Embulk core.

Once almost all the plugins catch up with v0.11+, they will be free to use their own Jackson versions, and they will be less impacted by changes in the Embulk core.

`org.embulk:embulk-api` and `org.embulk:embulk-spi` will be a contract between the Embulk core and plugins since Embulk v0.11. They are rarely updated, and carefully maintained to be compatible, so that plugins are not impacted by changes in the Embulk core. `org.embulk:embulk-core` will be updated more frequently for new features.

`org.embulk:embulk-api` and `org.embulk:embulk-spi` will have a different version scheme from `org.embulk:embulk-core`. Their versions will be just like `0.11`, `1.0`, `1.1`, and `1.2` unlike `org.embulk:embulk-core`. Note that their first digit and second digit will not be coupled. For example, `org.embulk:embulk-core:1.3.2` may expect `org.embulk:embulk-api:1.1` and `org.embulk:embulk-spi:1.1`.

`org.embulk:embulk-api:0.11` and `org.embulk:embulk-spi:0.11` are released at the same time with Embulk v0.11.0. `org.embulk:embulk-api:1.0` and `org.embulk:embulk-spi:1.0` are released at the same time with Embulk v1.0.0. But later versions will be released when API/SPI have changes -- it should not be so frequent.

##### Processing Embulk configurations

Embulk plugins have used `org.embulk.config.ConfigSource#loadConfig` and `org.embulk.config.TaskSource#loadTask` to process Embulk configurations. But they're deprecated. Please use an external library [`org.embulk:embulk-util-config`](https://dev.embulk.org/embulk-util-config/) instead.

```
compile "org.embulk:embulk-util-config:0.2.1"
```

The usage is not very complicated. At first, `@Config`, `@ConfigDefault`, and `Task` need to be replaced to `embulk-util-config`'s.

```
- import org.embulk.config.Config;
- import org.embulk.config.ConfigDefault;
- import org.embulk.config.Task;
+ import org.embulk.util.config.Config;
+ import org.embulk.util.config.ConfigDefault;
+ import org.embulk.util.config.Task;
```

Then, check your plugin's `Task` definition. If it extends only `Task` (`org.embulk.config.Task` => `org.embulk.util.config.Task`), it has no problems. If it extends another interface (a typical example is `TimestampParser.Task`) with old `org.embulk.config.Task`, find a newer alternative, or copy its methods into your `Task`, so that they are compatible for users.

```
public interface PluginTask extends Task, org.embulk.spi.time.TimestampParser.Task {  // => Remove TimestampParser.Task.
    // ...

    // From org.embulk.spi.time.TimestampParser.Task.
    @Config("default_timezone")
    @ConfigDefault("\"UTC\"")
    String getDefaultTimeZoneId();

    // From org.embulk.spi.time.TimestampParser.Task.
    @Config("default_timestamp_format")
    @ConfigDefault("\"%Y-%m-%d %H:%M:%S.%N %z\"")
    String getDefaultTimestampFormat();

    // From org.embulk.spi.time.TimestampParser.Task.
    @Config("default_date")
    @ConfigDefault("\"1970-01-01\"")
    String getDefaultDate();
}
```

Next, prepare an instance of `org.embulk.util.config.ConfigMapperFactory`.

```
import org.embulk.util.config.ConfigMapperFactory;

private static final ConfigMapperFactory CONFIG_MAPPER_FACTORY = ConfigMapperFactory.builder().addDefaultModules().build();
```

You can add your preferred Jackson [Module](https://fasterxml.github.io/jackson-databind/javadoc/2.6/com/fasterxml/jackson/databind/Module.html) if needed to map an Embulk configuration to a non-standard class.

```
private static final ConfigMapperFactory CONFIG_MAPPER_FACTORY =
        ConfigMapperFactory.builder().addDefaultModules().addModule(new SomeJacksonModule()).build();
```

You can enable validating a configuration by Apache BVal.

```
import javax.validation.Validation;
import javax.validation.Validator;
import org.apache.bval.jsr303.ApacheValidationProvider;

private static final Validator VALIDATOR =
        Validation.byProvider(ApacheValidationProvider.class).configure().buildValidatorFactory().getValidator();

private static final ConfigMapperFactory CONFIG_MAPPER_FACTORY =
        ConfigMapperFactory.builder().addDefaultModules().withValidator(VALIDATOR).build();
```

Finally, replace `loadConfig` and `loadTask` into `org.embulk.util.config.ConfigMapper#map` and `org.embulk.util.config.TaskMapper#map`, respectively.

```
+ import org.embulk.util.config.ConfigMapper;
+ import org.embulk.util.config.TaskMapper;

- PluginTask task = config.loadConfig(PluginTask.class);
+ final ConfigMapper configMapper = CONFIG_MAPPER_FACTORY.createConfigMapper();
+ final PluginTask task = configMapper.map(config, PluginTask.class);

- PluginTask task = taskSource.loadTask(PluginTask.class);
+ final TaskMapper taskMapper = CONFIG_MAPPER_FACTORY.createTaskMapper();
+ final PluginTask task = taskMapper.map(taskSource, PluginTask.class);
```

`embulk-util-config` depends on Jackson by itself. Your plugin will have its own dependencies on Jackson.

Please consider keeping your plugin with Jackson 2.6.7 for a while from Embulk v0.11.0 so that the plugin can still work with the old stable Embulk v0.9. Once many popular plugins are ready for v0.11+, and users start to use v0.11+, any Jackson version would work!

##### Handling date/time and time zones

[Joda-Time](https://www.joda.org/joda-time/) is removed from Embulk. `org.embulk.spi.time.TimestampFormatter` and `org.embulk.spi.time.TimestampParser` are deprecated. Please use Java 8's `java.time` standard classes, and an external library [`org.embulk:embulk-util-timestamp`](https://dev.embulk.org/embulk-util-timestamp/) instead.

```
compile "org.embulk:embulk-util-timestamp:0.2.1"
```

Embulk's [`org.embulk.spi.time.Timestamp`](https://dev.embulk.org/embulk-api/0.10.31/javadoc/org/embulk/spi/time/Timestamp.html) is deprecated. Use [`java.time.Instant`](https://docs.oracle.com/javase/jp/8/docs/api/java/time/Instant.html) instead.

Joda-Time's [`DateTime`](https://www.joda.org/joda-time/apidocs/org/joda/time/DateTime.html) would be replaced with [`java.time.OffsetDateTime`](https://docs.oracle.com/javase/8/docs/api/java/time/OffsetDateTime.html) or [`java.time.ZonedDateTime`](https://docs.oracle.com/javase/8/docs/api/java/time/ZonedDateTime.html). If `OffsetDateTime` is sufficient for you, we'd suggest `OffsetDateTime`. The complexity of time zones based on geographical regions would easily get everything messed up. (Imagine that your plugin receives `2017-03-12 02:30:00` against a default time zone setting `America/Los_Angeles`.)

Joda-Time's [`DateTimeZone`](https://www.joda.org/joda-time/apidocs/org/joda/time/DateTimeZone.html) would be replaced with [`java.time.ZoneOffset`](https://docs.oracle.com/javase/8/docs/api/java/time/ZoneOffset.html) or [`java.time.ZoneId`](https://docs.oracle.com/javase/8/docs/api/java/time/ZoneId.html). For reasons similar to the above, we'd suggest to restrict it to `ZoneOffset` if it is sufficient for you.

Embulk's `org.embulk.spi.time.TimestampFormatter` and `org.embulk.spi.time.TimestampParser` are deprecated. Use `embulk-util-timestamp`'s [`org.embulk.util.timestamp.TimestampFormatter`](https://dev.embulk.org/embulk-util-timestamp/0.2.1/javadoc/org/embulk/util/timestamp/TimestampFormatter.html) instead. The typical "as-is" migration would be like below:

```
// Define your own PluginTask with embulk-util-config's org.embulk.util.config.Task.
private interface PluginTask extends org.embulk.util.config.Task {
    // ...

    // From org.embulk.spi.time.TimestampParser.Task.
    @Config("default_timezone")
    @ConfigDefault("\"UTC\"")
    String getDefaultTimeZoneId();

    // From org.embulk.spi.time.TimestampParser.Task.
    @Config("default_timestamp_format")
    @ConfigDefault("\"%Y-%m-%d %H:%M:%S.%N %z\"")
    String getDefaultTimestampFormat();

    // From org.embulk.spi.time.TimestampParser.Task.
    @Config("default_date")
    @ConfigDefault("\"1970-01-01\"")
    String getDefaultDate();
}

// Define your own TimestampColumnOption compatible with org.embulk.spi.time.Timestamp{Formatter|Parser}.TimestampColumnOption.
private interface TimestampColumnOption extends org.embulk.util.config.Task {
    @Config("timezone")
    @ConfigDefault("null")
    java.util.Optional<String> getTimeZoneId();

    @Config("format")
    @ConfigDefault("null")
    java.util.Optional<String> getFormat();

    @Config("date")
    @ConfigDefault("null")
    java.util.Optional<String> getDate();
}

// Get the entire configuration.
final PluginTask task = configMapper.map(configSource, PluginTask.class);

// Get per-column option with embulk-util-config.
final TimestampColumnOption columnOption = configMapper.map(columnConfig.getOption(), TimestampColumnOption.class);

// Build TimestampFormatter.
final TimestampFormatter formatter = TimestampFormatter
        .builder(columnOption.getFormat().orElse(task.getDefaultTimestampFormat(), true)
        .setDefaultZoneFromString(columnOption.getTimeZoneId().orElse(task.getDefaultTimeZoneId()))
        .setDefaultDateFromString(columnOption.getDate().orElse(task.getDefaultDate()))
        .build();

Instant instant = formatter.parse("2019-02-28 12:34:56 +09:00");
System.out.println(instant);  // => "2019-02-28T03:34:56Z"

String formatted = formatter.format(Instant.ofEpochSecond(1009110896));
System.out.println(formatted);  // => "2017-12-23 12:34:56 UTC"
```

##### Using Jackson and JSON

`org.embulk:embulk-core` no longer has dependencies on Jackson. If your plugin needs Jackson for itself, it needs to have dependencies on Jackson by itself.

If your plugin has started to use `org.embulk:embulk-util-config` described above, your plugin should already have dependencies on Jackson (2.6.7). You can add some more Jackson sub-librarires by yourself.

Embulk's `org.embulk.spi.json.JsonParser` is deprecated. Use [`org.embulk:embulk-util-json`](https://dev.embulk.org/embulk-util-json/) instead. The usage is almost the same.

As described above, please consider keeping your plugin with Jackson 2.6.7 for a while from Embulk v0.11.0 so that the plugin can still work with the old stable Embulk v0.9.

##### Using Google Guava and/or Apache Commons Lang 3

`org.embulk:embulk-core` no longer has dependencies on Google Guava nor Apache Commons Lang 3. If your plugin uses them, it needs to be rewritten to Java 8's standard classes, or to have dependencies on them by itself.

Java 8 is powerful enough. Unless your plugin has a heavy use of Google Guava or Apache Commons Lang 3, we'd basically recommend you to rewrite them with Java 8's standard classes. They (especially Guava) often introduce incompatibility that could frustrates you.

At least, Guava's [`com.google.common.base.Optional`](https://guava.dev/releases/18.0/api/docs/com/google/common/base/Optional.html) no longer works with `embulk-util-config`. It needs to be replaced with [`java.util.Optional`](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html).

For typical use-cases of Guava:

* Guava's immutable collections could be replaced with [`java.util.Collections`](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html) `unmodifiable*`.
* Guava's [`Throwables`](https://guava.dev/releases/18.0/api/docs/com/google/common/base/Throwables.html) has been deprecated. Read [Guava's document](https://github.com/google/guava/wiki/Why-we-deprecated-Throwables.propagate).
* Guava's [`Preconditions`](https://guava.dev/releases/18.0/api/docs/com/google/common/base/Preconditions.html) can be replaced simply.

##### Using Guice

Stop using Guice in your plugin if possible.

If it does not seem possible, for example if another dependency has a dependency on Guice, we'd suggest to have a dependency on Guice explicitly. But, note that it may make your plugin not to work with older Embulk v0.9.

##### Using JRuby (`org.jruby` classes)

Stop using JRuby in your plugin.

A typical use-case of JRuby has been to use Ruby's date-time formatter. It can be replaced with [`embulk-util-timestamp`](https://dev.embulk.org/embulk-util-timestamp/) or [`embulk-util-rubytime`](https://dev.embulk.org/embulk-util-rubytime/).

##### Using FindBugs

`org.embulk:embulk-core` no longer has a dependency on FindBugs `com.google.code.findbugs:annotations` (for `@SuppressFBWarnings`).

[FindBugs](http://findbugs.sourceforge.net/) is known to be no longer maintained. We'd at least suggest to stop using FindBugs.

[SpotBugs](https://spotbugs.github.io/) can be a good alternative to FindBugs. But, the Embulk core will not provide any special support for plugins to use SpotBugs. Please give it a try by yourself.

##### Prepare for Java 11+

The Java eco-system needs to be prepared for later Java versions. A big jump there is from Java 8 to Java 11+. Java EE classes are removed from the Java runtime environments from Java 11. If your plugin uses those classes, it will stop working with Java 11+. JAXB `javax.xml.*` is a typical use-case. See [JEP 320](https://openjdk.java.net/jeps/320) for the details.

When Embulk (v0.11+) detects a plugin using a class that is removed by JEP 320, it just logs a warning message like an example below even on Java 8.

```
Class javax.xml.bind.JAXB is loaded by the parent ClassLoader, which is removed by JEP 320. The plugin needs to include it on the plugin side. See https://github.com/embulk/embulk/issues/1270 for more details.
```

Embulk, as a framework, will not provide any more special help with them. If your plugin uses those Java EE classes, please add alternative dependencies in the plugin by yourself.

Some typical examples are shown below.

```
// Adding dependencies on JAXB explicitly.

// JAXB 2.2.11 is chosen here because:
// 1. JDK 8's bundled JAXB is 2.2.8. Better with a closer version while we are on Java 8.
//    https://javaee.github.io/jaxb-v2/doc/user-guide/ch02.html#a-2-2-8
// 2. Neither com.sun.xml.bind:jaxb-core:2.2.8 nor com.sun.xml.bind:jaxb-impl:2.2.8 does not exist on Maven Central.
// 3. 2.2.11 looks to be used by the most Java libraries among JAXB 2.2.
//    https://mvnrepository.com/artifact/javax.xml.bind/jaxb-api
//    https://mvnrepository.com/artifact/com.sun.xml.bind/jaxb-core
//    https://mvnrepository.com/artifact/com.sun.xml.bind/jaxb-impl
// 4. JAXB 2.2.8 and 2.2.11 look to have the same set of classes.
//    Although their internal implementations are a bit different, class loaders would not be confused.
compile "javax.xml.bind:jaxb-api:2.2.11"
compile "com.sun.xml.bind:jaxb-core:2.2.11"
compile "com.sun.xml.bind:jaxb-impl:2.2.11"
```

```
// Adding a dependency on the Activation Framework explicitly.

// The Activation Framework is often required along with JAXB.
// 1.1.1 is chosen because it was the latest version when JAXB 2.2.11 was released.
compile "javax.activation:activation:1.1.1"
```

```
// Adding a dependency on the Annotation API explicitly.

// 1.2 is chosen because only 1.2 should have been available in the age of Java 8.
// https://mvnrepository.com/artifact/javax.annotation/javax.annotation-api
compile "javax.annotation:javax.annotation-api:1.2"
```

##### Log

`Exec.getLogger` is deprecated. Use [SLF4J](http://www.slf4j.org/)'s [`org.slf4j.LoggerFactory.getLogger`](https://www.javadoc.io/doc/org.slf4j/slf4j-api/1.7.30/org/slf4j/LoggerFactory.html) directly.

##### Other utility classes in `org.embulk:embulk-core`

* Processing file-like byte sequence (ex. `FileInputInputStream` and `FileOutputOutputStream` in `org.embulk.spi.util`)
    * Exported as [`embulk-util-file`](https://dev.embulk.org/embulk-util-file/)
* Processing text (ex. `LineDecoder` and `LineEncoder` in `org.embulk.spi.util`)
    * Exported as [`embulk-util-text`](https://dev.embulk.org/embulk-util-text/)
* Dynamic column setter (ex. `DynamicColumnSetter` in `org.embulk.spi.util`)
    * Exported as [`embulk-util-dynamic`](https://dev.embulk.org/embulk-util-dynamic/)
* Retries (ex. `RetryExecutor` in `org.embulk.spi.util`)
    * Exported as [`embulk-util-retryhelper`](https://dev.embulk.org/embulk-util-retryhelper/)

#### Summary of replacement

##### Processing Embulk configurations

| Old | New |
|---|---|
| `@Config` | `@org.embulk.util.config.Config` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `@ConfigDefault` | `@org.embulk.util.config.ConfigDefault` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `@ConfigInject` | `Exec.get???()` (for example, `Exec.getBufferAllocator`) |
| `ConfigSource#loadConfig` | `ConfigMapper#map` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `TaskSource#loadTask` | `TaskMapper#map` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `Task` | `org.embulk.util.config.Task` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `ColumnConfig` | `org.embulk.util.config.units.ColumnConfig` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `SchemaConfig` | `org.embulk.util.config.units.SchemaConfig` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `ModelManager` | Stop using it, and build your own Jackson `ObjectMapper` |
| `LocalFile` in `PluginTask` | Use `org.embulk.util.config.modules.LocalFileModule` and `org.embulk.util.config.units.LocalFile` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `Charset` in `PluginTask` | Use `org.embulk.util.config.modules.CharsetModule` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `Column` in `PluginTask` | Use `org.embulk.util.config.modules.ColumnModule` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `Schema` in `PluginTask` | Use `org.embulk.util.config.modules.SchemaModule` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `Type` in `PluginTask` | Use `org.embulk.util.config.modules.TypeModule` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |

##### Handling date/time and time zones

| Old | New |
|---|---|
| `org.joda.time.DateTime` | `java.time.OffsetDateTime` or `java.time.ZonedDateTime` |
| `org.joda.time.DateTimeZone` | `java.time.ZoneOffset` or `java.time.ZoneId` |
| `org.joda.time.DateTimeZone` in `PluginTask` | `java.time.ZoneId` with `org.embulk.util.config.modules.ZoneIdModule` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) if needed |
| `Timestamp` | `java.time.Instant` as far as possible |
| `TimestampFormatter` | `org.embulk.util.timestamp.TimestampFormatter` in [embulk-util-timestamp](https://dev.embulk.org/embulk-util-timestamp/) |
| `TimestampParser` | `org.embulk.util.timestamp.TimestampFormatter` in [embulk-util-timestamp](https://dev.embulk.org/embulk-util-timestamp/) |

##### Using Jackson and JSON

| Old | New |
|---|---|
| `org.embulk.spi.json.*` | `org.embulk.util.json.*` in [embulk-util-json](https://dev.embulk.org/embulk-util-json/) |

##### Using Google Guava and/or Apache Commons Lang 3

| Old | New |
|---|---|
| Guava `Optional` | `java.util.Optional` |
| Guava `Throwables` | Read [Guava's document](https://github.com/google/guava/wiki/Why-we-deprecated-Throwables.propagate) |

##### Using JRuby (`org.jruby` classes)

| Old | New |
|---|---|
| `org.jruby.*` | Stop using it -- [embulk-util-rubytime](https://dev.embulk.org/embulk-util-rubytime/) may work for parsing date-time  |

##### Using FindBugs

| Old | New |
|---|---|
| `@SuppressFBWarnings` | Remove it now |

##### Prepare for Java 11+

| Old | New |
|---|---|
| `javax.*` | See [JEP 320](https://openjdk.java.net/jeps/320), and check if it's removed in Java 11+ ([Issue](https://github.com/embulk/embulk/issues/1270)) |

##### Log

| Old | New |
|---|---|
| `Exec.getLogger` | `org.slf4j.LoggerFactory.getLogger` |

##### Other utility classes in `org.embulk:embulk-core`

| Old | New |
|---|---|
| `org.embulk.spi.util.*` | [embulk-util-file](https://dev.embulk.org/embulk-util-file/), [embulk-util-text](https://dev.embulk.org/embulk-util-text/), [embulk-util-dynamic](https://dev.embulk.org/embulk-util-dynamic/), or [embulk-util-retryhelper](https://dev.embulk.org/embulk-util-retryhelper/) |

#### Tests

Tests still need to depend on `org.embulk:embulk-core`, and in addition, `org.embulk:embulk-deps`.

```
testCompile "org.embulk:embulk-core:0.10.??"
testCompile "org.embulk:embulk-deps:0.10.??"
```

You may want to use some internal (`embulk-core`-only) classes in tests. Typical instances can be retrieved from `org.embulk.spi.ExecInternal`, such as `ExecInternal.getModelManager()` to get `org.embulk.config.ModelManager`.

Note that such internal classes may easily drop its compatibility. For example, `ModelManager` has already dropped `public <T> T readObject(Class<T>, com.fasterxml.jackson.core.JsonParser)` and `public <T> T readObjectWithConfigSerDe(Class<T>, com.fasterxml.jackson.core.JsonParser)`.

The testing framework would be improved through v0.11.
