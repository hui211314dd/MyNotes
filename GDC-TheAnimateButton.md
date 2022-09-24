# 一键生成动画-育碧蒙特利尔关于Mocap的自动化技术

## 前言
在[MotionMatching 中的代码驱动移动和动画驱动移动](https://zhuanlan.zhihu.com/p/432663486)的Creative章节中，Daniel Holden提到了Dan Lowe的本次演讲，可以借助自动化工具对动捕数据进行调整以符合SimulationObject的要求，后来写完[UE5中的MotionMatching-实践(一)](https://zhuanlan.zhihu.com/p/557039076)后想着使用动捕数据作为Database, 如果想使用动捕数据那么必须面临数据一致性的问题，这时候觉得有必要研究下Dan Lowe的这篇演讲了。

本次演讲的[视频链接](https://www.gdcvault.com/play/1023477/Animation-Bootcamp-The-Animate-Button)以及[PPT链接](https://www.gdcvault.com/play/1023114/Animation-Bootcamp-The-Animate-Button)

Dan Lowe目前在圣莫尼卡工作室，专注于《战神:诸神黄昏》的开发，今年他的职位也提升到了Principal Tech Animator，他经常活动于[推特](https://twitter.com/danlowlows)，并且经常探讨游戏动画相关的技术，他的[油管频道](https://www.youtube.com/user/danlowe1983)有一些3A游戏的动画技术分析视频，还有MotionBuilder相关的教程，他的[Github](https://github.com/Danlowlows)上有本次的脚本代码，大家可以关注学习。

## 正文
大家好，我是Dan Lowe，目前在育碧蒙特利尔担任动画TA，正如下图看到的，参与了很多游戏的开发。我知道不同的公司对于动画TA有不同的定义，基本上我每天大部分的时间跟其他动画师一样在参与动画的制作，但是我也会拿出一些时间做一些技术上的工作比如状态机，设计动画工具以及管线流程，确保每个人都在以最佳方式工作。我目前的工作是在为动画师做一些自动化工具，这也是接下来我们要讲的。本次演讲的内容主要是我和Zach Hall完成的。

![Dan参与的游戏](.\TheAnimateButtonPic/1.png)


## 我的总结

## Bonus!

脚本地址

https://zhuanlan.zhihu.com/p/34260822

https://zhuanlan.zhihu.com/p/34330297

## 不明白的点
* Layer原理
* Adjustment Blending核心算法
* MotionBuilder FullBodyIK在Adjustment Blending中充当的作用
* Best when adjusting existing motion什么意思，后面提到的比较amount怎么理解和操作？

# 单词

