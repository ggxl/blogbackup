---
title: monaco-editor代码编辑器
no_word_count: false
tags: [技术,前端,插件]
---

### monaco-editor介绍

官方介绍：

The Monaco Editor is the code editor that powers VS Code. A good page describing the code editor's features is here.

It is licensed under the MIT License and supports IE 11, Edge, Chrome, Firefox, Safari and Opera.

The Monaco editor is not supported in mobile browsers or mobile web frameworks.

Find more information at the Monaco Editor repo.

[官方文档](https://microsoft.github.io/monaco-editor/api/index.html)

[下载地址](https://registry.npmjs.org/monaco-editor/-/monaco-editor-0.13.1.tgz)也可以使用`npm install monaco-editor`

### 页面中使用

在页面中引用
```
<script src="monaco-editor/min/vs/loader.js"></script>
```

<!--more-->

##### 获取所有language

```
monaco.languages.getLanguages();
```

##### 设置language

```
var newModel = monaco.editor.createModel("class A{}", "java");
editor.setModel(newModel);
```

##### 共三个主题

>'vs' | 'vs-dark' | 'hc-black';

##### 设置主题

```
monaco.editor.setTheme("hc-black");
```

---



[在线demo](/FunctionPage/plug-in/monaco-editor/index.html)