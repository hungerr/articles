# NPM简介

### 更新npm
```
npm install npm@latest -g
```

### 设置源

```
npm config set registry https://registry.your-registry.npme.io/
```

### 下载包`npm install`

#### locally
```
npm install <package_name>
```
会在在当前目录下创建`node_modules`目录

#### globally
```
npm install -g <package_name>
```
install aliases
```
aliases: add, i, in, ins, inst, insta, instal, isnt, isnta, isntal, isntall
```

#### package
- a) a folder containing a program described by a `package.json` file
- b) a gzipped tarball containing (a)   The filename must use `.tar`, `.tar.gz`, or `.tgz` as the extension.
- c) a url that resolves to (b)
- d) a `[<@scope>/]<name>@<version>` that is published on the registry (see registry) with (c)
- e) a `<name>@<tag>` (see npm dist-tag) that points to (d)
- f) a `<name>` that has a "latest" tag satisfying (e)
- g) a `<git remote url>` that resolves to (a)

- `npm install (in a package directory, no arguments)`:
        
    Install the `dependencies` to the local node_modules folder.

    By default, npm install will install all modules listed as `dependencies` in `package.json`.

    With the `--production` flag (or when the NODE_ENV environment variable is set to production), npm will not install modules listed in `devDependencies`. To install all modules listed in `both` dependencies and devDependencies when NODE_ENV environment variable is set to production, you can use `--production=false`.

- `npm install <folder>`:

    If `<folder>` sits `inside` the root of your project, its dependencies will be installed and may be `hoisted` to the top-level node_modules as they would for other types of dependencies. If `<folder>` sits `outside `the root of your project, npm will `not install` the package dependencies in the directory `<folder>`, but it will `create a symlink` to `<folder>`.

    If you want to install the content of a directory like a package from the registry instead of creating a link, you would need to use the --install-links option.

    Example:
    ```
    npm install ../../other-package --install-links
    npm install ./sub-package
    ```

- `npm install <tarball file>`:

    Install a package that is sitting on the filesystem

    The filename must use `.tar`, `.tar.gz`, or `.tgz` as the extension.

    The package contents should reside in a subfolder inside the tarball(usually it is called `package/`). 

    The package must contain a `package.json` file with name and version properties.

    Example:
    ```
    npm install ./package.tgz
    ```

- `npm install <tarball url>`:

    ```
    npm install https://github.com/indexzero/forever/tarball/v0.5.6
    ```

- `npm install [<@scope>/]<name>`

    ```
    npm install sax
    npm install githubname/reponame
    npm install @myorg/privatepackage
    npm install node-tap --save-dev
    npm install dtrace-provider --save-optional
    npm install readable-stream --save-exact
    npm install ansi-regex --save-bundle
    ```

- `npm install <alias>@npm:<name>`

    ```
    npm install my-react@npm:react
    npm install jquery2@npm:jquery@2
    npm install jquery3@npm:jquery@3
    npm install npa@npm:npm-package-arg
    ```

- `npm install [<@scope>/]<name>@<tag>`

    ```
    npm install sax@latest
    npm install @myorg/mypackage@latest
    ```

 - `npm install [<@scope>/]<name>@<version>`

    ```
    npm install sax@0.1.1
    npm install @myorg/privatepackage@1.5.0
    ```

- `npm install [<@scope>/]<name>@<version range>`

    ```
    npm install sax@">=0.1.0 <0.2.0"
    npm install @myorg/privatepackage@"16 - 17"
    ```

- `npm install <git remote url>`

    ```
    <protocol>://[<user>[:<password>]@]<hostname>[:<port>][:][/]<path>[#<commit-ish> | #semver:<semver>]
    npm install git+ssh://git@github.com:npm/cli.git#v1.0.27
    npm install git+ssh://git@github.com:npm/cli#pull/273
    npm install git+ssh://git@github.com:npm/cli#semver:^5.0
    npm install git+https://isaacs@github.com/npm/cli.git
    npm install git://github.com/npm/cli.git#v1.0.27
    GIT_SSH_COMMAND='ssh -i ~/.ssh/custom_ident' npm install git+ssh://git@github.com:npm/cli.git
    ```

