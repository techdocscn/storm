---
layout: documentation
---
开发 topologies ，你需要把 Storm jar 包含进你项目的 classpath。你应该或者把解压缩的jar包含到你项目的 classpath 中，或者用 Maven 来把 Strom 作为一个开发依赖项。Storm 承载在 Clojars (一个 Maven 代码库). 向你项目的 pom.xml 中添加以下代码来把 Storm 包含进你的项目：

```xml
<repository>
  <id>clojars.org</id>
  <url>http://clojars.org/repo</url>
</repository>
```

```xml
<dependency>
  <groupId>storm</groupId>
  <artifactId>storm</artifactId>
  <version>0.7.2</version>
  <scope>test</scope>
</dependency>
```

[这里](https://github.com/nathanmarz/storm-starter/blob/master/m2-pom.xml)有一个 Storm 项目的 pom.xml 作为例子。

如果你不喜欢 Maven，请看 [leiningen](https://github.com/technomancy/leiningen). Leiningen 是一个 Clojure 的构建工具, 但是它也可以用在 Java 程序上。Leiningen 使得用 Maven 来构建项目和管理依赖项超级简单。这里有一个单纯的 Java Storm 项目的 project.clj 的例子：

```clojure
(defproject storm-starter "0.0.1-SNAPSHOT"
  :java-source-path "src/jvm"
  :javac-options {:debug "true" :fork "true"}
  :jvm-opts ["-Djava.library.path=/usr/local/lib:/opt/local/lib:/usr/lib"]
  :dependencies []
  :dev-dependencies [
                     [storm "0.7.2"]
                     ])
```

你可以用 `lein deps` 来抓取依赖项， `lein compile` 来构建项目, `lein uberjar` 来构建一个可以提交到集群上的 jar.

### 把 Storm 当做库使用

如果你想把 Storm 作为一个库来使用（例如使用分布式 RPC 客户端），让 Storm jar 依赖项随着你的程序分布，你可以使用另外一个 Maven 依赖项："storm/storm-lib"。 这个依赖项和常规的 "storm/storm" 唯一的区别是 storm-lib 不包括任何日志的设置。

### 开发 Storm

你需要：

	bash ./bin/install_zmq.sh   # 安装 jzmq 依赖项
	lein sub install

构建 javadocs：

	bash ./bin/javadoc.sh

### 构建 Storm 的一个正式版本

使用文件 `bin/build_release.sh` 来构建一个和下载得来一样的压缩文件 （和为了运行后台程序所需要的二进制文件一样）。