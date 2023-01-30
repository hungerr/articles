# Maven Build Lifecycle

## 三个内置的生命周期
Maven 基于生命周期的核心概念。有三个内置的生命周期：

- `clean`：负责清理项目；
- `default`：负责构建项目；
- `site`：负责建立项目站点。

每个生命周期都包含一些阶段`phase`，用户与 Maven 最直接的交互方式就是调用这些生命周期的`phase`。

### clean 生命周期及其阶段
`clean`生命周期包含以下阶段：

- `pre-clean`：在清理之前完成一些所需的工作；
- `clean`：删除之前构建出的所有文件；
- `post-clean`：在清理之后完成一些所需的工作。


### default 生命周期及其阶段
`default`生命周期包含了实际构建时需要执行的所有步骤，是最重要的生命周期。该生命周期包含以下阶段：

- `validate`：验证项目是否正确，所有必要信息是否可用；
- `initialize`：初始化构建状态，例如设置 property 或创建目录；
- `generate-sources`：生成要包含在编译中的任何源代码；
- `process-sources`：处理源代码，例如替换所有引用的值；
- `generate-resources`：生成要包含在包中的资源；
- `process-resources`：将资源复制并处理到目标目录中，准备打包；
- `compile`：编译项目的源代码；
- `process-classes`：对编译生成的文件进行后期处理，例如对 Java 类进行字节码增强；
- `generate-test-sources`：生成要包含在编译中的任何测试源代码；
- `process-test-sources`：处理测试源代码，例如替换所有引用的值；
- `generate-test-resources`：为测试创建资源；
- `process-test-resources`：将资源复制并处理到测试目标目录中；
- `test-compile`：将测试源代码编译到测试目标目录中；
- `process-test-classes`：对测试编译生成的文件进行后期处理，例如对 Java 类进行字节码增强；
- `test`：使用合适的单元测试框架运行测试。这些测试不应要求打包或部署代码；
- `prepare-package`：在实际打包之前，执行准备打包所需的任何操作。这通常会生成已处理但尚未打包的所有文件和目录;
- `package`：获取编译后的代码，并将其打包为可分发的格式，如 JAR；
- `pre-integration-test`：在执行集成测试之前执行所需的操作。这可能涉及一些事项，如设置所需环境等；
- `integration-test`：如有必要，处理包并将其部署到可以运行集成测试的环境中；
- `post-integration-test`：执行集成测试后执行所需的操作。这可能包括清理环境；
- `verify`：运行任何检查以验证包是否有效并符合质量标准；
- `install`：将包安装到本地仓库中，作为本地其他项目中的依赖项使用；
- `deploy`：在集成或发布环境中完成，将最终包复制到远程仓库，以便与其他开发人员和项目共享。

### site 生命周期及其阶段
- `pre-site`：在生成实际的项目站点之前执行所需的流程；
- `site`：生成项目的站点文档；
- `post-site`：执行完成站点生成所需的流程；
- `site-deploy`：将生成的站点文档部署到指定的 web 服务器。

### 常见的命令行调用
每个生命周期内的阶段都是有顺序的，且后面的阶段依赖于前面的阶段。当调用某个生命周期的某个阶段时，则会尝试先依次调用该生命周期内位于该阶段前面的各个阶段。比如，当调用 `clean` 生命周期的 `pre-clean` 阶段时，仅有 `pre-clean` 阶段得以执行；当调用 `clean` 生命周期的 `clean` 阶段时，则会依次执行 `pre-clean` 、`clean` 阶段；当调用 `clean` 生命周期的 `post-clean` 阶段时，则会依次执行 `pre-clean` 、`clean` 、`post-clean` 阶段。

这三个生命周期彼此都是独立的，用户可以仅调用 `clean` 生命周期的某个阶段，或者仅仅调用 `default` 生命周期的某个阶段，而不会对其他生命周期产生影响。

如果想要 `jar` 包，请运行 `mvn package`。如果要运行单元测试，请运行 `mvn test`。

如果你不确定你想要什么，那就调用 `mvn verify` 命令。在 `verify` 阶段执行验证之前，将按顺序执行 `default` 生命周期中在 `verify` 阶段之前的其他每个阶段（`validate`、`compile`、`package` 等），即只需要调用要执行的最后一个构建阶段即可。在大多数情况下，调用 `verify` 的效果与 `package` 相同。但是，如果存在集成测试，也将执行这些阶段。在 `verify` 阶段，可以进行一些额外的检查。例如，如果您的代码是根据预定义的 `checkstyle` 规则编写的。

`phases`和`goals`可以链式调用。

在构建环境中，使用 `mvn clean deploy` 以下调用来干净地构建工件并将其部署到共享仓库中。同一命令可用于多模块场景（即具有一个或多个子项目的项目），Maven 将遍历每个子项目并执行 `clean`，然后执行 `deploy`（包括所有先前的构建阶段步骤）。
