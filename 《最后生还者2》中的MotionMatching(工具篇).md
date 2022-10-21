# 《最后生还者2》中的MotionMatching(工具篇)

## 前言

为什么写这篇呢？ 两个原因，一是之前写过一篇[《最后生还者2》中的Motion Matching](https://zhuanlan.zhihu.com/p/403923793), 但那个时候没有听懂MMAssistant和LoopsInsertTool的讲解，所以在当时就挖了个坑; 二是写过[UE5中的MotionMatching-实践(一)](https://zhuanlan.zhihu.com/p/557039076),[一键生成动画-育碧蒙特利尔关于Mocap的自动化技术(上)](https://zhuanlan.zhihu.com/p/568048965)和[一键生成动画-育碧蒙特利尔关于Mocap的自动化技术(下)](https://zhuanlan.zhihu.com/p/569672931)后，想进一步看下其他公司如何高效编辑动捕动画，从我找到的资料中看除了顽皮狗以外，其他公司都没有重点介绍这块，但我在之前的文章中也提过，如果要达到产品级的效果，如何高效编辑动捕动画这个大坑必须得迈过去。基于这两个原因我硬着头皮再把顽皮狗的那次演讲又看了一遍，这次感觉收获颇丰，看了之后啊觉得常看常新...

![常看常新](.\TLOU2MotionMatchingToolsPic/1.png)

## MMAssistant

{MMAssistant演示视频.mp4}

MMAssistant是Maksym Zhuravlov提供给动画师的一个辅助工具，协助动画师制作用于MotionMatching所需的动画，该辅助工具人性化地设定了多个步骤，不管你是新手动画师还是经验丰富的动画师，只要按照辅助工具提供的步骤一步一步来，就能产出高质量可用的动画。

接下来我们看下MMAssistant的工作流程是怎么样的。

### Drop In Mocap Data And Finding the best take
插入多条动捕动画，在其中比对寻找最佳的动画片段，比如第一个例子制作的是StartForward的动画，我们从两者中找到了一个理想的动画。

![在所有的动捕动画中选取最合适的动画](.\TLOU2MotionMatchingToolsPic/2.png)

### Select Your Loco
这时候我们再选择下我们要编辑哪个人物哪种状态的什么类型的动画，MotionMatching具体需要多少种动画呢？没有人比动画TA更了解了，所以具体需要什么动画需要动画TA以yaml文件的形式填充，动画师只需要把所有的动画补齐就可以了。

可以看到主界面上方需要选择人物比如艾莉或者艾比，然后选择人物的状态比如relaxed，uneasy，explore，battle等，最后需要选择动画的类型比如walk, jog, run等等, 当选择完毕后下方会出现该人物在这个状态下所需的所有动画类别，下面列出的分别是：

* ellie-mm-explore-run-loops，所有loops相关的动画
* ellie-mm-explore-run-starts, 所有starts起步相关的动画
* ellie-mm-explore-run-stops, 所有stops停步相关的动画
* ellie-mm-explore-run-quickplants, 所有quickplants相关的动画 
* ellie-mm-explore-run-corners, 所有corners拐角相关的动画
* ellie-mm-explore-run-turns-on-spot, 所有原地转身的动画

点击ellie-mm-explore-run-starts后弹出了右侧所有待补充的starts动画列表:

![MMAssistant主界面](.\TLOU2MotionMatchingToolsPic/3.png)

## LoopsInsertTool

## Animation Analyzer

## DebugTools

## 我的总结

## Bonus!

## Next