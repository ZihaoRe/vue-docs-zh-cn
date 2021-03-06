# 插件开发指南

## 核心概念

- [Creator](#creator)
- [Service](#service)
- [CLI 插件](#cli-插件)
- [Service 插件](#service-插件)
- [Generator](#generator)
- [提示符](#提示符)

系统里有两个主要的部分：

- `@vue/cli`：全局安装的，暴露 `vue create <app>` 命令；
- `@vue/cli-service`：本地安装，暴露 `vue-cli-service` 命令。

两者皆应用了基于插件的架构。

### Creator

[Creator][creator-class] 是调用 `vue create <app>` 时创建的类。负责偏好提示符、调用 generator 和安装依赖。

### Service

[Service][service-class] 是调用 `vue-cli-service <command> [...args]` 时创建的类。负责管理内部的 webpack 配置、暴露服务和构建项目的命令等。

### CLI 插件

CLI 插件是一个可以为 `@vue/cli` 项目添加额外特性的 npm 包。它应该始终包含一个 [Service 插件](#service-插件)作为其主要导出，且可选的包含一个 [Generator](#generator) 和一个 [Prompt 文件](#prompts-for-3rd-party-plugins)。

一个典型的 CLI 插件的目录结构看起来是这样的：

```
.
├── README.md
├── generator.js  # generator (可选)
├── prompts.js    # prompt 文件 (可选)
├── index.js      # service 插件
└── package.json
```

### Service 插件

Service 插件会在一个 Service 实例被创建时自动加载——比如每次 `vue-cli-service` 命令在项目中被调用时。

注意我们这里讨论的“service 插件”的概念要比发布为一个 npm 包的“CLI 插件”的要更窄。前者涉及一个会被 `@vue/cli-service` 在初始化时加载的模块，也经常是后者的一部分。

额外的，`@vue/cli-service` 的[内建 command][commands] 和[配置模块][config]也是全部以 service 插件实现的。

一个 service 插件应该导出一个函数，这个函数接受两个参数：

- 一个 [PluginAPI][plugin-api] 示例

- 一个包含 `vue.config.js` 内指定的项目本地选项的对象，或者在 `package.json` 内的 `vue-cli` 字段。

这个 API 允许 service 插件针对不同的环境扩展/修改内部的 webpack 配置，并向 `vue-cli-service` 注入额外的命令。例如：

``` js
module.exports = (api, projectOptions) => {
  api.chainWebpack(webpackConfig => {
    // 通过 webpack-chain 修改 webpack 配置
  })

  api.configureWebpack(webpackConfig => {
    // 修改 webpack 配置
    // 或返回通过 webpack-merge 合并的配置对象
  })

  api.registerCommand('test', args => {
    // 注册 `vue-cli-service test`
  })
}
```

#### Service 插件中的环境变量

对于环境变量来说，有一个值得注意的重要的事情，就是了解它们何时被解析。一般情况下，像 `vue-cli-service serve` 或 `vue-cli-service build` 这样的命令会始终调用 `api.setMode()` 作为它的第一件事。然而，这也意味着这些环境变量可能在 service 插件被调用的时候还不可用：

``` js
module.exports = api => {
  process.env.NODE_ENV // 可能还没有解析出来

  api.registerCommand('build', () => {
    api.setMode('production')
  })
}
```

所以，在 `configureWebpack` 或 `chainWebpack` 函数中依赖环境变量更加安全，它们只有在 `api.resolveWebpackConfig()` 完全调用结束之后才会被懒执行：

``` js
module.exports = api => {
  api.configureWebpack(config => {
    if (process.env.NODE_ENV === 'production') {
      // ...
    }
  })
}
```

#### 在插件中解析 webpack 配置

一个插件可以通过调用 `api.resolveWebpackConfig()` 取回解析好的 webpack 配置。每次调用都会新生成一个 webpack 配置用于在未来修改。

``` js
api.registerCommand('my-build', args => {
  // 确认模式设置并加载环境变量
  api.setMode('production')

  const configA = api.resolveWebpackConfig()
  const configB = api.resolveWebpackConfig()

  // 针对不同的目的修改 `configA` 和 `configB`...
})
```

或者，一个插件也可以通过调用 `api.resolveChainableWebpackConfig()` 获得一个新生成的[链式配置](https://github.com/mozilla-neutrino/webpack-chain)：

``` js
api.registerCommand('my-build', args => {
  api.setMode('production')

  const configA = api.resolveChainableWebpackConfig()
  const configB = api.resolveChainableWebpackConfig()

  // 针对不同的目的链式修改 `configA` 和 `configB`...

  const finalConfigA = configA.toConfig()
  const finalConfigB = configB.toConfig()
})
```

#### 第三方插件的自定义选项

`vue.config.js` 的导出将会[通过一个 schema 的验证](https://github.com/vuejs/vue-cli/blob/dev/packages/%40vue/cli-service/lib/options.js#L3)以避免笔误和错误的配置值。然而，一个第三方插件仍然允许用户通过 `pluginOptions` 字段配置其行为。例如，对于下面的 `vue.config.js`：

``` js
module.exports = {
  pluginOptions: {
    foo: { /* ... */ }
  }
}
```

该第三方插件可以读取 `projectOptions.pluginOptions.foo` 来做条件式的决定配置。

### Generator

一个发布为 npm 包的 CLI 插件可以包含一个 `generator.js` 或 `generator/index.js` 文件。插件内的 generator 将会在两种场景下被调用：

- 在一个项目的初始化创建过程中，如果 CLI 插件作为项目创建预设选项的一部分被安装。

- 插件在项目创建好之后通过 `vue invoke` 独立调用时被安装。

这里的 [GeneratorAPI][generator-api] 允许一个 generator 向 `package.json`  注入额外的依赖或字段，并向项目中添加文件。

一个 generator 应该导出一个函数，这个函数接收三个参数：

1. 一个 `GeneratorAPI` 实例：

2. 这个插件的 generator 选项。这些选项会在项目创建提示符过程中被解析，或从一个保存在 `~/.vuerc` 中的预设选项中加载。例如，如果保存好的 `~/.vuerc` 像如下的这样：

    ``` json
    {
      "presets" : {
        "foo": {
          "plugins": {
            "@vue/cli-plugin-foo": { "option": "bar" }
          }
        }
      }
    }
    ```

    如果用户使用 `foo` 预设选项创建了一个项目，那么 `@vue/cli-plugin-foo` 的 generator 就会收到 `{ option: 'bar' }` 作为第二个参数。

    对于一个第三方插件来说，该选项将会解析自提示符或用户执行 `vue invoke` 时的命令行参数中 (详见[第三方插件的提示符](#第三方插件的提示符))。

3. 整个预设选项 (`presets.foo`) 将会作为第三个参数传入。

**示例：**

``` js
module.exports = (api, options, rootOptions) => {
  // 修改 `package.json` 字段
  api.extendPackage({
    scripts: {
      test: 'vue-cli-service test'
    }
  })

  // 复制并用 ejs 渲染 `./template` 内所有的文件
  api.render('./template')

  if (options.foo) {
    // 有条件的生成文件
  }
}
```

### 提示符

#### 内建插件的提示符

只有内建插件可以定制创建新项目时的初始化提示符，且这些提示符模块放置在 [`@vue/cli` 包的内部][prompt-modules]。

一个提示符模块应该导出一个函数，这个函数接收一个 [PromptModuleAPI][prompt-api] 实例。这些提示符的底层使用 [inquirer](https://github.com/SBoudrias/Inquirer.js) 进行展示：

``` js
module.exports = api => {
  // 一个特性对象应该是一个有效的 inquirer 选择对象
  cli.injectFeature({
    name: 'Some great feature',
    value: 'my-feature'
  })

  // injectPrompt 的预期是一个有效的 inquirer 提示符对象
  cli.injectPrompt({
    name: 'someFlag',
    // 确认你的提示符只在用户已经选取了你的特性的时候展示
    when: answers => answers.features.include('my-feature'),
    message: 'Do you want to turn on flag foo?',
    type: 'confirm'
  })

  // 当所有的提示符都完成之后，将你的插件注入到
  // 即将传递给 Generator 的选项中
  cli.onPromptComplete((answers, options) => {
    if (answers.features.includes('my-feature')) {
      options.plugins['vue-cli-plugin-my-feature'] = {
        someFlag: answers.someFlag
      }
    }
  })
}
```

#### 第三方插件的提示符

第三方插件通常会在一个项目创建完毕后被手动安装，且用户将会通过调用 `vue invoke` 来初始化这个插件。如果这个插件在其根目录包含一个 `prompt.js`，那么它将会在被调用的时候使用。这个文件应该导出一个用于 Inquirer.js 的[问题](https://github.com/SBoudrias/Inquirer.js#question)的数组。这些被解析的答案对象会作为选项被传递给插件的 generator。

或者，用户可以通过在命令行传递选项来忽略提示符直接初始化插件，比如：

``` sh
vue invoke my-plugin --mode awesome
```

## 开发核心插件的注意事项

> 这个章节只用于你在本仓库中的内建插件内部的工作。

一个带有为本仓库注入额外依赖的 generator 的插件 (比如 `chai` 会通过 `@vue/cli-plugin-unit-mocha/generator/index.js` 被注入) 应该将这些依赖列入其自身的 `devDependencies` 字段。这会确保：

1. 这个包始终存在于该仓库的根 `node_modules` 中，因此我们不必在每次测试的时候重新安装它们。

2. `yarn.lock` 会保持其一致性，因此 CI 程序可以更好的利用缓存。

[creator-class]: https://github.com/vuejs/vue-cli/tree/dev/packages/@vue/cli/lib/Creator.js
[service-class]: https://github.com/vuejs/vue-cli/tree/dev/packages/@vue/cli-service/lib/Service.js
[generator-api]: https://github.com/vuejs/vue-cli/tree/dev/packages/@vue/cli/lib/GeneratorAPI.js
[commands]: https://github.com/vuejs/vue-cli/tree/dev/packages/@vue/cli-service/lib/commands
[config]: https://github.com/vuejs/vue-cli/tree/dev/packages/@vue/cli-service/lib/config
[plugin-api]: https://github.com/vuejs/vue-cli/tree/dev/packages/@vue/cli-service/lib/PluginAPI.js
[prompt-modules]: https://github.com/vuejs/vue-cli/tree/dev/packages/@vue/cli/lib/promptModules
[prompt-api]: https://github.com/vuejs/vue-cli/tree/dev/packages/@vue/cli/lib/PromptModuleAPI.js
