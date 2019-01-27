title: 思考快与慢(一):高效开发之Intellij IDEA
date: 2019/1/27 12:51:50
toc  : true
tags: [高效,开发,快与慢]
categories: technology
description: <<思考快与慢>>系列之高效的Intellij IDEA原生快捷.

-----

在15年开始从Eclipse转入IDEA时,为了兼容当时的使用习惯,自定义了许多快捷键.但随着后面MacOS逐渐成为我主要的开发平台,原先在Windows平台上定义的很多快捷键与MacOS原生快捷键有很多的冲突,经过一个阶段的思考与实践之后,我决定放弃原有的习惯,以全新的视角接受Intellij IDEA中的快捷键.

> 下述快捷键适用于MacOS平台,基于`Mac 10.5+`.

## 编辑

| 快捷键               | 功能                                | 说明                      |
| -------------------- | ----------------------------------- | ------------------------- |
| Command+Space        | Basic code completion               | 基本代码补齐              |
| Command+Shift+Space  | Smart code completion               | 智能补齐                  |
| Command+Shift+Enter  | Complete statement                  | 语句补全(末尾自动加分号)  |
|     |                                     |             |
| Command+Alt+Enter    | Start New Line Before Current | 从上方开始一行            |
| Shift+Enter          | Start New Line | 从下方开始一行            |
| Shift+Alt+↑/↓        | Move Line Up/Down | 上移/下移一行             |
| Shift+Command+↑/↓    | Move Statement Up/Down | 上移/下移一个块           |
| Alt+↑/↓    | Extend/Shrink Selection | 选中/反选代码块   |
|     |                                     |             |
| Command+P            | Parameter info                      | 参数信息提示                  |
| Command+Y | Open quick definition lookup | 浮窗快速查看定义 |
| Ctrl+J | Quick documentation lookup | 文档快速预览 |
| Ctrl+O | Override methods | 可覆盖方法快速预览 |
| Crtl+I | Implement Code | 实现接口方法 |
|  |  |  |
| Command+Shift+delete | Last edit location                  | 跳转到上次编辑的地方      |
| Command+Z            | Undo                                | 撤销修改                  |
| Command+Shift+Z | Redo | 恢复刚才修改 |
| Command+/            | Comment/uncomment wiht line comment | 注释/反注释行             |
|                      |                                     |                           |
| Command+N            | Generate code                       | 生成代码                  |
| Command+Alt+T        | Surround with                       | 嵌入代码块，如try...catch |
| Command+Alt+L        | Reformat code                       | 格式化代码                   |
|                      |                                     |                           |
| Command+X            | Cut current line to clipboard       | 剪贴行到剪贴板            |
| Command+C            | Copy current line to clipboard      | 复制行                    |
| Command+V            | Paste from clipboard                | 粘贴                      |
| Command+D            | Duplicate current line              | 复制行                    |
| Command+Delete       | Delete line at caret                | 删除行                    |
| Command+W            | Close active editor tab             | 关闭当前编辑窗口          |

# 生产力

| 快捷键          | 功能                   | 说明                                                         |
| --------------- | ---------------------- | ------------------------------------------------------------ |
| Command+J       | Insert Live Template   | 模板插入                                                     |
| Alt+Enter       | Show Intention Actions | 意图预测与智能帮助,比如简单重构,移除死代码,结构调整,自动导包等 |
| Command+Shift+A | Find Action            | 命令查询                                                     |

## 模板补全

在使用Command+J选中模板关键字或者直接敲如模板关键字后,可以直接通过Tab或Enter键触发补全操作,其中`$1$2…`是我们要填充的模板变量,常见的模板操作如下

| 关键字  | 模板定义                             | 说明          |
| ------- | ------------------------------------ | ------------- |
| `ifn`   | `if ($1 == null) {}`                 | 判空操作      |
| `inn`   | `if ($1 != null) {}`                 | 判非空操作    |
| `fori`  | `for (int $1 = 0; $1 < $2; $1++) {}` | 创建索引循环  |
| `todo`  | `// TODO: $1`                        | 添加todo说明  |
| `fixme` | `// FIXME: 7/19/16 $1`               | 添加fixme说明 |
| `inst`  | `if ($1 instanceof $2) {}`           | 类型关系判断  |
| `sout`  | `System.out.println($1);`            | 标准输出      |



## 后缀完成

