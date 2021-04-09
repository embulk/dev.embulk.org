---
layout: posts
title: "Java plugins to catch up with Embulk v0.10 from v0.9"
date: 2020-07-21 17:56:50 +0900
last_modified_at: 2021-04-09 10:50:49 +0900
authors:
- "dmikurube"
---

**The page would be updated according to Embulk updates.**

### Apply the Gradle plugin `org.embulk.embulk-plugins`

Follow the guidance at: [https://github.com/embulk/gradle-embulk-plugins](https://github.com/embulk/gradle-embulk-plugins)

It needs Gradle 6.

#### Choose your own Maven `groupId`

Java-based Embulk plugins are to be released as Maven artifacts, in addition to Ruby gems. Set your own Maven `groupId` to make it available!

Note that `org.embulk` is only for plugins in [https://github.com/embulk/](https://github.com/embulk/). Use your own `groupId` for your own plugins, **not** `org.embulk`.

##### If you are not familiar with Maven's naming convention

Nice to read the [Guide to naming conventions on groupId, artifactId, and version](https://maven.apache.org/guides/mini/guide-naming-conventions.html) at first.

##### If you have your own domain name

A typical manner is to use the domain name of yours in a reverse order. For example:

```
group = "com.example.any_sub_domain_if_wanted)"  // NOTE: "com.example" is just an example. Do not use "com.example" actually!
```

##### If you don't have your own domain name

We observe that people often use `io.github.???`. See [`io.github` in Maven Central](https://repo1.maven.org/maven2/io/github/). It should be safe because the sub domain name `<your-github-username>.github.io` is allocated to your GitHub account unless you remove the GitHub account of yours.

```
group = "io.github.<your-github-username>.any_sub_domain_if_wanted"
```

If your GitHub username includes `-` (hyphen) or `_` (underscore), please make sure what sub domain name you own under `github.io`. Domain names must not include `_` (underscore) in general. So, GitHub may perform some conversion in your sub domain name under `github.io`, and the converted name may conflict with others.

#### Configure to release to your preferred Maven repository

We don't limit to specific Maven repositories to release Embulk plugins. You can choose your preferred one: Maven Central, Bintray, JCenter, GitHub Packages, or else... But anyway, you will need to configure your `build.gradle` for your destination.

For example, if you are releasing to Maven Central, the configuration would be like below. See [their requirements](https://central.sonatype.org/pages/requirements.html), jfyi.

```
publishing {
    publications {
        // ...
    }

    repositories {
        maven {  // publishMavenPublicationToMavenCentralRepository
            name = "mavenCentral"
            if (project.version.endsWith("-SNAPSHOT")) {
                url "https://oss.sonatype.org/content/repositories/snapshots"
            } else {
                url "https://oss.sonatype.org/service/local/staging/deploy/maven2"
            }

            credentials {
                username = project.hasProperty("ossrhUsername") ? ossrhUsername : ""
                password = project.hasProperty("ossrhPassword") ? ossrhPassword : ""
            }
        }
    }
}

signing {  // Maven Central needs signing.
    sign publishing.publications.maven
}
```

### If your plugin uses Jackson, Guava, Apache commons-lang3, or `javax.validation`

Include those libraries explicitly in `dependencies`. Use **the 100% same versions** with `org.embulk:embulk-core:0.10.29`.

```
"com.fasterxml.jackson.core:jackson-annotations:2.6.7"
"com.fasterxml.jackson.core:jackson-core:2.6.7"
"com.fasterxml.jackson.core:jackson-databind:2.6.7"
"com.fasterxml.jackson.datatype:jackson-datatype-guava:2.6.7"
"com.fasterxml.jackson.datatype:jackson-datatype-jdk8:2.6.7"
"com.fasterxml.jackson.datatype:jackson-datatype-joda:2.6.7"
"com.google.inject:guice:4.0"
"javax.validation:validation-api:1.1.0.Final"
"org.apache.commons:commons-lang3:3.4"
```

`embulk-core` would stop depending on those libraries at some point during `v0.10`. Including the 100% same versions of them keeps your plugin work before and after the removal in `embulk-core`.

### If your plugin uses Joda-Time in it

Use Java 8's standard `java.time` classes instead.

If your dependency uses Joda-Time transitively, include Joda-Time explicitly in your dependencies, exact 2.9.2. But we basically recommend to replace Joda-Time with `java.time`.

```
"joda-time:joda-time:2.9.2"
```

### If your plugin uses Guice, JRuby (`org.jruby` classes), or Logback (`ch.qos.logback` classes)

Stop using them, and find alternatives.

A typical use-case of JRuby has been to use Ruby's date-time formatter. It can be replaced with [`embulk-util-timestamp`](https://dev.embulk.org/embulk-util-timestamp/) or [`embulk-util-rubytime`](https://dev.embulk.org/embulk-util-rubytime/) now.

### If your plugin uses the standard Java EE classes, typically `javax.xml.*`

Java 9 and 10 need a command line option to use those classes. Java 11+ no longer includes those classes. See [JEP 320](https://openjdk.java.net/jeps/320).

To make the plugin work even with Java 9+, include them explicitly in your dependencies.

Embulk will never provide those classes by itself for plugins. It will be of plugin's responsibility.

### If your plugin uses Embulk's `TimestampFormatter` or `TimestampParser`

Those are deprecated. Use [`embulk-util-timestamp`](https://dev.embulk.org/embulk-util-timestamp/) instead.

### (Almost all the plugins) If your plugin uses Embulk's `ConfigSource#loadConfig` or `TaskSource#loadTask`

They will be gradually deprecated. Try using [`embulk-util-config`](https://dev.embulk.org/embulk-util-config/) instead.

### If your plugin uses the followings :

| Old | New |
|---|---|
| `Exec.getLogger` | `org.slf4j.LoggerFactory.getLogger` |
| `org.embulk.spi.Timestamp` | `java.time.Instant` as far as possible |
| Guava `Optional` | `java.util.Optional` |
| Guava `Throwables` | Read [Guava's document](https://github.com/google/guava/wiki/Why-we-deprecated-Throwables.propagate) |
| `org.joda.time.*` | `java.time.*` |
| `org.jruby.*` | Stop using it -- [embulk-util-rubytime](https://dev.embulk.org/embulk-util-rubytime/) may work for parsing date-time  |
| `javax.*` | See [JEP 320](https://openjdk.java.net/jeps/320), and check if it's removed in Java 11+ ([Issue](https://github.com/embulk/embulk/issues/1270)) |
| `SuppressFBWarnings` | Remove it -- FindBugs is no longer supported |
| `ModelManager` | Stop using it, and build your own Jackson `ObjectMapper` |
| `@Config` | `@org.embulk.util.config.Config` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `@ConfigDefault` | `@org.embulk.util.config.ConfigDefault` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `@ConfigInject` | `Exec.get???()` |
| `ConfigSource#loadConfig` | `ConfigMapper#map` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `TaskSource#loadTask` | `TaskMapper#map` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `Task` | `org.embulk.util.config.Task` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `ColumnConfig` | `org.embulk.util.config.units.ColumnConfig` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `SchemaConfig` | `org.embulk.util.config.units.SchemaConfig` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `LocalFile` in `PluginTask` | Use `org.embulk.util.config.modules.LocalFileModule` and `org.embulk.util.config.units.LocalFile` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `Charset` in `PluginTask` | Use `org.embulk.util.config.modules.CharsetModule` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `Column` in `PluginTask` | Use `org.embulk.util.config.modules.ColumnModule` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `Schema` in `PluginTask` | Use `org.embulk.util.config.modules.SchemaModule` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `Type` in `PluginTask` | Use `org.embulk.util.config.modules.TypeModule` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `Timestamp` in `PluginTask` | (deprecated) Use `org.embulk.util.config.modules.TimestampModule` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `DateTimeZone` in `PluginTask` | Use `java.time.ZoneId` with `org.embulk.util.config.modules.ZoneIdModule` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) |
| `org.embulk.spi.json.*` | `org.embulk.util.json.*` in [embulk-util-json](https://dev.embulk.org/embulk-util-json/) |
| `org.embulk.spi.util.*` | `org.embulk.util.file.*` in [embulk-util-file](https://dev.embulk.org/embulk-util-file/), or `org.embulk.util.text.*` in [embulk-util-text](https://dev.embulk.org/embulk-util-text/) |
| `RetryExecutor` | `org.embulk.util.retryhelper.RetryExecutor` in [embulk-util-retryhelper](https://dev.embulk.org/embulk-util-retryhelper/) |
| `TimestampFormatter` | `org.embulk.util.timestamp.TimestampFormatter` in [embulk-util-timestamp](https://dev.embulk.org/embulk-util-timestamp/) |
| `TimestampParser` | `org.embulk.util.timestamp.TimestampFormatter` in [embulk-util-timestamp](https://dev.embulk.org/embulk-util-timestamp/) |

### If your plugin depends on `org.embulk:embulk-core` for the main code

Remove `org.embulk:embulk-core`, and depend only on the following two instead.

```
compileOnly "org.embulk:embulk-api:0.10.29"
compileOnly "org.embulk:embulk-spi:0.10.29"
```

Once you have done the replacements explained above, `org.embulk:embulk-core` should no longer be required. (If you see an error by removing `org.embulk:embulk-core`, it means that you still have something to replace.)

### Tests

Tests still need to depend on `org.embulk:embulk-core`, and in addition, `org.embulk:embulk-deps`.

```
testCompile "org.embulk:embulk-core:0.10.29"
testCompile "org.embulk:embulk-deps:0.10.29"
```

### Release your plugin to RubyGems.org, and your preferred Maven repository in addition

Java-based Embulk plugins are to be released as Maven artifacts, in addition to Ruby gems. When releasing your plugin, run two tasks like that, to release a Maven artifact and a Ruby gem.

```
./gradlew publishMavenPublicationTo<???>Repository gemPush  # NOTE: "<???>" would depend on your configuration.
```
