---
title: Protobuf Gradle Plugin 的用例 
date: 2022-03-08
author: tison
categories: tison
tags:
    - Gradle
    - Java
    - Protobuf
---

近日尝试利用 [Apache Ratis](https://github.com/apache/ratis) 这个项目包装一个 Raft 协议驱动的状态机，遇到了需要用 Protobuf 传输数据的场景。由于 Gradle 构建工具的门槛和 Java 语言项目的某些惯例碰到了使用上的问题，这里记录一下我在这个玩具项目当中的用例。

<!-- more -->

首先介绍一下整个项目的主要目录结构，这里只包含最小复现需要的集合

```
project/
project/proto/
project/proto/RMap.proto
project/build.gradle
project/settings.gradle
```

其中 `settings.gradle` 只有一行默认生成的 `rootProject.name = 'dryad'` 信息，`RMap.proto` 是一个普通的不包含 gRPC 定义的 proto 文件。`RMap.proto` 文件内容如下

```protobuf
syntax = "proto3";
option java_package = "org.tisonkun.dryad.proto.rmap";
option java_outer_classname = "RMapProtos";
option java_generate_equals_and_hash = true;

package dryad.rmap;

message GetRequest {
    bytes key = 1;
}

message GetResponse {
    bool found = 1;
    bytes key = 2;
    bytes value = 3;
}

message PutRequest {
    bytes key = 1;
    bytes value = 2;
}

message PutResponse {
}
```

主要使用 [Protobuf Gradle Plugin](https://github.com/google/protobuf-gradle-plugin) 的逻辑都在 `build.gradle` 文件里，文件内容如下

```groovy
plugins {
    id 'java'
    id 'com.google.protobuf' version '0.8.18'
    id 'com.github.johnrengelman.shadow' version '7.1.2'
}

repositories {
    mavenCentral()
}

sourceCompatibility = 17
targetCompatibility = 17

dependencies {
    implementation 'com.google.protobuf:protobuf-java:3.19.2'
    implementation 'org.apache.ratis:ratis-thirdparty-misc:0.7.0'

    protobuf files("proto/")
}

protobuf {
    protoc {
        artifact = 'com.google.protobuf:protoc:3.12.0'
    }
}

shadowJar {
    configurations = []
    relocate 'com.google.protobuf', 'org.apache.ratis.thirdparty.com.google.protobuf'
}
```

编译和构建工具采用的版本信息如下

```
------------------------------------------------------------
Gradle 7.4
------------------------------------------------------------

Build time:   2022-02-08 09:58:38 UTC
Revision:     f0d9291c04b90b59445041eaa75b2ee744162586

Kotlin:       1.5.31
Groovy:       3.0.9
Ant:          Apache Ant(TM) version 1.10.11 compiled on July 10 2021
JVM:          17.0.2 (Eclipse Adoptium 17.0.2+8)
OS:           Mac OS X 10.15.7 x86_64
```

这个用例当中有两个注意点。

**第一个注意点是 protobuf 的配置方式。**

可以看到在 `dependencies` 配置块中声明了 proto 文件的路径。我不记得是不是有默认的查询路径比如 `<project>/src/main/proto` 这样的，但是建议还是明确写出来为好，毕竟业界也没有什么公认的标准，每个插件工具的假设不一定采用同一套约定。

另外就是 `protobuf` 配置块中声明了 `protoc` 工具的版本。Protobuf Gradle Plugin 的[官方文档](https://github.com/google/protobuf-gradle-plugin#customizing-protobuf-compilation)当中还介绍了如何整合 gRPC 等插件等控制 `protoc` 编译过程的方式。玩具项目当中不需要，因此略过。

最后是 `protoc` 和 `protobuf-java` 的版本不一样，如果还要引入 gRPC 的插件和 JAR 包依赖，还会有其他不一样的版本。这是因为 Protobuf 生态并不是整体同步发布的，而是各个组件很大程度上自主开发和发布的缘故。具体的兼容矩阵我没有研究过，但是一般来说锁定了某个版本就不太会轻易升级了。比如 Apache Hadoop 的 Protobuf 版本一直停留在 2.5.0 版本上。印象中 3.0 版本以后的兼容性还是比较好的，3.10+ 版本之间的升级还算顺滑。

**第二个注意点是 Gradle Shadow Plugin 插件的使用。**

[Gradle Shadow Plugin](https://imperceptiblethoughts.com/shadow/) 很大程度上是 [Maven Shade Plugin](https://maven.apache.org/plugins/maven-shade-plugin/) 的同位替代。也就是说，服务于需要把依赖项一起打成一个大 JAR 包的场景。

通常来说，Maven 或 Gradle 项目打包的时候，依赖项都不会进入到最终产物当中。因为打包就只是对你写的这些代码编译出来的 class 文件打包，而不是像 C / Rust 这种产生二进制可执行文件的思路。Java 语言程序运行起来，是需要程序员把所有的依赖项都写进 classpath 里，再指定要运行的类，执行其 Main 方法启动的。这种情况下打包不需要把依赖项都搭进去。

这种方案对于企业自己管理所有依赖，大部分软件是自包含少依赖的大型软件的场景是比较合理的。但是随着互联网的兴起和合作开发效率提升，一个项目依赖大量其他项目的情形越来越多，这些其他项目也有自己的开发周期，往往会产生多个版本的 JAR 包发布产物。这种情况下再要求程序员自己去管理依赖项，管理 classpath 的内容，在生产上就是既繁琐有不可靠的了。

因此，Gradle Shadow Plugin 和 Maven Shade Plugin 解决的问题就是把所有依赖在打包的时候也打进构建产物当中，产生一个 `project-all.jar` 文件。用户可以直接把这一个 JAR 包加入 classpath 就能保证所有的依赖都已经就绪。甚至在 MANIFEST 文件中写好默认的 MainClass 信息，就能通过 `java -jar` 命令将大 JAR 包以一种形如二进制可执行文件的方式运行起来。

不过，我们这里用上的不是打一个大 JAR 包的功能，而是在这个大需求下解决 package relocation 问题的功能。

Java 语言程序依靠全限定名来识别一个类，每个 ClassLoader 都对每个全限定名都只会加载一个类实例。如果 classpath 当中存在两个相同全限定名的类，那么根据 ClassLoader 的实现策略，可能会加载其中任意一个，或者报错。

对于服务端应用例如 Apache Flink 和 Apache Ratis 而言，它们自己需要依赖 protobuf 或 akka 等三方库，同时它们自己的用户也有可能依赖这些三方库，那么用户内部逻辑使用的三方库版本，跟用户逻辑需要跟服务端打交道时使用的三方库版本，就有可能在 classpath 当中同时存在。如果这两个版本不兼容，就会出现运行时错误。

由于服务端应用往往受众更广，通常来说解决方案是用户应用程序采用跟服务端相同的依赖版本。但是如果用户不是直接依赖跟服务端可能冲突的三方库，而是间接依赖，那么这个版本对齐的工作往往就很难做了。

另一种形式是形如 akka 生态当中的 play 框架，直接暴露操作 akka 底层数据结构的接口，用户自己不依赖 akka 而是通过 play 提供的接口使用 akka 的能力。但是这种形式只对 akka 和 play 这样由同一个团队开发的软件是比较合适的，放在更加复杂的开源软件生态当中就很难配合了。

因此从服务端的角度出发，为了避免用户遇到这一难题，一个彻底的解决方法就是 package relocation 更改自己依赖的三方库的全限定名。

比如上面 `build.gradle` 里配置项显示的

```groovy
shadowJar {
    configurations = []
    relocate 'com.google.protobuf', 'org.apache.ratis.thirdparty.com.google.protobuf'
}
```

这意味着把所有 `com.google.protobuf` 的文本都替换成 `org.apache.ratis.thirdparty.com.google.protobuf` 的字样，也包括字符串当中的情况，以应对动态加载的用例。

这样，服务端最终打出来的 JAR 包里，使用的类全限定名就不是 `com.google.protobuf.Message` 而是 `org.apache.ratis.thirdparty.com.google.protobuf.Message` 了，这也就跟用户依赖的 `com.google.protobuf.Message` 不同，从而不会起冲突。

当然，这种 package relocation 不仅仅在服务端的使用上会改掉全限定名，也需要类的实现本身也是以新的全限定名来提供的。因此 Apache Ratis 项目提供了 [`ratis-thirdparty-misc`](https://github.com/apache/ratis-thirdparty) 库，Apache Flink 项目提供了 [`flink-shaded`](https://github.com/apache/flink-shaded) 库。其中的内容就是把服务端依赖的软件以 relocate 之后的名称重新发布。

对于这个玩具项目来说，它需要的是保持跟 Apache Ratis 服务端一样的 protobuf 依赖的全限定名，保证能够嵌入到 Apache Ratis 的服务端实现当中。对于其中的 proto 定义部分，它并不需要真的把依赖项也打进自己的 JAR 包里，这个打大 JAR 包的工作会交给最终的 dist package 完成。所以我们还需要把 Gradle Shadow Plugin 默认打入所有运行时依赖的行为变掉。这就是 `configurations = []` 一行起的作用，把打入最终 JAR 包的依赖项置空，这样就只会包含 proto 文件编译出来的 class 文件了。这样的用例，其实与 Maven Shade Plugin 的惯用法有较大的差别，更像是 [Maven Replacer Plugin](https://code.google.com/archive/p/maven-replacer-plugin/) 的用法。

最后作为小 tip 值得一提的是，上面提到 package relocation “也包括字符串当中的情况，以应对动态加载的用例”。这其实导致了 akka 项目很难利用常规的 package relocation 插件来完成这个工作。惯例上，Java 语言项目的全限定名以域名开头，形如 `com.google.protobuf` 或 `org.apache.ratis` 等等。一般而言这种形式的字符串只会出现在类的全限定名当中。然而，akka 作为一个 Scala 项目采用了 `akka.actor` 形式的全限定名前缀。不幸的是，这种前缀模式跟 akka 的配置项是重叠的。这就导致 package relocation 会同时改变配置项的名称。这其实不是我们想要的，因为这样用户也要跟着改配置项的名称才能跟 relocate 之后的 akka 库交互，这通常来说是非常难做到并且与大部分开发者的直觉和生态项目的假设是冲突的。
