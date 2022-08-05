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
![ActivePose](.\UE5MotionMatchingDebuggerPic\A-Region.png)
ActivePose表示当前正在播放的那个动画，也可以理解成被MotionMatching选中的动画。有个特别特别重要的点，就是**MotionMatching不是每帧都执行的**，ActivePose是当前或者以前几帧的时候被MotionMatching系统选中并播放的，因此你经常看到，当前的ActivePose明明不是Cost最小的而偏偏被播放了，原因可能是它在前面因为Cost最小被选中播放了并且当前因为SearchThrottleTime的缘故没有Search。

ActivePose页签内最多只能有一行动画数据，并且这一行数据可以被选中，当被选中时，可以看到编辑器视口内出现了这个PoseIdx所对于FeatureVector的调试信息，颜色为**绿色**(蓝色的为QueryPoseVector的调试信息)

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
![ContinuingPose](.\UE5MotionMatchingDebuggerPic\B-Region.png)
ContinuingPose表示上一个ActivePose所在的动画如果继续播放下去，应该播放的那个PoseIdx。我们再次强调**MotionMatching不是每帧都执行的**，如果SearchThrottleTime设置的为1，那么表示1秒强制进行一次Search(其实表述不太准确，如果动画继续播放已经到动画末尾时也会强制Search)，那么本次Search后的1s时间内，MotionMatching是不会Search，我们假设本次Search的结果是A动画PoseIdx为10的动画帧，那么在接下来的1s内会播放A动画PoseIdx为11，12，13，14，15的动画直到1s后的Search，1s后再次Search的结果是B动画PoseIdx为80的动画帧，那么A区域的ActivePose会显示B动画PoseIdx为80的信息，而B区域的ContinuingPose会显示A动画PoseIdx为16的动画，这样会很直观的看出，为什么不再继续播放A动画了，特别方便进行比较。要知道，使用MotionMatching时会出现大量的类似场景“刚才为什么跳转动画了呢，应该继续播放原来的动画才对呀”

ContinuingPose页签内最多只能有一行动画数据，并且这一行数据可以被选中，当被选中时，可以看到编辑器视口内出现了这个PoseIdx所对于FeatureVector的调试信息，颜色为**灰色**(蓝色的为QueryPoseVector的调试信息)

![当ContinuingPose被选中时显示的灰色调试信息](.\UE5MotionMatchingDebuggerPic\4.png)

### C区域
![PoseDatabase](.\UE5MotionMatchingDebuggerPic\C-Region.png)
PoseDatabase显示了所有的候选Pose，当然也包括上面所说的ActivePose以及ContinuingPose，从这里也看到ActivePose是如何战胜其他候选人而成为天之骄子的。

![百里挑一](.\UE5MotionMatchingDebuggerPic\5.jpeg)

有个需要注意的点是，可能在实践中发现明明某个Pose的Cost最低但是并没有被选中，是不是有黑幕什么的，其实并不是，有些Pose有可能在AnimSequence中被标记了BlockTransition(通过PoseSearchingNotifyState_BlockTransition), 表示该Pose不允许被跳转，只不过属性界面里没有被标识出来而已。

PoseDatabase页签内有众多Pose数据，但最多只能有一行动画数据可以被选中，当被选中时，可以看到编辑器视口内出现了这个PoseIdx所对于FeatureVector的调试信息，颜色为**红色**(蓝色的为QueryPoseVector的调试信息)

![当PoseDatabase某项被选中时显示的红色调试信息](.\UE5MotionMatchingDebuggerPic\6.png)

### D区域
![MotionMatchingState](.\UE5MotionMatchingDebuggerPic\D-Region.png)
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
DrawOptions主要负责编辑器视口中的调试信息绘制，Query页签下的负责QueryPoseVector的绘制，SelectedPose负责选中Pose的FeatureVector的绘制

* Disable: 下面三个控制选项都禁用
* DrawBoneNames: 绘制骨骼名称，但由于DrawDebugString的实现原因，目前DrawBoneNames在RewindDebugger暂不可用
* DrawSampleLabels: 绘制Trajectory每个Sample点的信息，但由于DrawDebugString的实现原因，目前DrawSampleLabels在RewindDebugger暂不可用
* DrawSampleWithColorGradient: 绘制Trajectory每个Sample点颜色时是否使用渐变色来区分Sample
* DrawActiveSkeleton: 是否绘制当前ActivePose的完整骨骼图，绿色显示
* DrawSelectedSkeletion: 是否绘制选中Pose的完整骨骼图，蓝色显示

我自己使用过程中一般会关闭DrawActiveSkeleton和DrawSelectedSkeletion，这样视口会清爽很多

