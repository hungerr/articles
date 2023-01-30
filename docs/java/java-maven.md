# Java与Maven

## JDK与JRE

JDK：Java Development Kit 是Java的标准开发工具包

JRE：Java runtime environment 是运行基于Java语言编写的程序所不可缺少的运行环境，用于解释执行Java的字节码文件

普通用户只需要安装 JRE来运行 Java 程序。而程序开发者必须安装JDK来编译、调试程序

JVM：Java Virtual Machine 是Java的虚拟机，是JRE的一部分。它是整个java实现跨平台的最核心的部分，负责解释执行字节码文件，是可运行java字节码文件的虚拟计算机。

## Java安装

### OpenJDK发行版本

- [Amazon Corretto](https://aws.amazon.com/cn/corretto/?filtered-posts.sort-by=item.additionalFields.createdDate&filtered-posts.sort-order=desc "Amazon Corretto")
- [Microsoft](https://learn.microsoft.com/zh-cn/java/openjdk/download "Microsoft")
- [OpenJDK](http://openjdk.java.net/ "OpenJDK")
- [Ali Dragonwellto](https://dragonwell-jdk.io/#/index "Ali Dragonwellto")
- [腾讯 Kona](https://cloud.tencent.com/product/tkjdk "腾讯 Kona")

安装后安装路径`jdk\bin`加入环境变量`PATH`

查看版本`java -version`

## Maven安装

[下载](https://maven.apache.org/download.cgi "下载")

安装后安装路径`\apache-maven\bin`加入环境变量`PATH`

查看版本`mvn -v`

## 仓库

在`Maven`的术语中，仓库是一个位置（place）。

Maven 仓库是项目中依赖的第三方库，这个库所在的位置叫做仓库。

在`Maven`中，任何一个依赖、插件或者项目构建的输出，都可以称之为构件。

Maven 仓库能帮助我们管理构件（主要是JAR），它就是放置所有JAR文件（WAR，ZIP，POM等等）的地方。

Maven 仓库有三种类型：

- 本地仓库（local）
- 中央仓库（central）
- 远程仓库（remote）

### 本地仓库
Maven 的本地仓库，在安装 Maven 后并不会创建，它是在第一次执行 maven 命令的时候才被创建。

运行 Maven 的时候，Maven 所需要的任何构件都是直接从本地仓库获取的。如果本地仓库没有，它会首先尝试从远程仓库下载构件至本地仓库，然后再使用本地仓库的构件。

每个用户在自己的用户目录下都有一个路径名为 `.m2/repository/` 的仓库目录。

要修改默认位置，`settings.xml` 文件中定义另一个路径:
```XML
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 
   http://maven.apache.org/xsd/settings-1.0.0.xsd">
      <localRepository>C:/MyLocalRepository</localRepository>
</settings>
```

### 中央仓库
Maven 中央仓库是由 Maven 社区提供的仓库，其中包含了大量常用的库。

### 远程仓库

由于最原始的本地仓库是空的，Maven必须知道至少一个可用的远程仓库，才能在执行Maven命令的时候下载到需要的构件。中央仓库就是这样一个默认的远程仓库，Maven的安装文件自带了中央仓库的配置。

### 远程仓库的认证
大部分的远程仓库不需要认证，但是如果是自己内部使用，为了安全起见，还是要配置认证信息的。

配置认证信息和配置远程仓库不同，远程仓库可以直接在`pom.xml`中配置，但是认证信息必须配置在`settings.xml`文件中。这是因为pom往往是被提交到代码仓库中供所有成员访问的，而`settings.xml`一般只存在于本机。因此，在`settings.xml`中配置认证信息更为安全。
```XML
<settings>
      ...
      <!--配置远程仓库认证信息-->
      <servers>
          <server>
              <id>releases</id>
              <username>admin</username>
              <password>admin123</password>
          </server>
      </servers>
     ...
</settings>
```
这里除了配置账号密码之外，值关键的就是id了，这个id要跟你在`pom.xml`里面配置的远程仓库repository的id一致，正是这个id将认证信息与仓库配置联系在了一起。

### 远程仓库的配置
在平时的开发中，我们往往不会使用默认的中央仓库，默认的中央仓库访问的速度比较慢，访问的人或许很多，有时候也无法满足我们项目的需求，可能项目需要的某些构件中央仓库中是没有的，而在其他远程仓库中有，如JBoss Maven仓库。这时，可以在`pom.xml`中配置该仓库，代码如下：
```XML
<!-- 配置远程仓库 -->
    <repositories>
        <repository>
            <id>jboss</id>
            <name>JBoss Repository</name>
            <url>http://repository.jboss.com/maven2/</url>
            <releases>
                <enabled>true</enabled>
                <updatePolicy>daily</updatePolicy>
            </releases>
            <snapshots>
                <enabled>false</enabled>
                <checksumPolicy>warn</checksumPolicy>
            </snapshots>
            <layout>default</layout>
        </repository>
        <repository>
            <id>spring</id>
            <url>https://maven.aliyun.com/repository/spring</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
    </repositories>
```

- `repository`: 在repositories元素下，可以使用repository子元素声明一个或者多个远程仓库。
- `id`：仓库声明的唯一id，尤其需要注意的是，Maven自带的中央仓库使用的id为`central`，如果其他仓库声明也使用该id，就会覆盖中央仓库的配置。
- `name`：仓库的名称，让我们直观方便的知道仓库是哪个，暂时没发现其他太大的含义。
- `url`：指向了仓库的地址
- `releases`和`snapshots`：用来控制Maven对于发布版构件和快照版构件的下载权限。需要注意的是enabled子元素，该例中releases的enabled值为true，表示开启JBoss仓库的发布版本下载支持，而snapshots的enabled值为false，表示关闭JBoss仓库的快照版本的下载支持。根据该配置，Maven只会从JBoss仓库下载发布版的构件，而不会下载快照版的构件。
- `layout`：元素值default表示仓库的布局是Maven2及Maven3的默认布局，而不是Maven1的布局。基本不会用到Maven1的布局。
- 其他：对于`releases`和`snapshots`来说，除了enabled，它们还包含另外两个子元素`updatePolicy`和`checksumPolicy`。
- `updatePolicy`: 用来配置Maven从远处仓库检查更新的频率，默认值是daily，表示Maven每天检查一次。其他可用的值包括：never-从不检查更新；always-每次构建都检查更新；interval：X-每隔X分钟检查一次更新（X为任意整数）。
- `checksumPolicy`: 用来配置Maven检查校验和文件的策略。当构建被部署到Maven仓库中时，会同时部署对应的检验和文件。在下载构件的时候，Maven会验证校验和文件，如果校验和验证失败，当checksumPolicy的值为默认的warn时，Maven会在执行构建时输出警告信息，其他可用的值包括：fail-Maven遇到校验和错误就让构建失败；ignore-使Maven完全忽略校验和错误。

### 镜像
如果仓库X可以提供仓库Y存储的所有内容，那么就可以认为X是Y的一个镜像。用过Maven的都知道，国外的中央仓库用起来太慢了，所以选择一个国内的镜像就很有必要，我推荐国内的阿里云镜像。

阿里云镜像：配置很简单，修改conf文件夹下的`settings.xml`文件，添加如下镜像配置：
```XML
<mirrors>
    <mirror>
      <id>alimaven</id>
      <name>aliyun maven</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>        
    </mirror>
</mirrors>
```

上例子中，mirrorOf的值为central,表示该配置为中央库的镜像，任何对于中央仓库的请求都会转至该镜像，用户也可以用同样的方法配置其他仓库的镜像

这里介绍下`<mirrorOf>`配置的各种选项

- `<mirrorOf>*<mirrorOf>`: 匹配所有远程仓库。
- `<mirrorOf>external:*<mirrorOf>`: 匹配所有远程仓库，使用localhost的除外，使用file://协议的除外。也就是说，匹配所有不在本机上的远程仓库。
- `<mirrorOf>repo1,repo2<mirrorOf>`: 匹配仓库repo1h和repo2，使用逗号分隔多个远程仓库。
- `<mirrorOf>*,!repo1<mirrorOf>`: 匹配所有远程仓库，repo1除外，使用感叹号将仓库从匹配中排除。

需要注意的是，由于镜像仓库完全屏蔽了被镜像仓库，当镜像仓库不稳定或者停止服务的时候，Maven仍将无法访问被镜像仓库，因而将无法下载构件。

## 设置

### settings.xml

位于`USER_HOME/.m2`目录

如果没有该文件，则复制Maven/conf/settings.xml)

添加[阿里云](https://developer.aliyun.com/mvn/guide "阿里云")镜像
```xml
<mirror>
  <id>aliyunmaven</id>
  <mirrorOf>*</mirrorOf>
  <name>阿里云公共仓库</name>
  <url>https://maven.aliyun.com/repository/public</url>
</mirror>
```
如果想使用其它代理仓库，可在`<repositories></repositories>`节点中加入对应的仓库使用地址。以使用 spring 代理仓为例：
```XML
<repository>
  <id>spring</id>
  <url>https://maven.aliyun.com/repository/spring</url>
  <releases>
    <enabled>true</enabled>
  </releases>
  <snapshots>
    <enabled>true</enabled>
  </snapshots>
</repository>
```
在你的`pom.xml`文件`<denpendencies></denpendencies>`节点中加入你要引用的文件信息：
```XML
<dependency>
  <groupId>[GROUP_ID]</groupId>
  <artifactId>[ARTIFACT_ID]</artifactId>
  <version>[VERSION]</version>
</dependency>
```

## 使用

### 创建项目
`mvn archetype:generate -DgroupId=com.mycompany.app -DartifactId=my-app -DarchetypeArtifactId=maven-archetype-quickstart -DarchetypeVersion=1.4 -DinteractiveMode=false
`

参数说明:
- DgroupId: 组织名，公司网址的反写 + 项目名称
- DartifactId: 项目名-模块名
- DarchetypeArtifactId: 指定 ArchetypeId，maven-archetype-quickstart，创建一个简单的 Java 应用
- DinteractiveMode: 是否使用交互模式

### mvn package

take the compiled code and package it in its distributable format, such as a JAR

TEST`java -cp target/my-app-1.0-SNAPSHOT.jar com.mycompany.app.App`

### Java 9 or later
By default your version of Maven might use an old version of the `maven-compiler-plugin` that is not compatible with Java 9 or later versions. To target Java 9 or later, you should at least use version `3.6.0` of the `maven-compiler-plugin` and set the `maven.compiler.release` property to the Java release you are targetting (e.g. 9, 10, 11, 12, etc.).

In the following example, we have configured our Maven project to use version 3.8.1 of maven-compiler-plugin and target Java 11:

```XML
    <properties>
        <maven.compiler.release>11</maven.compiler.release>
    </properties>
 
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.8.1</version>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
```

### maven install

install the package into the local repository, for use as a dependency in other projects locally
 
### mvn verify
run any checks to verify the package is valid and meets quality criteria

typical invocation for building a Maven project uses a Maven life cycle phase. E.g.

Just creating the package and installing it in the local repository for re-use from other projects can be done with

### mvn clean
清除各个模块target目录及里面的内容

### mvn compile
静态编译，根据xx.java生成xx.class文件