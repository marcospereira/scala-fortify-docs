# Fortify SCA for Scala

This page documents Scala support in Fortify SCA.  For an
overview of Fortify SCA in general,
[visit this Micro Focus page](https://software.microfocus.com/en-us/products/static-code-analysis-sast/overview).

## Requirements

This release supports translating and scanning Scala source code on
the following platforms:

- Linux
- MacOS
- Windows

The following Java Virtual Machine versions are supported:

- Java 8

The following Scala versions are supported:

- Scala 2.11.12, 2.11.11, 2.11.8
- Scala 2.12.4, 2.12.3

The latest patch releases are recommended (2.11.12 and 2.12.4, as of
January 2018).

To translate Scala code for Fortify to scan, you must be a current
Lightbend subscriber.

To actually scan translated code for vulnerabilities, you must be a
licensed Fortify SCA user. Fortify SCA version 17.20 (or newer)
is required.

## Supported language features

All of Scala is supported by the translator.

## Fortify on Demand

[Micro Focus Fortify on Demand](https://software.microfocus.com/en-us/products/application-security-testing/overview),
version 17.4, also supports Scala. Please see the Fortify on Demand
documentation for more information on preparing Scala applications. To
access the documentation, log in to Fortify on Demand, select
Documentation from the account menu, and run a search for "Preparing
Scala Application Files".

## License installation

The license file for using the Scala translator is a standard Lightbend
Enterprise Suite license.  Lightbend subscribers can obtain their
license by visiting
[https://www.lightbend.com/account/license](https://www.lightbend.com/account/license).

The license contents should look like:

    ===== LIGHTBEND ENTERPRISE SUITE LICENSE =====

    DO NOT TAMPER WITH THIS FILE - ...

All current Lightbend subscribers should have the `fortify` grant in
the `grants:` line of the license.  You may need to retrieve a new
license if the old one was created before mid-July 2017. (Note that
Lightbend Enterprise Suite trial licenses do not have this grant.)

Copy and paste the text from that web page into a file in the correct
platform-dependent location.  On MacOS and Linux platforms the default
location is `~/.lightbend/license` and on the Windows platform it is
`%homePath%\.lightbend\license`.

You can ask the compiler plugin to look in a different location by
specifying `-P:fortify:license=...`, substituting any full path you
like for `...`.

## Getting and using the translator (via sbt)

Prerequisite: install the sbt build tool
([link](http://www.scala-sbt.org/download.html)).

Translating Scala source code for Fortify is done by a Scala
compiler plugin.

You can have sbt resolve the compile plugin if you either

* set up authentication for Lightbend's commercial-releases repository
* or, mirror the compiler plugin to an internal repository used at your organization.
  This is usually done in larger organizations.

To authenticate to the commercial-releases repo, create a
`~/.lightbend/commercial.credentials` file (on MacOS or Linux
platforms), or a `%homePath%\.lightbend\commercial.credentials` file
(on Windows) that looks like:

    realm=Bintray
    host=dl.bintray.com
    user=********-****-****-****-************@lightbend
    password=****************************************

but substitute `user` and `password` values that you retrieve
from https://portal.lightbend.com/ReactivePlatform/Credentials .

And in your project's build definition, add:

```scala
credentials += Credentials(
  Path.userHome / ".lightbend" / "commercial.credentials")
resolvers += Resolver.url(
  "lightbend-commercial-releases",
  new URL("http://repo.lightbend.com/commercial-releases/"))(
  Resolver.ivyStylePatterns)
```

If the compiler plugin has already been mirrored to an internal repository at
your organization, you can skip the preceding steps.

Then, add the following to your top-level `build.sbt`:

```scala
addCompilerPlugin(
  "com.lightbend" %% "scala-fortify" % "1.0.2"
    classifier "assembly"
    cross CrossVersion.patch)

scalacOptions += s"-P:fortify:build=myproject"
```

Substituting any build id you like for `myproject`.  Specifying the
build id causes translated files to be written to the usual location
used by SCA (`$HOME/.fortify/sca17.2/build/myproject`).  (If you
prefer to specify the output directory directly, use
`-P:fortify:out=...` instead, filling in a filesystem path for `...`.)

You may also want to add:

```scala
scalacOptions += "-Ystop-before:jvm"
```

to make compilation stop before any actual JVM classfiles are
generated.  (You can omit this if you want to let compilation
finish.)

These changes to your build will cause the compiler plugin to run and
translated files (with an `.nst` extension) to be generated whenever
your code is compiled (e.g., with the `compile` task).

For the scan step, supplying the build id again will allow the
scanner to find the translated files.  For example:

    sourceanalyzer -b myproject -f scan.fpr -scan

### Multi-project builds

If your build has subprojects, then you'll need to adapt the
above `libraryDependencies += ...` and `scalacOptions += ...`
settings accordingly.  (By default, the settings will apply
only to your root project.)

The best way to do this depends on how your build is set up
and whether you want to enable Fortify SCA on every subproject, or
only some of them.

If you want to enable Fortify SCA in every subproject and you don't
mind all of the translated files getting mixed together in the
top-level `target` directory, then simply add `in ThisBuild` to the
settings.
[The sbt documentation on build-level settings](http://www.scala-sbt.org/1.x/docs/Scopes.html#Build-level+settings)
for details on how this works.

To enable Fortify on a particular subproject, and to keep that
subproject's translated files in their own `target` directory,
add the `libraryDependencies` and `scalacOptions` to those
subprojects only.

If you want to do this across multiple subprojects without
copy-and-paste, you can store the `libraryDependencies` and
`scalacOptions` settings in a variable, and then add that variable in
each subproject.  This technique is shown in
[the sbt documentation on common settings](http://www.scala-sbt.org/1.x/docs/Multi-Project.html#Common+settings).
The resulting build definition will look something like this:

```scala
lazy val fortifySettings = Seq(
  libraryDependencies += ...,
  scalacOptions += ...)
lazy val project1 = project ... (
  .settings(fortifySettings,
    // other settings
  )
lazy val project2 = project ... (
  .settings(fortifySettings,
    // other settings
  )
```

## Getting and using the translator (with Gradle)

Here is a sample `build.gradle` file showing how to use the translator
in a [Gradle](https://gradle.org/) build.

It is assumed that your Lightbend credentials, retrieved from
https://portal.lightbend.com/ReactivePlatform/Credentials,
are in `~/.gradle/gradle.properties` as:

```properties
lightbendUser=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx@lightbend
lightbendPassword=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Sample `build.gradle`:

```groovy
apply plugin: 'scala'

repositories {
    mavenCentral()
    ivy {
        credentials {
            username lightbendUser
            password lightbendPassword
        }
        url = 'https://repo.lightbend.com/commercial-releases'
        layout 'pattern', {
          artifact '[organisation]/[module]/[revision]/[ext]s/[artifact](-[classifier]).[ext]' // by default gradle is missing `(-[classifier])`
          ivy '[organisation]/[module]/[revision]/[artifact]/[artifact](.[ext])'
        }
    }
}

ext {
    scalaBinaryVersion = '2.12'
    scalaVersion = '2.12.4'
    fortifyPluginVersion = '1.0.2'
}

configurations {
    lightbendFortifyPlugin
}

dependencies {
    compile("org.scala-lang:scala-library:$scalaVersion") // specify Scala version
    lightbendFortifyPlugin group: 'com.lightbend', name: "scala-fortify_$scalaVersion", version: fortifyPluginVersion, classifier: "assembly"
}

tasks.withType(ScalaCompile) {
  scalaCompileOptions.additionalParameters = [
    "-Xplugin:" + configurations.lightbendFortifyPlugin.asPath,
    "-Xplugin-require:fortify",
    "-P:fortify:build=myproject"
  ]
}
```

### When using with Play Framework

Gradle also has [support for Play Framework](https://docs.gradle.org/current/userguide/play_plugin.html) and you mix both Fortify and Play configuration in your project, for example:


```groovy
plugins {
    id 'play'
    id 'scala'
}

ext {
    playVersion = '2.6.11'
    scalaVersion = '2.12.4'
    scalaBinaryVersion = '2.12'
    fortifyPluginVersion = '1.0.2'
}

model {
    components {
        play {
            platform play: playVersion, scala: scalaBinaryVersion, java: '1.8'
            injectedRoutesGenerator = true

            sources {
                twirlTemplates {
                    defaultImports = TwirlImports.SCALA
                }
            }
        }
    }
}

configurations {
    lightbendFortifyPlugin
}

dependencies {
    // Example of play dependencies
    play "com.typesafe.play:play-guice_$scalaBinaryVersion:$playVersion"
    play "com.typesafe.play:play-ahc-ws_$scalaBinaryVersion:$playVersion"
    play "com.typesafe.play:play-logback_$scalaBinaryVersion:$playVersion"
    play "com.typesafe.play:filters-helpers_$scalaBinaryVersion:$playVersion"

    lightbendFortifyPlugin group: 'com.lightbend', name: "scala-fortify_$scalaVersion", version: fortifyPluginVersion, classifier: "assembly"
}

repositories {
    jcenter()
    maven {
        name "lightbend-maven-releases"
        url "https://repo.lightbend.com/lightbend/maven-release"
    }
    ivy {
        name "lightbend-ivy-release"
        url "https://repo.lightbend.com/lightbend/ivy-releases"
        layout "ivy"
    }

    // Fortify repository
    ivy {
        credentials {
            username lightbendUser
            password lightbendPassword
        }
        url = 'https://repo.lightbend.com/commercial-releases'
        layout 'pattern', {
            artifact '[organisation]/[module]/[revision]/[ext]s/[artifact](-[classifier]).[ext]' // by default gradle is missing `(-[classifier])`
            ivy '[organisation]/[module]/[revision]/[artifact]/[artifact](.[ext])'
        }
    }
}

tasks.withType(ScalaCompile) {
    scalaCompileOptions.additionalParameters = [
            "-Xplugin:" + configurations.lightbendFortifyPlugin.asPath,
            "-Xplugin-require:fortify",
            "-P:fortify:build=play-webgoat"
    ]
}
```

You can also see a [sample application here](https://github.com/lightbend/play-webgoat).

## Getting and using the translator (manually)

Prerequisite: install the Scala compiler
([link](https://www.scala-lang.org/download/)).

In the following instructions, substitute the actual full Scala 2.11.x
or 2.12.x version you are using; we use 2.12.4 as the example version.

Using the username and password that you retrieve
from https://portal.lightbend.com/ReactivePlatform/Credentials ,
you can download the compiler plugin JAR from:

    https://lightbend.bintray.com/commercial-releases/com.lightbend/scala-fortify_2.12.4/1.0.2/jars/scala-fortify_2.12.4-assembly.jar

Then, supposing you have `scala-fortify_2.12.4-assembly.jar` in your
current working directory, you can do e.g.:

    scalac -Xplugin:scala-fortify_2.12.4-assembly.jar \
      -Xplugin-require:fortify \
      -Ystop-before:jvm \
      -P:fortify:build=myproject \
      *.scala

Substituting your own build id for `myproject`.  (Or, use the
`-P:fortify:out=...` to specify the output directory directly.)

When build id support is enabled, it is currently assumed that
you are running a version of Fortify SCA in the 17.2 series,
so that translated files are written to `~/.fortify/sca17.2`.
If you are using some other version of Fortify SCA, for
example some version in the 18.1 series, add e.g.:

    -P:fortify:scaversion=18.1

Including `-Ystop-before:jvm`, as shown above, makes compilation stop
before any actual JVM classfiles are generated.  You can also
omit it if you want to let compilation finish.

For the scan step, supplying the build id again will allow the
scanner to find the translated files.  For example:

    sourceanalyzer -b myproject -f scan.fpr -scan

## Using the translator (other build tools)

If you are using a build tool other than sbt, then:

* Obtain the compiler plugin JAR using the instructions in the previous section
* Use whatever your build tool's mechanism is for customizing the
  options passed to scalac, and pass the additional options shown in
  the previous section.

## Known issues

To report an issue, please open a support ticket with either Lightbend
or Micro Focus.

* Java 6, 7, and 9 may work but are not currently officially
  supported. (issue 222, issue 223)
* Issues found in Twirl templates are reported as occurring in the
  generated `.scala` file, not in the original template. (issue 159)
* Mixed Scala and Java codebases require separately translating
  the Scala and Java sources. Depending on the dependency structure
  of the code, not all issues involving both languages may be found.
  (issue 180)
* Some kinds of user code cause "There is more than one class named
  'scala.PartialFunction'" warnings to be printed.  We expect the
  problem to be fixed in the forthcoming SCA 18.10 release.
  (issue 239)
* Some pattern matches may produce spurious
  "Dead Code : Expression is Always false" reports. (issue 221)
* Use of `do ... while` may produce spurious
  "null dereference" reports. (issue 208)
* When method calls are nested, all issues are reported as occurring
  on the line of code containing the outermost call. (issue 204)
* Not all characters are supported in build ids; `-` and `_` work
  but other special characters may not. (issue 174)
* On Scala 2.11, the `-Ydelambdafy` flag is not supported and results
  in "unknown code Function" errors. Disable this flag when using the
  compiler plugin. (issue 215)

## Release notes

### 1.0.2 (January 11, 2018)

* the location of the Lightbend license file is now configurable with
  `-P:fortify:license=...` (issue 263)

### 1.0.1 (December 20, 2017)

* enabling the plugin no longer interferes with Scaladoc generation
  (issue 260)

### 1.0.0 (December 6, 2017)

* first general release
