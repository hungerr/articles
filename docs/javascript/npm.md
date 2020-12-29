# NPM简介

### 更新
```
npm install npm@latest -g
```

### 下载包
#### locally
```
npm install <package_name>
```
会在在当前目录下创建`node_modules`目录

#### globally
```
npm install -g <package_name>
```

### package.json包版本

#### Patch release 最新的patch
`1.0` or `1.0.x` or `~1.0.4`
#### Minor releases 最新的minor版本
`1` or `1.x` or `^1.0.4`
#### Major releases 最新的major版本
`*` or `x`

### 设置源

```
npm config set registry https://registry.your-registry.npme.io/
```