- `npm install github:<githubname>/<githubrepo>[#<commit-ish>]`

    Install the package at `https://github.com/githubname/githubrepo` by attempting to clone it using git.

    If `#<commit-ish>` is provided, it will be used to clone exactly that commit.
    ```
    npm install mygithubuser/myproject
    npm install github:mygithubuser/myproject
    ```

- `npm install gist:[<githubname>/]<gistID>[#<commit-ish>|#semver:<semver>]`

    Install the package at `https://gist.github.com/gistID` by attempting to clone it using git. The GitHub username associated with the gist is optional and will not be saved in `package.json`.

    ```
    npm install gist:101a11beef
    ```

- `npm install gitlab:<gitlabname>/<gitlabrepo>[#<commit-ish>]`

    Install the package at `https://gitlab.com/gitlabname/gitlabrepo` by attempting to clone it using git.

    ```bash
    npm install gitlab:mygitlabuser/myproject
    npm install gitlab:myusr/myproj#semver:^5.0
    ```

- `npm install bitbucket:<bitbucketname>/<bitbucketrepo>[#<commit-ish>]`

    Install the package at https://bitbucket.org/bitbucketname/bitbucketrepo by attempting to clone it using git

    ```
    npm install bitbucket:mybitbucketuser/myproject
    ```

### 强制install

The `-f` or `--force` argument will force npm to fetch remote resources even if a local copy exists on disk.

```
npm install sax --force
```

### 忽略`package-lock.json`

If `package-lock` set to `false`, then ignore `package-lock.json` files when installing

### package.json包版本

