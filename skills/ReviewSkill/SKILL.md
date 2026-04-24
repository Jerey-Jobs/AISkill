---
name: ReviewSkill
inclusion: always
description: 当你在阅读和优化dart/flutter代码时使用此skill。
---
<!------------------------------------------------------------------------------------
   Add rules to this file or a short description that will apply across all your workspaces.

   Learn about inclusion modes: https://kiro.dev/docs/steering/#inclusion-modes
-------------------------------------------------------------------------------------> 
-----------> # 代码技能指南

## 代码解释
当用户请求解释代码时，请遵循以下原则：

### 解释结构
1. 概述：简要说明代码的整体功能和目的
2. 核心逻辑：逐步解析关键代码段的工作原理
3. 数据流：描述数据如何在代码中流动和转换
4. 依赖关系：说明与其他模块/函数的交互

### 解释风格
- 使用中文进行解释，但要口语话，不要太死板
- 避免逐行翻译，聚焦于逻辑和意图
- 对复杂算法使用类比或图示说明
- 标注潜在的边界情况和异常处理

### Flutter/Dart 特定说明
- 解释 Widget 树结构和状态管理
- 说明 BuildContext 的作用域
- 解析异步操作和 Future 处理

#### Flutter/Dart 通用优化
- 减少 Widget 重建范围
- 使用 `const` 构造函数
- 合理使用 `ValueListenableBuilder`
- 避免在 build 方法中进行耗时操作
- 避免改动Widget引起动画异常
- 性能至上，避免一切不必要的CPU计算
- 一切的优化都要关注性能，并注释好为什么这么改
- 评估问题的时候，可以去查看flutter源码，确认下是否有这种问题，不要瞎猜
- 改动记得检查编译错误，别有什么找不到import,参数不对的情况

#### Flutter/Dart 专家级优化
1. Matrix4.identity() 每次调用会分配 128 字节堆内存（Float64List(16)）+ 16 次浮点数赋值。在动画 builder 中每帧调用会导致 GC 压力和 CPU 开销。
   正确的写法是：复用成员变量，使用一个成员变量缓存，在动画开始时候进行初始化，动画结束的时候销毁至空，动画期间复用该字符串

2. Opacity在0和1.0的情况下，也会有开销。对于动画写法，最佳的方案是使用AnimatedBuilder，在没有动画的时候去掉Opacity/Transform相关的包裹，使得Widget的层级变少

3. Widget深度越小越好。但并不是每一个 Widget 都会影响性能，只有创建 layer 的 Widget 才会影响性能。比如 transform/ClipRect/ClipRRect/Opacity/fade/scale/ 
   


