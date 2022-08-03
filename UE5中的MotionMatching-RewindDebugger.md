# UE5中的MotionMatching(六) 调试工具RewindDebugger

## 前言
在我看来，想要使用好MotionMatching有两件事情需要重点做好，一是之前常说的数据一致性即动画数据与MotionModel运动要有一致性或者相似性，想要做好这点项目组内必须得有资深的动画TA作为支持，另一个就是今天要说的Debug工具。Debug工具为什么如此重要？因为在使用MotionMatching时经常出现如下场景：“刚才那段行走动画不太对，动画为什么跳转呢” “刚才为什么跳转了呢，应该继续播放才对呀”，MotionMatching经常要处理的问题: **为什么使用了动画A而不是动画B**, 在这个时候一个强有力的Debug工具能帮你找出问题所在。

强烈建议在阅读本文前先按照虚幻官方文档[RewindDebugger](https://docs.unrealengine.com/5.0/zh-CN/animation-rewind-debugger-in-unreal-engine/)使用下该工具，这一点特别重要！

我的UE5-Main是在2022.8.1日更新的，PoseSearch(UE5对MotionMaching的称呼)本身就处于试验阶段，所以不确定将来是否会有大的改动(其实最近一段时间一直有提交)。

PoseSearch插件路径：UnrealEngine\Engine\Plugins\Experimental\Animation\PoseSearch

如果你对MotionMatching感兴趣，可以看下我的其他文章。

[MotionMatching 中的代码驱动移动和动画驱动移动](https://zhuanlan.zhihu.com/p/432663486)

[《荣耀战魂》中的MotionMatching](https://zhuanlan.zhihu.com/p/401890149)

[《最后生还者2》中的MotionMatching](https://zhuanlan.zhihu.com/p/403923793)

[《Control》中的MotionMatching](https://zhuanlan.zhihu.com/p/405873194)

[游戏开发中的Pose Matching](https://zhuanlan.zhihu.com/p/424382326)

[MotionMatching中的DataNormalization](https://zhuanlan.zhihu.com/p/414438466)

[UE5中的MotionMatching(一) MotionTrajectory](https://zhuanlan.zhihu.com/p/453659782)

[UE5中的MotionMatching(二) 创建可运行的PoseSearch工程](https://zhuanlan.zhihu.com/p/455983339)

[UE5中的MotionMatching(三) PoseMatching](https://zhuanlan.zhihu.com/p/492266731)

[UE5中的MotionMatching(四) MotionMatching](https://zhuanlan.zhihu.com/p/507268359)

[UE5中的MotionMatching(五) Mirroring Animation](https://zhuanlan.zhihu.com/p/516358714)

## 使用RewindDebugger调试MotionMatching
上面的官方文档已经详细说明了RewindDebuger的使用方法，要知道RewindDebugger可不是只为MotionMatching服务的，你面对的大部分动画相关的问题都可以用RewindDebugger复现并解决。

如果你的角色使用了MotionMatching并且通过RewindDebugger完成了录制，你可以发现在对象大纲视图(ObjectOutliner)中出现了PoseSearch以及众多动画资源:

![RewindDebugger录制数据](.\UE5MotionMatchingDebuggerPic\1.png)

绿色框内的是被MotionMatching选中并播放的动画资源，红色框内PoseSearch表明当前使用了MotionMatching。我们通过Tools/Debug/Rewind Debugger Details可以打开属性详情界面，然后点击红色框内的PoseSearch, 随着我们拖动时间轴会发现属性详细界面在实时更新：

![点击PoseSearch显示的详情界面](.\UE5MotionMatchingDebuggerPic\2.png)

我们本文的重点就是详细介绍这个属性面板内各个属性的含义

## 属性详情界面介绍
### A区域
![DrawOptions](.\UE5MotionMatchingDebuggerPic\A-Region.png)
ActivePose表示当前正在播放的那个动画，也可以理解成被MotionMatching选中的动画。有个特别特别重要的点，就是**MotionMatching不是每帧都执行的**，ActivePose是当前或者以前几帧的时候被MotionMatching系统选中并播放的，因此你经常看到，当前的ActivePose明明不是Cost最小的而偏偏被播放了，原因可能是它在前面因为Cost最小被选中播放了并且当前因为SearchThrottleTime的缘故没有Search。

ActivePose页签内最多只能有一行动画数据，并且这一行数据可以被选中，当被选中时，可以看到编辑器视口内出现了这个PoseIdx所对于FeatureVector的调试信息，颜色为**绿色**(蓝色的为QueryFeatureVector的调试信息)

![当ActivePose被选中时显示的绿色调试信息](.\UE5MotionMatchingDebuggerPic\3.png)

我们再看下列属性分别代表什么:
* Asset: 动画资源信息
* Type: Sequence或者BlendSpace, 目前就这两种，BlendSpace我们下篇再讲
* Cost：总的Cost，MotionMatching判断使用哪个动画的主要标准
* Cost[0]~Cost[3]: 各个FeatureChannel的Cost, Cost[0]中的0表示ChannelIdx，指的是该Channel在Schema Channels中的索引值
* CostModifier: 其他可能影响Cost的设置，比如ContinuingBias, MirrorMismatchCostBias等
* Frame: 当前播放帧在Asset动画内的索引信息
* Mirrored: 当前是否为镜像版本的动画
* Looping: 当前是否为循环动画
* BlendParameters: 当Type为BlendSpace时的参数
* PoseIndex: 采样Pose的编号

### B区域
![DrawOptions](.\UE5MotionMatchingDebuggerPic\B-Region.png)
ContinuingPose表示上一个ActivePose所在的动画如果继续播放下去，应该播放的那个PoseIdx。我们再次强调**MotionMatching不是每帧都执行的**，如果SearchThrottleTime设置的为1，那么表示1秒强制进行一次Search(其实表述不太准确，如果动画继续播放已经到动画末尾时也会强制Search)，那么本次Search后的1s时间内，MotionMatching是不会Search，我们假设本次Search的结果是A动画PoseIdx为10的动画帧，那么在接下来的1s内会播放A动画PoseIdx为11，12，13，14，15的动画直到1s后的Search，1s后再次Search的结果是B动画PoseIdx为80的动画帧，那么A区域的ActivePose会显示B动画PoseIdx为80的信息，而B区域的ContinuingPose会显示A动画PoseIdx为16的动画，这样会很直观的看出，为什么不再继续播放A动画了，特别方便进行比较。要知道，使用MotionMatching时会出现大量的类似场景“刚才为什么跳转动画了呢，应该继续播放原来的动画才对呀”

ContinuingPose页签内最多只能有一行动画数据，并且这一行数据可以被选中，当被选中时，可以看到编辑器视口内出现了这个PoseIdx所对于FeatureVector的调试信息，颜色为**灰色**(蓝色的为QueryFeatureVector的调试信息)

![当ContinuingPose被选中时显示的灰色调试信息](.\UE5MotionMatchingDebuggerPic\4.png)

### C区域
![DrawOptions](.\UE5MotionMatchingDebuggerPic\C-Region.png)
PoseDatabase显示了所有的候选Pose，当然也包括上面所说的ActivePose以及ContinuingPose，从这里也看到ActivePose是如何战胜其他候选人而成为天之骄子的。

![百里挑一](.\UE5MotionMatchingDebuggerPic\5.jpeg)

有个需要注意的点是，可能在实践中发现明明某个Pose的Cost最低但是并没有被选中，是不是有黑幕什么的，其实并不是，有些Pose有可能在AnimSequence中被标记了BlockTransition(通过PoseSearchingNotifyState_BlockTransition), 表示该Pose不允许被跳转，只不过属性界面里没有被标识出来而已。

PoseDatabase页签内有众多Pose数据，但最多只能有一行动画数据可以被选中，当被选中时，可以看到编辑器视口内出现了这个PoseIdx所对于FeatureVector的调试信息，颜色为**红色**(蓝色的为QueryFeatureVector的调试信息)

![当PoseDatabase某项被选中时显示的红色调试信息](.\UE5MotionMatchingDebuggerPic\6.png)

### D区域
![DrawOptions](.\UE5MotionMatchingDebuggerPic\D-Region.png)
通过名字可以猜到，MotionMatchingState显示的是MotionMatching内部的一些状态信息.

* CurrentDatabase: 当前所使用的Database。 MotionMatching支持动态更换Database;
* ElapsedPoseJumpTime: 这个属性特别特别重要，表示距离上次跳转累计多长时间了，上面我们提到过SearchThrottleTime，当ElapsedPoseJumpTime小于SearchThrottleTime时并且ContinuingPose还有有效Pose, 那么是不会Search的。调试中我们可以重点观察下这个值，当发现这个数值变成0了说明Pose发生跳转了;
* FollowUpAnimation: Pose跳转一般存在两种情况，一是直接Cost比较直接跳转到其他帧上，另外一种情况是在配置Database时，某个非循环动画A配置了FollowUpAnimation B,A动画在继续播放时发现已经到动画末尾了，但是配置了FollowUpAnimation B(动画B不仅是A动画的FollowUpAnimation, 而且还作为一个SourceEntry储存在Database中)，这时候会跳转到B的首帧上继续开始播放。此状态表示是否是第二种情况;
* AssetPlayerAssetName: 当前要播放的动画资源名称;
* AssetPlayerTime: 当前要播放的动画Local时间索引;
* LastDeltaTime: DeltaTime
* SimLinearVelocity: 目前Simulation的线性速度值
* SimAngularVelocity: 目前Simulation的旋转速度值，单位为角度
* AnimLinearVelocity: 当RootMotionDriven时，动画的线性速度值
* AnimAngularVelocity: 当RootMotionDriven时，动画的旋转速度值，单位为角度

### E区域
![DrawOptions](.\UE5MotionMatchingDebuggerPic\E-Region.png)
DrawOptions主要负责编辑器视口中的调试信息绘制，Query页签下的负责QueryFeatureVector的绘制，SelectedPose负责选中Pose的FeatureVector的绘制

* Disable: 下面三个控制选项都禁用
* DrawBoneNames: 绘制骨骼名称，但由于DrawDebugString的实现原因，目前DrawBoneNames在RewindDebugger暂不可用
* DrawSampleLabels: 绘制Trajectory每个Sample点的信息，但由于DrawDebugString的实现原因，目前DrawSampleLabels在RewindDebugger暂不可用
* DrawSampleWithColorGradient: 绘制Trajectory每个Sample点颜色时是否使用渐变色来区分Sample
* DrawActiveSkeleton: 是否绘制当前ActivePose的完整骨骼图，绿色显示
* DrawSelectedSkeletion: 是否绘制选中Pose的完整骨骼图，蓝色显示

我自己使用过程中一般会关闭DrawActiveSkeleton和DrawSelectedSkeletion，这样视口会清爽很多

### F区域
![DrawOptions](.\UE5MotionMatchingDebuggerPic\F-Region.png)

## 举例

## 注意事项和不足

## Bonus
其他大厂Debugger工具比较

## 下一步计划