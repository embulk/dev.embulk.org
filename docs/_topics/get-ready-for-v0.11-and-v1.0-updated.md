---
layout: posts
title: "For Embulk plugin developers: Get ready for v0.11 and v1.0! [Updated]"
date: 2023-09-29
authors:
- "dmikurube"
---

[Embulk v0.11.0 has been released!](https://github.com/embulk/embulk/releases/tag/v0.11.0)  For Embulk users, an introduction of v0.11 is here: "[Embulk v0.11 is coming soon](https://www.embulk.org/articles/2023/04/13/embulk-v0.11-is-coming-soon.html)"

This post is for Embulk plugin developers how to get your Embulk plugin(s) ready for v0.11.

### Before all: Ruby plugins

The JRuby integration with Embulk will not be maintained actively. We will not remove it sooner, but Ruby-based plugins will not be supported actively as a matter of high priority.

If you need to maintain your Ruby-based Embulk plugin, consider rewriting it to Java.

### Simple: Fully catch up with Embulk v0.11 and v1.0, no longer with v0.9

Fully catching up with Embulk v0.11, and getting ready for upcoming v1.0, mean dropping support for Embulk v0.9. It would be easier, simpler, and recommended unless yourself or your users still need Embulk v0.9. We'd explain the full catch up at first.

#### Upgrade the Gradle wrapper to 7.6.1

Embulk v0.11+ expects that plugins are built with [the `org.embulk.embulk-plugins` Gradle plugin](https://plugins.gradle.org/plugin/org.embulk.embulk-plugins). Its recent version 0.6.1 needs Gradle 7.6.1. Gradle 8 is not officially supported yet.

```
./gradlew wrapper --gradle-version=7.6.1
```

Note that you may need to replace `compile` dependencies to `implementation` when upgrading Gradle from 6 or earlier to 7.

#### Apply the `org.embulk.embulk-plugins` Gradle plugin

Apply the latest [`org.embulk.embulk-plugins` Gradle plugin](https://plugins.gradle.org/plugin/org.embulk.embulk-plugins), and rewrite your `build.gradle`.

Follow [`README.md` of the Gradle plugin](https://github.com/embulk/gradle-embulk-plugins/blob/master/README.md) to check how to rewrite `build.gradle`.

#### Prepare for publishing your plugin to Maven Central

[Maven Central](https://central.sonatype.com/) would be the first choice to publish an Embulk plugin after Embulk v0.11 unless your plugin is closed proprietary.

[`README.md` of the Gradle plugin](https://github.com/embulk/gradle-embulk-plugins/blob/master/README.md) explains the basics to publish your plugin to Maven Central. Picking up the key points :

* Choose an appropriate Maven group ID.
  * Do not use `org.embulk` unless your plugin is maintained under [https://github.com/embulk](https://github.com/embulk)
  * Use your own domain (ex. "com.example"), or
  * Use your GitHub account (ex. "io.github.your-github-user")
* Configure `publishing` appropriately
  * `javadocJar` and `sourcesJar` are mandatory for Maven Central
  * Some `pom.xml` configurations are mandatory for Maven Central (ex. `licenses`, `developers`, `scm`)
* Configure `signing` appropriately
  * Signing packages with PGP is mandatory for Maven Central
  * See: [https://central.sonatype.org/publish/requirements/gpg/](https://central.sonatype.org/publish/requirements/gpg/)

#### Prepare for publishing your plugin to RubyGems.org

For backward compatibility, you may still want to publish your plugin to [RubyGems.org](https://rubygems.org/) for a while. Use the `gem` and `gemPush` tasks defined in [`org.embulk.embulk-plugins` Gradle plugin](https://plugins.gradle.org/plugin/org.embulk.embulk-plugins) to publish your plugin to [RubyGems.org](https://rubygems.org/).

However, please keep it in your mind that the RubyGem style is no longer the first choice for Embulk plugins.

#### Cleanup dependencies: Overall

As described in [`README.md` of the Gradle plugin](https://github.com/embulk/gradle-embulk-plugins/blob/master/README.md), Embulk plugins should no longer depend on `org.embulk:embulk-core`. An Embulk plugin should depend only on `org.embulk:embulk-spi` from the Embulk core. An Embulk plugin should use neither a class only in `org.embulk:embulk-core`, nor a library that is depended from `embulk-core`.

Classes in `org.embulk:embulk-core` have been categoried into three types. The first type is SPI (Service Provider Interface), which is an official contract between the Embulk core and plugins. SPI classes are included in the new Maven artifact `org.embulk:embulk-spi`. Embulk plugins can depend on `embulk-spi`, and use the SPI classes. They are kept. See [Javadoc](https://dev.embulk.org/embulk-spi/0.11/javadoc/) of the SPI classes.

The second type is Embulk's internal classes. They are only in the Maven artifact `org.embulk:embulk-core`. Embulk plugins should no longer use them. They have no guarantee about their behavior, compatibility, and existence.

The third type is utility classes. They are deprecated only in the Maven artifact `org.embulk:embulk-core`. They have been exported as external librarires `org.embulk:embulk-util-*` under different Java package names. Embulk plugins should start using those exported libraries, instead of the core utility classes.

In addition, many library dependenceis from `embulk-core` are removed, such as Apache Commons, Guava, Guice, Jackson, and Joda-Time. An Embulk plugin has to include those dependencies by itself to continue using them.

#### Cleanup dependencies: SPI

As explained above, SPI classes are moved into `org.embulk:embulk-spi` out of `embulk-core`. All Embulk plugins are expected to depend on `embulk-spi`.

This `embulk-spi` has a different version scheme from the Embulk core. Its version has only two digits, such as `0.11`. The minor version is incremented for any update on `embulk-spi`. The major version is incremented for any incompatible update.

Note that Embulk SPI versions will not align with Embulk core versions. The Embulk SPI may stay `0.*` for a while even after the Embulk core v1.0 is released. It would go `1.0` when the Embulk SPI drops some deprecated things, such as the dependency on `org.msgpack:msgpack-core`.

#### Cleanup dependencies: Configuration processor

Embulk plugins had used `ConfigSource#loadConfig` and `TaskSource#loadTask` to process Embulk configurations. They are deprecated. The configuration processor has been exported as an external library [`org.embulk:embulk-util-config`](https://dev.embulk.org/embulk-util-config/). Almost all the Embulk plugins would use this library.

```
    // build.gradle
    implementation "org.embulk:embulk-util-config:0.3.4"
```

It is not very complicated how to use it. At first, replace `@Config`, `@ConfigDefault`, and `Task` into `embulk-util-config`'s.

```
- import org.embulk.config.Config;
- import org.embulk.config.ConfigDefault;
- import org.embulk.config.Task;
+ import org.embulk.util.config.Config;
+ import org.embulk.util.config.ConfigDefault;
+ import org.embulk.util.config.Task;
```

Then, check your plugin's `Task` definition. It has no problems if it extends only `Task` (`org.embulk.config.Task` => `org.embulk.util.config.Task`).

If it extends another interface (a typical example is `org.embulk.spi.time.TimestampParser.Task`), find a newer alternative, or copy its original methods into your `Task`, so that they are compatible for users. (See: [`TimestampParser.Task` in Embulk v0.9.25](https://github.com/embulk/embulk/blob/v0.9.25/embulk-core/src/main/java/org/embulk/spi/time/TimestampParser.java#L160-L183))

```
public interface PluginTask extends Task {  // Remove extending "TimestampParser.Task"
    // ...

    // Copy this from TimestampParser.Task
    @Config("default_timezone")
    @ConfigDefault("\"UTC\"")
    String getDefaultTimeZoneId();

    // Copy this from from TimestampParser.Task
    @Config("default_timestamp_format")
    @ConfigDefault("\"%Y-%m-%d %H:%M:%S.%N %z\"")
    String getDefaultTimestampFormat();

    // Copy this from TimestampParser.Task
    @Config("default_date")
    @ConfigDefault("\"1970-01-01\"")
    String getDefaultDate();
}
```

Next, prepare an instance of `org.embulk.util.config.ConfigMapperFactory`.

```
import org.embulk.util.config.ConfigMapperFactory;

private static final ConfigMapperFactory CONFIG_MAPPER_FACTORY =
        ConfigMapperFactory.builder().addDefaultModules().build();
```

You can add your preferred Jackson [Module](https://fasterxml.github.io/jackson-databind/javadoc/2.6/com/fasterxml/jackson/databind/Module.html) if you need to map an Embulk configuration to a non-standard class.

```
private static final ConfigMapperFactory CONFIG_MAPPER_FACTORY =
        ConfigMapperFactory.builder().addDefaultModules().addModule(new SomeJacksonModule()).build();
```

You can also enable a validator by Apache BVal.

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

If your plugin creates a configuration instance by [`Exec.newConfigDiff()`](https://dev.embulk.org/embulk-spi/0.11/javadoc/org/embulk/spi/Exec.html#newConfigDiff--), [`Exec.newConfigSource()`](https://dev.embulk.org/embulk-spi/0.11/javadoc/org/embulk/spi/Exec.html#newConfigSource--), [`Exec.newTaskReport()`](https://dev.embulk.org/embulk-spi/0.11/javadoc/org/embulk/spi/Exec.html#newTaskReport--), and/or [`Exec.newTaskSource()`](https://dev.embulk.org/embulk-spi/0.11/javadoc/org/embulk/spi/Exec.html#newTaskSource--), replace them with [`CONFIG_MAPPER_FACTORY.newConfigDiff()`](https://dev.embulk.org/embulk-util-config/0.3.4/javadoc/org/embulk/util/config/ConfigMapperFactory.html#newConfigDiff--), [`CONFIG_MAPPER_FACTORY.newConfigSource()`](https://dev.embulk.org/embulk-util-config/0.3.4/javadoc/org/embulk/util/config/ConfigMapperFactory.html#newConfigSource--), [`CONFIG_MAPPER_FACTORY.newTaskReport()`](https://dev.embulk.org/embulk-util-config/0.3.4/javadoc/org/embulk/util/config/ConfigMapperFactory.html#newTaskReport--), and/or [`CONFIG_MAPPER_FACTORY.newTaskSource()`](https://dev.embulk.org/embulk-util-config/0.3.4/javadoc/org/embulk/util/config/ConfigMapperFactory.html#newTaskSource--), respectively. The old `Exec.new...()` methods are for the old `org.embulk.config`, not for `embulk-util-config`.

| Old | New |
| --- | --- |
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

#### Cleanup dependencies: Timestamps

[Joda-Time](https://www.joda.org/joda-time/) has been removed from Embulk. `org.embulk.spi.time.Timestamp`, `org.embulk.spi.time.TimestampFormatter` and `org.embulk.spi.time.TimestampParser` are deprecated.

Embulk now uses the Java standard [`java.time.Instant`](https://docs.oracle.com/javase/8/docs/api/java/time/Instant.html) as the `TIMESTAMP` column type. The timestamp parser and formatter have been exported as an external library [`org.embulk:embulk-util-timestamp`](https://dev.embulk.org/embulk-util-timestamp/).

```
    // build.gradle
    implementation "org.embulk:embulk-util-timestamp:0.2.2"
```

At first, use the following new methods in the Embulk SPI to start using `java.time.Instant` as the `TIMESTAMP` column type.

* [`PageBuilder#setTimestamp(Column, Instant)`](https://dev.embulk.org/embulk-spi/0.11/javadoc/org/embulk/spi/PageBuilder.html#setTimestamp-org.embulk.spi.Column-java.time.Instant-)
* [`PageBuilder#setTimestamp(int, Instant)`](https://dev.embulk.org/embulk-spi/0.11/javadoc/org/embulk/spi/PageBuilder.html#setTimestamp-int-java.time.Instant-)
* [`PageReader#getTimestampInstant(Column)`](https://dev.embulk.org/embulk-spi/0.11/javadoc/org/embulk/spi/PageReader.html#getTimestampInstant-org.embulk.spi.Column-)
* [`PageReader#getTimestampInstant(int)`](https://dev.embulk.org/embulk-spi/0.11/javadoc/org/embulk/spi/PageReader.html#getTimestampInstant-int-)

Next, replace use of `org.embulk.spi.time.TimestampFormatter` and `org.embulk.spi.time.TimestampParser` into [`org.embulk.util.timestamp.TimestampFormatter`](https://dev.embulk.org/embulk-util-timestamp/0.2.2/javadoc/org/embulk/util/timestamp/TimestampFormatter.html) in `embulk-util-timestamp`.

The typical "as-is" migration would be like below:

```
// Define your own PluginTask with org.embulk.util.config.Task of embulk-util-config (explained above).
public interface PluginTask extends org.embulk.util.config.Task {
    // ...

    // Copy this from TimestampParser.Task
    @Config("default_timezone")
    @ConfigDefault("\"UTC\"")
    String getDefaultTimeZoneId();

    // Copy this from from TimestampParser.Task
    @Config("default_timestamp_format")
    @ConfigDefault("\"%Y-%m-%d %H:%M:%S.%N %z\"")
    String getDefaultTimestampFormat();

    // Copy this from TimestampParser.Task
    @Config("default_date")
    @ConfigDefault("\"1970-01-01\"")
    String getDefaultDate();
}

// Define your own TimestampColumnOption so that the configuration will be compatible for users.
//
// See also the old TimestampFormatter and TimestampParser.
// https://github.com/embulk/embulk/blob/v0.9.25/embulk-core/src/main/java/org/embulk/spi/time/TimestampParser.java#L185-L208
// https://github.com/embulk/embulk/blob/v0.9.25/embulk-core/src/main/java/org/embulk/spi/time/TimestampFormatter.java#L120-L139
public interface TimestampColumnOption extends org.embulk.util.config.Task {
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

Instant instant = formatter.parse("2019-02-28 12:34:56 +09:00");  // => "2019-02-28T03:34:56Z"

String formatted = formatter.format(Instant.ofEpochSecond(1009110896));  // => "2017-12-23 12:34:56 UTC"
```

| Old | New |
| --- | --- |
| `org.joda.time.DateTime` | `java.time.OffsetDateTime` or `java.time.ZonedDateTime` |
| `org.joda.time.DateTimeZone` | `java.time.ZoneOffset` or `java.time.ZoneId` |
| `org.joda.time.DateTimeZone` in `PluginTask` | `java.time.ZoneId` with `org.embulk.util.config.modules.ZoneIdModule` in [embulk-util-config](https://dev.embulk.org/embulk-util-config/) if needed |
| `Timestamp` | `java.time.Instant` |
| `TimestampFormatter` | `org.embulk.util.timestamp.TimestampFormatter` in [embulk-util-timestamp](https://dev.embulk.org/embulk-util-timestamp/) |
| `TimestampParser` | `org.embulk.util.timestamp.TimestampFormatter` in [embulk-util-timestamp](https://dev.embulk.org/embulk-util-timestamp/) |

#### Cleanup dependencies: JSON and Jackson

[`org.msgpack.value.Value`](https://javadoc.io/static/org.msgpack/msgpack-core/0.8.24/org/msgpack/value/Value.html) in [MessagePack for Java](https://github.com/msgpack/msgpack-java) had been used to represent the `JSON` column type in Embulk. This use of `Value` is deprecated. Represent the `JSON` column type by the new [`org.embulk.spi.json.JsonValue`](https://dev.embulk.org/embulk-spi/0.11/javadoc/org/embulk/spi/json/JsonValue.html) instead. `JsonValue` has almost the same features with `Value` with a little bit different method names.

The `Value`-based SPI methods still exist in Embulk v0.11 and SPI `0.11`, but they will be removed at some point in Embulk v1.0 and SPI `1.0`. Embulk plugins have to start using the new `JsonValue`-based SPI methods.

`org.embulk.spi.json.JsonParser`, which parses JSON into `Value`, will be also removed. Use [`org.embulk.util.json.JsonValueParser`](https://dev.embulk.org/embulk-util-json/0.3.0/javadoc/org/embulk/util/json/JsonValueParser.html) in [`embulk-util-json`](https://dev.embulk.org/embulk-util-json/) instead. Note that [`embulk-util-json`](https://dev.embulk.org/embulk-util-json/) has had [`org.embulk.util.json.JsonParser`](https://dev.embulk.org/embulk-util-json/0.3.0/javadoc/org/embulk/util/json/JsonParser.html), but it will be removed, too.

```
    // build.gradle
    implementation "org.embulk:embulk-util-json:0.3.0"
```

Embulk plugins can use newer versions of Jackson librarires from Embulk v0.11 and SPI `0.11`. The plugin may stop working with Embulk v0.9, but we'll eventually go that way.

| Old | New |
| --- | --- |
| `org.msgpack.value.Value` | `org.embulk.spi.json.JsonValue` in [embulk-spi](https://dev.embulk.org/embulk-spi/) |
| `org.msgpack.value.ArrayValue` | `org.embulk.spi.json.JsonArray` in [embulk-spi](https://dev.embulk.org/embulk-spi/) |
| `org.msgpack.value.MapValue` | `org.embulk.spi.json.JsonObject` in [embulk-spi](https://dev.embulk.org/embulk-spi/) |
| `org.msgpack.value.BooleanValue` | `org.embulk.spi.json.JsonBoolean` in [embulk-spi](https://dev.embulk.org/embulk-spi/) |
| `org.msgpack.value.FloatValue` | `org.embulk.spi.json.JsonDouble` in [embulk-spi](https://dev.embulk.org/embulk-spi/) |
| `org.msgpack.value.IntegerValue` | `org.embulk.spi.json.JsonLong` in [embulk-spi](https://dev.embulk.org/embulk-spi/) |
| `org.msgpack.value.NilValue` | `org.embulk.spi.json.JsonNull` in [embulk-spi](https://dev.embulk.org/embulk-spi/) |
| `org.msgpack.value.StringValue` | `org.embulk.spi.json.JsonString` in [embulk-spi](https://dev.embulk.org/embulk-spi/) |
| `org.embulk.spi.json.JsonParser` | `org.embulk.util.json.JsonValueParser` in [embulk-util-json](https://dev.embulk.org/embulk-util-json/) |
| `org.embulk.util.json.JsonParser` | `org.embulk.util.json.JsonValueParser` in [embulk-util-json](https://dev.embulk.org/embulk-util-json/) |

#### Cleanup dependencies: Logging

`Exec.getLogger` has been deprecated. Use [SLF4J](http://www.slf4j.org/)'s [`org.slf4j.LoggerFactory.getLogger`](https://www.javadoc.io/doc/org.slf4j/slf4j-api/1.7.30/org/slf4j/LoggerFactory.html) directly.

| Old | New |
| --- | --- |
| `Exec.getLogger` | `org.slf4j.LoggerFactory.getLogger` |

#### Cleanup dependencies: Guava and Guice

Embulk no longer contains [Google Guava](https://github.com/google/guava) nor [Google Guice](https://github.com/google/guice). An Embulk plugin has to have dependencies on them by itself when it needs Guava and/or Guice.

However, it would be nice to consider removing Guava from your plugin. Almost all the typical use-cases of Guava can be realized by Java 8 standard classes, for example :

* Guava's immutable collections could be replaced with [`java.util.Collections`](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html).
* Guava's [`Preconditions`](https://guava.dev/releases/18.0/api/docs/com/google/common/base/Preconditions.html) could be replaced with [`java.util.Objects`](https://docs.oracle.com/javase/8/docs/api/java/util/Objects.html).
* Guava's [`Throwables`](https://guava.dev/releases/18.0/api/docs/com/google/common/base/Throwables.html) has been deprecated. Read [Guava's document](https://github.com/google/guava/wiki/Why-we-deprecated-Throwables.propagate).

At least, Guava's [`com.google.common.base.Optional`](https://guava.dev/releases/18.0/api/docs/com/google/common/base/Optional.html) no longer works with `embulk-util-config`. It needs to be replaced with [`java.util.Optional`](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html).

Guice is no longer recommended to use in an Embulk plugin, but it is sometimes unavoidable due to transitive dependencies. Note that the plugin may stop working with Embulk v0.9 in that case.

| Old | New |
| --- | --- |
| Guava `Optional` | `java.util.Optional` |
| Guava `Preconditions` | `java.util.Objects` |
| Guava `Throwables` | Read [Guava's document](https://github.com/google/guava/wiki/Why-we-deprecated-Throwables.propagate) |

#### Cleanup dependencies: Apache Commons Lang

Embulk no longer contains [Apache Commons Lang](https://commons.apache.org/proper/commons-lang/). An Embulk plugin has to have dependencies by itself when it needs Apache Commons Lang.

However, it would be nice to consider removing Apache Commons Lang from your plugin. Almost all the typical use-cases of Apache Commons Lang can be realized by Java 8 standard classes.

#### Cleanup dependencies: JRuby

Stop using JRuby classes (`org.jruby`) in an Embulk plugin.

A typical use-case of a JRuby class has been Ruby's date/time formatter and parser. It can be replaced with [`embulk-util-timestamp`](https://dev.embulk.org/embulk-util-timestamp/).

| Old | New |
| --- | --- |
| `org.jruby.*` | Stop using it |

#### Cleanup dependencies: FindBugs

Embulk no longer contains a dependency on FindBugs `com.google.code.findbugs:annotations` (for `@SuppressFBWarnings`).

Note that [FindBugs](http://findbugs.sourceforge.net/) is known to be no longer maintained. [SpotBugs](https://spotbugs.github.io/) can be a good alternative to FindBugs.

Anyway, an Embulk plugin has to maintain FindBugs or SpotBugs by itself.

#### Cleanup dependencies: File-like streams

The Embulk core had provided utility classes for file-like byte streams, but they are deprecated. They have been exported as an external library [`org.embulk:embulk-util-file`](https://dev.embulk.org/embulk-util-file/). Use it instead.

| Old | New |
| --- | --- |
| `org.embulk.spi.util.FileInputInputStream` | `org.embulk.util.file.FileInputInputStream` in `embulk-util-file` |
| `org.embulk.spi.util.FileOutputOutputStream` | `org.embulk.util.file.FileOutputOutputStream` in `embulk-util-file` |
| `org.embulk.spi.util.InputStreamFileInput` | `org.embulk.util.file.InputStreamFileInput` in `embulk-util-file` |
| `org.embulk.spi.util.InputStreamTransactionalFileInput` | `org.embulk.util.file.InputStreamTransactionalFileInput` in `embulk-util-file` |
| `org.embulk.spi.util.ListFileInput` | `org.embulk.util.file.ListFileInput` in `embulk-util-file` |
| `org.embulk.spi.util.OutputStreamFileOutput` | `org.embulk.util.file.OutputStreamFileOutput` in `embulk-util-file` |
| `org.embulk.spi.util.ResumableInputStream` | `org.embulk.util.file.ResumableInputStream` in `embulk-util-file` |

#### Cleanup dependencies: Text processing

The Embulk core had provided utility classes for processing texts, but they are deprecated. They have been exported as an external library [`org.embulk:embulk-util-text`](https://dev.embulk.org/embulk-util-text/). Use it instead.

| Old | New |
| --- | --- |
| `org.embulk.spi.util.LineDecoder` | `org.embulk.util.text.LineDecoder` in `embulk-util-text` |
| `org.embulk.spi.util.LineDelimiter` | `org.embulk.util.text.LineDelimiter` in `embulk-util-text` |
| `org.embulk.spi.util.LineEncoder` | `org.embulk.util.text.LineEncoder` in `embulk-util-text` |
| `org.embulk.spi.util.Newline` | `org.embulk.util.text.Newline` in `embulk-util-text` |

#### Cleanup dependencies: Dynamic column setter

The Embulk core had provided utility classes for dynamic column setting, but they are deprecated. They have been exported as an external library [`org.embulk:embulk-util-dynamic`](https://dev.embulk.org/embulk-util-dynamic/). Use it instead.

| Old | New |
| --- | --- |
| `org.embulk.spi.util.DynamicColumnNotFoundException` | `org.embulk.util.dynamic.DynamicColumnNotFoundException` in `embulk-util-dynamic` |
| `org.embulk.spi.util.DynamicColumnSetter` | `org.embulk.util.dynamic.DynamicColumnSetter` in `embulk-util-dynamic` |
| `org.embulk.spi.util.DynamicPageBuilder` | `org.embulk.util.dynamic.DynamicPageBuilder` in `embulk-util-dynamic` |
| `org.embulk.spi.util.dynamic.AbstractDynamicColumnSetter` | `org.embulk.util.dynamic.AbstractDynamicColumnSetter` in `embulk-util-dynamic` |
| `org.embulk.spi.util.dynamic.BooleanColumnSetter` | `org.embulk.util.dynamic.BooleanColumnSetter` in `embulk-util-dynamic` |
| `org.embulk.spi.util.dynamic.DefaultValueSetter` | `org.embulk.util.dynamic.DefaultValueSetter` in `embulk-util-dynamic` |
| `org.embulk.spi.util.dynamic.DoubleColumnSetter` | `org.embulk.util.dynamic.DoubleColumnSetter` in `embulk-util-dynamic` |
| `org.embulk.spi.util.dynamic.JsonColumnSetter` | `org.embulk.util.dynamic.JsonColumnSetter` in `embulk-util-dynamic` |
| `org.embulk.spi.util.dynamic.LongColumnSetter` | `org.embulk.util.dynamic.LongColumnSetter` in `embulk-util-dynamic` |
| `org.embulk.spi.util.dynamic.NullDefaultValueSetter` | `org.embulk.util.dynamic.NullDefaultValueSetter` in `embulk-util-dynamic` |
| `org.embulk.spi.util.dynamic.SkipColumnSetter` | `org.embulk.util.dynamic.SkipColumnSetter` in `embulk-util-dynamic` |
| `org.embulk.spi.util.dynamic.StringColumnSetter` | `org.embulk.util.dynamic.StringColumnSetter` in `embulk-util-dynamic` |
| `org.embulk.spi.util.dynamic.TimestampColumnSetter` | `org.embulk.util.dynamic.TimestampColumnSetter` in `embulk-util-dynamic` |

#### Cleanup dependencies: Retry helper

The Embulk core had provided utility classes for retries, but they are deprecated. They have been exported as an external library [`org.embulk:embulk-util-dynamic`](https://dev.embulk.org/embulk-util-dynamic/). Use it instead.

| Old | New |
| --- | --- |
| `org.embulk.spi.util.RetryExecutor` | `org.embulk.util.retryhelper.RetryExecutor` in `embulk-util-retryhelper` |
| `org.embulk.spi.util.RetryExecutor.Retryable` | `org.embulk.util.retryhelper.Retryable` in `embulk-util-retryhelper` |
| `org.embulk.spi.util.RetryExecutor.RetryGiveupException` | `org.embulk.util.retryhelper.RetryGiveupException` in `embulk-util-retryhelper` |

#### Prepare for Java 11+

The Java eco-system needs to be prepared for later Java versions, but Java had a big jump from Java 8 to Java 11+.

For example, Java EE classes are removed from the Java runtime environments from Java 11. If an Embulk plugin uses Java EE classes, it stops working with Java 11+. A typical use-case is JAXB `javax.xml.*`. See [JEP 320](https://openjdk.java.net/jeps/320) for the details.

When Embulk detects a plugin using a class that is removed by JEP 320, Embulk logs a warning message like below, even on Java 8.

```
Class javax.xml.bind.JAXB is loaded by the parent ClassLoader, which is removed by JEP 320. The plugin needs to include it on the plugin side. See https://github.com/embulk/embulk/issues/1270 for more details.
```

However, Embulk will not provide any more special help with them by itself. If your plugin uses the Java EE classes, add alternative dependencies in the plugin by yourself.

A typical example for JAXB follows.

```
    // build.gradle

    // Adding dependencies on JAXB explicitly.
    //
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
    implementation "javax.xml.bind:jaxb-api:2.2.11"
    implementation "com.sun.xml.bind:jaxb-core:2.2.11"
    implementation "com.sun.xml.bind:jaxb-impl:2.2.11"
```

```
    // build.gradle

    // Adding a dependency on the Activation Framework explicitly.
    //
    // The Activation Framework is often (not always) required along with JAXB.
    // 1.1.1 is chosen because it was the latest version when JAXB 2.2.11 was released.
    implementation "javax.activation:activation:1.1.1"
```

```
    // build.gradle

    // Adding a dependency on the Annotation API explicitly.
    //
    // The Annotation API is often (not always) required along with JAXB.
    //
    // 1.2 is chosen because only 1.2 should have been available in the age of Java 8.
    // https://mvnrepository.com/artifact/javax.annotation/javax.annotation-api
    implementation "javax.annotation:javax.annotation-api:1.2"
```

#### Use of `embulk-standards`

The Maven artifact `org.embulk:embulk-standards` no longer exists. Some of Embulk plugins had depended on `embulk-standards` for tooling, but they will stop working.

Some of standard plugin features have been exported as external librarires. Consider using those external libraries instead.

* [`embulk-util-csv`](https://dev.embulk.org/embulk-util-csv/) from CSV Parser Plugin
* [`embulk-util-guess`](https://dev.embulk.org/embulk-util-guess/) from Guess Plugins

#### Tests

`org.embulk:embulk-test` had been renamed to `org.embulk:embulk-junit4`.

```
    // build.gradle
    testImplementation "org.embulk:embulk-junit4:0.11.0"
```

Tests still need to depend on `org.embulk:embulk-core`, and in addition, `org.embulk:embulk-deps`.

```
    // build.gradle
    testCompile "org.embulk:embulk-core:0.11.0"
    testCompile "org.embulk:embulk-deps:0.11.0"
```

You may want to use some internal `embulk-core` classes in tests. Typical instances can be retrieved from `org.embulk.spi.ExecInternal`, such as `ExecInternal.getModelManager()` to get `org.embulk.config.ModelManager`.

Note that such internal classes has been incompatible so far, and can be incompatible in the future. For example, `ModelManager` has already dropped `public <T> T readObject(Class<T>, com.fasterxml.jackson.core.JsonParser)` and `public <T> T readObjectWithConfigSerDe(Class<T>, com.fasterxml.jackson.core.JsonParser)`.

We plan to improve the plugin testing framework through v0.11.

### Advanced: Keep compatibility with Embulk v0.9

If you need to keep your Embulk plugin compatible with Embulk v0.9 for a while, for some reasons, take care of the following points.

* Keep the Jackson versions on `2.6.7` while including those Jackson libraries explicitly in the dependencies
* Keep the Guava version on `18.0` while including Guava explicitly in the dependencies
* Keep using `org.msgpack.value.Value` for the JSON column type.

However, note you will have to give up someday. For example, support for `org.msgpack.value.Value` will be removed at some point in v1.0.