### F区域
![PoseVectors](.\UE5MotionMatchingDebuggerPic\F-Region.png)
“凭什么MotionMatching选择A动画而不是B动画，B动画在计算Cost时哪方面吃亏了？”，当你有这方面疑问时，PoseVectors提供的信息能帮助我们找到答案。

* QueryPoseVector: 向MotionMatching申请Search时传入的QueryVector, 数据为归一化前的数据，比如位置为(10, 30, 0)等，数组的数量等于所有Feature的数量，依赖于Schema Channels的设置;
* ActivePoseVector: 当前ActivePose的PoseVector, 数据为归一化前的数据，数量与QueryPoseVector相等;
* SelectedPoseVector: 当PoseDatabase中有Pose被选中时，SelectedPoseVector显示的是选中Pose的PoseVector，数据为归一化前的数据，数量与QueryPoseVector相等;
* CostVector: 当PoseDatabase中有Pose被选中时，CostVector显示的是选中PoseCost的详细，数量与QueryPoseVector相等;
* CostVectorDifference: SelectedPoseCost与ActivePoseCost的差，数量与QueryPoseVector相等。当数值为负值时，说明SelectedPoseCost更小，更占优，如果为正值，说明SelectedPoseCost更大，具有劣势。可以看到该属性在解决上面那个问题时特别有用;

### G区域
![Misc](.\UE5MotionMatchingDebuggerPic\G-Region.png)
* PlaySelectedAsset: 在编辑器视口播放在PoseDatabase选中的动画
* AssetPlayRate: 设置动画播放时的速率

## 举例
{RewindDebugger在使用MotionMatching时的演示.mp4}

我们发现在静止状态下，角色似乎一直在跳转动画，导致角色动画不流畅，我们期望的结果是静止状态下角色一直在Idle动画上循环播放，我们通过RewindDebugger录制并回放发现，角色每1.5s会发生一次跳转，而我们的SearchThrottleTime设置的就是1.5s，所以结论是只要Search就会跳转。我们查看按帧步进发现在发生跳转的那一刻，ContinuingPose应该是193，并且Cost为0，而ActivePose变成了325，Cost也为0，因此原因如下：因为SearchThrottleTime为1.5s的缘故，MotionMatching系统每1.5s调用一次Search并且进行比较，发现所有的Cost都相同，所以就找到了一个存储位置靠后的Pose作为最佳结果返回并跳转(其实引擎代码有bug，在进行float比较时没有考虑ErrorTolerance，导致比较出现了错误)，那么如何解决呢？ 我们在[《最后生还者2》中的MotionMatching](https://zhuanlan.zhihu.com/p/403923793)看到过Bias相关的设置，幸运的是，PoseSearch也添加了相关的设置，我们将ContinuingPoseCostBias设置成-0.12就可以了~

## 注意事项和不足
不同的FeatureChannel的绘制代码在UPoseSearchFeatureChannel::DebugDraw中实现，需要特别注意的是，绘制Trajectory时，LinearVelocity和FacingDirection很难区别，因为使用的是同一种颜色并且有些时候两个方向的长度还是等长的...

![](.\UE5MotionMatchingPracticePic/3.png)

通过UPoseSearchFeatureChannel_Trajectory::DebugDraw代码我们可以知道，绘制LinearVelocity时，向量的长度表示了速度的大小，而FacingDirection却始终是单位向量，所以可以得出结论，如果向量的长度等于圆球的半径大小那就是FacingDirection, 如果向量长度有长有短，那表示的就是LinearVelocity.

虽说RewindDebugger表现已经相当不错了，但仍然有些地方可以改进的：
* DrawDebugString貌似在RewindDebugger中无效，导致DrawBoneNames和DrawSampleLabels不可用;
* F区域中虽说列出了哪些Cost大哪些小，没有直接指出是Position还是Rotation还是Direction，需要自己算;
* FacingDirection和LinearVelocity使用同种颜色，无法很直观的区分;
* 前面反复提到过，**MotionMatching不是每帧都执行的**, 但目前没有Flag能够标识出当前是否执行过Search;
* PoseDatabase中没有BlockTranstion的标识列，导致有可能该PoseCost最小，但不会被选中，令开发者疑惑;
* 性能问题，拖动时间轴的情况下Details界面反应会比较慢;

## Bonus
前言中已经说过，Debug工具是MotionMatching核心的功能之一，直接会影响MotionMatching的开发效率，所以各个大厂对于debug工具都特别重视，下面视频列出了Naughty Dog(顽皮狗)和Remedy(绿美迪娱乐)所开发的调试工具，仅供参考


## 下一步计划

* MM中的BlendSpace

* MM FootLock(FootPlacement)

* MM DynamicPlayRateSettings

* MM 数据的标准化处理
  
* MM应用篇