SEE [semver](https://github.com/npm/node-semver "semver")

- `latest`  最新
- `version` Must match version exactly
- `>version` Must be greater than version
- `>=version` etc
- `<version`
- `<=version`
- `~version` Patch release 最新的patch `1.0` or `1.0.x` or `~1.0.4`
- `~1.2.3` := `>=1.2.3 <1.(2+1).0` := `>=1.2.3 <1.3.0-0`
- `^version` Minor releases 最新的minor版本 `1` or `1.x` or `^1.0.4`  
- `^1.2.3`:= `>=1.2.3 <2.0.0-0`
- `1.2.x` 1.2.0, 1.2.1, etc., but not 1.3.0 `>=1.2.0 <1.3.0-0`
- `1` := `1.x.x` := `>=1.0.0 <2.0.0-0`
- `1.2` := `1.2.x` := `>=1.2.0 <1.3.0-0`
- `http://...` a tarball URL `http://asdf.com/asdf.tar.gz`
- `*` Matches any version `>=0.0.0`
- `""` (just an empty string) Same as `*`
- `version1 - version2` Same as `>=version1 <=version2`.
- `range1 || range2` Passes if either `range1` or `range2` are satisfied.
- `git...` See `Git URLs as Dependencies` below
- `user/repo` See `GitHub URLs` below
- `tag` A specific version tagged and published as tag See `npm dist-tag`
- `path/path/path` See `Local Paths` below

#### Git URLs as Dependencies
```
<protocol>://[<user>[:<password>]@]<hostname>[:<port>][:][/]<path>[#<commit-ish> | #semver:<semver>]
```
```
git+ssh://git@github.com:npm/cli.git#v1.0.27
git+ssh://git@github.com:npm/cli#semver:^5.0
git+https://isaacs@github.com/npm/cli.git
git://github.com/npm/cli.git#v1.0.27
```

#### GitHub URLs
As of version 1.1.65, you can refer to GitHub urls as just `"foo": "user/foo-project"`. Just as with git URLs, a `commit-ish` suffix can be included. For example:
```JSON
{
  "name": "foo",
  "version": "0.0.0",
  "dependencies": {
    "express": "expressjs/express",
    "mocha": "mochajs/mocha#4727d357ea",
    "module": "user/repo#feature\/branch"
  }
}
```

#### npm-dist-tag

```
npm dist-tag add <package-spec (with version)> [<tag>]
npm dist-tag rm <package-spec> <tag>
npm dist-tag ls [<package-spec>]

alias: dist-tags
```

```
npm install <name>@<tag>
```

#### Local Paths
As of version 2.0.0 you can provide a path to a local directory that contains a package. Local paths can be saved using `npm install -S` or `npm install --save`, using any of these forms:
```
../foo/bar
~/foo/bar
./foo/bar
/foo/bar
```
in which case they will be normalized to a relative path and added to your package.json. For example:
```JSON
{
  "name": "baz",
  "dependencies": {
    "bar": "file:../foo/bar"
  }
}
```

### 更新packages

全局
```
npm update -g
```
本地包
```
cd /path/to/project
npm update
```
To test 更新
```
npm outdated
```
Example 包版本
```JSON
{
  "dist-tags": { "latest": "1.2.2" },
  "versions": [
    "1.2.2",
    "1.2.1",
    "1.2.0",
    "1.1.2",
    "1.1.1",
    "1.0.0",
    "0.4.1",
    "0.4.0",
    "0.2.0"
  ]
}
```
If app's `package.json` contains:
```JSON
"dependencies": {
  "dep1": "^1.1.1"
}
```
Then `npm update` will install `dep1@1.2.2`, because 1.2.2 is latest and 1.2.2 satisfies ^1.1.1.

However, if app's package.json contains:
```JSON
"dependencies": {
  "dep1": "~1.1.1"
}
```
In this case, running `npm update` will install `dep1@1.1.2`. 

### Uninstalling packages

```
npm uninstall <package_name>
npm uninstall -g <package_name>
```
To remove a package from the dependencies in package.json, use the --save flag
```
npm uninstall --save <package_name>
```

### `package-lock.json`

`package-lock.json`的出现，保证了依赖版本的一致性。

在npm@5.4.2版本后的表现：


- 无`package-lock.json`
    `npm i` 根据`package.json`进行安装，并生成`package-lock.json`

- `package.json`和`package-lock.json`的版本不兼容
    `npm i`会以`package.json`为准进行安装，并更新`package-lock.json`

- `package.json`和`package-lock.json`的版本兼容
    `npm i`会以`package-lock.json`为准进行安装

### npm ci
由于以上1、2点的存在，即使有`package-lock.json`文件，配合`npm i`，我们也不能保证线上构建时的依赖版本与本地开发时的一致。

`npm ci`是类似于`npm i`的命令，适用于`ci`时安装依赖。`npm ci`与`npm i`主要的差异有：

- 使用`npm ci`的项目必须存在`package-lock.json`或`npm-shrinkwrap.json`文件，否则无法执行（即以上1的情况）
- 如果p`ackage-lock.json`或`npm-shrinkwrap.json`中的依赖与`package.json`中不一致（即以上2的情况），`npm ci`会报错并退出，而不是更新lock文件
- `npm ci`只会安装整个项目，无法单独安装某个依赖项目
- 如果项目中已有`node_modules`，该文件夹会在`npm ci`执行安装前自动被移除
- 该命令不会写入`package.json`或`lock`文件，安装的行为是完全被固定的。

基于以上几种特性，使用npm ci能够有效防止线上构建的依赖与开发者本地不一致。

### 如何解决`package-lock.json`的冲突
`package-lock.json`文件不由开发者自行写入，在协同开发时，某一方若更新了依赖，很容易产生大量冲突，且难以逐个解决。

npm文档中推荐的两种方式：

- 从`5.7.0`版本的npm开始，解决`package-lock.json`的冲突可以通过解决`package.json`的冲突后，运行 `npm install [--package-lock-only]` 来完成。npm会自动解析`package-lock.json`中的冲突，并生成一个有着合理的tree结构，且包含了两个分支的所有依赖的lock文件。
- 文档中还提到了一个`npm-merge-driver`工具，可以帮助开发者解决`package-lock.json`的冲突。

至于旧版本的npm，可以尝试对于`package-lock.json`文件，忽略掉冲突，并重新`npm i`。以下是一些可行的具体操作。

- 在`vscode`上选择`Accept All Incoming/Current Changes`后`npm i`。
- 将`package-lock.json`文件checkout到之前某个commit的状态，`git checkout [commitId] -- path/to/package-lock.json`，再`npm i`

最好将文件重置到非自己开发分支的状态，便于合并之后的验证。例如，对于将master合进自己开发的特性分支的情况（如`git pull origin master`），对于冲突，选择`Accept All Incoming`。
