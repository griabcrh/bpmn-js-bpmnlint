# 基于bpmn-js的流程设计器校验实现

## [bpmnlint](https://github.com/bpmn-io/bpmnlint)简介

它根据一组已定义的规则来验证您的图表，并将其报告为错误或警告。它可以从命令行检查您的BPMN图，或者通过[bpmn-js-bpmnlint](https://github.com/bpmn-io/bpmn-js-bpmnlint)将其集成到我们的[BPMN建模器中](https://bpmn.io/toolkit/bpmn-js/)：

![](https://bpmn.io/assets/attachments/blog/2018/012-bpmnlint.gif)



## 核心规则

库的核心是用于检测BPMN图中某些模式的规则。每个规则都是由一段代码定义的，该代码可以检测并报告从丢失标签到检测到特定的易于出错的建模模式这一事实。

为了让您更好地了解规则可能是什么，这是到今天[为止](https://github.com/bpmn-io/bpmnlint/tree/master/rules)内置在库[中的规则列表](https://github.com/bpmn-io/bpmnlint/tree/master/rules)：

| 规则名称                   | 描述                           |
| :------------------------- | :----------------------------- |
| `conditional-flows`        | 报告缺少条件的外向流。         |
| `end-event-required`       | 报告缺少的结束事件。           |
| `fake-join`                | 报告实际上为空的隐式连接。     |
| `label-required`           | 报告缺少的标签。               |
| `no-complex-gateway`       | 报告复杂的网关。               |
| `no-disconnected`          | 报告未连接的元素。             |
| `no-gateway-join-fork`     | 报告同时分叉和加入的网关。     |
| `no-implicit-split`        | 报告隐式拆分。                 |
| `no-inclusive-gateway`     | 报告包含的网关。               |
| `single-blank-start-event` | 报告范围中的多个空白开始事件。 |
| `single-event-definition`  | 报告具有多个定义的事件。       |
| `start-event-required`     | 报告缺少的开始事件。           |



## 从零到bpmnlint

让我们对bpmnlint的配置和可扩展性有更好的了解。首先，签出并运行[bpmnlint-playground](https://github.com/bpmn-io/bpmnlint-playground)，这是一个专门设计用于模型验证项目的项目。

```sh
git clone git@github.com:bpmn-io/bpmnlint-playground.git

cd bpmnlint-playground

npm install
npm start
```

执行时，`npm start`将打开带有浏览器应用程序的浏览器窗口，该应用程序已支持了lint支持。

## 配置可用规则

一个`.bpmnlintrc`摆在当前工作目录定义文件，它的规则来适用，以及是否将其当作错误或警告。操场上拿着一个[`.bpmnlintrc`](https://github.com/bpmn-io/bpmnlint-playground/blob/master/.bpmnlintrc)看起来像这样的东西：

```json
{
  "extends": [
    "bpmnlint:recommended",
    "plugin:playground/recommended"
  ],
  "rules": {
    "playground/no-manual-task": "warn"
  }
}
```

该`extends`块告诉bpmnlint从两个预定义的规则集继承配置：`bpmnlint:recommended`和`playground/recommended`，后者由游乐场插件提供。

该`rules`块会覆盖特定规则的报告。该示例将设置`playground/no-manual-task`为警告（而不是错误）。我们可以选择任何规则，例如[内置](https://github.com/bpmn-io/bpmnlint/tree/master/rules)规则，也可以完全将其关闭：

```json
{
  ...
  "rules": {
    ...
    "bpmnlint/label-required": "off"
  }
}
```

在运动场应用程序中，我们可以看到短绒棉不再报告没有标签的开始事件。



## 创建自定义规则

自定义现有规则的报告非常有用，但是并不能满足每个用例。有时，用户或组织希望识别与他们的特定建模样式相关的领域特定模式。bpmnlint通过允许您贡献自定义规则和规则集来解决此问题。

例如，如果我们要制定一个规则来强制每个流节点的标签中包含表情符号，该怎么办？让我们跳入“游乐场” [`plugin`文件夹](https://github.com/bpmn-io/bpmnlint-playground/tree/master/plugin)并`emoji-label-required`在`rules/emoji-label-required.js`文件中创建规则：

```js
const {
  isAny
} = require('bpmnlint-utils');

const emojiRegex = require('emoji-regex');

/**
 * Detect and report missing emojis in element names.
 */
module.exports = function() {

  function check(node, reporter) {
    if (isAny(node, [
      'bpmn:FlowNode',
      'bpmn:SequenceFlow',
      'bpmn:Participant',
      'bpmn:Lane'
    ])) {

      const name = (node.name || '').trim();

      if (!emojiRegex().test(name)) {
        reporter.report(node.id, 'Element must have an emoji');
      }
    }
  }

  return {
    check
  };
};
```

该规则公开了`check(node, reporter)`仅在BPMN标签缺少表情符号时才报告的功能。该[表情符正则表达式](https://www.npmjs.com/package/emoji-regex)实用程序将执行我们的检查。我们必须将其作为依赖项安装在插件目录中，以使规则运行：

```sh
cd plugin
npm install emoji-regex
```

然后，我们需要调整配置以使用该`emoji-label-required`规则。由于这不是一个内置规则，因此我们为其添加前缀（在本例中为`playground`）：

```json
{
  "rules": {
    ...
    "playground/emoji-label-required": "error"
  }
}
```

回到操场上的应用程序，短毛猫现在将报告没有表情符号的标签错误：

![img](https://bpmn.io/assets/attachments/blog/2018/012-bpmnlint-emoji.gif)验证标签中表情符号的存在。

这就完成了我们对bpmnlint可扩展性的快速了解。

查看包含[表情符号标签](https://github.com/bpmn-io/bpmnlint-playground/tree/emoji-label-required)的“游乐场” [分支](https://github.com/bpmn-io/bpmnlint-playground/tree/emoji-label-required)，该[分支](https://github.com/bpmn-io/bpmnlint-playground/tree/emoji-label-required)包含此博客文章中描述的实现。要了解有关规则打包和测试的更多信息，请查看[示例插件](https://github.com/bpmn-io/bpmn-js-bpmnlint)。

以上是bpmn官方教程



## 第一种方案



### 1.下载依赖

在package.json中加入如下依赖

```js
"bpmnlint": "^6.4.0",
"bpmn-js-bpmnlint": "^0.15.0",
"bpmnlint-loader": "^0.1.4",
"file-drops": "^0.4.0",
```



### 2.新建规则文件



在项目中新建`.bpmnlintrc`文件，并使用该`extends`块从通用配置继承：

```js
{
  "extends": "bpmnlint:recommended"
}
```



使用`rules`块添加或自定义规则：

```js
{
  "extends": "bpmnlint:recommended",
  "rules": {
    "label-required": "off"
  }
}
```



### 3.在中配置加载程序`webpack.config.js`

```js
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.bpmnlintrc$/,
        use: [
          {
            loader: 'bpmnlint-loader',
          }
        ]
      }
    ]
  }
};
```

这将确保您的构建可以使用[bpmnlint配置文件](https://github.com/bpmn-io/bpmnlint#configuration)。



### 4.将linter集成到[bpmn-js中](https://github.com/bpmn-io/bpmn-js)

```js
import lintModule from 'bpmn-js-bpmnlint';

import BpmnModeler from 'bpmn-js/lib/Modeler';

import bpmnlintConfig from './.bpmnlintrc';

var modeler = new BpmnModeler({
  linting: {
    bpmnlint: bpmnlintConfig
  },
  additionalModules: [
    lintModule
  ]
});
```



如果在项目运行报错提示`.bpmnlintrc`无法识别，可以使用第二种解决办法



## 第二种方案



在第一种方案的基础上添加如下依赖：

```js
npm i -g bpmnlint bpmnlint-pack-config
```



然后在命令行中执行:

```js
bpmnlint-pack-config -c .bpmnlintrc -o packed-config.js -t es
```

生成`packed-config.js`文件



之后就是同样的方式，将规则文件引入`bpmn-js`中

```js
import * as bpmnlintConfig from './packed-config';

var modeler = new BpmnModeler({
  linting: {
    bpmnlint: bpmnlintConfig
  },
  additionalModules: [
    lintModule
  ]
});
```



转化成`packed-config.js`文件的好处就是可以自己写逻辑

比如我将规则翻译成了中文；比如我加入了：如果画布中拖入了用户任务，用户任务就必须有左右连接线，不能单独存在画布中。



## 可控制是否校验

```js
import fileDrop from 'file-drops';

var modeler = new BpmnModeler({
    linting: {
        bpmnlint: bpmnlintConfig,
        active: that.getUrlParam('linting')
    },
    additionalModules: [
        lintModule
    ],
});

modeler.on('linting.toggle', function(event) {
    const active = event.active;
    that.setUrlParam('linting', active);
});

const dndHandler = fileDrop('Drop BPMN Diagram here.', function(files) {
    this.bpmnModeler.importXML(files[0].contents);
});
document.querySelector('body').addEventListener('dragover', dndHandler);


// 流程校验使用
setUrlParam = (name, value) => {

    var url = new URL(window.location.href);

    if (value) {
        url.searchParams.set(name, 1);
    } else {
        url.searchParams.delete(name);
    }

    window.history.replaceState({}, null, url.href);
}
// 流程校验使用
getUrlParam = (name) => {
    var url = new URL(window.location.href);

    return url.searchParams.has(name);
}
```

主要代码就是这部分，最终效果图如下：

<img src="./validprocess2.png" style="zoom:80%;" />


test




