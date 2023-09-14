---
layout: post
title: "Git Commit规范"
subtitle: "Git Commit Specification"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - Git
---

一直想规范一下自己的git commit message，之前都是随便写的，太业余了（，并且代码的维护成本也很大（虽然还没咋维护过，也没有回滚过。不过，具体的公司应该会对代码和提交都有相应的规范。还是写一下来规范personal projet。

参考Angular规范和[如何规范你的Git commit？—阿里云开发者](https://zhuanlan.zhihu.com/p/182553920)

---

**Git Commit Message格式**

```shell
<type>(<scope>): <subject>
```

1. type (required) ：说明此次提交的类别，只允许使用下面的类别。

     - feat (feature)：新功能/特性

     - fix/to：修复bug。fix适合一次提交直接修复bug；to适合多次提交直到最终修复bug（fix）

     - docs (documentation)：文档

     - style：修改格式，注释也算

     - refactor：重构，非新增功能和修复bug，修改文件名也算

     - perf：优化性能、体验

     - test：增加测试

     - chore：构建过程或辅助工具的变动

     - revert：回滚到上一个版本

     - merge：代码合并

     - sync：同步主线




2. scope (optional)：说明commit影响的范围（看了看一些著名的repo都没有这个，感觉用的不多



3. subject (required)：commit的简短描述，不超过50字符