---
layout: docs
toc_group: native-image
link_title: Build Configuration
permalink: /reference-manual/native-image/BuildConfiguration/
---
# Native Image Build Configuration

* [Embedding a Configuration File](#embedding-a-configuration-file)
* [Configuration File Format](#configuration-file-format)
* [Memory Configuration for Native Image Build](#memory-configuration-for-building-a-native-executable)
* [Runtime vs Build Time Initialization](#runtime-vs-build-time-initialization)

Native Image supports a wide range of options to configure the `native-image` tool.

## Embedding a Configuration File

We recommend that you provide the configuration for the `native-image` tool by embedding a _native-image.properties_ file into a project JAR file.
The `native-image` tool will also automatically pick up all configuration options provided in the _META-INF/native-image/_ directory (or any of its subdirectories) and use it to construct `native-image` command line arguments.

To avoid a situation when constituent parts of a project are built with overlapping configurations, we recommended you use subdirectories within _META-INF/native-image_: a JAR file built from multiple maven projects cannot suffer from overlapping `native-image` configurations.
For example:
* _foo.jar_ has its configurations in _META-INF/native-image/foo_groupID/foo_artifactID_
* _bar.jar_ has its configurations in _META-INF/native-image/bar_groupID/bar_artifactID_

The JAR file that contains `foo` and `bar` will then contain both configurations without conflict.
Therefore the recommended layout to store configuration data in JAR files is as follows:
```
META-INF/
└── native-image
    └── groupID
        └── artifactID
            └── native-image.properties
```

Note that the use of `${.}` in a _native-image.properties_ file expands to the resource location that contains that exact configuration file.
This can be useful if the _native-image.properties_ file refers to resources within its subdirectory, for example, `-H:SubstitutionResources=${.}/substitutions.json`.
Always make sure you use the option variants that take resources, that is, use `-H:ResourceConfigurationResources` instead of `-H:ResourceConfigurationFiles`.
Other options that work in this context are:
* `-H:DynamicProxyConfigurationResources`
* `-H:JNIConfigurationResources`
* `-H:ReflectionConfigurationResources`
* `-H:ResourceConfigurationResources`
* `-H:SubstitutionResources`
* `-H:SerializationConfigurationResources`

By having such a composable _native-image.properties_ file, building a native executable does not require any additional arguments on the command line.
It is sufficient to run the following command:
```shell
$JAVA_HOME/bin/native-image -jar target/<name>.jar
```

To identify which configuration is applied when building a native executable, use `native-image --verbose`.
This shows from where `native-image` picks up the configurations to construct the final composite configuration command line options for the native image builder.
```shell
native-image --verbose -jar build/basic-app-0.1-all.jar
Apply jar:file://~/build/basic-app-0.1-all.jar!/META-INF/native-image/io.netty/common/native-image.properties
Apply jar:file://~/build/basic-app-0.1-all.jar!/META-INF/native-image/io.netty/buffer/native-image.properties
Apply jar:file://~/build/basic-app-0.1-all.jar!/META-INF/native-image/io.netty/transport/native-image.properties
Apply jar:file://~/build/basic-app-0.1-all.jar!/META-INF/native-image/io.netty/handler/native-image.properties
Apply jar:file://~/build/basic-app-0.1-all.jar!/META-INF/native-image/io.netty/codec-http/native-image.properties
...
Executing [
    <composite configuration command line options for the image builder>
]
```

Typical examples of configurations that use a configuration from _META-INF/native-image_ can be found in [Native Image configuration examples](https://github.com/graalvm/graalvm-demos/tree/master/native-image-configure-examples).

## Configuration File Format

A _native-image.properties_ file is a Java properties file that specifies native image configurations. The following properties are
supported.

**Args**

Use this property if your project requires custom `native-image` command line options to build correctly.
For example, the `native-image-configure-examples/configure-at-runtime-example` contains `Args = --initialize-at-build-time=com.fasterxml.jackson.annotation.JsonProperty$Access` in its _native-image.properties_ file to ensure the class `com.fasterxml.jackson.annotation.JsonProperty$Access` is initialized at executable build time.

**JavaArgs**

Sometimes it can be necessary to provide custom options to the Java VM that runs the `native-image` tool.
Use the `JavaArgs` property in this case.

**ImageName**

This property specifies a user-defined name for the executable.
If `ImageName` is not used, a name is automatically chosen:
    * `native-image -jar <name.jar>` has a default executable name `<name>`
    * `native-image -cp ... fully.qualified.MainClass` has a default executable name `fully.qualified.mainclass`

Note that using `ImageName` does not prevent the user overriding the name via the command line.
For example, if `foo.bar` contains `ImageName=foo_app`:
    * `native-image -jar foo.bar` generates the executable `foo_app` but
    * `native-image -jar foo.bar application` generates the executable `application`

### Order of Arguments Evaluation
The arguments passed to `native-image` are evaluated from left to right.
This also extends to arguments that are passed indirectly via configuration files in the _META-INF/native-image_ directory.
Consider the example where there is a JAR file that includes _native-image.properties_ containing `Args = -H:Optimize=0`.
You can override the setting that is contained in the JAR file by using the `-H:Optimize=2` option after `-cp <jar-file>`.

### Specifying Default Options for Native Image
If you need to pass the same options every time you build a native executable, for example, to always generate an executable in verbose mode (`--verbose`), you can make use of the `NATIVE_IMAGE_CONFIG_FILE` environment variable.
If the variable is set to the location of a Java properties file, the `native-image` tool will use the default setting defined in there on each invocation.

Write a configuration file and export `NATIVE_IMAGE_CONFIG_FILE=$HOME/.native-image/default.properties` in _~/.bash_profile_.
Every time `native-image` is run it will implicitly use the arguments specified as `NativeImageArgs`, plus the arguments specified on the command line.
Here is an example of a configuration file, saved as _~/.native-image/default.properties_:

```
NativeImageArgs = --configurations-path /home/user/custom-image-configs \
                  -O1
```

### Changing the Default Configuration Directory

Native Image by default stores configuration information in the user's home directory: _$HOME/.native-image/_.
To change this default, set the environment variable `NATIVE_IMAGE_USER_HOME` to a different location. For example:
```shell
export NATIVE_IMAGE_USER_HOME= $HOME/.local/share/native-image
```

## Memory Configuration for Building a Native Executable

The `native-image` tool runs on a Java VM and uses the memory management of the underlying platform.
The usual Java command-line options for garbage collection apply to the `native-image` tool.

During the creation of a native executable, the representation of the whole application is created to determine which classes and methods will be used at runtime.
It is a computationally intensive process that uses the following default values for memory usage:
```
-Xss10M \
-Xms1G \
```
These defaults can be changed by passing `-J + <jvm option for memory>` to the `native-image` tool.

The `-Xmx` value is computed by using 80% of the physical memory size, but no more than 14G per host.
You can provide a larger value for `-Xmx` on the command line, for example, `-J-Xmx26G`.

By default, the `native-image` tool uses up to 32 threads (but not more than the number of processors available). For custom values, use the option `-H:NumberOfThreads=...`.

For other related options available to the `native-image` tool, see the output from the command `native-image --expert-options-all`.

## Runtime vs Build-Time Initialization

When you build a native executable, you can decide which elements of your application are run at build time, and which elements are run at executable run time.

By default, all class-initialization code (static initializers and static field initialization) of the application is run at executable run time.
Sometimes it is beneficial to run class initialization code when the executable is built. For example, for faster startup if some static fields are initialized to values that are runtime-independent.
This is controlled with the following `native-image` options:

* `--initialize-at-build-time=<comma-separated list of packages and classes>`
* `--initialize-at-run-time=<comma-separated list of packages and classes>`

In addition to that, arbitrary computations are allowed at build time that can be put into `ImageSingletons` that are accessible at run time. For more information, see [Native Image configuration examples](https://github.com/graalvm/graalvm-demos/tree/master/native-image-configure-examples).

## Specifying Types Required to Be Defined at Build Time

A well-structured library or application should handle linking of Java types (ensuring all reachable Java types are fully defined at build time) when building a native executable by itself.
The default behavior is to throw linking errors, if they occur, at run time. 
However, you can prevent unwanted linking errors by specifing which classes are required to be fully linked at build time.
For that, use the `--link-at-build-time` option. 
If the option is used in the right context (see below), you can specify required classes to link at build time without explicitly listing classes and packages.
It is designed in a way that libraries can only configure their own classes, to avoid any side effects on other libraries.
You can pass the option to the `native-image` tool on the command line, embed it in a `native-image.properties` file on the module-path or the classpath.

Depending on how and where the option is used it behaves differently:

* If you use `--link-at-build-time` without arguments, all classes in the scope are required to be fully defined. If used without arguments on command line, all classes will be treated as "link-at-build-time" classes. If used without arguments embedded in a `native-image.properties` file on the module-path, all classes of the module will be treated as "link-at-build-time" classes. If you use `--link-at-build-time` embedded in a `native-image.properties` file on the classpath, the following error will be thrown:
    ```shell
    Error: Using '--link-at-build-time' without args only allowed on module-path. 'META-INF/native-image/org.mylibrary/native-image.properties' in 'file:///home/test/myapp/MyLibrary.jar' not part of module-path.
    ```
* If you use the  `--link-at-build-time` option with arguments, for example, `--link-at-build-time=foo.bar.Foobar,demo.myLibrary.Name,...`, the arguments should be fully qualified class names or package names. When used on the module-path or classpath (embedded in `native-image.properties` files), only classes and packages defined in the same JAR file can be specified. Packages for libraries used on the classpath need to be listed explicitly. To make this process easy, use the `@<prop-values-file>` syntax to generate a package list (or a class list) in a separate file automatically.

Another handy option is `--link-at-build-time-paths` which allows to specify which classes are required to be fully defined at build time by other means.
This option variant requires arguments that are of the same type as the arguments passed via `-p` (`--module-path`) or `-cp` (`--class-path`):

```shell
--link-at-build-time-paths <class search path of directories and zip/jar files>
```

The given entries are searched and all classes inside are registered as `--link-at-build-time` classes.
This option is only allowed to be used on command line.

# Related Documentation
* [Class Initialization in Native Image](ClassInitialization.md)
* [Assisted Configuration of Native Image Builds](Agent.md#assisted-configuration-of-native-image-builds)
* [Building Native Image with Java Reflection Example](Agent.md#building-native-image-with-java-reflection-example)
* [Agent Advanced Usage](Agent.md#agent-advanced-usage)