---
layout: documentation
---
这一页概述了如何建立一个 Storm 项目来进行开发。步骤如下:

1. 将 Storm jars 加入到 classpath
2. 如果使用 multilang, 将 multilang 目录加入到 classpath

按照如下指示在 Eclipse 中建立 [storm-starter](http://github.com/nathanmarz/storm-starter) 项目。

### 将 Storm jars 加入到 classpath

开发 Storm topologies 需要首先保证 storm jars 已经加入到 classpath中。我们强烈推荐使用 [Maven](Maven.html) 来开发。 [这里](https://github.com/nathanmarz/storm-starter/blob/master/m2-pom.xml) 是一个如何给 Storm 项目建立你的 pom.xml 的事例。如果你不想使用 Maven, 那么你可以将 Storm 版本中的 jars 加入到你自己的 classpath。

[storm-starter](http://github.com/nathanmarz/storm-starter) 使用 [Leiningen](http://github.com/technomancy/leiningen) 来进行编译和相关性解析。你可以下载 [这个脚
本](https://raw.github.com/technomancy/leiningen/stable/bin/lein) 来安装 leiningen, 把它放到自定义的路径下, 然后将它变成可执行文件。 在项目根目录下运行 `lein deps` 即可获取 Storm 的 dependencies。

若想在 Eclipse 下建立 classpath, 需要建立一个新的 Java 项目, 将 `src/jvm/` 设为源路径, 并保证所有 `lib/` 和 `lib/dev/` 下的 jars 都在项目 `Referenced Libraries` 部分里。

### 如果使用 multilang, 将 multilang dir 加入到 classpath

如果你不想使用 Java 语言来实现 spouts 或者 bolts, 那么这些实现应该放在项目 `multilang/resources/` 目录下。 为了让 Storm 在本地模式下找到这些文件, `resources/` 目录需要放到 classpath 里。你可以在 Eclipse 下将 `multilang/` 加为源文件夹来实现. 并需要将 multilang/resources 加为源目录.

请参考 [使用 non-JVM 语言和 Storm](Using-non-JVM-languages-with-Storm.html)来获取更多如何用其他语言来实现 topologies。

现在你可以 `Run` 文件 `WordCountTopology.java` 来测试 Eclipse 下是否一切运行正常。 如果运行正常，将会有信息在 console 下出现并持续大约10秒钟。