[后缀完成](https://blog.jetbrains.com/idea/2014/03/postfix-completion/)是JetBrains系IDE的一项新功能,目的是减少光标经常性的前后移动,其使用惯例为:先使用元素,再考虑变量声明或结构补全.它的主要功能是根据当前元素的属性,提供可能的行为建议,例如使用if-else结构包围,判(非)空,格式化,进行类型转换等,常用的后缀声明如下:

| 后缀完成关键字    | 定义                           | 说明                              |
| ----------------- | ------------------------------ | --------------------------------- |
| element.`sout`    | `System.out.println(element);` | 将当前内容element定位到标准输出流 |
| element.`return`  | `return element;`              | 将当前元素element作为返回值       |
| element.`null`    | `if (string == null) {}`       | 元素判空操作                      |
| element.`notnull` | if (string != null) {}         | 元素判非空操作                    |

# 错误

| 快捷键      | 功能                           | 说明                                    |
| ----------- | ------------------------------ | --------------------------------------- |
| F2/Shift+F2 | Next/previous hghlighted error | 下一个/上一个错误处,常配合Alt+Enter使用 |
| Command+F1  | Show desciptions of error      | 错误提示信息                            |



# 重构

| 快捷键        | 功能              | 说明               |
| ------------- | ----------------- | ------------------ |
| Shift+F6      | Rename            | 重命名类/方法/变量 |
| Ctrl+T        | Refactor This     | 开始重构,重构一切  |
|               |                   |                    |
| Command+F6    | Change Signature  | 修改方法签名       |
| Command+Alt+M | Extract Method    | 重构成方法         |
| Command+Alt+V | Extract Variable  | 重构成变量         |
| Command+Alt+F | Extract Field     | 重构成字段         |
| Command+Alt+C | Extract Constant  | 重构成静态变量     |
| Command+Alt+P | Extract Parameter | 重构参数列表       |
|               |                   |                    |
| F5            | Copy              | 拷贝类/目录等      |
| F6            | Move              | 移动类/目录等      |

# 导航

| 快捷键            | 功能                           | 说明                                |
| ----------------- | ------------------------------ | ----------------------------------- |
| Command+O         | Go to class                    | 搜索类                              |
| Command+Shift+O   | Go to file                     | 搜索文件                            |
| Command+Alt+O     | Go to sysbol                   | 搜索符号                            |
|                   |                                |                                     |
| Command+L         | Go to line                     | 跳转到某行                          |
| Command+U         | Go to super-method/super-class | 跳转到父级方法/类                   |
| Command+E         | Recent files popup             | 跳到最近编辑过的文件列表            |
| Command+B         | Go to declaration              | 跳转到定义行                        |
| Command+Alt+B     | Go to implementation(s)        | 进入接口实现方法                    |
|                   |                                |                                     |
| Command+F12       | File structure popup           | 文件结构弹窗,包含本类及所有基类方法 |
| Ctrl+H            | Type hierarchy                 | 类继承体系                          |
| Ctrl+Alt+H        | Call hierarchy                 | 调用层次结构                        |
|                   |                                |                                     |
| Command+Shift+F12 | Hide All Tools Window          | 隐藏所有工具窗口,代码区最大         |
| Command+1~9       | Open corresponding tool window | 显示对应工具窗口                    |

# 搜索与替换

| 快捷键                    | 功能                     | 说明                              |
| ------------------------- | ------------------------ | --------------------------------- |
| Shift+Shift               | Search everywhere        | 全局搜索                          |
| Alt+F7                    | Find usages              | 查找引用点                        |
| Command+Alt+F7            | Show usages              | 展示引用点                        |
| Command+Shift+F7          | Highlight usages in file | 高亮文件内引用点                  |
|                           |                          |                                   |
| Command+F                 | Find                     | 文件内查找                        |
| Command+Shift+F           | Find in path             | 项目内查找                        |
| Command+G/Command+Shift+G | Find next/previous       | 按选择字符查找下一个/上一个       |
|                           |                          |                                   |
| Command+R                 | Replace                  | 替换字符                          |
| Command+Shift+R           | Replace in path          | 项目内替换                        |
|                           |                          |                                   |
| Ctrl+G                    | Select next occurrence   | 选择下一个所选文本内容,可同时编辑 |
| Ctrl+Command+G            | Select all occurrences   | 选择全部相同文本，可同时编辑      |

